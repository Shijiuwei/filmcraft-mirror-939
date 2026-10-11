# Architecture

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** trivial (status line, revision history and roadmap link added; content checked against the crate list) · **Target:** Adobe Premiere Pro 2026 (26.5.2)

FilmCraft is a Cargo workspace of small crates with strictly enforced layering. The engine is
headless: every feature can be reached without a window, and the egui UI is one client among the
CLI, the JSON control channel and the MCP server.

Design principles:

1. **Engine-first.** Project-changing actions go through `Session::execute(id, params)`.
2. **Everything is a command.** Stable id, label, menu path, shortcut, parameter doc, `enabled()`
   predicate with a human-readable reason, and `run()`.
3. **Exact time.** Integer ticks, rational frame rates. No `f64` seconds in edit math.
4. **Copy-on-write snapshots.** The project is an `Arc<Project>`. Undo is a stack of snapshots, and
   background readers (playback, export) hold a snapshot without locking.
5. **CPU reference, GPU fast path.** The CPU compositor is the oracle. The GPU path is tested
   against it.
6. **Pure Rust, clean-room.** Codecs and containers are written from public specifications
   (see [AGENTS.md](../AGENTS.md)).

## 1. Layers

```text
 L6  apps/filmcraft · apps/filmcraft-cli · apps/filmcraft-web
 L5  ui-egui · automation · platform
 L4  engine
 L3  render · gpu · export · golden (test-only)
 L2  edit · codecs · interchange · captions · speech · tts · tts-text
 L1  frame · media · project · audio-dsp · text
 L0  foundation: time · geom · color · bitstream · testkit (dev-dependency only)
     codecs/containers: isobmff · matroska · mxf · cfb · mpegts · ogg · h264 · h264enc · hevc · vp9 · av1 · mpeg2v · prores · dnx · apv · aac · ac3 · opus
```

Crates are named `filmcraft-<dir>` (`crates/time` is `filmcraft-time`). The apps are `filmcraft`
and `filmcraft-cli`.

