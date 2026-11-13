# RFC-0031: Hardware video in a helper process

Moved here from [storytold/effectcraft#3648] at a maintainer's request, after a maintainer said on
the ArtCraft Discord (2026-11-13) that hardware video should be revisited.

Related issues: [storytold/effectcraft#805] (hardware encoding), [storytold/effectcraft#498]
(NVENC), [storytold/effectcraft#804] (hardware decoding), [storytold/effectcraft#2240] (slow
export).

## In short

- Video exports are slow because all video decoding and encoding runs on the CPU. The GPU's
  built-in video engine is never used.
- Using it means calling the graphics driver. A driver bug can crash the app and lose unsaved
  work, so EffectCraft has kept driver code out.
- Proposal: run the driver code in a **separate helper program**, the way EffectCraft already runs
  native plug-ins. If the driver crashes, only the helper stops; the app keeps running and falls
  back to its own encoder.
- It is opt-in, and nothing changes for anyone who doesn't turn it on.

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

## What exists

- **FilmCraft already uses the video engine.** Its [ADR 0001][filmcraft-adr] allows `unsafe`
  code in one crate, `filmcraft-platform`, which supports hardware decoding on every desktop
  platform and hardware encoding on Mac and NVIDIA. AMD encoding on Linux is still in review
  ([storytold/filmcraft#1612]).
- **EffectCraft declined it in process.** [storytold/effectcraft#819] tried to bring that crate
  into EffectCraft. It was declined to keep driver code out of the app, for the same reason as
  [storytold/effectcraft#498]: a crash inside a driver can't be caught, and the user loses their
  work. [storytold/effectcraft#1840] takes the same approach and is waiting on a rebase.
- **EffectCraft already runs risky native code out of process.** Its [ADR 0001][effectcraft-adr]
  runs native plug-ins as separate helper programs. If one crashes or hangs, it is stopped and
  restarted, and after 3 crashes it is turned off.
- **The suite's direction fits.** [RFC-0022](0022-gpu-layer.md) keeps wgpu as the GPU baseline and
  puts vendor-specific acceleration behind a seam in a leaf crate. A helper program is that seam
  for video.

This RFC proposes running hardware video the same way EffectCraft runs native plug-ins.

## Decision

- **D1 No driver code inside the app.** A helper program, `effectcraft-helper-video`, talks to the
  driver. The app starts it and sends it work. The app itself keeps `unsafe_code = "forbid"`. The
  helper is built from FilmCraft's existing code and released separately from the app.
- **D2 Reuse EffectCraft's plug-in helper system.** The helper is found, started, timed out,
  restarted and blocklisted exactly like effect helpers. Only new message types for video are
  added.
- **D3 Plug in at two points.** Exports get a helper-backed encoder next to the built-in ones.
  Footage decoding gets a helper-backed decoder next to FilmCraft's.
- **D4 Opt-in.** Each output module gets a **Hardware Encoding: Off / Auto** setting, Off at
  first. Auto uses the helper only when it is installed and supports the export settings;
  otherwise the built-in encoder runs and the render log says why. Default exports stay exactly as
  they are today.
- **D5 Fail gracefully.** No helper problem can stop the app or harm the project:
  - before using the hardware, ask the driver what it supports, and skip it if unsupported;
  - if the helper crashes, hangs or misbehaves, it is stopped. Decoding falls back to the
    built-in decoder for that frame. An export restarts with the built-in encoder, and the log
    says so;
  - after 3 crashes the helper is turned off until the user retries in Settings;
  - hardware-decoded pictures must match the built-in decoder's, or that file isn't decoded in
    hardware.
- **D6 Encoding first.** Exports spend the most CPU on encoding, and it is the simplest to
  connect: frames go one way and there is no seeking.

## Plan

| Phase | What | Notes |
| --- | --- | --- |
| 1 | H.264 / HEVC hardware encoding in EffectCraft | Mac and NVIDIA code exists in FilmCraft; AMD on Linux is in [storytold/filmcraft#1612] |
| 2 | Hardware decoding, for export and preview | Code and measurements exist in [storytold/effectcraft#819] |
| 3 | AMD encoding on Windows | Not in FilmCraft yet (Media Foundation or AMF) |

## Costs and trade-offs

- **Nothing changes without the helper.** The app, the web build and default exports stay the
  same.
- **A copy per frame.** Each frame is passed to the helper and back: about 3 MB at 1080p, 12 MB at
  4K. Phase 1 measures this; it should be small next to the 165 ms of encoding it saves.
- **Output varies by machine.** Hardware encoders produce different files on different GPUs and
  drivers, and often lower quality at the same file size. The render log records which encoder,
  GPU and driver were used.
- **More to release and test.** The helper needs its own builds and signing on each platform. CI
  can test the helper system with a stand-in helper, but not real GPUs, so a manual test list
  (NVIDIA, AMD and Intel on Windows and Linux, plus Apple silicon) is kept with the helper.

## Other options considered

| Option | Why not (for now) |
| --- | --- |
| `filmcraft-platform` inside EffectCraft ([storytold/effectcraft#819], [storytold/effectcraft#1840]) | Simplest and fastest, but declined in [storytold/effectcraft#819]. If maintainers prefer it after all, [storytold/effectcraft#1840]'s settings and CLI work can be reused |
| Our own GPU encoder in shaders | No driver code, but most encoding steps can't run in parallel, so it's months of work for a small gain |
| ffmpeg as a separate program | The apps allow ffmpeg only for tests, never shipped |
| Doing nothing | Exports stay CPU-bound and strong GPUs stay idle |

## Not in this RFC

- **FilmCraft's in-process `filmcraft-platform`.** It stays as its ADR 0001 describes.
- **Zero-copy from GPU textures to the encoder.** A later step, if the per-frame copy turns out to
  matter.
- **Using EffectCraft's GPU and CPU compositors at once.** Today a Render Queue job uses one and the
  other idles. That needs no driver code and can go ahead on its own.
- **Other codecs** (AV1, VP9, ProRes) in hardware.

## Technical notes (EffectCraft)

- **Messages (sketch).** `hello` lists what the machine supports. Encoding:
  `encode.open {codec, width, height, rate, bitrate, colour}` → `encode.frame` → packets →
  `encode.flush` → `encode.close`. Decoding: `decode.open {codec, sample entry}` →
  `decode.sample` → picture. Pictures and packets travel through the effect helpers' frame
  exchange file, never through the pipe. The full protocol would go in EffectCraft's
  `docs/external-helpers.md`.
- **Encoder seam.** `crates/export/src/encode.rs`, where `movie` picks the encoder. H.264 frames
  already arrive there as 4:2:0 planes, so the helper gets exactly what the built-in encoder gets.
- **Decoder seam.** `MediaPool` in `crates/media/src/pool.rs`, where it opens FilmCraft's decoder
  today.
- **Limits.** The crash limit and timeouts are the effect-helper ones (`effectcraft-helper-host`).
  Export settings the hardware can't handle (for example HDR, odd sizes or non-square pixels) are
  declined up front, as FilmCraft does.

## Questions for the maintainers

1. Is a separate helper process the right approach, or would you rather have it inside the app?
2. Can the installer include the helper, or should it be a separate optional download?
3. Where should the helper's code live (FilmCraft, craft-libs or its own repository), and who
   builds and signs it?
4. If the helper fails during an export, should the export restart with the built-in encoder
   (D5), or stop with an error, as FilmCraft does?
5. Encoding first (D6), or decoding first, since [storytold/effectcraft#819]'s code already
   exists?
6. Hardware Encoding Off by default, or Auto once the manual tests pass?
7. How should this work together with [storytold/effectcraft#1840] and its author?

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
