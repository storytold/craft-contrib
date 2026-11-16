# RFC-0031: Hardware video in a helper process

Moved here from [storytold/effectcraft#3648] at a maintainer's request, after a maintainer said on
the ArtCraft Discord (2026-11-13) that hardware video should be revisited.

Related issues: [storytold/effectcraft#805] (hardware encoding), [storytold/effectcraft#498]
(NVENC), [storytold/effectcraft#804] (hardware decoding), [storytold/effectcraft#2240] (slow
export).

**Changes after review (2026-11-15).** The safety claim is narrowed to hardware video calls (D1).
Starting the helper needs the user's consent and it is found only in the app's install folder
(D2). Video gets its own frame format, limits and version check (D3). A failed hardware export
stops with a clear error and never replaces an existing file (D5). Decoding fallback rebuilds
the decoder's state (D6). Each GPU vendor's API and the choice between two GPUs are named (D7).
The plan has pass/fail gates, and the open questions are split into decided and still open.

## In short

- Video exports are slow because all video decoding and encoding runs on the CPU. The GPU's
  built-in video engine is never used.
- Using it means calling the hardware video parts of the graphics driver. A bug there can crash
  the app and lose unsaved work, so EffectCraft has kept those calls out.
- Proposal: make those calls in a **separate helper program**, the way EffectCraft already runs
  native plug-ins. If they crash or hang, only the helper stops; the app keeps running and the
  export ends with a clear error.
- It is off by default, and nothing changes for anyone who doesn't turn it on.

## Affected apps

- **EffectCraft**: the first user of the helper.
- **FilmCraft**: its hardware video code is what the helper is built from. Its own in-process
  setup is not changed by this RFC.
- **Any other Craft app that decodes or encodes video** can use the same helper later, instead of
  each app solving this again.

## The problem

Every frame of an export is decoded, composited and encoded. In EffectCraft only compositing can
use the GPU. Decoding and encoding always run on the CPU, with the suite's own codecs, so on long
exports the CPU is maxed out while the GPU waits. Users with strong GPUs report this
([storytold/effectcraft#3426], [storytold/effectcraft#3624]).

Every modern GPU has a video engine built for exactly this job: VCN on AMD, NVENC / NVDEC on
NVIDIA, Quick Sync on Intel, and the media engine on Apple silicon.

How much it could help:

| Measurement | Result |
| --- | --- |
| EffectCraft's CPU cost of a 1080p30 H.264 export (12-thread laptop, `main`, 2026-11-13) | ≈ 425 ms per frame: ≈ 165 ms encoding, ≈ 130 ms decoding the source |
| Hardware decoding ([storytold/effectcraft#819]) | 5–31× less CPU per frame, identical pictures |
| 4K HEVC export, software encoder on a Mac ([storytold/effectcraft#1840]) | 0.23 frames per second |
| Same export, Mac hardware encoder ([storytold/effectcraft#1840]) | 15.3 frames per second |
| Same export, NVIDIA RTX 3090 / AMD RX 5500 ([storytold/effectcraft#1840]) | 11.6 / 5.0 frames per second |

These numbers come from in-process code. They show why the work is worth doing, but they don't
prove the helper will be as fast. Phase 1 measures that (see Plan).

## What exists

- **FilmCraft already uses the video engine.** Its [ADR 0001][filmcraft-adr] allows `unsafe`
  code in one crate, `filmcraft-platform`, which supports hardware decoding on every desktop
  platform and hardware encoding on Mac and NVIDIA. VA-API encoding on Linux is in review
  ([storytold/filmcraft#1612], so far tested on AMD).
- **EffectCraft declined it in process.** [storytold/effectcraft#819] tried to bring that crate
  into EffectCraft. It was declined to keep driver code out of the app, for the same reason as
  [storytold/effectcraft#498]: a crash inside a driver can't be caught, and the user loses their
  work. [storytold/effectcraft#1840] takes the same approach and is waiting on a rebase.
- **EffectCraft already runs risky native code out of process.** Its [ADR 0001][effectcraft-adr]
  runs native plug-ins as separate helper programs. They start only when the user asks, and if
  one crashes or hangs, it is stopped and restarted, and after 3 crashes it is turned off.
- **The suite's direction fits.** [RFC-0022](0022-gpu-layer.md) keeps wgpu as the GPU baseline and
  puts vendor-specific acceleration behind a seam in a leaf crate. A helper program is that seam
  for video.

## Decision

- **D1 Hardware video calls run outside the app.** A helper program, `effectcraft-helper-video`,
  makes all hardware video encoding and decoding calls. The app keeps `unsafe_code = "forbid"`
  and never loads those drivers. This is all the boundary does:
  - The app still uses the GPU for compositing, so a GPU reset or a system-wide driver failure
    can still affect it.
  - A separate process is not a security sandbox.
- **D2 The user decides when it runs, and only the right program runs.**
  - Nothing starts the helper unless the user asks, the same consent effect helpers need today.
    Opening a project that has Hardware Encoding set to Auto does not start it. Agents confined to
    folders, and scripts without write and network permission, can't start it either.
  - The helper is found only in the app's own install folder. Never from `PATH`, the current
    folder, the settings folder or another folder the user can write to.
  - The app checks the protocol and helper version in `hello`, before it sends any work. A
    mismatch means the helper isn't used, and the message says which version to install.
- **D3 Video gets its own frame format.** Effect helpers send images as float RGBA (16 bytes per
  pixel: about 33 MB per 1080p frame, 133 MB at 4K). Video uses its own format instead, agreed in
  `hello` as a separate capability, so existing effect helpers are unaffected:
  - pictures are 8-bit 4:2:0 to start (about 3.1 MB per 1080p frame, 12.4 MB at 4K); 10-bit
    comes later as another agreed format;
  - each message gives every plane's offset, stride, size and byte length, plus the colour
    information (primaries, transfer, matrix, range);
  - the app owns every region of the exchange file. It says where the helper may write packets,
    and checks every offset and length before reading;
  - at most a few frames are in flight, with a cap on the bytes queued and in flight together,
    so a slow GPU makes the export wait rather than use more memory;
  - each job and frame has an id; `cancel` stops a job, and replies for a cancelled or old job
    are ignored.
- **D4 Plug in at two points.** Exports get a helper-backed encoder next to the built-in ones.
  Footage decoding gets a helper-backed decoder next to FilmCraft's.
- **D5 A failed hardware export fails cleanly.**
  - Hardware and software output are never mixed in one file.
  - A hardware export writes to a temporary file and replaces the destination only when it
    finishes. A failed or cancelled export leaves any existing file as it was. (EffectCraft's
    exports write straight to the destination today, so this is new.)
  - In phase 1, a failure ends the export with a clear error that says what failed. The format is
    never changed silently.
  - Automatic retry with the built-in encoder can come later, only with its tests. It would start
    the whole export again from a fresh temporary file, at most once, and keep cancellation
    working. EffectCraft has built-in encoders for both H.264 and HEVC, so both could retry.
    (FilmCraft has no software HEVC encoder, so this doesn't carry over there.)
  - Every frame has its own timeout. A GPU that stops answering ends the helper instead of making
    the export look busy forever.
  - After 3 crashes the helper is turned off until the user retries in Settings.
- **D6 Decoding falls back by rebuilding state.** Video frames depend on earlier frames, so a
  decoding failure can't be fixed by swapping one frame. As FilmCraft's fallback does, the
  built-in decoder replays from the last restart point and drops pictures already returned,
  keeping the order, timestamps, seeks, flushes and colour information right. Whether hardware
  pictures match the built-in decoder's is checked with fixture files and the hardware tests, not
  by decoding every frame twice. A stream type that fails those checks isn't decoded in hardware.
- **D7 One API per vendor, and a clear choice between GPUs.**
  - Linux: VA-API for AMD and Intel (through their Mesa / Intel drivers), NVENC / NVDEC for
    NVIDIA. Intel on Linux uses the same VA-API path but is untested so far.
  - Windows: NVENC for NVIDIA; AMD and Intel come in phase 3. macOS: VideoToolbox.
  - `hello` reports each device, its driver and API, and what it can encode and decode.
  - When two GPUs can do the same codec (a laptop with integrated and discrete graphics), Auto
    picks one by a fixed rule and the render log says which.
- **D8 Safe start-up inside the helper.** The helper starts its hardware APIs (CUDA, VA-API and
  others) on one thread, in a fixed order, before it accepts any work. Starting NVENC and VA-API
  at the same time on two threads can deadlock with NVIDIA's VA-API driver (found while
  investigating [storytold/filmcraft#4083]).
- **D9 Encoding first.** Exports spend the most CPU on encoding, and it is the simplest to
  connect: frames go one way and there is no seeking.

## Plan

| Phase | What | Done when |
| --- | --- | --- |
| 1 | H.264 / HEVC hardware encoding in EffectCraft | The gates below pass. Mac and NVIDIA code exists in FilmCraft; VA-API is in [storytold/filmcraft#1612] |
| 2 | Hardware decoding, for export and preview | D6's fallback and fixture checks pass. Code and measurements exist in [storytold/effectcraft#819] |
| 3 | AMD and Intel encoding on Windows | Not in FilmCraft yet (Media Foundation or AMF) |

**Gates for phase 1:**

- **Tests with a stand-in helper, in CI on every platform:** a crash, a hang, a garbage reply, a
  reply that's too large, a protocol version mismatch, cancellation, a missing helper, a lost GPU
  device, and a failure while an earlier export already exists at the destination.
- **Real hardware reports** for each tested setup record the backend and driver, codec and bit
  depth, quality settings, end-to-end wall time and peak memory, including the frame copies.
  Reviewers have offered to test on Linux with NVIDIA and AMD, including a laptop with both. Intel
  and sandboxed (Flatpak) setups still need testers.

## Costs and trade-offs

- **Nothing changes without the helper.** The app, the web build and default exports stay the
  same.
- **A copy per frame.** Each frame is passed to the helper and back: about 3.1 MB at 1080p,
  12.4 MB at 4K (D3). Phase 1 measures this against the encoding time it saves.
- **Output varies by machine.** Hardware encoders produce different files on different GPUs and
  drivers, and often lower quality at the same file size. The render log records which encoder,
  GPU and driver were used.
- **More to release and test.** The helper needs its own builds, signing and version on each
  platform, and a person who owns them. Sandboxed Linux packages (Flatpak) need GPU access and a
  matching driver extension, and hardware encoding there is often unreliable, so they are tested
  separately, not assumed to work.

## Other options considered

| Option | Why not (for now) |
| --- | --- |
| `filmcraft-platform` inside EffectCraft ([storytold/effectcraft#819], [storytold/effectcraft#1840]) | Simplest and fastest, but declined in [storytold/effectcraft#819] |
| Our own GPU encoder in shaders | No driver code, but most encoding steps can't run in parallel, so it's months of work for a small gain |
| ffmpeg as a separate program | The apps allow ffmpeg only for tests, never shipped |
| Doing nothing | Exports stay CPU-bound and strong GPUs stay idle |

## Not in this RFC

- **FilmCraft's in-process `filmcraft-platform`.** It stays as its ADR 0001 describes.
- **Zero-copy from GPU textures to the encoder.** A later step, if the per-frame copy turns out to
  matter.
- **Using EffectCraft's GPU and CPU compositors at once.** Today a Render Queue job uses one and the
  other idles. That needs no driver code and can go ahead on its own.
- **The CLI render path and the choice of compositor.** `effectcraft-cli render` renders on the CPU
  unless it is started with `--gpu`, and a GPU render that runs out of memory can fall back to the
  CPU without saying so. Hardware encoding alone will not close the gap that CLI users see
  ([storytold/effectcraft#3706]; related: [storytold/effectcraft#897],
  [storytold/effectcraft#2423], [storytold/effectcraft#2789]). That needs its own fix.
- **Other codecs** (AV1, VP9, ProRes) in hardware.

## Technical notes (EffectCraft)

- **Messages (sketch).** `hello` agrees the protocol version and video capabilities, and lists
  devices (D7). Encoding: `encode.open {codec, width, height, rate, bitrate, colour}` →
  `encode.frame` → packets → `encode.flush` → `encode.close`. Decoding:
  `decode.open {codec, sample entry}` → `decode.sample` → picture. Both have `cancel`. The full
  protocol would go in EffectCraft's `docs/external-helpers.md`.
- **Encoder seam.** `crates/export/src/encode.rs`, where `movie` picks the encoder. H.264 frames
  already arrive there as 4:2:0 planes, the format D3 sends to the helper.
- **Decoder seam.** `MediaPool` in `crates/media/src/pool.rs`, where it opens FilmCraft's decoder
  today.
- **Output file.** `crates/export/src/out.rs` creates the destination directly today. D5's
  temporary file and rename go there.
- **Limits.** The crash limit is the effect-helper one (`effectcraft-helper-host`). Export
  settings the hardware can't handle (for example HDR, odd sizes or non-square pixels) are
  declined up front, as FilmCraft does.

## Decided in review

- A separate helper process, not in-process.
- Hardware encoding first, then decoding.
- Hardware Encoding **Off by default**.
- In phase 1 a failed hardware export ends with a clear error (D5).
- The helper is a **separately versioned, optional** program with an app/helper compatibility
  check (D2), not part of the default install at first.
- It reuses FilmCraft's backend; it does not fork it.
- The helper is **not distributed until an owner is named** to build, sign and release it, and the
  app/helper compatibility policy (D2) is written down.

## Still open

1. Who builds, signs and releases the helper? Until someone is named, the helper is not
   distributed.
2. Where does its code live (FilmCraft, craft-libs or its own repository)? To be decided with
   FilmCraft's maintainers.
3. What fixed rule should Auto use to pick between two GPUs (D7)?
4. How does this work together with [storytold/effectcraft#1840]? Its author is to be contacted
   before any settings or CLI work is duplicated.

[effectcraft-adr]: https://github.com/storytold/effectcraft/blob/main/docs/adr/0001-external-effect-helpers.md
[filmcraft-adr]: https://github.com/storytold/filmcraft/blob/main/docs/adr/0001-platform-ffi.md
[storytold/effectcraft#498]: https://github.com/storytold/effectcraft/issues/498
[storytold/effectcraft#804]: https://github.com/storytold/effectcraft/issues/804
[storytold/effectcraft#805]: https://github.com/storytold/effectcraft/issues/805
[storytold/effectcraft#819]: https://github.com/storytold/effectcraft/pull/819
[storytold/effectcraft#1840]: https://github.com/storytold/effectcraft/pull/1840
[storytold/effectcraft#2240]: https://github.com/storytold/effectcraft/issues/2240
[storytold/effectcraft#3426]: https://github.com/storytold/effectcraft/issues/3426
[storytold/effectcraft#3624]: https://github.com/storytold/effectcraft/issues/3624
[storytold/effectcraft#3648]: https://github.com/storytold/effectcraft/pull/3648
[storytold/filmcraft#1612]: https://github.com/storytold/filmcraft/pull/1612
[storytold/filmcraft#4083]: https://github.com/storytold/filmcraft/pull/4083
[storytold/effectcraft#897]: https://github.com/storytold/effectcraft/issues/897
[storytold/effectcraft#2423]: https://github.com/storytold/effectcraft/issues/2423
[storytold/effectcraft#2789]: https://github.com/storytold/effectcraft/issues/2789
[storytold/effectcraft#3706]: https://github.com/storytold/effectcraft/issues/3706