| Crate | Layer | Purpose |
|---|---|---|
| `time` | L0 | `Tick`, `FrameRate`, `TimeRange`, timecode parse/format (NDF/DF, frames, feet+frames, samples) |
| `geom` | L0 | `Vec2`, `Rect`, `Affine`, Motion-transform composition |
| `color` | L0 | colour spaces, transfer functions, YUV↔RGB matrices, LUTs |
| `bitstream` | L0 | bit reader/writer, Exp-Golomb, emulation prevention |
| `isobmff` | L0 | MP4/MOV demux and mux |
| `matroska` | L0 | MKV/WebM demux |
| `mxf` | L0 | MXF demux and mux (SMPTE ST 377-1): OP1a / OP-Atom, index tables, AVC / VC-3 / ProRes / MPEG-2 identification, PCM / AES3 sound, timecode; writer for OP1a / OP-Atom with VC-3, ProRes, AVC and PCM ([README](../crates/mxf/README.md)) |
| `cfb` | L0 | Compound File Binary ([MS-CFB] structured storage) reader and writer, the container of AAF ([README](../crates/cfb/README.md)) |
| `mpegts` | L0 | MPEG-2 Systems demux (H.222.0): transport streams (188 / 192-byte BDAV / 204), program streams (MPEG-2 / MPEG-1, VOB, MOD), PSI, PES, PTS / PCR, access-unit index built on open ([README](../crates/mpegts/README.md)) |
| `ogg` | L0 | Ogg demux (RFC 3533), Ogg Opus timing (RFC 7845 granules, pre-skip, end trimming), Vorbis headers |
| `h264`, `h264enc` | L0 | H.264 decoder; H.264 encoder |
| `hevc` | L0 | H.265 Main/Main 10 decoder |
| `prores` | L0 | ProRes decoder and encoder |
| `dnx` | L0 | DNxHD / DNxHR (SMPTE ST 2019-1 VC-3) decoder and DNxHR encoder |
| `apv` | L0 | Advanced Professional Video (IETF RFC 9924) decoder and encoder ([README](../crates/apv/README.md)) |
| `av1` | L0 | AV1 decoder (Main profile; bit-exact with libdav1d; see its README for the stage table) |
| `mpeg2v` | L0 | MPEG-2 video (Main / 4:2:2 profile; frame and field pictures, dual prime) and MPEG-1 video decoder ([README](../crates/mpeg2v/README.md)) |
| `aac` | L0 | AAC-LC decoder and encoder |
| `ac3` | L0 | AC-3 (ATSC A/52) decoder ([README](../crates/ac3/README.md)) |
| `testkit` | L0 | test-only helpers, used only as a dev-dependency: ffmpeg/ffprobe discovery, fixture dirs, golden images ([testing.md](testing.md)) |
| `frame` | L1 | `VideoFrame` (planar YUV / RGBA8 / linear RGBA f32, colour metadata), `AudioBuffer` |
| `media` | L1 | `MediaSource` trait, probing/openers, frame cache, generators, stills, image sequences, WAV / Broadcast WAV |
| `project` | L1 | document model, effect definitions, keyframes |
| `audio-dsp` | L1 | loudness metering (BS.1770 / R128), audio effects, channel layouts (BS.775 up/downmix, 5.1 panner), music remix analysis; no dependencies |
| `text` | L1 | text engine: font database (bundled OFL fonts + system fonts), shaping (harfrust), bidi, line breaking, paragraph layout, glyph/path rasteriser, strokes ([crates/text/README.md](../crates/text/README.md)) |
| `edit` | L2 | pure edit algebra (insert, overwrite, razor, ripple, roll, slip, slide, rate stretch…; text-based editing: `edit::transcript`) |
| `speech` | L2 | speech-to-text: `Transcriber` trait, Whisper model catalogue + verified downloader (feature `download`), pure-Rust Whisper inference on candle with word timestamps (feature `whisper`), speaker labelling ([transcripts.md](transcripts.md)) |
| `tts` | L2 | text to speech: `Voice` trait, voice catalogue, pause markers, built-in formant voices; Kokoro-82M natural voices on candle and their pinned package catalogue (feature `kokoro`). Same-layer edge to `tts-text` |
| `tts-text` | L2 | text front end for narration: normalizer (numbers, money, dates, acronyms, pauses) and English pronunciation (CMUdict + letter-to-sound) to Kokoro phonemes |
| `codecs` | L2 | container + codec hub: MP4/MOV, MKV, MXF, Ogg and MPEG TS / PS / video elementary stream sources, GOP-aware seeking, decoder registry, audio decoding |
| `interchange` | L2 | EDL, FCP7 XML, FCPXML, OTIO, AAF (on `cfb`) and OMF 2.0 import/export (no file I/O; the engine supplies rendered audio essence) ([README](../crates/interchange/README.md)) |
| `render` | L3 | sequence evaluation, CPU compositor, video effects (`effects`, `vfx`; effects needing other frames or tracks read them through `vfx::FxEnv`), transitions, audio mix |
| `gpu` | L3 | wgpu compositor (WGSL) |
| `golden` | L3 | test-only: golden-image tests of the CPU renderer and GPU-vs-CPU parity; empty library, dev-dependencies only |
| `scopes` | L3 | video scope maths: waveform, parade, histogram, YUV / HLS vectorscopes, numeric summaries ([crates/scopes/README.md](../crates/scopes/README.md)) |
| `export` | L3 | render → encode → mux pipeline, progress/cancel |
| `engine` | L4 | `Session`, command registry, undo history, media pool, jobs, interchange glue |
| `ui-egui` | L5 | the egui frontend: docking, panels, timeline, monitors, playback, control-channel handlers |
| `automation` | L5 | MCP server (`rmcp`, stdio), headless or bridged to the running app |
| `platform` | L5 | OS media FFI only: hardware video decoding (VideoToolbox H.264 / HEVC on macOS; Media Foundation / Direct3D 11 H.264 / HEVC / VP9 / AV1 on Windows; VA-API H.264 / HEVC on Linux, with our own parsers and DPBs driving the stateless hardware; a no-op elsewhere) behind `codecs::VideoDecoder`, with transparent fallback to our decoders, and hardware H.264 encoding (VideoToolbox, opt-in) and H.265 encoding (VideoToolbox, the only H.265 encoder; the format exists only where a hardware encoder does) and NVIDIA NVENC H.264 (Windows and 64-bit Linux, opt-in) and H.265 (Windows, FilmCraft's only H.265 encoder there) encoding behind `export::VideoEncoder`; H.264 declines to the built-in encoder for what the hardware does not take. The one crate allowed `unsafe` ([ADR 0001](adr/0001-platform-ffi.md), [README](../crates/platform/README.md)) |
| `filmcraft` | L6 | desktop binary: eframe/wgpu window, cpal audio output, file dialogs, native macOS menu, TCP control server |
| `filmcraft-cli` | L6 | headless CLI: `exec`, `run`, `inspect`, `describe`, `commands`, `import`, `export`, `render`, `probe`, `mcp`; `--bridge` targets the running app |
| `filmcraft-web` | L6 | the browser app (wasm32): eframe web runner on WebGPU/WebGL2, Blob-backed services, OPFS recovery, WebAudio, WebCodecs, `window.filmcraft` API ([web.md](web.md)) |

### What `cargo xtask layers` enforces

The table of layers lives in `xtask/src/main.rs` (`LAYERS`). It also reserves names for planned
crates. The check reads `cargo metadata` and looks at normal and build dependencies (dev-dependencies
are exempt):

| Rule | Detail |
|---|---|
| Every crate has a layer | A new crate fails the check until it is added to `LAYERS`. |
| Only downward edges | A crate may not depend on a crate in a higher layer. |
| Same-layer edges are listed | From L1 up, a same-layer edge must be in `SAME_LAYER`: `media→frame`, `project→media`, `project→frame`, `gpu→render`, `export→render`, `cli→filmcraft`, plus a few reserved for planned crates. |
| L0 codecs stay standalone | L0 crates other than `time`, `geom`, `color`, `bitstream`, `testkit` may depend on no workspace crate except `filmcraft-bitstream`. External crates such as `thiserror` and `rayon` are allowed. |
| No UI/OS crates below L5 | `egui`, `eframe`, `egui-wgpu`, `winit`, `rfd`, `cpal`, `muda` are allowed only in L5 and L6. |

`cargo xtask wasm` runs `cargo check --target wasm32-unknown-unknown` on every L0–L4 crate, the
egui UI and the web app, so everything up to the engine stays web-portable and the web app builds
([web.md](web.md)). `unsafe_code = "forbid"` applies workspace-wide, except in `platform`
(`deny`, allowed only on its FFI modules; [ADR 0001](adr/0001-platform-ffi.md)).

## 2. Time base

All time is `filmcraft_time::Tick(i64)` at `TICKS_PER_SECOND = 254_016_000_000`.

That number divides evenly into the frame duration of every broadcast rate (23.976, 24, 25, 29.97,
30, 48, 50, 59.94, 60, 120…) and the sample duration of every common audio rate (8 kHz to 192 kHz,
including the 44.1 kHz family). So frame and sample positions are exact integers, edits never drift
at 29.97, and audio and video line up to the sample.

| Type | Use |
|---|---|
| `Tick` | timeline and media positions and durations |
| `FrameRate { num, den }` | `frame_duration()`, `tick_of(frame)`, `frame_at(tick)`, `snap(tick)` |
| `TimeRange` | half-open `start + duration` |
| Timecode | display only (SMPTE NDF/DF, frames, feet+frames, samples); `parse_timecode` and `format_time` |

Commands take time as `time` (ticks), `frame`, `seconds` or `timecode`; the engine converts once at
the boundary.

## 3. Data model (`filmcraft-project`)

```text
Project
├─ root: Bin                         tree of bins
├─ items: map ItemId → ProjectItem   flat
│    kind: Media(MediaClip) | Sequence(Sequence) | Subclip{..} | AdjustmentLayer{..} | Graphic{..}
└─ next_id

Sequence
├─ settings: frame rate, size, sample rate, …
├─ video_tracks / audio_tracks: Vec<Track>
├─ markers, mark_in / mark_out
Track
├─ locked, sync lock, targeting, mute/solo/visibility
├─ items: Vec<TrackItem>             sorted, never overlapping
├─ transitions: Vec<Transition>
└─ audio: volume_db, pan, effects (mixer inserts), mixer: MixerStrip
     (automation mode + lanes, sends, output, record arm, solo safe, input map)
Sequence (audio) ─ submix_tracks: Vec<Track>, master_volume_db / master_effects / master_mixer
TrackItem (a clip instance)
├─ item: ItemId, start (timeline ticks), source_in (media ticks), duration, speed
├─ link group, label, enabled
└─ effects: Vec<EffectInstance>      intrinsic Motion/Opacity/Volume… first, then standard effects
EffectInstance
├─ effect id, enabled, params: id → constant value or keyframe track
└─ masks: Vec<Mask>                  path (keyframable Bézier), feather, opacity, expansion, inverted, mode
```

- **Graphic clips** ([graphics.md](graphics.md)) reference a `Graphic` canvas item; their text and
  shape layers are hidden `graphic_text` / `graphic_shape` effect instances on the track item.
- Everything is plain serde data. `Sequence::check()` validates the invariants (no overlaps, unique
  ids), and the engine runs it after every sequence edit.
- Timeline positions are sequence ticks; `source_in` and keyframes are in media time, so trims and
  splits never move keyframes.
- **Effect definitions are data.** `project::effect::effect_defs()` lists every video effect,
  audio effect and transition with its parameter schema. The Effects panel tree, the Effect
  Controls rows and the parameter docs agents see are all generated from it.
- **Project files** (`.fcproj`) are the project serialised as JSON. Saves are atomic: write a
  temporary sibling file, then rename it over the target.

## 4. Command system (`filmcraft-engine`)

```rust
pub struct CommandSpec {
    pub id: &'static str,                 // "sequence.addEdit"
    pub label: &'static str,              // "Add Edit"
    pub menu: &'static [&'static str],    // ["Sequence"]; empty = not in menus
    pub shortcut: Option<&'static str>,   // "Cmd+K" (Cmd = ⌘ on macOS, Ctrl elsewhere)
    pub params: &'static str,             // r#"{"time":ticks?}"#, shown to agents
    pub enabled: fn(&Session) -> Result<(), String>,  // Err carries the reason
    pub run: fn(&mut Session, &Value) -> Result<Value>,
    pub journal: bool,                    // false for read-only queries
}
```

- All commands are in `crates/engine/src/commands.rs` (`cmd!` for actions, `query!` for read-only
  queries such as `project.inspect`, `sequence.inspect`, `effects.list`, `jobs.list`). Ids follow
  the menu structure: `file.*`, `edit.*`, `clip.*`, `sequence.*`, `markers.*`, `timeline.*`,
  `effects.*`…
- `Session::execute(id, params)` finds the spec, checks `enabled`, runs it and appends it to the
  journal.
- **Undo.** Edits go through `Session::edit(label, |project, state| …)` or
  `Session::edit_sequence(label, |seq, ctx, state| …)`. These clone the project, apply the closure,
  and on success push the old `Arc<Project>` with the label onto the undo stack (200 entries).
  On error nothing changes. `edit.undo` and `edit.redo` swap snapshots. Thanks to structural sharing
  a snapshot costs little.
- **Editor state** (`EditorState`: active sequence, playheads, selection, targeting, edit points…)
  is serde, so agents can read it with `state.inspect`.
- **Events** (`ProjectChanged`, `Toast`, `OpenSequence`, `OpenSource`) are drained by frontends
  each frame.
- **Event log** (`Session::log`, `engine::panels`): every failed top-level command (an error; a
  disabled one is a warning), messages and error toasts, auto-save errors, and background jobs
  starting, finishing, failing or being cancelled. Repeats of the newest entry bump its count. The
  Events panel shows it; `events.list {level?, since?}` and `events.clear` reach it headless.
- **UI-only commands** (tools, playback, zoom, panels, workspaces) live in
  `crates/ui-egui/src/menus.rs` (`UI_COMMANDS`). The menu bar is built from the engine registry plus
  this table, and `menus::invoke` is the single entry point for menus, shortcuts and the control
  channel.

## 5. Media, render and playback pipeline

```text
file ──► codecs (MP4/MOV, MKV, audio)        demux + decode, GOP-aware seek
          │   decoder registry: h264, hevc, vp9, av1, prores, dnx, mjpeg (+ any registered first:
          │   platform's VideoToolbox (macOS) or Media Foundation (Windows) H.264 / HEVC, falling back to ours)
          ▼
        media::MediaSource ──► frame cache (byte-budgeted LRU, shared)
          ▼
        render::render_sequence(project, seq, t, scale)        CPU reference
          per track bottom→top: map timeline t → media t (speed), fetch frame,
          standard effects → Motion → Opacity/blend, transitions, composite
          in linear-light premultiplied f32
          │
          └─ render::plan::plan_frame → gpu::GpuCompositor     GPU path
               layers = decoded YUV/RGBA frames + matrix + opacity + blend mode;
               anything the shaders don't cover is pre-rendered on the CPU
          ▼
        ui-egui frames.rs worker pool ──► monitors (program/source), thumbnails, prefetch
```

- **Sources.** `media::MediaSource` yields `video_frame(FrameRequest)` and
  `audio(start, frames, rate)` in media time. Sources are `Send + Sync` and shared by monitors,
  thumbnails, playback and export. The engine's `MediaPool` creates one per project item, lazily,
  through registered openers (`codecs::openers()`: MP4/MOV, MKV/WebM, audio files).
- **Pixel aspect ratio.** Demuxers report it in `VideoStreamInfo::par` (MP4/MOV `pasp`, Matroska
  display size, MXF aspect ratio, MPEG sequence header / VUI); Interpret Footage's
  `Interpretation::par` overrides it (`clip.interpretFootage {pixelAspect}`), and
  `SequenceSettings::par` is the sequence's. Both go through `project::checked_par` (a zero term
  or more than 8:1 either way = square). `render::motion_matrix` works in square display units,
  so a 1440 x 1080 clip with 4:3 pixels fills a 1920 x 1080 square-pixel sequence at 100 %
  (Scale to Frame Size, Set to Frame Size and Fit / Fill use `Project::conformed_source_size`);
  graphics and adjustment layers are in sequence pixels. New Sequence From Clip copies the clip's
  ratio, and the Source and Program monitors draw the picture at its display aspect. The
  renderer does not read decoded frames' own `VideoFrame::par`.
