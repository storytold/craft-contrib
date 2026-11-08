# RFC-0028: One AAC decoder for the suite, in craft-libs

The Craft apps decode AAC in two ways, and neither way decodes HE-AAC:

- FilmCraft's MP4 path and EffectCraft use FilmCraft's clean-room
  `filmcraft-aac` (`crates/aac` in storytold/filmcraft). EffectCraft
  pins it by git revision from the FilmCraft repository. It decodes only
  the AAC-LC core of an HE-AAC stream, so the high band is never rebuilt
  and the audio sounds muffled (storytold/effectcraft#2893).
- SoundCraft (`crates/audio-io`, symphonia 0.6), DeckCraft
  (`crates/media`, symphonia 0.5.5) and FilmCraft's own audio-file path
  (`crates/codecs`, symphonia 0.5.5) use `symphonia-codec-aac`.
  Symphonia's AAC decoder implements only the LC profile. Its support
  table lists HE-AAC and HE-AAC v2 as not implemented. The same files
  import at half length and double speed (storytold/soundcraft#712), and
  HE-AAC v2 files import as mono (storytold/soundcraft#740).

So one file sounds different in each app, and every AAC fix has to be
made twice. storytold/filmcraft#4507 adds SBR (HE-AAC v1) and parametric
stereo (HE-AAC v2) to `filmcraft-aac`. This RFC decides where that
decoder lives once it does, so the work lands once and every app gets it.

## Decision

- **A shared crate.** `filmcraft-aac` moves to craft-libs as
  `craft-aac` (`crates/aac`), with its encoder, decoder, ADTS and
  `AudioSpecificConfig` parsing, tests and probing tools. Its git
  history comes with it. It keeps its layer rule: no dependencies beyond
  bit I/O and `thiserror` (optional `rayon`), no `unsafe`, and it builds
  for `wasm32-unknown-unknown`. Bit I/O comes from `craft-media` with only
  its `bitstream` feature, which has the same `BitReader` API as
  `filmcraft-bitstream`.
- **craft-audio uses it.** `craft-audio`'s `decode-native` decodes AAC
  in MP4/MOV, Matroska and ADTS through `craft-aac`, with the priming
  and edit-list timing those demuxers already report, plus the SBR
  decoder delay. Its `symphonia` feature stops turning on symphonia's
  `aac` feature. It keeps `isomp4` and `mkv`, which ALAC, MP3 and FLAC
  in those containers still need. AAC that `craft-aac` can't decode
  (Main, SSR, LTP, ER object types) returns `Unsupported`, as symphonia
  does today.
- **Every app decodes AAC through it.** No app turns on symphonia's
  `aac` feature. Each app moves when its tests pass on the shared crate:
  - FilmCraft: the MP4 path uses `craft-aac`, `crates/codecs` drops
    symphonia's `aac`, and `crates/aac` is removed. Other crates in the
    workspace that depend on `filmcraft-aac` switch to `craft-aac`.
  - EffectCraft: pins craft-libs instead of FilmCraft for the AAC
    decoder.
  - SoundCraft and DeckCraft: decode AAC through `craft-audio` or
    `craft-aac`, and drop symphonia's `aac` feature.
- **Same file, same result.** Each app's import tests decode the same
  HE-AAC files and check the same length, sample rate and channel count.
- **Clean room.** The decoder is written from ISO/IEC 14496-3 only. The
  crate README records the edition used. FFmpeg, FAAD2, fdk-aac and the
  3GPP reference code (TS 26.410, TS 26.411) are not read. The ffmpeg
  binary is used only as a test oracle and fixture generator. Conformance
  streams stay in an external corpus unless their licence allows
  committing them.
- **Order.** SBR and parametric stereo are written in FilmCraft's
  `crates/aac`, where the decoder and its tests are now (#4507 steps 1–3).
  The move to craft-libs follows when this RFC is accepted, as one pull
  request that adds the crate unchanged, so the review there is about
  the move and not the codec. Until then EffectCraft keeps pinning
  FilmCraft.
- **Ownership.** craft-libs maintainers own the crate. Changes to it go
  through craft-libs pull requests, with a maintainer of an app that
  decodes AAC invited to review codec changes.

## What this asks of each app

- **FilmCraft:** switch the MP4 and audio-file paths to `craft-aac` and
  delete `crates/aac` once nothing in the workspace depends on it.
- **EffectCraft:** replace the `filmcraft-aac` git dependency with
  `craft-aac`, and keep `filmcraft-media` until a separate change moves
  it.
- **SoundCraft and DeckCraft:** decode AAC through `craft-audio` or
  `craft-aac`, and turn off symphonia's `aac` feature.
- **Every app above:** import tests for HE-AAC v1 and v2 files that check
  length, rate and channels.

## Not decided here

- **Symphonia's licence.** Symphonia is `MPL-2.0`, which FilmCraft's and
  EffectCraft's own `AGENTS.md` allow when used unmodified and the shared
  allowlist does not. That is RFC-0027 (storytold/craft-contrib#41).
- **Symphonia for other formats.** It still decodes MP3, FLAC, Vorbis,
  ALAC and others in these apps. Replacing it for those is a separate
  decision.
- **Other FilmCraft codecs.** Only the AAC crate moves.

## Alternatives considered

- **Add HE-AAC to symphonia.** It would still leave two AAC decoders in
  the suite, and the work would follow another project's review and
  release schedule.
- **Symphonia's libfdk-aac adapter.** fdk-aac is C under Fraunhofer's
  own licence, which is not on the allowlist, and the apps are Rust only.
- **OS decoders** (Media Foundation, AudioToolbox), the way FilmCraft's
  `crates/platform` decodes video. Linux, FreeBSD and the web build would
  stay muffled.
- **Keep the decoder in FilmCraft and depend on it by git**, as EffectCraft
  does now. Apps would depend on another app's repository, and craft-libs
  crates may not depend on an app, so `craft-audio`, which lists HE-AAC
  as decoded through symphonia, could not use it.