- **Offline media and proxies.** The pool caches each item's source per reference (path, offline
  flag), so relinking and undo take effect at once. Media that can't be opened renders the offline
  slate (`render::offline`) instead of failing. With proxies enabled, an item with a proxy reads
  it through `ProxySource`, which reports the original's size. Export always uses
  `MediaPool::full_res_provider`. See [project-files.md](project-files.md#media-offline-relinking-proxies-ingest).
- **Seeking.** `codecs::Mp4Source` seeks to the preceding sync sample and decodes forward, caching
  every frame of the GOP. Sequential playback reuses the decoder. Decoders implement
  `codecs::VideoDecoder`. `register_video_decoder` puts a factory in front of the built-in ones, so a
  hardware decoder can take precedence. The GOP cache never holds its lock while decoding.
- **Hardware decoding.** The apps (desktop, CLI / headless MCP, bench) call
  `filmcraft_platform::register()` at startup, which on macOS registers a VideoToolbox factory for
  `avcC` / `hvcC` streams (8 / 10-bit, 4:2:0 and 4:2:2; elsewhere it does nothing), and on Windows a
  Media Foundation one (below). The factory
  declines (our decoder is used) when Settings ▸ Playback ▸ Hardware decoding is Off
  (`codecs::hw::set_hardware_decoding`, applied by the engine whenever preferences change), for
  formats it does not take, and when the OS cannot create a *hardware* session for the stream.
  Its decoder is a `platform::HybridDecoder`: VideoToolbox decodes asynchronously (two access
  units in flight), a reorder buffer of the stream's own depth (`max_num_reorder_frames` /
  `sps_max_num_reorder_pics`, from `codecs::hw::NalStreamInfo`) restores presentation order, and
  the decoded biplanar `CVPixelBuffer` is copied into planar `Yuv8` / `Yuv16`. Colour, pixel
  aspect, random-access and disposable answers come from the same helpers as the software
  decoders (`video::vui_color`, `sar_par`, `h264_disposable`, `hevc_disposable`), so the two are
  interchangeable (bit-exact in the parity tests). If the hardware fails mid-stream (decode error,
  session lost to a GPU change or sleep, changed in-band parameter sets), the hybrid builds
  `codecs::software_video_decoder` (built-in factories only), replays the samples since the last
  restart point and continues in software for that instance; it is logged and counted. A source
  already open keeps its decoder when the setting changes, until it is reopened. `perf.stats`
  `decode.hardware` reports hardware vs software frames, sessions, declines and fallbacks, and the
  registered `backend`.
  On Windows the decoder (`platform::media_foundation`) is a Direct3D-aware decoder MFT driven at
  the level of single access units (no Source Reader: the demuxer above already delivers the
  samples): the `avcC` / `hvcC` sample becomes Annex B, the MFT is given the process's Direct3D 11
  video device through an `IMFDXGIDeviceManager` so it decodes with DXVA, and its NV12 (8-bit) /
  P010 (10-bit) texture is read back through a staging texture (the one GPU to CPU copy) into
  planar `Yuv8` / `Yuv16`. H.264 Baseline / Main / High, HEVC Main / Main 10, VP9 profiles 0 / 2 and AV1 main
  (8- and 10-bit), 4:2:0, progressive (VP9 / AV1 samples go in as they are, not as Annex B);
  everything else, and any stream the GPU's DXVA decoder does not list, is declined. A decoder that
  would hand back system-memory pictures (Microsoft's decoder MFTs do that when DXVA is not
  available) fails the stream, so the hybrid continues with our decoder and Windows' software
  decoding is never used in its place. Zero-copy (the texture straight to wgpu) is not done yet.
  Decoders run slices on rayon, so an export worker waiting inside a decode can pick up another
  frame of the same source. A request that finds the shared decoder busy decodes with a private
  decoder.
- **Hardware encoding.** On Windows and 64-bit Linux, `platform::nvenc::export::factory` is registered with
  `filmcraft_export::register_encoder` (`register()` does this once), in front of the software H.264
  encoder. It takes an export only when Export ▸ Hardware encoding (`ExportSettings.hardwareEncoding`,
  `HardwareEncoding::Auto`) is Auto, which is off by default: hardware streams differ from ours, and
  exports are otherwise byte-identical from run to run. NVENC (`platform::nvenc`, the driver's
  `nvEncodeAPI64.dll` / `libnvidia-encode.so.1` loaded at run time) then takes RGBA frames, converted to 4:2:0 by the software
  encoder's own conversion, and produces the H.264 samples and `avcC` of an MP4 / MOV. It declines
  (the software encoder runs, counted in `export.hardware.declined`) two-pass VBR, HDR, MXF,
  interlaced output, sizes outside NVENC's limits, and systems without an NVIDIA GPU or driver.
  A failure during an export ends it with an error: the software encoder cannot take over a hardware
  stream. The same NVENC session also encodes H.265 (HEVC Main 8-bit 4:2:0 SDR, or Main 10 HDR: PQ / HLG,
  BT.2020, limited range, from the float R'G'B' pictures through `rgbf_to_yuv420_10` into P010 input
  buffers, with the HDR10 static metadata as SEI on every IDR for PQ; the `hvcC` is built from the SPS
  the encoder wrote). An HDR sequence exports as HDR H.265 only where a registered probe says so
  (`filmcraft_export::register_hdr_probe`, `hdr_available`; NVENC with 10-bit support), otherwise it is
  tone-mapped to SDR as on macOS: `register()` hands `platform::nvenc::hevc_available` to
  `filmcraft_export::register_format_probe`, so the H.265 format exists where an HEVC encoder does,
  and choosing it is the opt-in (the Hardware encoding toggle governs H.264 only). With no software
  H.265 encoder, what NVENC declines is counted and ends the export with an error that says why.
- **Compositor.** `render` is the reference for monitors, thumbnails and export. `render::plan`
  turns a frame into GPU layers, each with its opacity and blend mode. All 27 blend modes run on the
  GPU (`filmcraft-gpu`): Normal and Dissolve with fixed-function "over" blending, the others by
  copying the accumulator under the layer into a backdrop texture and compositing in the fragment
  shader with the CPU reference's formulas. The common standard effects run on the GPU too
  (`render::gpufx`, `gpu::fx`): Brightness & Contrast, ProcAmp, Tint, Black & White, Color
  Balance, Leave Color, Change to Color, Color Pass, Color Replace, Channel Mix, ASC CDL, Gamma
  Correction, Levels, Extract, Invert, Posterize, Alpha Adjust, Gaussian Blur and Directional Blur
  (and their legacy aliases), Camera Blur, Sharpen, Unsharp Mask, Crop, Edge Feather, Transform,
  Horizontal / Vertical Flip, Mirror, Offset, Vignette and Video Limiter. When every enabled effect of a media clip is in
  that set (unmasked, with finite parameters, and no Transform shrinking the picture below half
  size, which the CPU pre-filters), the plan hands the GPU the clip's source with
  the effects' parameters evaluated at that time: the source is drawn into an `Rgba32Float`
  working image at the size the CPU decodes it, each effect runs as compute passes with the CPU
  reference's math (effects, then Motion, then Opacity / blend, as on the CPU), and the result is
  placed like any layer. Other standard effects, effect and opacity masks, adjustment layers,
  nested sequences (except a plain one, whose own layers go into the plan: same frame size and
  colour settings as its parent, nothing on the clip that changes the picture, every layer inside
  blended normally) and non-dissolve transitions are rendered on the CPU for that layer or frame
  and handed to the GPU as an image (a layer image keeps its clip's blend mode), so both paths
  give the same picture. Setting `FILMCRAFT_CPU_COMPOSITE=1`
  forces the CPU path in the desktop app.
- **CPU compositor shortcuts.** The CPU compositor, which is what export runs, works on
  premultiplied linear `f32` images, 33 MB at 1080p, so what it does not allocate, convert or mix
  is time saved. Three shortcuts, each of which gives the general path's bits (compared bit for
  bit, with every blend mode, in `render/src/region_tests.rs`): the canvas stays unallocated
  until a layer needs it, and an opaque bottom layer (an 8-bit or 16-bit Y'CbCr frame without an
  alpha plane, opacity 100 %, Normal) simply becomes the canvas; a plain media frame with an alpha
  plane (a ProRes 4444 banner) is converted and mixed only inside the rectangle where its alpha is
  not zero (`VideoFrame::alpha_region`, `to_linear_f32_region`, `blend::composite_at`), the rest
  being transparent anyway; and working images are recycled through `filmcraft_frame::pool`
  (`take_f32_overwritten` / `recycle_f32`: the buffer comes back with its old contents, so there is
  no zero-fill) by the export pipeline once a frame is converted to 8 bits; a standalone export's
  pipeline frees the idle float images (`pool::trim_f32`) when it goes away, so a finished export
  leaves none behind (parts of a batch, such as render-preview segments, keep them for the next part). A clip
  with effects, opacity masks, frame blending, colour management, or a picture that is not placed
  one to one on the output takes the general path.
- **Frame scheduling.** `crates/ui-egui/src/frames.rs` runs a small pool of worker threads with
  prioritised jobs: the frame on screen first, then playback prefetch, then thumbnails. The UI never
  decodes. It shows the exact frame when it is ready and holds the nearest cached frame meanwhile.
  Play waits for the first frames (`PREROLL_FRAMES`, at most 0.5 s) before starting the clock.
  Every refresh, `schedule_playback` asks for the next frames and drops (or cancels, through
  `filmcraft_media::cancel`) jobs for frames the playhead has passed; a new on-screen frame
  replaces the one asked for before (scrubbing); Stop cancels the prefetch. When frames cost more
  than the workers can render in real time (CPU effects), it starts only frames that can still be
  on time and spaces them evenly (`playback_plan`). When decoding is what falls behind, each job
  tells its sources which frames are already late (`filmcraft_media::cancel::with_catch_up`: the
  playhead's distance, two seconds for a scrubbed-to frame), and the GOP cache skips
  non-reference pictures of late frames on the way (`VideoDecoder::is_disposable`, H.264
  `nal_ref_idc` 0 and HEVC sub-layer non-reference pictures; the wanted frame decodes exactly as
  before). With Settings ▸ Playback ▸ Draft decoding (off by default), frames played at 1/2
  resolution or lower are requested as draft frames (`FrameKey::draft`,
  `filmcraft_media::cancel::with_draft`): the H.264 decoder skips deblocking of non-reference
  pictures (`VideoDecoder::set_draft`), the GOP cache serves those draft frames to draft
  requests only (a paused frame, render or export decodes them again exactly), and draft plans
  hand the GPU box-decimated Y'CbCr planes at the drawn size. `perf.stats` reports the counters.
  GPU plans carry their texels already converted for upload (`filmcraft_gpu::prepare`), so the UI
  thread only copies them.
- **Audio clock.** The desktop app passes a cpal output (`apps/filmcraft/src/audio.rs`) to the UI as
  `AudioOut`. While playing, the samples played by the sound card drive the playhead and video follows.
  Without an audio device, playback falls back to the wall clock. `PlaybackMeter` counts timeline
  frames: shown when the exact picture was on screen while due, dropped otherwise (including frames
  passed over without a refresh, not while the window is hidden). `cargo xtask bench-playback`
  measures the whole path headlessly ([testing.md](testing.md) §5).
  Sequence audio goes through the mixer graph (`render::mixer`, §5.1); clip audio effects run on
  `audio-dsp` via `render::audio_fx`.

### 5.1 Audio mixer

```text
clip: gain → clip effects → Volume / Channel Volume / Panner (clip keyframes, media time)
      → audio transitions → summed per track                           render::audio::track_input
track / submix strip:  input map, mono fold → pre-fader inserts → pre-fader sends → mute
      → fader (volume) → meter → post-fader inserts → post-fader sends → pan / balance → output
Mix:  bus sum → pre-fader inserts → fader → meter → post-fader inserts → out    render::mixer
```

- **Model** (`project::mixer`). Every audio track, submix and the Mix has a `MixerStrip`. Static
  values stay in `Track::volume_db`, `pan`, `muted` and the send/effect parameters; automation is
  keyframes in sequence ticks: lanes `volume`, `pan`, `mute` (hold), `send.<i>.level` in
  `MixerStrip::lanes`, and insert parameters (`fx.<slot>.<param>`) in the effect's own keyframes.
  Up to 5 inserts (`EffectInstance::post_fader` picks the side) and 5 sends per strip. Submixes feed
  the Mix or a submix after them (no feedback). All fields have serde defaults, so older projects
  load unchanged.
- **Graph** (`render::mixer::mix_graph`). Lanes are evaluated per sample; effect parameters update
  on an absolute 64-sample grid, so the output does not depend on how callers cut the timeline into
  requests (export batches and device callbacks give identical samples). Inserts that report
  latency delay their strip; each route into a bus gets a compensation delay and the graph is read
  ahead by its total latency. Graph state (DSP, delay lines) is cached per structure and continued by
  sequential readers; other requests start fresh with a pre-roll (effect tails, ≤ 3 s). Tracks run
  in parallel (rayon), buses in order. Mono tracks pan with the −3 dB constant-power law; stereo
  tracks and sends use balance. Solo keeps soloed and solo-safe strips plus everything feeding them
  or fed by them. 24 tracks × (EQ + Dynamics + Studio Reverb) + a compressed submix renders at
  ~6× realtime on one core (release).
- **Automation modes** (Premiere semantics). Off ignores lanes; Read plays them; Latch records from
  the first touch and holds the last value until playback stops; Touch records while held and ramps
  back to the existing automation over the **automatch time** (Preferences ▸ Audio, 1 s); Write
  records every control from playback start (then switches to Touch unless "Switch to Touch after
  Write" is off).
- **Recording** (`engine::mixer`). Playback start runs `mixer.recordStart`, stop runs
  `mixer.recordStop`. Fader and knob drags send `mixer.touch` (value, playhead) while held and
  `mixer.release` when let go. Held values go to `render::mixer::LiveMix`, which the playing mix
  reads, so moves are heard at once. At stop each gesture stream is thinned (linear keyframe
  thinning, optional minimum time interval) and written over its time range as one undo step,
  with boundary keyframes that keep the automation outside the range unchanged.
- **Live state.** `PreviewStore::live` (`LiveMix`) also carries per-strip meter peaks posted by the
  mix (one per channel; Track Mixer and Audio Meters read them), the voice-over input level (Meter
  Input(s) Only) and the newest project snapshot, which the audio callback uses, so edits made
  during playback are heard.
- **Audio Clip Mixer automation.** Each audio track has a Clip Mixer mode (`clipMixer.setMode`,
  session state `EditorState::clip_mixer_modes`, Read by default). During a pass the Clip Mixer's
  fader and pan knob are gestures on the track's `clip.volume` / `clip.pan` lanes
  (`clipMixer.touch` / `clipMixer.release`): the clip under the playhead plays the held value at
  once (`render::audio::track_input_live`), and Latch / Touch / Write follow the track rules. At
  `mixer.recordStop` each stream is thinned and written as **clip keyframes** (Volume `level`,
  Panner `balance`, in media time) into every clip it covers, with boundary keyframes that keep the
  clip's values outside the range; one undo step for the whole pass.
- **Track Mixer view.** The panel menu (`mixer.menu`) has Show/Hide Tracks (`UiState::mixer_hidden`)
  and Meter Input(s) Only (`UiState::mixer_meter_input_only`: record-armed tracks meter the
  voice-over input). The transport's record button runs `voiceover.recordToggle`.

### 5.1.1 Multichannel and 5.1

```text
5.1 track (6 ch: L R C LFE Ls Rs) ─┐                         ┌─► 5.1 Mix ─► 6-ch export / 5.1 device
stereo / mono track ──5.1 panner───┼─► 5.1 submix / 5.1 Mix ─┤
5.1 strip ──BS.775 downmix─────────┴─► stereo bus            └─► stereo device: 5.1 Mixdown Type fold
```

- **Model.** `Track::channels` (Standard = Stereo, Mono, 5.1, Adaptive) and
  `SequenceSettings::audio_master` (the Mix: Stereo, Mono, 5.1, Adaptive) existed already;
  `file.newSequence {mix, trackType}` and `sequence.settings {mix}` set them, `mixer.setStrip
  {channels}` / `mixer.addSubmix {channels}` per strip. The 5.1 panner is four lanes in
  `MixerStrip::lanes` (`pan51.x`, `pan51.y` −100…100, `pan51.center` 0…100 %, `pan51.lfe` dB): a
  lane without keyframes stores the static value, so no new project fields were needed and the
  puck is automatable like any lane.
- **Maths** (`audio_dsp::channels`). 5.1 order is L, R, C, LFE, Ls, Rs everywhere inside FilmCraft.
  Downmix is ITU-R BS.775 (`Lo = L + k·C + k·Ls`, `Ro = R + k·C + k·Rs`, mono
  `M = k·L + k·R + C + ½·Ls + ½·Rs`, `k = 1/√2`, LFE omitted); upmix puts mono on C and stereo on
  L/R. The panner splits a point source front/rear and left/right with the sine/cosine
  constant-power law and gives the centre speaker `center·(1 − |x|)` of the front power (Σg² = 1).
  Stereo sources sit at the puck ± 1 in x; 5.1 sources keep their layout moved by the puck's offset
  from front-centre (the default puck is the identity). LFE gets the source LFE (5.1) or the mean of
  the source channels (mono/stereo, default −∞) times the LFE level.
- **Graph.** Buses are 2 or 6 channels wide (`render::mixer::width_of`); inserts are built for the
  strip's width; routes convert between widths (5.1 panner into 5.1 buses, BS.775 into stereo
  buses); meters have one value per channel. `mix_graph` returns the Mix's width;
  `audio::mix_sequence` always returns stereo, `mix_sequence_layout` any layout and mixdown. Clips
  on 5.1 tracks play the source's first six channels (or the six picked with Modify ▸ Audio
  Channels), mono/stereo sources upmixed; clip effects run on all six channels.
- **Playback.** On a device with ≥ 6 output channels a 5.1 Mix plays as six channels; on a stereo
  device it is folded with Preferences ▸ Audio ▸ 5.1 Mixdown Type (Front Only, Front + Rear
  Surround = BS.775, Front + LFE, Front + Rear Surround + LFE; `PreviewStore::mix_layout`).
- **Export.** Audio Channels Mono / Stereo / 5.1: WAV writes `WAVE_FORMAT_EXTENSIBLE` with channel
  mask `0x3F`; QuickTime PCM (ProRes, DNxHR, MJPEG) adds a `chan` atom
  (`kAudioChannelLayoutTag_MPEG_5_1_A`); AAC uses channel configuration 6 (our encoder already
  supported 1–8 channels; the exporter reorders to AAC's C, L, R, Ls, Rs, LFE). A stereo export of a
  5.1 Mix is the BS.775 downmix.

### 5.1.2 Voice-over recording

**Voice-over recording** (`engine::voiceover`, `audio.voiceover.*`). The record point R is the playhead, or the In point when In/Out are set (punch-in; punch-out at Out). Playback starts the pre-roll before it (C = max(0, R − pre-roll)) and the input is captured from C. When the UI's audio clock really starts, `audio.voiceover.sync` restarts the capture there. `audio.voiceover.stop` keeps the audio from R to min(stop, Out), writes a mono 32-bit float WAV `<Name> <n>.wav` (Scratch Disks ▸ Captured, else next to the project, else the data or temporary directory), imports it and overwrites it onto the record track at R as one undo step. The record track is the given one, else the record-armed one, else the first targeted one. Input goes through the `AudioInput` trait: cpal in the desktop app (`apps/filmcraft/src/audio_in.rs`, on its own thread), and `SyntheticInput` headless, which produces exactly the samples the timeline asks for so recordings land sample-accurately. Voice-Over Record Settings (Source, Input channel, Name, Countdown Sound Cues, Pre-/Post-roll) are preferences (`voiceOver`). The track header's microphone records or stops, and a right-click opens the settings dialog. Countdown beeps (1 kHz, 100 ms, each whole second of pre-roll and at R) are mixed into playback, an overlay counts down, and playback stops at Out + post-roll.

### 5.1.2a Text to Speech (narrations)

**Narrations** (`engine::narration`, `tts.*`; voices in `filmcraft-tts`). The Text to Speech panel (Window ▸ Text to Speech) writes a script with pause markers (`[pause 1s]`), a voice, a vocal pitch and a pace; `tts.create` synthesizes it, writes `Narration <n>.wav` (mono 32-bit float, 24 kHz, where voice-over recordings go), imports it and places it at the playhead on the first free targeted audio track, else the first free audio track, else a new one, as one undo step. The settings are kept per generated item in `Project::narrations` (schema v14). Selecting a narration clip loads it into the panel; `tts.edit` writes a new file (beside the previous one) and swaps the clip to it, keeping the clip's start, duration and speed: longer speech is kept whole in the file but plays only to the clip's end; shorter speech is padded with silence to the clip's length. The old item and narration stay, so undo is exact. `tts.preview` synthesizes without changing the project and the UI plays `Session::tts_preview` on the audio output. Built-in voices are an original formant synthesizer (robotic, no download). The natural voices (feature `neural-voices` of the desktop app, off by default like `whisper`) are Kokoro-82M on candle (`filmcraft-tts`) fed by `filmcraft-tts-text` (normalizer, CMUdict + letter-to-sound); their 336 MB package is downloaded once after the user confirms (`tts.downloadVoices`, a job). The UI synthesizes with `tts.render` (a job filling `Session::tts_cache`) and then runs create / edit / preview, which use the cache.

### 5.1.3 Remix

```text
clip media (source In … +duration) ─► audio_dsp::remix::analyze: spectral-flux onsets → tempo (autocorrelation, 50–200 BPM)
  → DP beat tracker → per-beat chroma + mel cepstra → beat self-similarity
  ─► plan(target, Segments, Variations): intro … joints at beat boundaries … outro, length within one beat
  ─► hidden clip effect `remix` (target, sliders, original duration, plan in ticks) ─► render::remix::read (20 ms equal-power crossfades)
```

Clip ▸ Remix ▸ Enable Remix / Remix Properties… / Revert Remix (`clip.remix.*`; agents can use `clip.remix {duration|seconds…}`) and the Remix tool in the Rate Stretch group (drag a music clip's Out edge) retime a music clip to a target duration. The analysis finds the beat and compares every pair of beats by harmony (chroma) and timbre (cepstra). Each joint leaves the music at one beat boundary and continues at the beat boundary whose four beats on either side sound most alike. The intro and outro are kept, and the result lands within one beat of the target. Segments (0–100) sets the number of joints (1–3, more for long extensions). Variations (0–100) sets how far a joint may move from its evenly spaced position (±1–8 beats). The state is a hidden effect instance with no definition, so panels, the effect chain and Paste Attributes ignore it. The plan is stored in ticks, so playback and export read it without re-analysing. A remix never moves other clips: one that would overlap the next clip is refused, and Revert restores the original duration. Results are deterministic and the analysis is cached per media range.

### 5.2 Essential Sound

```text
clip.essential (type + settings)  ──apply()──►  clip effects marked `essential`, clip gain, Volume, Panner
essentialSound.autoMatch   BS.1770 integrated loudness of render::audio::clip_signal (mono clips as one channel) → match gain (clip gain)
essentialSound.generateDucking   trigger clips' summed level (10 ms hops) → activity regions → Volume keyframes
```

- **Model** (`project::essential`). A clip's `essential: Option<EssentialSound>` holds its audio type
  (Dialogue / Music / SFX / Ambience) and per-type sections: Loudness (match gain, measured and target
  LUFS), Repair (Reduce Noise, Reduce Rumble, DeHum 50/60 Hz, DeEss, Reduce Reverb), Clarity
  (Dynamics, EQ preset + amount, Enhance Speech), Creative (Reverb preset + amount, Stereo Width for
  Ambience), Ducking (against types, sensitivity, reduce by, fades), Pan, Clip Volume and Mute.
- **Effects under the hood.** `essential::apply(item, old, t)` turns the settings into ordinary clip
  effects (Highpass, DeNoise, DeHummer, DeEsser, DeReverb, Dynamics Processing, Parametric Equalizer,
  Enhance Speech, Stereo Width, Studio Reverb) flagged `EffectInstance::essential`, in that order, ahead
  of the user's own effects. Only parameters whose derived value changed are written (at the playhead),
  so keyframes added in Effect Controls survive. A section switch bypasses its effects, a slot switch
  removes its effect, clearing the type removes them all. Auto-match gain, Clip Volume (an offset on
  the Volume level or on all its keyframes) and Pan are applied as deltas, so clearing restores the
  clip. Because these are normal effects, playback, the mixer and export need nothing special.
- **Loudness.** The clip signal (clip gain + effects, before Volume) is measured with the BS.1770
  meter; the gain is linear after the effects, so one measurement hits the target exactly. Targets are
  preferences (`audio.dialogueTargetLufs` −23, `musicTargetLufs` −25, `sfxTargetLufs` −21,
  `ambienceTargetLufs` −30).
- **Ducking.** Sensitivity 0…10 maps to a threshold −20 − 4·s dBFS on the summed trigger signal (50 ms
  window); regions shorter than 100 ms are dropped and pauses under 250 ms bridged; each region gets a
  fade-down before it and a fade-up after it (`audio_dsp::ducking::duck_keyframes`), written as Volume
  level keyframes that replace earlier ones.
- **Presets** (our own names and values) per type; user presets are saved in preferences
  (`essentialSound.userPresets`). Music remixing to a duration is Clip ▸ Remix (§5.1.3).
- **Enhance Speech** is a DSP chain (high-pass, de-mud, presence and air EQ, expander, compressor), not a
  model: no speech-enhancement model with an open licence that we could ship and verify is bundled.
  DeepFilterNet (MIT/Apache-2.0, Rust inference via tract) is the candidate for a future optional
  integration behind a trait.

### 5.3 Masks

Every video effect and the intrinsic Opacity carry `EffectInstance::masks` (`project::mask`).
A mask is a closed cubic Bézier `MaskPath` in clip pixels (ellipse = four smooth vertices with
circular tangents, 4-point polygon = corner vertices, the pen draws arbitrary vertices), stored as a
`ParamValue::Path` so the ordinary keyframe engine animates it (vertex-wise interpolation; paths
with different vertex counts hold). Feather, Opacity and Expansion are ordinary float parameters.

- **Coverage** (`render::mask`): the path is flattened (≤ 0.05 working px chord error); the
  signed distance to the polygon (nonzero winding) plus Expansion goes through a falloff of width
  max(Feather, 1) centred on the edge (linear = exact box-filtered antialiasing at Feather 0,
  blending into smoothstep as Feather grows). Masks combine top to bottom with Add / Subtract /
  Intersect / Lighten / Darken / Difference; Inverted and Opacity apply per mask.
- **Semantics.** A masked effect is `lerp(original, effected, coverage)` per premultiplied channel
  (the effect only applies inside); Opacity masks scale the clip's layer before Motion, so they
  follow the clip's transform. Adjustment-layer masks are in sequence pixels.
- **GPU.** `filmcraft-gpu::GpuMask` evaluates the same coverage and mix in WGSL (compute), tested
  to agree with the CPU within 3·10⁻⁶. Layers with masks are CPU-rendered images in frame plans.
- **Editing.** `masks.*` commands (add / remove / set / moveVertex / translate / addVertex /
  removeVertex / toggleVertexSmooth / roundCorners / select / list; `masks.add` takes a
  `cornerRadius` for a rounded rectangle); keyframe commands take `"mask": n`. The
  Program monitor overlay drags vertices, Bézier handles, the whole mask and the feather /
  expansion handles; drags merge into one undo step.
- **Tracking** (`render::track`, `masks.track`): Shi–Tomasi features inside the mask, pyramidal
  Lucas–Kanade (4 levels, 15×15 window) with a forward–backward check, then RANSAC + least squares
  for Position / Position & Rotation / Position, Scale & Rotation (2D Procrustes). Each frame's
  transform moves the path, written as Mask Path keyframes while the background job runs (one undo
  step per run; `jobs.cancel` stops and keeps what was tracked). Frames are tracked at ≤ 960 px
  wide. On synthetic footage with known similarity motion the path stays within 0.3 px over 20 frames.

### 5.4 Colour management

```text
frame (Y'CbCr/RGB + metadata) ─► source colour space: Interpret Footage override, else VUI/colr/MKV Colour
  ─► decode table (sRGB/BT.709, PQ, HLG scene light, camera log) ─► HLG OOTF ─► 3×3 to working gamut
  ─► BT.2390 tone map (HDR/log into an SDR sequence, Auto Tone Map Media) ─► gamut compression
  ─► effects + compositing in working linear (1.0 = reference white = SDR white = 203 cd/m²)
  ─► monitors: working → SDR BT.709 (tone map from HDR, gamut map from BT.2020)
  ─► HDR export: working → PQ/HLG BT.2020 R'G'B' → Y'CbCr (BT.2020 NCL) + VUI/colr/mdcv/clli/SEI
```

- **Model.** `SequenceSettings::color` (`ColorPipeline`: working space Rec. 709 / Rec. 2100 PQ /
  Rec. 2100 HLG, wide gamut, auto tone map) and `Interpretation::color_space` (per media item;
  `None` = from metadata). Commands: `sequence.colorSettings`, `clip.interpretFootage`,
  `color.spaces`, `media.colorInfo`.
- **Maths** in `filmcraft-color` (`transform`, `log`, `spaces`; formulas and sources in
  [crates/color/README.md](../crates/color/README.md)); the renderer side is `render::colorman`.
- **Fast path.** A Rec. 709 sequence without wide gamut and media whose metadata says Rec. 709 /
  sRGB decodes exactly as before, and its layers stay on the GPU. Log, HDR or wide-gamut media,
  and every layer of an HDR/wide-gamut sequence, are converted on the CPU (the GPU path draws them
  as pre-rendered images, like layers with effects).
- **Outputs.** `RenderOptions::working_output` returns working-space pixels (HDR exports,
  scopes); otherwise the top-level render is converted for an SDR monitor. The other colour
  effects work on display-encoded values clamped to 0..1 (they clip HDR highlights above
  reference white).
- **HDR grading (Lumetri).** `FxCtx::working` carries the sequence's working space. In a PQ or HLG
  sequence Lumetri grades the working space's own signal normalised to **HDR White** (cd/m²,
  Basic Correction; the curves, wheels, looks and HSL Secondary use the Curves section's **HDR
  Range**): `filmcraft_color::GradeSpace` (PQ: `PQ⁻¹(nits/10000) / PQ⁻¹(white/10000)`; HLG: the
  BT.2100 inverse OOTF of a display with that peak, then the OETF). So the sliders and curves span
  0 … HDR White like black … white in SDR, exposure is stops of light, and highlights above HDR
  White pass through (contrast leaves them alone; **HDR Specular** scales them). Rec. 709
  sequences grade exactly as before. HSL Secondary ▸ Refine: **Denoise** (median of the key,
  radius 1–3 px) and **Blur** (Gaussian of the key, σ ≤ 20 px), both scaled with the playback
  resolution.
- **HDR metadata.** `VideoStreamInfo::hdr` (`HdrMetadata`: ST 2086 mastering luminance, MaxCLL,
  MaxFALL) is read from `mdcv` / `clli` (MP4 / MOV) and Matroska `MasteringMetadata` / `MaxCLL` /
  `MaxFALL`. PQ media tone mapped into an SDR working space uses it as the BT.2390 source peak
  (MaxCLL, else the mastering peak; 1000 cd/m² without metadata), so a 4000-nit master keeps its
  highlight detail instead of clipping at 1000. `media.colorInfo` reports it (`hdrMetadata`,
  `toneMapPeakNits`). The field is optional and serde-defaulted (no schema change).
- **HDR export.** H.264 and ProRes exports of a PQ/HLG sequence encode BT.2020 PQ/HLG (ProRes
  10-bit; our H.264 encoder is 8-bit, so H.264 HDR is 8-bit) and signal it in the VUI / ProRes
  frame header, `colr`, and for PQ `mdcv` (BT.2020 / D65, 1000 / 0.0001 cd/m²) and `clli`
  (MaxCLL/MaxFALL 0 = unknown, they are not measured) plus the matching H.264 SEI. `sdr: true`
  exports the tone-mapped SDR picture instead; render previews always do.
- **Monitors / display colour management.** The monitors are SDR (sRGB-encoded RGBA8 textures):
  HDR sequences are shown tone mapped. macOS EDR (extended-range `CAMetalLayer` output) is not
  wired up: there is no `platform` crate yet and eframe/wgpu do not expose an EDR surface, so HDR
  values above SDR white are never sent to the display. The scopes of an HDR sequence show the
  working-space values (waveform in cd/m² on a PQ scale, BT.2020 vectorscope).
- **LUTs.** Lumetri Input LUT and Creative Look reference `lib:<id>` (the project's LUT library,
  `Project::luts`, embedded `.cube`/`.3dl` text) or `builtin:<id>` (code-generated camera
  conversions and looks). `filmcraft-gpu::GpuLut` is the WGSL tetrahedral counterpart, tested for
  parity.

### 5.4 Multi-camera and synchronisation

```text
clips ──sync (in | out | timecode[±hours] | marker | audio)──► anchors (media time ↔ common instant)
  ├─ clip.synchronize        moves selected timeline clips (link groups together) onto the reference
  ├─ clip.mergeClips         video + ≤16 audio clips → a merged-clip sequence
  └─ clip.createMulticam     cameras → a multi-camera source sequence (one video track per angle)
multi-camera clip = nested source + TrackItem::multicam {enabled, angle}
  render: only the angle's video track · audio: camera 1 | all | the angle (Switch Audio)
```

- **Model** (`project::multicam`). `Sequence::multicam` (`MulticamSource`: cameras with their video
  track, audio tracks, name, shown flag and source item; audio mode) marks a multi-camera source;
  `Sequence::merged` a merged clip. `TrackItem::multicam` (`MulticamSel`) makes a nested clip a
  multi-camera clip; any nest can be one (its video tracks are then the angles,
  `Sequence::cameras()`). Editing a multi-camera source into a sequence gives an enabled clip on the
  first angle (`Project::make_track_item`). Project schema v7.
- **Sync** (`engine::sync`). Each method reduces a clip to an *anchor* (the media time that lines up
  with the common instant). Audio uses `audio_dsp::sync::find_offset`: DC removal, windowed-sinc
  decimation to ≤ 8 kHz, GCC-PHAT-β (β = 0.75) via one packed complex FFT for the coarse lag, then
  the same at the full rate on the loudest common window (≤ 2.7 s) and parabolic interpolation.
  Recordings with different gains, microphones (filtered), 0 dB SNR noise or a strong echo are
  aligned to the sample; two 10-minute recordings take ~2.4 s (release). Clips are placed on frame
  boundaries with the sub-frame remainder taken from their source In, so video stays on frames
  while audio keeps sample accuracy.
- **Render.** `render::item_layer` renders a multi-camera clip's angle track only
  (`render_seq_tracks`); `render::audio` mixes the nested source with only the audible tracks
  (`Sequence::with_angle_audio`). Nested audio now plays at all (it was skipped when the nest had
  no media source) and is limited to the clip's range. `render::multicam::render_grid_page` renders
  one page of the shown angles at the cell scale in parallel (rayon) and tiles them: the
  Multi-Camera view is one frame job (`frames::Target::MulticamGrid(source, layout, page)`),
  prefetched while playing like the program. Only the page's angles are decoded.
- **Multi-Camera view settings** (`EditorState::multicam_view`, Program ▸ wrench menu): grid
  layout (automatic = smallest square up to 4×4, or fixed 2×2 / 3×3 / 4×4) with pages when the
  angles don't fit (`render::multicam::page_layout`; `multicam.gridLayout`, `multicam.page`,
  page arrows; keys 1–9 pick cameras on the shown page, `multicam.cut {camera}` stays absolute);
  Multi-Camera Selection Top Down (stacked multi-camera clips: topmost instead of lowest);
  Show Multi-Camera Preview Monitor (off: the grid fills the monitor); Auto-Adjust Multi-Camera
  Playback Quality (`grid_cell_scale`: ½ / ¼ of the cell scale while playing); Transmit
  Multi-Camera View (stored; there is no transmit device yet). `multicam.grid` returns the page.
  Edit Cameras… shows a thumbnail per angle (`frames::Target::MulticamAngle`).
- **Editing** (`edit::multicam`, `engine::multicam`). `multicam.switchAngle` (click an angle,
  Ctrl/⌘-click for video only), `multicam.selectCamera1…9` (keys 1–9) and `cutToCamera1…9`
  (Ctrl+1–9), Enable/Flatten, Edit Cameras, Audio Follows Video (`EditorState`). Live switching:
  playback in the Multi-Camera view runs `multicam.recordStart`; each key/click is a
  `multicam.cut` that is applied at once (the program shows it) by re-applying the whole pass to
  the project from before the pass (`edit_merged`), so a pass is one undo step; Stop runs
  `multicam.recordStop`, which ends the last angle at the stop point. Through edits inside the
  recorded range are healed, so pressing the angle already showing adds no edit. Flatten replaces
  a clip by the clip(s) its angle shows (outer effects carried over; linked pairs stay linked).

## 6. Export jobs (`filmcraft-export`)

```text
file.exportMedia {path, preset?, settings?, format?, range?, …}     export.quick {preset?, path?}
  → engine::export_tools: preset ⊕ settings JSON ⊕ flat overrides → ExportSettings; range resolved
  → a Job {id, label, progress, result} on a background thread (export.queue.* runs them in order)
  → export: per output frame render (output rate) → fit into the output frame → overlays → limiter
            → encode in order → mux; audio mixed at the output rate, folded to mono/stereo,
            loudness-normalized (measuring pass + gain + true-peak limiter) per batch
  → jobs.list shows progress; jobs.cancel sets the shared cancel flag
```

| Format | Encoder | Container |
|---|---|---|
| `h264` | `filmcraft-h264enc` (profile, level, CBR / VBR 1-pass / VBR 2-pass, keyframe distance) + `filmcraft-aac` | MP4 or QuickTime (`isobmff`, the Multiplexer setting) |
| `prores` | `filmcraft-prores` (Proxy / LT / 422 / HQ) | MOV |
| `dnxhr` | `filmcraft-dnx` (LB / SQ / HQ / HQX) | MOV (`AVdh`) |
| `mjpeg` | built in | MOV |
| `aac` | `filmcraft-aac` (AAC-LC, the audio settings' bitrate) | audio only: MPEG-4 audio `.m4a` (`isobmff`, priming in an edit list), or raw ADTS when the path ends in `.aac` |
| `mxf-op1a` | DNxHR (default), ProRes or H.264 (Annex B, long GOP), `mxfVideoCodec`; PCM 16/24-bit | MXF OP1a (`filmcraft-mxf` writer: frame-wrapped, index with temporal offsets, start timecode) |
| `mxf-opatom` | as `mxf-op1a` | Avid-style MXF OP-Atom: clip-wrapped picture at `path`, one mono PCM file per channel (`<stem>_A1.mxf` …, `Report::extra_files`) |
| `png`, `tiff`, `bmp`, `jpg` | `image` | numbered stills `<name>000.<ext>`, `<name>001.<ext>` … |
| `tga`, `dpx`, `exr` | built in (`raster.rs`: uncompressed Targa 24/32-bit; DPX 10-bit RGB, SMPTE ST 268; OpenEXR half-float linear RGB(A), uncompressed scan lines) | numbered stills, as above |
| `gif`, `wav`, `aiff` | built in / `image` | GIF / RIFF WAVE (`WAVE_FORMAT_EXTENSIBLE` for 5.1) / AIFF (16- or 24-bit PCM) |

- **Reproducible.** The same project and settings give the same file on every machine. Frames render
  in batches of one per core, but the muxer gets them in fixed groups of 16 output frames
  (`INTERLEAVE` in `crates/export/src/job.rs`), each followed by its audio, and H.264 uses one slice
  per four macroblock rows instead of one per core. MXF files are the exception: their UMIDs and
  modification date come from the clock, as SMPTE ST 330 / ST 377 expect.
- **Settings.** `ExportSettings` (serde, camelCase, every field optional) holds Video (frame size or
  Match Source, frame rate, Scale to Fit / Fill / Stretch, pixel aspect, field order — progressive
  only, profile / level, bitrate encoding, target / maximum / adaptive bitrate, keyframe distance,
  maximum render quality), Audio (codec, sample rate, channels, AAC bitrate, PCM sample size),
  Multiplexer (audio channels: mono, stereo or 5.1, §5.1.1), Captions (burn-in or SRT / WebVTT sidecar), Effects (image / name / timecode
  overlays, video limiter, loudness normalization), Metadata (`udta` `©nam`, `©ART`, `©cpy`, `©des`,
  `©cmt`). `settings.summary()` / `estimate_bytes()` feed Export mode's Summary.
- **Presets.** `export::presets::builtin_presets()` are our own definitions (Match Source adaptive
  H.264 at 0.2 / 0.1 / 0.05 bits per pixel, 1080p / 2160p delivery, vertical 1080×1920, ProRes,
  DNxHR, MXF OP1a (DNxHR HQ / ProRes 422 HQ / H.264) and OP-Atom (DNxHR, Avid), image sequences, GIF, WAV / AIFF). User presets and favourites persist in
  `<data dir>/export-presets.json` (`export.presets.*`).
- **Queue.** `export.queue.add` snapshots the project and the resolved settings (several sequences
  or ranges add several items); `export.queue.start` encodes ready items one after another as
  ordinary jobs, advanced by `Session::poll_persistence` / `pump_jobs` (or synchronously with
  `wait`). Cancel, retry, reorder, remove and clear work per item. The queue is session state, not
  part of the project file.

Video encoders implement `export::VideoEncoder`. Codec crates plug in with `register_encoder` and
`register_audio_encoder`.

Timelines can be exchanged as EDL, FCP7 XML, FCPXML, OTIO, AAF or OMF. `file.import` detects these
formats and merges the result into the project as one undoable step (AAF / OMF embedded audio is
written next to the document first). `file.import {replace: true}` starts from an empty project
with the same name (`file.newProject`) so a headless import→export round-trip contains only the
imported timeline. `file.exportInterchange`, `file.exportEdl`,
`file.exportFcpxml`, `file.exportOtio`, `file.exportAaf` and `file.exportOmf` write them.
`file.exportInterchange {format}` takes `edl`, `xml` (the default), `fcpxml`, `otio`, `aaf` or `omf`;
any other value is a parameter error and nothing is written. For AAF and
OMF the engine (`engine::aaf_omf`) first prepares the audio the document references: it lists the
used ranges (`interchange::essence::audio_needs`), decodes or renders them (clip effects through the
export audio pipeline), embeds them or writes WAV / AIFF files, and optionally renders a video
mixdown. A nested sequence is exported the way Premiere Pro does it: a composition of its own in
AAF (the media of the clips inside it is prepared with the rest), its sound mixed into the document
in OMF, a nested sequence in FCP7 XML, FCPXML and OTIO, and one `AX` event in an EDL (table in the
[interchange README](../crates/interchange/README.md)).

## 7. Automation surfaces

All of these dispatch the same command ids.

| Surface | Where | Scope |
|---|---|---|
| UI | `ui-egui` menus, shortcuts, panels | `menus::invoke` → engine or UI command |
| CLI | `filmcraft-cli` | `exec <id> key=value…`, `run script.jsonl` (one `{"id","params"}` per line), `inspect`, `import`, `export`, `render`, `probe`; `--save`, `--bridge` |
| Control channel | `filmcraft --control <port>` | JSON lines on loopback TCP: engine commands plus synthetic input, inspection and screenshots of the live UI |
| MCP | `filmcraft-cli mcp` | stdio MCP server: headless in-process session, or `--bridge` to the control channel |

- **Automation ids.** Every interactive widget calls `app.auto.add(id, rect, label)` each frame
  (`crates/ui-egui/src/automation.rs`). Agents click by id, e.g. `tools.Razor`,
  `timeline.clip.<id>`, `effects.item.gaussian_blur`, `panel.Timeline`.
- **UI state** that is not project data (tool, workspace, dock layout, zoom, scroll, monitor
  settings) is in `crates/ui-egui/src/state.rs` as serde structs, so `ui.inspect` and `ui.set` can
  read and write it.

- **Panels with engine data** (M8.9 / M12.6): Lumetri Scopes (`scopes.read` returns the same
  scopes as numbers), Metadata (`metadata.get` / `metadata.set`, one undo step per edit, stored in
  `ProjectItem::metadata`), Events (`events.list` / `events.clear`), Progress (`jobs.list` /
  `jobs.cancel`), Project Notes (`project.notes` / `project.setNotes`, plain text in
  `Project::notes`, schema v15; one typing session is one undo step). Timecode and Reference Monitor are frontend-only views. Their settings are
  `UiState::panels` (`ui.set {"panels": {...}}`).
- **Project panel and Media Browser** (M12.7). View settings (List / Icon / Freeform, thumbnail
  and font size, Preview Area, Hover Scrub), the List view's columns, widths and sort, and ten view
  presets are preferences (`Preferences::project_panel`; `project.view.*`, `project.columns.*`,
  `project.sort`, `project.viewPreset.*`). Freeform positions, Clip Size, stacks and saved
  arrangements are item metadata (`Freeform Position`, `Freeform Stack`,
  `Freeform Arrangement: <name>`; `project.freeform.*`), so they are undoable and saved with the
  project without a schema change. The Media Browser lists directories through
  `Services::list_entries` / `volumes` / `home_dir` (native: `std::fs`; web: the virtual file
  table; tests: a fake filesystem): `engine::media_browser` keeps navigation history and the
  selection in `Session::browser`, Favorites / recent directories / file types / columns in
  `Preferences::media_browser` (`mediaBrowser.*`). Bins opened in a tab or a window, the inline
  rename, dialogs and the hover-scrubbed card are `UiState::project_panel`; the browser's tree
  state is `UiState::media_browser` (`ui.set {"projectPanel": …, "mediaBrowser": …}`).

Protocol reference: [control-protocol.md](control-protocol.md). Agent guide: [agents.md](agents.md).

## 8. Not built yet

The layer table reserves names for crates that don't exist yet: `riff`, `mjpeg`,
`keyframe`, `effects`, `audio` and `playback`. (`platform` exists but holds only OS media FFI;
other OS integration stays in `apps/filmcraft`.)
Until they exist, that work lives elsewhere: keyframes and effect definitions in `project`, effects
and the audio mix in `render`, playback in `ui-egui`, and OS integration (cpal, rfd,
native menus) in `apps/filmcraft`. [roadmap.md](roadmap.md) has the milestone status and [gaps.md](gaps.md) what is missing.

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | trivial | Status line and revision history added (craftrules progress-docs standard); milestone link points at docs/roadmap.md |


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://cbmh.wtpuscm.cn/pingce/feedback-392.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://kjxf.tcti.cn/wangluo/analysis-38342468.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://dcwb.tcti.cn/pingtai/logo-94628488.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://kpdu.wtpuscm.cn/wenzhang/tool-361996.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://qosu.tcti.cn/kaifa/customer-04537940.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://www.ai-hao123.com/gongsi/security-96352150.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://qlxr.wtpuscm.cn/zhinan/audience-663145.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://ankmz.cn/anquan/tag-68558329.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://qbxm.wtpuscm.cn/baogao/creative-666814.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://www.yx-sf.com/wiki/54236.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://www.yx-sf.com/wiki/50932.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://mcca.wtpuscm.cn/yingyong/accessibility-524.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://aqwk.tcti.cn/wangluo/server-55719004.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://rhjb.wtpuscm.cn/gongsi/lead-669.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://yphk.tcti.cn/chanpin/system-71763527.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://www.ai-hao123.com/yingyong/policy-84365701.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://www.yx-sf.com/wiki/28157.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://rfoe.tcti.cn/anli/community-07462915.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://www.ai-hao123.com/zhineng/global-67421128.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://www.ai-hao123.com/zhineng/feedback-35196122.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://www.yx-sf.com/tech/85538.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://mjii.wtpuscm.cn/qiye/whitepaper-901.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://wwls.wtpuscm.cn/keji/prospect-755868.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://xtzp.wtpuscm.cn/wenzhang/domain-558290.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)

</details>



---

<details open>
<summary><b>📚 Architecture 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Architecture」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Architecture 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Architecture 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-25512.html)
* [Architecture 内部组件解耦与事件状态机规范 (Node-96)](https://ankmz.cn/anquan/tag-68558329-2-7843.html)
* [【官方规范】Architecture 8. Not built yet 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-60889.html)
* [Architecture 内部组件解耦与事件状态机规范 (v2.0-GA)](https://iucy.wtpuscm.cn/gongsi/platform-237776.html)
* [Architecture 内部组件解耦与事件状态机规范 (Verified)](https://eapy.wtpuscm.cn/ziyuan/fashion-079.html)
* [【官方规范】Architecture Last reviewed: 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358-6-43932.html)
* [基于 Architecture 的高吞吐 8. Not built yet 设计白皮书](https://tjeu.tcti.cn/yingxiao/milestone-04113521.html)
* [【官方规范】Architecture 1. Layers 核心运行拓扑标准](https://www.mw-wm.com/huodong/presentation-45443121.html)
* [现代 Architecture 架构演进之路 —— Architecture 深度实践](https://www.yx-sf.com/wiki/81687.html)
* [【官方规范】Architecture 高韧性系统架构设计 核心运行拓扑标准](https://www.ai-hao123.com/fuwu/study-81613627.html)
* [【官方规范】Architecture 5.4 Colour management 核心运行拓扑标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-44165.html)
* [【Architecture】f64 核心系统架构与设计规约 (Core/f64)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-2253.html)
* [Architecture 核心系统架构与设计规约 (Draft-04)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-57459.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Architecture 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-40230.html)
* [【官方规范】Architecture 5.1 Audio mixer 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-19612.html)
* [【官方规范】Architecture 可信存活健康度量 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-45099.html)
* [Architecture 内部组件解耦与事件状态机规范 (RFC-888)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-55928.html)
* [面向大规模网络的 Architecture 工业级架构基准](https://gynw.tcti.cn/fenxi/recipe-50406879-18-62599.html)
* [2. Time base 核心系统架构与设计规约 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-41221.html)
* [Architecture 分布式数据通道与 5.4 Colour management 技术规范 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-37191.html)
* [【官方规范】Architecture Last updated: 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-2940.html)
* [7. Automation surfaces 核心系统架构与设计规约 (Core/7. Aut)](https://lase.tcti.cn/fenxi/lesson-67970267-22-31749.html)
* [分布式状态机一致性 核心系统架构与设计规约 (Draft-01)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-34934.html)
* [现代 Change: 架构演进之路 —— Architecture 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-18465.html)
* [5.3 Masks 核心系统架构与设计规约 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-10714.html)
* [现代 5.2 Essential Sound 架构演进之路 —— Architecture 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-60181.html)
* [Architecture 分布式数据通道与 5.1 Audio mixer 技术规范 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-8752.html)
* [基于 Architecture 的高吞吐 Last reviewed: 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-17946.html)
* [【Architecture】run() 核心系统架构与设计规约 (Node-58)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-25593.html)
* [Architecture 内部组件解耦与事件状态机规范 (Core/Change)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-2630.html)
* [【官方规范】Architecture 分布式状态机一致性 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-59938.html)
* [现代 Last reviewed: 架构演进之路 —— Architecture 深度实践](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-45129.html)
* [Architecture 分布式数据通道与 Architecture 技术规范 (Node-56)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-15094.html)
* [现代 5.1.3 Remix 架构演进之路 —— Architecture 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016-34-4230.html)
* [【官方规范】Architecture 5.1.3 Remix 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-63822.html)
* [7. Automation surfaces 核心系统架构与设计规约 (Draft-03)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-201.html)
* [现代 Arc<Project> 架构演进之路 —— Architecture 深度实践](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-50088.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Architecture 核心 API 接口契约与客户端调用指南](https://szjp.tcti.cn/shichang/retention-16647312-38-51273.html)
* [【集成指南】2. Time base 服务端接入准则与 Architecture 实战](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-1779.html)
* [【生产手册】Architecture 模块通信与请求穿透标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-42034.html)
* [【集成指南】8. Not built yet 服务端接入准则与 Architecture 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-10381.html)
* [【集成指南】5.1 Audio mixer 服务端接入准则与 Architecture 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-43650.html)
* [Architecture vs 业界主流方案：Architecture 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-28960.html)
* [Architecture 插件生态规范与 Target: 扩展手册 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-50325.html)
* [基于 Architecture 的自动化部署与生产环境配置实践](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-37370.html)
* [Architecture vs 业界主流方案：f64 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-62783.html)
* [Architecture vs 业界主流方案：2. Time base 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-40431.html)
* [Architecture 插件生态规范与 Change: 扩展手册 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-39137.html)
* [Architecture 异步中间件流水线与 高韧性系统架构设计 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-17891.html)
* [Architecture 插件生态规范与 Arc<Project> 扩展手册 (Node-10)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-40133.html)
* [【集成指南】f64 服务端接入准则与 Architecture 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-15433.html)
* [Architecture 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-52-28986.html)
* [Architecture 插件生态规范与 5.1 Audio mixer 扩展手册 (Node-19)](https://www.poykq.cn/redian/tutorial-673489-53-28066.html)
* [【集成指南】enabled() 服务端接入准则与 Architecture 实战](https://whkw.wtpuscm.cn/wangluo/expense-432933.html)
* [Architecture 插件生态规范与 Last updated: 扩展手册 (Core/Last u)](https://nwew.wtpuscm.cn/zhinan/management-610.html)
* [Architecture vs 业界主流方案：Target: 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-28952.html)
* [Architecture vs 业界主流方案：Engine-first. 深度技术选型对比](https://ythn.tcti.cn/huodong/about-80681637.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Architecture 实战](https://www.mw-wm.com/kuangjia/seminar-45535209.html)
* [Architecture 异步中间件流水线与 f64 接入规范](https://www.yx-sf.com/tech/52870.html)
* [Architecture 异步中间件流水线与 5.3 Masks 接入规范](https://www.ai-hao123.com/pingce/api-15096147.html)
* [Architecture 异步中间件流水线与 Target: 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-46244.html)
* [Architecture 异步中间件流水线与 5.1 Audio mixer 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-57203.html)
* [Architecture 插件生态规范与 enabled() 扩展手册 (Core/enable)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-25253.html)
* [Architecture 插件生态规范与 f64 扩展手册 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-18044.html)
* [【集成指南】Target: 服务端接入准则与 Architecture 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-4249.html)
* [Architecture 异步中间件流水线与 Engine-first. 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-11456.html)
* [Architecture 异步中间件流水线与 2. Time base 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-19146.html)
* [Architecture 异步中间件流水线与 Last updated: 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-41585.html)
* [Architecture 异步中间件流水线与 生产环境运维调优手册 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-21136.html)
* [Architecture 插件生态规范与 5.1 Audio mixer 扩展手册 (Draft-08)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-19820.html)
* [Architecture 插件生态规范与 enabled() 扩展手册 (Draft-05)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-1297.html)
* [Architecture 插件生态规范与 8. Not built yet 扩展手册 (Draft-06)](https://lase.tcti.cn/fenxi/lesson-67970267-72-26752.html)
* [Architecture vs 业界主流方案：5.1 Audio mixer 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-36802.html)
* [【集成指南】Engine-first. 服务端接入准则与 Architecture 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-58428.html)

#### 3. ⚡ Architecture 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Architecture 官方毫秒级实时数据广播节点](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-44549.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Draft-08)](https://owfm.tcti.cn/shangye/education-63674207-76-62812.html)
* [Architecture 官方高可用镜像注册节点 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-46022.html)
* [Architecture 亚太与欧美多活集群数据同步中枢](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-6676.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (RFC-321)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-33387.html)
* [全球权威拓扑节点：Architecture 实时镜像与索引入口](https://aagj.tcti.cn/jianzhan/audience-84449488-80-63009.html)
* [Architecture 官方高可用镜像注册节点 (Spec-v1.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-34137.html)
* [Architecture 官方高可用镜像注册节点 (Node-79)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-33399.html)
* [Architecture 官方高可用镜像注册节点 (Draft-04)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-57531.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (RFC-607)](https://iuuv.tcti.cn/hezuo/app-80239016-84-38240.html)
* [冷热数据分层镜像：Architecture Exact time. 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-60271.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Node-73)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-63811.html)
* [冷热数据分层镜像：Architecture 8. Not built yet 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-37972.html)
* [Architecture 去中心化数据同步源与拓扑寻址规约](https://szjp.tcti.cn/shichang/retention-16647312-88-41849.html)
* [Architecture 官方高可用镜像注册节点 (Core/Archit)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-53318.html)
* [冷热数据分层镜像：Architecture 高韧性系统架构设计 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-65296.html)
* [Architecture 官方高可用镜像注册节点 (Core/Exact )](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-3038.html)
* [Architecture: 5.1.3 Remix 5.1.3 Remix (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-39438.html)
* [Architecture: Exact time. Exact time. (Node-13)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-29060.html)
* [Architecture 官方高可用镜像注册节点 (Node-23)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-55094.html)
* [冷热数据分层镜像：Architecture Target: 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-39962.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-35248.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-15144.html)
* [冷热数据分层镜像：Architecture 2. Time base 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-48646.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-23007.html)
* [Architecture 官方高可用镜像注册节点 (RFC-997)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-49792.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Node-51)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-49527.html)
* [Architecture 官方高可用镜像注册节点 (Node-66)](https://ankmz.cn/anquan/tag-68558329-102-663.html)
* [Architecture: 2. Time base 2. Time base (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-103-41368.html)
* [Architecture 官方高可用镜像注册节点 (Draft-07)](https://pmmd.wtpuscm.cn/yunsuan/products-046976.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (RFC-429)](https://esby.wtpuscm.cn/fenxi/search-045.html)
* [Architecture 官方高可用镜像注册节点 (Core/5.2 Es)](https://www.tcti.cn/hulianwang/feedback-96499358-106-55811.html)
* [Architecture 官方高可用镜像注册节点 (Verified)](https://zszy.tcti.cn/xinwen/landing-28021421.html)
* [Architecture 官方高可用镜像注册节点 (RFC-533)](https://www.mw-wm.com/youhua/careers-39885699.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Core/5.2 Es)](https://www.yx-sf.com/tech/80736.html)
* [冷热数据分层镜像：Architecture 模块化解耦与协议标准 权威归档源](https://www.ai-hao123.com/wangluo/blog-37870245.html)
* [冷热数据分层镜像：Architecture 5.4 Colour management 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-27947.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Architecture 权威网络权重传递与收录基准规范](https://keil.spidernode-mesh.net/yunying/server-16840384-112-16887.html)
* [Architecture 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-44769.html)
* [Architecture 节点连通性、存活性探测与防作弊指标](https://fjoc.tcti.cn/yingyong/contact-00610313-114-51751.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-33905.html)
* [【评测基准】Architecture 吞吐抖动度量与健康检查协议](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-12877.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Core/5.1.3 )](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-47062.html)
* [Architecture 高负载场景下 Architecture 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-18678.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-929)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-55308.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Draft-06)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-45355.html)
* [Architecture 高负载场景下 生产环境运维调优手册 基准评测报告](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-25522.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (RFC-920)](https://lase.tcti.cn/fenxi/lesson-67970267-122-28516.html)
* [Architecture 高负载场景下 f64 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-3134.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Draft-06)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-27433.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v2.5)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-61402.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Spec-v1.1)](https://owfm.tcti.cn/shangye/education-63674207-126-63782.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-24284.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Node-29)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-28597.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Draft-05)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-63301.html)
* [Architecture 高负载场景下 run() 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-27621.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (RFC-196)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-48262.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v1.5)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-63236.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Node-44)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-3287.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-830)](https://iuuv.tcti.cn/hezuo/app-80239016-134-57127.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-33520.html)
* [Architecture 高负载场景下 enabled() 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-59636.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-17087.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312-138-13560.html)
* [Architecture: run() run() (Draft-03)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-30029.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Node-58)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-22229.html)
* [Architecture 高负载场景下 Exact time. 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-29798.html)
* [Architecture 高负载场景下 可信存活健康度量 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-32413.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-721)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-3186.html)
* [Architecture 高负载场景下 8. Not built yet 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-23067.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-898)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-55718.html)
* [Architecture 高负载场景下 Revision history 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-61874.html)
* [Architecture 高负载场景下 2. Time base 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-13020.html)
* [Architecture 高负载场景下 5.2 Essential Sound 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-1151.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Core/1. Lay)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-2451.html)
* [Architecture 高负载场景下 5.4 Colour management 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-115.html)

</details>

