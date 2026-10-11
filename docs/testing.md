# Testing

```sh
cargo test --workspace                          # everything; oracle tests skip without ffmpeg
cargo test -p filmcraft-h264                    # one crate
cargo test --release -p filmcraft-hevc          # codec tests are much faster in release
cargo xtask ci                                  # all gates (fmt, clippy, tests, layers, assets, wasm)
cargo xtask fixtures                            # pre-generate the ffmpeg fixture matrix
FILMCRAFT_REQUIRE_ORACLES=1 cargo test --workspace   # CI: missing ffmpeg fails instead of skipping
```

## 1. Kinds of tests

| Kind | Where | Examples |
|---|---|---|
| Unit | `src/` `#[cfg(test)]` modules, `src/tests.rs` | CABAC engines, VLC tables (prefix-freeness, Kraft sums), colour matrices, frame cache |
| Property (`proptest`) | `time`, `bitstream`, `edit`, `isobmff/tests/roundtrip.rs`, `interchange/tests/*` | tick↔frame round trips for every rate, DF timecode, edit invariants (no overlaps, durations conserved), mux→demux round trips, interchange export→import |
| Synthetic streams | `h264/src/synth_tests.rs`, `hevc/src/synth_tests.rs` | hand-built bitstreams for features the reference encoders don't emit; expected output known exactly |
| Scripted UI | `crates/ui-egui/tests/scripted.rs` | headless app (egui_kittest) driven over the control channel: razor/undo, insert, apply effect, playback, click by automation id (§4) |
| Monitor view / Graphics menu | `crates/ui-egui/tests/view_ui.rs`, `crates/engine/src/graphics_tests.rs` | View menu items and checkmarks, playback/paused resolution, channel / comparison / multi-camera / waveform display modes, magnification and Hand-tool panning, guides dragged from the rulers (move, lock, remove, Add Guide…, templates in the preferences), snapping a graphic to the frame centre and a guide; align to frame / as group / to selection, distribute (centres and gaps), arrange, select next/previous graphic and layer, reset parameters / duration, vertical text, New Layer from file (`FILMCRAFT_UI_SHOTS=<dir>` writes `view-*.png`) |
| Settings | `crates/engine/src/settings_tests.rs`, `crates/ui-egui/tests/settings_ui.rs` | schema keys and defaults, persistence in the data directory, v1 → v2 migration and repair of bad values, validation, per-category reset; each wired setting (still / transition durations, step many, label defaults and colours, media scaling, timebase, recent projects, media cache policy, output mapping, smart quotes, auto-transcribe); the dialog: every category from its command and Cmd+,, edits by automation id, OK / Cancel / Escape / Reset…, theme, label names in Edit ▸ Label, tooltips, frame cache (`FILMCRAFT_UI_SHOTS=<dir>` writes `settings-*.png`) |
| Scopes and panels (M8.9 / M12.6) | `crates/scopes/src/tests.rs`, `crates/engine/src/{scopes,panels}_tests.rs`, `crates/ui-egui/tests/panels_ui.rs` | scope maths on generated frames (flat colours → exact histogram bins, ramps → waveform row = code value, 75 % bars → parade levels and vectorscope target cells for BT.601/709/2020, HLS hue angles, YC chroma, Clamp Signal, decimation, NaN); `scopes.read` on a colour matte (exact bins, 78.43 %) and on Bars and Tone (the six targets); metadata edits (one undo step, read-only fields refused, saved in the project), the event log (failed / disabled commands, repeats, jobs started / finished / failed / cancelled); every panel from Window ▸, the scopes' wrench menu, presets, five-scope grid, typing into a Metadata field, Timecode rows and modes, Events filter / Clear All, Progress cancel, Reference Monitor park / gang / scopes (`FILMCRAFT_UI_SNAPSHOT_DIR=<dir>` writes `panels-*.png`); `perf_scopes_at_1080p` (ignored) times each scope |
| Keyframe navigator and Effect Controls ruler | `crates/engine/src/tests.rs` (`add_keyframe_toggles_the_keyframe_at_the_playhead`), `crates/ui-egui/tests/keyframes_ui.rs`, `lane_tests` in `crates/ui-egui/src/panels/effect_controls.rs` | `effects.addKeyframe` adds a keyframe at the playhead or removes the one there (the value falls back to the remaining keyframes; undo, redo, parameters that name nothing); headless: the Properties diamond turns animation on, adds and removes the keyframe at the playhead (Scale 50 % → 100 % again), its ◀ ▶ arrows and Effect Controls' step between keyframes, Crop is applied before its first keyframe; a click on the Effect Controls time ruler and a drag of the playhead's handle move the playhead, and no playhead is drawn off the clip; ruler labels are sequence timecode, and clip times, frame rates and panel widths at their limits don't panic (`FILMCRAFT_UI_SHOTS=<dir>` writes `keyframes-*.png`) |
| Golden images | `crates/golden/tests/golden.rs` | CPU renders vs committed PNGs; GPU vs CPU on the same scenes (§3) |
| Engine / command | `crates/engine/src/tests.rs` | run commands on the demo project, assert the sequence, undo/redo, disabled cases |
| Render | `crates/render/src/tests.rs` | compositing, opacity, Motion, cross dissolve midpoint, ½-res vs full, GPU plan vs reference, audio mix, audio-effect continuity |
| Multi-camera / sync | `crates/audio-dsp/src/sync.rs`, `crates/engine/src/multicam_tests.rs`, `crates/ui-egui/tests/multicam_ui.rs` | audio offsets within one sample: Rust-generated multi-mic speech (gains −20…+10 dB, SNR 40/25/10/3 dB, zero-phase colouring, fractional delays, a 0 dB-SNR room with a −4 dB echo) and ffmpeg-made camera/recorder files (pink-noise bursts, independent noise; exact lags recovered); timecode (± hours) and marker sync within a frame; Merge Clips; timeline Synchronize keeps frame-aligned starts; multicam structure, angle render (PSNR vs the camera), Switch Audio mix = the angle's track, live switching from a simulated playback clock (cuts, healing, one undo step), Ctrl+N cuts, flatten (frames identical), save/load; 20 angles: paging (auto / 2×2 / 3×3), page-relative keys, absolute cameras beyond 16, a page decoding only its angles at the cell scale (recording source), Auto-Adjust scale, Selection Top Down; headless: page arrows, angle 17–20 cells, grid-only view, Edit Cameras thumbnails; `perf_grid_of_four_1080p_angles` (ignored) prints the grid cost |
| Project panel and Media Browser (M12.7) | `crates/engine/src/{project_panel,media_browser}_tests.rs`, `crates/ui-egui/tests/project_panel_ui.rs` | view settings and their preference round trip; Metadata Display columns (custom fields, Freeform keys hidden), numeric vs text sorting, view presets (save / save as / restore / rename / delete, ten slots, persisted in `preferences.json`); Freeform moves keeping offsets, snapping, Align to Grid, Clip Size, stacks, arrangements, Reset (each one undo step, saved in the project); bin rename / nesting; the Media Browser against a fake filesystem (folders first, hidden and unsupported files left out, file-type filter, back / forward / up, recent directories, Favorites, import of a selection or a folder, Open in Source Monitor, ingest, probe columns). Headless: List view (sort by header, drag a column wider, inline rename), Icon view (hover scrub + I / O set the clip's In / Out, Shift+] / [), Freeform (drag a card, stack from the card menu, options dialog), panel menu, Metadata Display dialog, view presets, bins in place / tab / window, footer buttons, Media Browser navigation, Favorites, import, Shift+O, Edit Columns…, thumbnails, drag to the Timeline (`FILMCRAFT_UI_SNAPSHOT_DIR=<dir>` writes `project-*.png`, `media-browser-*.png`) |
| Menu long tail (M3.11) | `crates/render/src/scene.rs`, `crates/engine/src/{scene_detect,sequence_extras,project_tools}_tests.rs`, `crates/interchange/src/ale.rs`, `crates/project/src/find.rs`, `crates/ui-egui/tests/menus_ui.rs` | Scene Edit Detection on synthetic frames (hard cuts vs pans, flashes, sensitivity) and on generated movies with known cut points (our own ProRes render of demo scenes; an ffmpeg-made H.264 concat of testsrc2 / SMPTE bars / Mandelbrot): cuts with linked audio, clip markers, subclips, one undo step, background job and cancel; Normalize Mix Track to a target peak; Simplify Sequence; Find / Find Next and search bins; Automate to Sequence (overlap, transitions, insert, unnumbered markers); ALE round trip; Project Settings and scratch disks; menu order and shortcuts against Premiere's; the dialogs by automation id |
| Mixer | `crates/render/src/mixer_tests.rs`, `crates/engine/src/mixer_tests.rs` | sample-exact fader gain and pan laws, automation at block boundaries (bit-identical however requests are cut), solo/mute/solo-safe, sends and submix routing, latency-compensation alignment, render-vs-playback identity, Touch ramp-back, recorder modes (Latch/Touch/Write), thinning, Audio Gain, transition curves; `perf_24_tracks_3_effects_realtime_factor` prints the realtime factor |
| 5.1 / multichannel | `crates/audio-dsp/src/channels.rs`, `crates/render/src/mixer_tests.rs`, `crates/export/src/surround_tests.rs`, `crates/engine/src/mixer_tests.rs`, `crates/ui-egui/tests/audio51_ui.rs` | BS.775 downmix / upmix coefficients exact, mixdown types; 5.1 panner point gains equal-power (Σg² = 1 over a 21×21 grid × 3 centre settings), speaker positions exact, 5.1 through the default puck bit-exact identity; graph: stereo/mono tracks into a 5.1 Mix (L/R, C, rear right, room centre −6 dB each, LFE = mean), 5.1 track through a 5.1 Mix and folded into stereo, a 5.1 submix, `pan51` automation block-invariant, six-channel meters; export: 6-ch WAV (`WAVE_FORMAT_EXTENSIBLE`, mask 0x3F) and ProRes MOV (6 × 24-bit PCM) decoded by ffmpeg **sample-exact**, ffprobe `channels` 6 / `channel_layout` 5.1; AAC 5.1 in MP4: every decoded channel's own tone ≥ 30 dB above the others' (no swapped channels), stereo export of a 5.1 Mix = BS.775 within 2·10⁻⁶; `sequence.settings {mix}`, `file.newSequence {mix, trackType}`, panner lanes, Write pass records the puck; UI: puck drag (one undo step), Center / LFE, six meter bars |
| Audio Clip Mixer automation, Track Mixer view | `crates/engine/src/mixer_tests.rs`, `crates/ui-egui/tests/audio51_ui.rs` | Touch: a 201-point fader ramp thins to ≤ 6 clip keyframes, values within 0.1 dB, live override while held, values outside the gesture unchanged, one undo step; Latch holds until stop; Write records volume and pan from the start; a gesture across two clips writes both; Read commits once without a pass; UI: Clip Mixer mode dropdown, a recorded fader drag writes clip keyframes; Show/Hide Tracks, Meter Input(s) Only, transport record button |
| Voice-over | `crates/engine/src/voiceover_tests.rs`, `crates/ui-egui/tests/voiceover_ui.rs` | synthetic click train lands in `mix_sequence` exactly at R + k·period (pre-roll dropped, exact sample count), punch-in between In/Out (ramp source, every sample checked), one undo step + undo/redo, take numbering, record-arm/targeting track choice, sync/discard/zero-length/locked cases, cue times; UI: right-click Mic → Voice-Over Record Settings dialog (edits persist on OK, Cancel discards), the Mic button starts and stops a take on its track with pre-roll and overlay |
| Text to Speech | `crates/tts/src/**/*.rs` (Kokoro tests need `FILMCRAFT_KOKORO_DIR`), `crates/tts-text`, `crates/engine/src/narration_tests.rs`, `crates/ui-egui/tests/tts_ui.rs` | built-in voices: determinism, finite samples, exact-silence pause markers, pace scales length, measured pitch per voice and per semitone, hostile scripts; engine: narration placed at the playhead with clip length = speech (sample-exact), never overwrites (busy tracks → next free track → new track), occupied/locked/unknown track refused with no change, edit keeps start/length/speed (longer speech: whole speech in the file; shorter: file padded with exact zeros), edits write beside the previous file, undo/redo restore the project exactly, enabled only with a narration clip (or `clip`), 16 hostile parameter sets refused without change, preview changes nothing, save/reload keeps narrations, the narration is audible in `mix_sequence`; UI: type + Add pause + Add to timeline, the new clip is selected and loaded, clicking the clip on the timeline loads its script, Save edits it in place as one undo step, Advanced twirl, panel opens beside Properties; natural voices: 50-sentence Whisper-base WER 2.1–3.6 % for all 9 voices (crate README), loaders fuzzed with hostile files, `tts.render` cache, download refused without the feature |
| Remix | `crates/audio-dsp/src/remix.rs`, `crates/render/src/remix.rs`, `crates/engine/src/remix_tests.rs`, `crates/ui-egui/tests/remix_ui.rs` | generated rhythmic music (Rust-synthesised drums, bass, chord sections at 90–128 BPM, 22.05/44.1/48 kHz): tempo within 0.3 BPM, every beat within 10 ms (measured ≤ 8.9 ms); remix targets from 0.4× to 2.2× the source within one beat (measured ≤ 12 ms at default sliders), every cut on a detected beat, intro and outro kept, pieces ≥ 4 beats, deterministic; silence, tone and noise refused; rendering independent of request cuts, source-exact outside the 20 ms equal-power crossfades, no clicks; the engine's mix equals the source pieces (< 1e-5); undo, redo and revert exact; overlaps refused; Remix Properties dialog and Remix tool drag driven by automation id |
| Oracle | `crates/*/tests/*oracle*.rs`, `conformance.rs` | compare with ffmpeg/ffprobe (§2) |
| Robustness / fuzz | `*/tests/robustness.rs`, `*/tests/fuzz.rs` | seeded mutation and truncation of real and synthetic files; nothing may panic |
| Program monitor while dragging | `crates/ui-egui/tests/perf_ui.rs` | headless: a parameter changed on every UI pass (a value dragged in Effect Controls or Properties) and a backward scrub leave the Program monitor's picture at most one step behind (it shows the frame asked for the pass before while the frame due renders; it used to freeze until the mouse rested); a frame still rendering is kept when the same frame is asked for at a newer revision, so frames slower than a refresh (CPU effects) are shown too; `perf.stats` reports the picture's frame and revision against the playhead and the project |
| Performance | `*/tests/perf.rs` (`#[ignore]`) | §5 |

Commit `*.proptest-regressions` files so failing cases are re-run.

## 2. ffmpeg oracle tests

ffmpeg and ffprobe are **external processes** used to generate fixtures and to check results. They
are never linked or shipped ([AGENTS.md](../AGENTS.md) §2).

- The tools are found by `filmcraft-testkit` (a dev-dependency-only crate, `crates/testkit`), in
  this order: `FILMCRAFT_FFMPEG` / `FILMCRAFT_FFPROBE` (explicit paths; a wrong path is an error),
  then every directory on `PATH` (`ffmpeg` and `ffmpeg.exe`), then well-known install directories
  (`/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `C:\ffmpeg\bin`, …).
- If a tool is missing the test prints `SKIPPED (<test>): ffmpeg not found …` and passes. Set
  **`FILMCRAFT_REQUIRE_ORACLES=1`** (CI) to make every such skip a failure. In new tests use
  `let ff = filmcraft_testkit::require_ffmpeg!();` (also `require_ffprobe!`, `require_oracles!`).
- Fixtures are generated on first use into `<workspace>/target/fixtures/<crate>/`
  (`filmcraft_testkit::fixtures_dir`; independent of `CARGO_TARGET_DIR`, so agents with private
  target dirs share them; `FILMCRAFT_FIXTURES_DIR` overrides the root) and reused afterwards.
  Generators write to a per-process/thread temporary name (`testkit::temp_path`) and rename it into
  place, so concurrent tests never see half-written files. Delete the directory to regenerate.
  Never commit media.
- Every crate's `tests/common/mod.rs` delegates discovery and fixture paths to testkit, except
  `crates/codecs/src/tests.rs`, which still has its own lookup (the crate was being edited
  concurrently; migrate it when convenient).
- `cargo xtask fixtures [crate…]` pre-generates the whole fixture matrix up front (useful before a
  parallel test run or on a fresh machine). It runs each crate's ignored `generate_fixtures` test —
  the same generators the oracle tests call — for `h264`, `hevc`, `isobmff`, `matroska`, `dnx`,
  `prores`, `mpeg2v`, `ac3` and `codecs` (MXF, Ogg and MPEG TS / PS), and prints one `made` / `cached` / `skipped` line per fixture plus a summary. The
  `aac`, `opus` and `h264enc` oracles generate small per-test signals on demand and are not part of
  the matrix.
- Fixture sources are synthetic: `testsrc2`, `mandelbrot`, SMPTE bars, noise and fades, sine tones.
  H.264 and HEVC fixtures need ffmpeg built with libx264 and libx265. VideoToolbox fixtures are
  generated only on macOS. The Media Foundation parity tests (`crates/platform/tests/media_foundation.rs`,
  Windows only) make their 640x360, 1080p and 2160p H.264 / HEVC / Main 10 fixtures with ffmpeg and skip
  without a Direct3D 11 video device or the HEVC / VP9 / AV1 codec extensions of the Microsoft Store. The VP9 and
  AV1 fixtures (libvpx-vp9, libaom-av1; 360p, 1080p, 2160p, hidden alt-ref frames, two GOPs) need an ffmpeg with those encoders.

The NVENC tests (`crates/platform/tests/nvenc.rs` and `nvenc_export.rs` for H.264, `nvenc_rgba_input.rs` for
the GPU's RGB → 4:2:0 conversion, `nvenc_hevc.rs`,
`nvenc_hevc_export.rs`, `nvenc_hevc_probe.rs` and `nvenc_hevc_warm.rs` for H.265, `nvenc_hevc_main10.rs` and
`nvenc_hevc_hdr_export.rs` for Main 10 HDR; Windows only) skip, printing `SKIPPED`, without an NVIDIA GPU
with NVENC (H.265: when the HEVC probe says there is no HEVC encoder; Main 10: when the Main 10 probe says
there is no 10-bit encoder; on a machine that has one, a configuration NVENC refuses fails the test). The FFI layout tests in
`crates/platform/src/nvenc/abi_tests.rs` were generated from a C program built with MSVC
(`cl` after `vcvars64.bat`) against NVIDIA's MIT-licensed `nvEncodeAPI.h` (12.1); the header and the
program are not in the repository. To regenerate them, put the header in a scratch directory and
write a program that prints, for every type, field, constant and GUID of `src/nvenc/ffi.rs` that the
asserts name (H.264, HEVC and the HEVC picture parameters with their SEI payload array), `sizeof` / `alignof` / `offsetof`, the enum and macro values cast to
64-bit integers and the GUIDs as 128-bit hex; for the bit-fields of the `flags` words, which
`offsetof` cannot take, zero a structure, set that one field to its maximum and print the 32-bit word.
Compare the numbers with the asserts (a difference is drift, which must be understood before
anything is "fixed" to match) and add asserts for anything new.

### Pass criteria per codec

| Crate | Oracle check | Criterion |
|---|---|---|
| `h264` | decode every fixture single-threaded and frame-threaded; compare with `ffmpeg -f rawvideo -pix_fmt yuv420p` | **bit-exact**, every frame |
| `hevc` | same, `yuv420p` / `yuv420p10le` | **bit-exact** (8- and 10-bit) |
| `h264enc` | `ffmpeg -v warning -err_detect +crccheck+bitstream+buffer+explode` decodes our stream | no output at all (a `corrupt decoded frame` warning fails; a negative control checks dropped and truncated slices are reported), exact frame count, **bit-identical to the encoder's reconstruction**; B-frame order checked with ffprobe; rate-control targets |
| `prores` decode | ffmpeg `prores_ks` / `prores_aw` fixtures, compared at native depth | within **±1 LSB** (different integer IDCT); alpha bit-exact |
| `prores` encode | ffmpeg decodes with `-xerror` | no errors; agrees with our decoder within ±1 |
| `dnx` decode | ffmpeg VC-3 fixtures (13 DNxHD CIDs, DNxHR LB/SQ/HQ/HQX/444), compared at native depth | within **±2 LSB**, ≤ 100 samples per million beyond ±1 (ffmpeg's integer IDCT; we evaluate the exact IDCT) |
| `dnx` encode | ffmpeg decodes with `-xerror` | no errors; agrees with our decoder within ±2; HQ luma PSNR ≥ 45 dB |
| `aac` decode | ffmpeg's decode of the same stream | max abs error ~1e-7 (float) |
| `aac` encode | ffmpeg decodes our stream | no errors, no clipping, CBR within ±5% of target; SNR reported |
| `isobmff` | `ffprobe -show_packets` on ffmpeg-made MP4/MOV | packet offsets, sizes, pts/dts, durations, key flags and stream parameters equal; remuxed files decode with `ffmpeg -v error` silent |
| `matroska` | `ffprobe -show_packets` | every packet (stream, size, key flag, pts, duration) equal; seeks land on the latest keyframe ≤ target |
| `audio-dsp` loudness | `tests/loudness_oracle.rs`: signals generated in Rust (997 Hz sine, pink noise at 48/96 kHz, speech-like bursts at 48/44.1 kHz, stereo with silence and sub-gate passages, an fs/4 inter-sample-peak tone) written as float WAV and measured with `ffmpeg -af ebur128=peak=true:metadata=1` | momentary and short-term every 100 ms **±0.1 LU**, integrated **±0.1 LU**, LRA **±0.5 LU**, true peak **±0.2 dB** (against the analytic value when one exists). Measured: ΔM/ΔS ≤ 0.0005 LU, ΔI ≤ 0.007 LU, ΔLRA ≤ 0.04 LU, ΔTP ≤ 0.045 dB against ffmpeg. On the fs/4 tone ffmpeg's own true peak is +0.6 dB high (−0.32 vs the analytic −0.92 dBTP); ours is −0.05 dB |
| `audio-dsp` effects | `src/effects/premiere_tests.rs`, `src/design.rs`, registry tests in `src/effects/mod.rs`: generated tones, noise and impulses | every effect: neutral settings = delayed identity (≤ 2·10⁻⁴), bit-exact determinism, latency constant and equal to where an impulse lands, block-size independence, finite / denormal-free tails at extreme settings, > 1× realtime; per effect a level or frequency-response check (measured tone vs analytic `response_db` within 0.1–0.2 dB, Butterworth vs closed form 10⁻⁶ dB, Chebyshev/elliptic ripple and stop-band bounds, LR4 crossover sum flat within 0.02 dB, comb-notch depth, echo positions to the sample, Schroeder RT60 of generated impulses within 25 %, click repair −20 dB) |
| `mxf` (via `codecs/tests/mxf_oracle.rs`) | ffmpeg-written OP1a / OP-Atom / D-10 files decoded by ffmpeg | H.264 (long GOP, B pictures without temporal offsets) **bit-exact** every frame and at 25 random seeks; DNxHR ±2, ProRes ±1; PCM / AES3 **sample-exact**; frame count = ffprobe packets; timecode (25 fps, 29.97 DF); MPEG-2 long GOP, XDCAM HD422 (1080i 4:2:2) and D-10 / IMX (720×608 4:2:2) within ±4 (`mpeg2v` criterion); truncation/mutation never panics |
| `mpeg2v` | ffmpeg-encoded MPEG-1/2 elementary streams (progressive, TFF/BFF interlaced, 4:2:2 intra and long GOP, 1080i, escapes + custom matrices, MPEG-1) and our synthetic streams with field pictures / 16x8 / dual prime / concealment vectors, decoded by ffmpeg (`-xerror`) | per frame max ±4 and PSNR ≥ 58 dB, ≤ 3 % of samples differing (ffmpeg's integer IDCT vs our IEEE 1180-accurate one; measured max ±1-3, 65-75 dB); frame count, display order, picture types, interlacing and field order **exact** (ffprobe); IEEE 1180 procedure passes |
| MPEG TS / PS (via `codecs/tests/mpeg_oracle.rs`) | ffmpeg-written TS (MPEG-2 + MP2, H.264 + AAC, HEVC + LATM AAC), BDAV `.m2ts` / `.mts` (H.264 + Blu-ray LPCM / AC-3), VOB (MPEG-2 + MP2 / DVD LPCM 16 and 24-bit / AC-3), MPEG-1 system streams, `.m2v`; MPEG-2 in MOV (XDCAM `xd5c`), MP4 (`mp4v`), MKV | frame count and every frame's **PTS exact** (ffprobe); H.264 / HEVC **bit-exact**, MPEG-2 within the `mpeg2v` criterion; random seeks = sequential decode; LPCM **sample-exact**; AAC ≤ 2.1·10⁻⁷, MP2 (symphonia) ≤ 3.2·10⁻⁵, AC-3 ≤ 5·10⁻³ (decoder-specific dither; measured 2.3·10⁻³); random-access audio = sequential |
| `ac3` | ffmpeg AC-3 encodes (2/0 with coupling and rematrixing, 1/0 32 kHz, 3/2+LFE 448 kb/s, noise) decoded by ffmpeg | SNR per channel ≥ 40-60 dB on tonal content (measured 66-94 dB), ≥ 18 dB on noise (27.7 dB, the same as between two of our own dither seeds) |
| `ogg` (via `codecs/tests/ogg_oracle.rs`) | ffmpeg (libopus, Vorbis) files decoded by libopus / ffmpeg | length **exact** (pre-skip, end trimming); SNR ≥ 50 dB CELT (measured 77.7), ≥ 10 dB SILK, Vorbis 139 dB; random seeks within 6·10⁻⁶ of the continuous decode |
| MP4 audio at another rate (`codecs/tests/mp4_oracle.rs`) | ffmpeg-written AAC in MP4 (48 kHz, edit list skipping the priming) | reads resampled to 44.1 kHz in frame-sized requests equal the native decode interpolated at the same times (difference < 10⁻⁴, peak ≤ the source level) |
| image sequences / BWF (`codecs/tests/media_oracle.rs`) | ffmpeg-written PNG/TIFF/BMP/JPEG stills read by ffmpeg's image2 demuxer; ffmpeg BWF (`-write_bext`) | PNG/TIFF/BMP **exact**, JPEG ±12 (measured 3); missing frames hold the previous one; BWF TimeReference = ffprobe's, start timecode, samples exact |
| `export` | our own demuxer/decoder reads the file back; ffprobe counts frames when present | expected size, duration, colour, audio level; exact frame count |
| `export` settings and presets (M6.5) | `export/src/settings_tests.rs`, engine `export_tests.rs`: every built-in preset exports a slice of the demo sequence; ffprobe reads codec, profile, size, rate, audio format and data rate; our decoders read pixels back | codec / profile / container / size / `r_frame_rate` / sample format exact; H.264 bitrate ≤ VBR max × 1.1 and ≥ ½ target over 3 s; ProRes ≤ 1.5 × nominal; DNxHR within 25 % of nominal; loudness normalization within **0.5 LU** of the target (WAV and AAC), true peak ≤ ceiling + 0.1 dB; image sequences numbered `<name>000…`; limiter, overlays, metadata (`title`/`copyright` tags), MOV multiplexer, two-pass/CBR progress |

Each codec README has the full fixture matrix and the measured results.

## 3. Rendering: CPU reference and GPU parity

- The CPU compositor in `filmcraft-render` is the reference. Its tests check computed values
  (opacity blends, dissolve midpoints, Motion placement, ½-resolution against downsampled full
  resolution) and that `render::plan` matches `render_sequence`.
- `crates/gpu/src/tests.rs` composites the same frame plan (a YUV layer plus a transformed,
  semi-transparent RGBA layer) on the GPU and on the CPU. It requires a 99th-percentile channel
  difference of ≤ 6 and a mean of < 1.5 (8-bit levels). It skips when no GPU adapter is available.
- Effects implemented only on the CPU need no GPU test: the plan pre-renders those layers on the
  CPU.
- **Golden images** (`crates/golden`, a test-only crate): `tests/golden.rs` builds small procedural
  projects (demo-generator footage, bars, no media files) at 320×180 and renders one frame of each
  through the CPU compositor: Motion transform + opacity, four blend modes (Multiply, Screen,
  Overlay, Difference), Gaussian Blur, Lumetri basic correction, Crop, Cross Dissolve at 50 %, Dip
  to Black at 25 %, Wipe at 50 %, Timecode / Clip Name burn-in text, graphic clips (also a built-in
  graphics template with overridden properties and per-character styles with a pinned box), and colour
  management: S-Log3/S-Gamut3.Cine footage interpreted into Rec. 709 (`log_to_rec709`) and
  Rec. 2100 PQ footage tone mapped into Rec. 709 (`hdr_tone_map`; the footage is demo frames
  re-encoded by the test's `Encoded` source). Each frame is compared
  with `crates/golden/goldens/<scene>.png` by `filmcraft_testkit::golden`:
  **PSNR ≥ 45 dB, max abs ≤ 12, 99th-percentile per-pixel max channel difference ≤ 2** (8-bit
  sRGB levels; `Tolerance::RENDER`). On failure the actual frame and a ×8 difference image go to
  `<workspace>/target/golden-failures/`. Six `vfx_*` scenes cover the M5.11 effects (Corner Pin over Turbulent
  Displace, Ultra Key, Wonder Glow + Glint, Spacer + Stroke + Long Shadow over Gradient, VR Rotate Sphere,
  Channel Mix + Color Emboss + Rounded Crop); per-effect unit tests live in `crates/render/src/vfx/tests.rs`
  (identity at neutral parameters, determinism, range, a known value each, temporal/track effects through
  the compositor). `scenes_are_distinct` guards against blank or duplicate
  scenes.
- **Video transitions** (`crates/render/src/transitions/tests.rs`) are checked for every transition
  (84 modern, 21 Legacy, 7 hidden Obsolete): p = 0 is exactly the outgoing frame and p = 1 exactly
  the incoming one, near both ends the output is close to that frame (continuity), the midpoint differs
  from both, the output is finite and deterministic (also on 1×1 / odd sizes and NaN progress), every
  parameter visibly changes the picture, and every direction differs. Golden *fingerprints* (4×4 mean
  RGB grid at p = 0.25/0.5/0.75, ±4 levels) live in `src/transitions/goldens.txt`; re-bless with
  `FILMCRAFT_BLESS=1 cargo test -p filmcraft-render golden_fingerprints` and review the diff.
  `FILMCRAFT_TRANSITION_SHEETS=<dir> cargo test -p filmcraft-render contact_sheets -- --ignored`
  writes one contact-sheet PNG per folder for eyeballing (never commit them).
- **Blessing:** `FILMCRAFT_BLESS=1 cargo test -p filmcraft-golden` rewrites the references (and
  writes a `.attribution` sidecar for any new one; it prints the `ATTRIBUTION.md` row to add).
  References are original work rendered by FilmCraft, must stay under 50 KB (enforced on bless),
  and every one needs its sidecar and index row (`cargo xtask assets`). Look at a re-blessed PNG
  before committing it.
- The same scenes run through `plan_frame` + `GpuCompositor` when a GPU adapter exists
  (`gpu_matches_cpu_on_golden_scenes`, same p99 ≤ 6 / mean < 1.5 criterion; skipped otherwise).
  This found a real bug: the GPU upload cache was keyed by pixel-buffer address without keeping
  the buffer alive, so a new frame allocated at a freed frame's address was drawn with the stale
  texture (fixed; regression test `upload_cache_keeps_buffers_alive`).

## 4. UI: control channel and screenshots

### Scripted UI tests (headless)

`crates/ui-egui/tests/scripted.rs` runs the real `FilmcraftApp` under
[`egui_kittest`](https://ubjm.tcti.cn/hezuo/contact-66995532.html) (`build_eframe`): no window, no GPU, no OS event loop.
A small `Driver` opens the demo project, sends requests through the same control channel agents use
(`ControlRequest` → `control::handle`, exactly as the TCP server does) and steps egui frames until
each reply arrives. Synthetic input queued by `ui.click` / `ui.drag` / `ui.key` is moved into the
next frame by the app's own `raw_input_hook`, so clicks by automation id work headless. Covered:

| Test | Asserts |
|---|---|
| `demo_project_opens_headless_and_registers_widgets` | active sequence, > 50 registered widgets, panel and tool ids, 6 clips on V1 (`sequence.inspect`) |
| `razor_then_undo_through_the_control_channel` | `timeline.razor` adds a clip, the timeline can locate it on screen, `edit.undo` via `ui.menu.invoke`, redo |
| `insert_from_source_ripples_the_sequence` | `source.open` + marks + `source.insert` lengthens the sequence |
| `apply_effect_appears_in_effect_controls` | `effects.apply` adds Gaussian Blur in the model and `effectControls.effect.gaussian_blur` appears in the panel; undo removes it |
| `playback_toggle_and_stop` | `playback.toggle` / `ui.playback stop` and `ui.inspect` playback state |
| `clicking_a_tool_button_by_automation_id` | `ui.click {id: "tools.Razor"}` changes the tool (real egui input path) |
| `unknown_methods_and_commands_fail_cleanly` | errors come back as `{"ok": false}` |

`crates/ui-egui/tests/mixer_ui.rs` drives the Audio Track Mixer, Audio Clip Mixer, timeline track
keyframes and the Audio Gain dialog by automation id and with multi-frame pointer drags (press, move
over several frames, release). With `FILMCRAFT_UI_SNAPSHOT_DIR=<dir>` it also renders the window
offscreen through wgpu (`Harness::render`) and writes `mixer-*.png`; this works without a visible
window (for example on a locked screen, where `ui.screenshot` cannot capture).
`crates/ui-egui/tests/essential_sound_ui.rs` does the same for the Essential Sound panel (type buttons,
switches, a slider drag as one undo step, section bypass, Auto-Match, ducking, Browse presets;
`essential-sound-*.png`).
`crates/ui-egui/tests/clip_audio_ui.rs` covers the audio of video clips (#223): the linked audio's
Volume, Channel Volume and Panner in Effect Controls and Properties, and the Volume line on audio
clips in the Timeline (a drag is one undo step, Pen-tool keyframes, keyframe drags, a click or
right-click on the line still reaches the clip; `FILMCRAFT_UI_SHOTS=<dir>` writes `volume-*.png`).

Essential Sound engine tests (`crates/engine/src/essential_sound_tests.rs`) build projects from
generated speech-like and tonal WAVs: Auto-Match lands within ±0.5 LU of the target (measured: 0.000 LU,
with and without a repair/clarity chain) and measures a mono clip as one channel (3.01 LU below the
same signal as dual mono, within 0.1 LU of a one-channel meter over the file); ducking on a dialogue +
music project gives keyframes within 60 ms of the expected times (measured 20 ms) and −15.00 dB in the
mix; each repair stage improves its metric through the render path (hum −39 dB, rumble −22 dB, noise
floor −13 dB, sibilance −19 dB, reverb tail −10 dB); the mix is bit-identical however requests are cut
and the WAV export equals it.
`perf_full_dialogue_chain_realtime_factor` (ignored; run with `--release`) prints the realtime factor of
all nine Dialogue effects on one clip (22× on one core).

`crates/ui-egui/tests/export_ui.rs` drives Export mode: the settings column and Summary, editing a
setting turning the preset into Custom, the Preset Manager (search, favourite star, save, apply,
delete), the queue panel (Send to Queue, Up/Down, Cancel, Start, Retry, Clear) with real encodes,
the Export button and the header's Quick Export popup (`export-*.png` snapshots).

`crates/ui-egui/tests/color_ui.rs` drives colour features the same way (`color-*.png` snapshots):
Lumetri Input/Look LUT menus and section switches, the Interpret Footage ▸ Color Management and
Sequence Color Management dialogs, and the HDR scopes of a PQ sequence.

Colour science tests: `filmcraft-color` (curve round trips and published reference values per
camera log curve, BT.709/BT.2087 matrices, BT.2390 EETF, gamut mapping, LUT tetrahedral vs a
brute-force barycentric reference, `.cube`/`.3dl` round trips), `filmcraft-gpu`
(`gpu_lut_matches_cpu_tetrahedral`: WGSL vs CPU, max |Δ| ≈ 2e-7), `render::colorman`,
`render::color_match`, engine `color_tests.rs`, and `export::tests::hdr_exports_signal_pq_and_hlg`
(PQ/HLG exports read back by our demuxer and by ffprobe: `color_transfer`, `color_primaries`,
`color_space`, mastering display and content light side data).

Not covered headless: `ui.screenshot` (needs a real viewport), the wgpu monitor path (the harness
runs the CPU texture path), audio output, and wall-clock playback advance (kittest frames do not
advance real time). Playing sound is covered without a device: `play_ahead` tests drive the
mixed-ahead ring with a wall-clock device (order, underruns stay in time, mixer panic, stop), and
`cargo run --release -p filmcraft-ui-egui --example bench_audio -- <project.fcproj>` plays a real
project's sequences through a simulated 48 kHz device and counts dropouts, mixing in the callback
(`direct`, as before) against mixing ahead (`ahead`). Use `Driver` for new UI regressions: `d.exec(command, params)`,
`d.ok(method, params)`, `d.frames(n)`.

### Interactive checks

For visual work, drive the real app:

```sh
cargo run --release -p filmcraft -- --control 9876
```

Then script it over the control channel or MCP: run commands, click by automation id,
`ui.inspect` / `ui.elements` to assert state, and `ui.screenshot {"path": "…"}` to capture the window
or a single panel (`{"panel": "Timeline"}`). Look at every screenshot. Examples are in
[agents.md](agents.md). The headless path (`filmcraft-cli run script.jsonl`, MCP `--demo`) covers
engine behaviour without a window.

## 5. Performance

| Benchmark | Command |
|---|---|
| H.264 decode, 1080p, threads sweep | `cargo test --release -p filmcraft-h264 --test perf -- --ignored --nocapture` |
| HEVC decode, 1080p / 2160p | `cargo test --release -p filmcraft-hevc --test perf -- --ignored --nocapture` |
| ProRes decode/encode | `cargo test --release -p filmcraft-prores --test perf -- --ignored --nocapture` |
| Encoder speed / PSNR / bitrate | `cargo run --release -p filmcraft-h264enc --example h264enc_synth -- …` |
| Decode a real file through the media stack | `cargo run --release -p filmcraft-cli -- bench-decode file.mp4 --frames 120` |
| Coding-tool coverage of the fixtures | `cargo test --release -p filmcraft-h264 --test conformance coverage_report -- --ignored --nocapture` (same for `hevc`) |
| **Whole-app suite**: decode fps per codec, playback, scrubbing, timeline UI, export, project save/open, peak RSS | `cargo xtask bench` ([performance.md](performance.md)) |

`cargo xtask bench` (the example `crates/ui-egui/examples/bench/`) runs six sections, each in its
own process under `/usr/bin/time` so peak RSS is per section: `decode` (every frame of 1080p and
2160p H.264, HEVC, VP9, AV1 and ProRes through the media stack), `playback` (the scenarios below),
`scrub` (random seeks and playhead drags: time until the exact frame is on screen), `timeline`
(the real app under `egui_kittest` with a 1000-clip / 20-track sequence: update, tessellation and
wgpu render ms per frame), `export` (H.264 + AAC and ProRes) and `project` (save / open 5000 clips).
Options: `--sections decode,scrub`, `--only <substring>` (fixture or scenario), `--repeat N`,
`--quick`, `--cpu`, `--hw auto|off` (Settings ▸ Playback ▸ Hardware decoding; default `auto`, as the app), `--label NAME` (output `target/bench/bench-<label>.{json,md}`),
`--section NAME --json FILE` (one section in-process, e.g. under a profiler). Fixtures are made
with ffmpeg in `target/fixtures/playback/` (or `$FILMCRAFT_FIXTURES/playback`). The suite's
`playback` section plays `h264-1080`, `stack3` and `stack3-blend` at Full, `h264-2160` at Full, 1/2 and 1/4 and with
draft decoding (Settings ▸ Playback ▸ Draft decoding) at 1/2 and 1/4, and `hevc-2160`, `vp9-2160`
and `av1-2160` at Full, 1/2 and 1/2 draft; its `draft` column counts the frames decoded in draft
mode.

The perf tests check bit-exactness before they time anything. Results go in the crate README's
performance table, with machine and thread count. Headline numbers go in
[ROADMAP.md](../ROADMAP.md). Measure on an idle machine: parallel agent builds distort timings.

### Playback benchmark

`cargo xtask bench-playback` (the example `crates/ui-egui/examples/bench_playback.rs`) plays
sequences headlessly through the Program monitor's own frame scheduler: the `FrameServer` worker
pool, `schedule_playback` (prefetch order and stale-job dropping), its caches, and the
`PlaybackMeter` that counts shown/dropped frames in the app. On the GPU path it also composites
each plan with `filmcraft-gpu`, as the monitor does on the UI thread.

```sh
cargo xtask bench-playback                                   # every scenario, GPU path, Full and Half
cargo xtask bench-playback --scenario stack3 --res full --cpu
cargo xtask bench-playback --json target/bench-playback.json # machine-readable results
```

| Scenario | What plays |
|---|---|
| `h264-1080` | one 1080p23.976 H.264 clip (testsrc2 + grain, ~45 Mbit/s, 250-frame GOP) |
| `stack3` | three 1080p H.264 clips on V1–V3; V2/V3 scaled, positioned, rotated, 70–85 % opacity |
| `stack3-blend` | `stack3` with V2 in Screen and V3 in Overlay (blend modes on the GPU) |
| `h264-2160` | one 2160p23.976 H.264 clip |
| `demo` | the built-in demo project (procedural footage, transitions, effects) |
| `after-preview` | 1080p H.264 + Lumetri/Sharpen/Levels/Tint: a live play, Render Effects In to Out, then two plays of the green segment |
| `seek-storm` | 40 jumps to random frames 150 ms apart (scrubbing), time until the exact frame shows |

Options: `--res full,half,quarter`, `--cpu` (CPU compositor + texture conversion instead of the
GPU path), `--seconds`, `--refresh` (display Hz), `--workers`, `--repeat`, `--json <file>`.
Fixtures are made with ffmpeg in `target/fixtures/playback/` (set `FILMCRAFT_FIXTURES` to share one
set between worktrees).

Columns: **shown/drop** as counted in the app (a frame is shown when its exact picture was on
screen at a refresh while it was due; frames passed over without a refresh count as dropped);
**ontime** = due frames whose job finished before they were due; **lat** = queue→ready per job,
**svc** = worker time per job; **cpu/j**, **src/j**, **srcC/j** = worker thread CPU, source fetch
(decode) wall and thread CPU per job; **ui** = time on the UI thread to present a frame (GPU upload
and draw, or texture conversion); **cpu ms/f** = process CPU per frame and **cores** = the cores that
needs at the sequence frame rate; **seeks/dec** = decoder restarts and samples decoded; **waste** =
jobs for frames that were never due.

Wall-clock columns (shown/drop, ontime, latencies) depend on machine load, so each row prints the
load average. CPU columns (thread and process CPU time) and the structural counters (seeks,
samples decoded, wasted jobs) do not, and are what to compare between runs on a busy machine.

Results on an M4 Pro (14 cores), GPU path, 8 s plays, median of 2 alternating runs of the M4.6
baseline (commit `5170376`) and the result of M4.6, on a machine shared with parallel agent
builds (**load average 86–207**, so wall-clock columns are pessimistic; CPU ms/frame is not):

| Scenario | shown/dropped before | after | CPU ms/frame before → after | decoder seeks |
|---|---|---|---|---|
| h264-1080 Full / Half | 106/86, 118/74 | **192/0, 192/0** | 80 → 46, 95 → 46 | 3–4 → 1 |
| stack3 (3 × 1080p) Full / Half | 185/7, 174/18 | **192/0, 192/0** | 136 → 109, 117 → 111 | 4 → 3 (one per source) |
| h264-2160 Full / Half | 0/192, 0/192 | 5/187, 19/173 | 110 → 135, 170 → 145 | 4–5 → 1–2 |
| demo Full / Half | 63/129, 124/68 | 97/95, 152/40 | 140 → 80, 93 → 40 | |
| after-preview, 1st play after render (Full) | 80/112 | **190/2** | 119 → 26 | 104 → 0 |
| after-preview, 2nd play (Full) | 145/47 | **192/0** | 76 → 26 | 132 → 0 |
| after-preview, live effects (Half) | 18/174 | 69/123 (frames skipped evenly) | 109 → 95 | |
| seek-storm (40 jumps, 150 ms each) | 0/40 shown | 6/40 | 827 → 666 per jump | |

At load ~25–60 the same final build plays h264-1080, stack3 and h264-2160 at Full with 0 dropped
in 3 of 3 runs (192/0) and render previews 192/0. The 4K fixture needs ~4 cores of decode per
real-time second (≈160 ms CPU per frame at 170 Mbit/s); the demo project's procedural footage
~190 ms per Full-resolution frame.

### Audio mixer throughput

`perf_24_tracks_3_effects_realtime_factor` (24 tracks, 72 inserts and a compressed submix on one
core) must run at least 4x realtime in release builds (1x in debug). On Unix it times the test
thread's CPU time, so it runs with the rest of the suite. Hosts without a per-thread CPU clock
(Windows) fall back to wall time, which depends on load, so there it is ignored by default; run it
explicitly with
`cargo test --release -p filmcraft-render --lib perf_24_tracks_3_effects_realtime_factor -- --ignored --nocapture`
and record the machine with the result.


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://dpgc.ai-hao123.com/peixun/local-86996340.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://skwa.wtpuscm.cn/qiye/sales-452431.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://vkoa.tcti.cn/fenxi/learning-41863024.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://lsex.ai-hao123.com/gongju/calendar-17741275.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://exsw.yx-sf.com/youhua/budget-75395254.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://vkoa.tcti.cn/fenxi/learning-41863024.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://exsw.yx-sf.com/youhua/budget-75395254.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://aalj.wtpuscm.cn/sheji/engagement-807379.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://yyak.yx-sf.com/liuliang/learning-84631732.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://aabe.tcti.cn/shichang/chapter-43416989.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://bxxq.yx-sf.com/wenzhang/discount-10594767.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://zkba.tcti.cn/fenxi/services-68521778.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://aalj.wtpuscm.cn/sheji/engagement-807379.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://exsw.yx-sf.com/youhua/budget-75395254.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://dpgc.ai-hao123.com/peixun/local-86996340.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://nlhq.tcti.cn/zixun/message-75753271.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://skyy.tcti.cn/shuju/price-06442157.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://rpfb.yx-sf.com/zhinan/strategy-40918346.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://skyy.tcti.cn/shuju/price-06442157.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://yjdn.ai-hao123.com/guanjianci/interface-80887664.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://klnd.yx-sf.com/xinwen/market-27283395.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://onxh.wtpuscm.cn/chuangxin/conversion-186515.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://onxh.wtpuscm.cn/chuangxin/conversion-186515.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://onxh.wtpuscm.cn/chuangxin/conversion-186515.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://nlhq.tcti.cn/zixun/message-75753271.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://yjdn.ai-hao123.com/guanjianci/interface-80887664.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://aabe.tcti.cn/shichang/chapter-43416989.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://zkba.tcti.cn/fenxi/services-68521778.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://exsw.yx-sf.com/youhua/budget-75395254.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://khaw.ai-hao123.com/anli/behavior-35439337.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://lsex.ai-hao123.com/gongju/calendar-17741275.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://srec.ai-hao123.com/chuangxin/photo-93384708.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://yjdn.ai-hao123.com/guanjianci/interface-80887664.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://yjdn.ai-hao123.com/guanjianci/interface-80887664.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://onxh.wtpuscm.cn/chuangxin/conversion-186515.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://jjlo.tcti.cn/xuexi/reminder-15742330.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://yyak.yx-sf.com/liuliang/learning-84631732.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://aaoa.wtpuscm.cn/zhinan/forum-595217.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://rpfb.yx-sf.com/zhinan/strategy-40918346.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://yyak.yx-sf.com/liuliang/learning-84631732.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://krqb.tcti.cn/fenxi/privacy-40467252.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://ptve.ai-hao123.com/zhineng/promotion-94588549.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://bxxq.yx-sf.com/wenzhang/discount-10594767.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://skwa.wtpuscm.cn/qiye/sales-452431.html)

</details>



---

<details open>
<summary><b>📚 Testing 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Testing」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Testing 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Testing 内部组件解耦与事件状态机规范 (Draft-01)](https://skwa.wtpuscm.cn/qiye/sales-452431.html)
* [Testing 内部组件解耦与事件状态机规范 (Draft-07)](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [【官方规范】Testing 模块化解耦与协议标准 核心运行拓扑标准](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [现代 ±1 LSB 架构演进之路 —— Testing 深度实践](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Testing 深度实践](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [基于 Testing 的高吞吐 Testing 设计白皮书](https://skyy.tcti.cn/shuju/price-06442157.html)
* [基于 Testing 的高吞吐 模块化解耦与协议标准 设计白皮书](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [现代 Audio mixer throughput 架构演进之路 —— Testing 深度实践](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [【官方规范】Testing sample-exact 核心运行拓扑标准](https://vkoa.tcti.cn/fenxi/learning-41863024.html)
* [【Testing】Testing 核心系统架构与设计规约 (Node-43)](https://skyy.tcti.cn/shuju/price-06442157.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Testing 深度实践](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [面向大规模网络的 Testing 工业级架构基准](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [Testing 分布式数据通道与 模块化解耦与协议标准 技术规范 (Verified)](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [Testing 分布式数据通道与 5. Performance 技术规范 (v2.0-GA)](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [【官方规范】Testing ±2 LSB 核心运行拓扑标准](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [1. Kinds of tests 核心系统架构与设计规约 (v2.0-GA)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [2. ffmpeg oracle tests 核心系统架构与设计规约 (Spec-v2.2)](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [Pass criteria per codec 核心系统架构与设计规约 (Node-85)](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [Testing 内部组件解耦与事件状态机规范 (Verified)](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [基于 Testing 的高吞吐 ±2 LSB 设计白皮书](https://www.poykq.cn/redian/tutorial-673489.html)
* [Testing 分布式数据通道与 sample-exact 技术规范 (Core/sample)](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [Testing 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [现代 1. Kinds of tests 架构演进之路 —— Testing 深度实践](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [Testing 内部组件解耦与事件状态机规范 (RFC-367)](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [【Testing】±2 LSB 核心系统架构与设计规约 (Spec-v2.7)](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [【Testing】±1 LSB 核心系统架构与设计规约 (v2.0-GA)](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [基于 Testing 的高吞吐 Interactive checks 设计白皮书](https://vkoa.tcti.cn/fenxi/learning-41863024.html)
* [Interactive checks 核心系统架构与设计规约 (Core/Intera)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [基于 Testing 的高吞吐 5. Performance 设计白皮书](https://eqrt.wtpuscm.cn/paiming/discovery-059548.html)
* [Testing 内部组件解耦与事件状态机规范 (Core/生产环境运维)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [基于 Testing 的高吞吐 Playback benchmark 设计白皮书](https://sgnf.yx-sf.com/chuangxin/security-35610453.html)
* [现代 2. ffmpeg oracle tests 架构演进之路 —— Testing 深度实践](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [基于 Testing 的高吞吐 sample-exact 设计白皮书](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [Testing 内部组件解耦与事件状态机规范 (Node-61)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [【官方规范】Testing 2. ffmpeg oracle tests 核心运行拓扑标准](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing 内部组件解耦与事件状态机规范 (Node-75)](https://krqb.tcti.cn/fenxi/privacy-40467252.html)
* [Testing 分布式数据通道与 2. ffmpeg oracle tests 技术规范 (Core/2. ffm)](https://vkoa.tcti.cn/fenxi/learning-41863024.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Testing 核心 API 接口契约与客户端调用指南](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing 插件生态规范与 ±1 LSB 扩展手册 (RFC-759)](https://srec.ai-hao123.com/chuangxin/photo-93384708.html)
* [【生产手册】Testing 模块通信与请求穿透标准](https://www.poykq.cn/redian/tutorial-673489.html)
* [Testing vs 业界主流方案：Playback benchmark 深度技术选型对比](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [【集成指南】±2 LSB 服务端接入准则与 Testing 实战](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [Testing 插件生态规范与 分布式状态机一致性 扩展手册 (Node-12)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [Testing 插件生态规范与 5. Performance 扩展手册 (v2.0-GA)](https://srec.ai-hao123.com/chuangxin/photo-93384708.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Testing 实战](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [Testing 异步中间件流水线与 sample-exact 接入规范](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [【集成指南】Audio mixer throughput 服务端接入准则与 Testing 实战](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [基于 Testing 的自动化部署与生产环境配置实践](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [Testing 插件生态规范与 ±1 LSB 扩展手册 (v2.0-GA)](https://onxh.wtpuscm.cn/chuangxin/conversion-186515.html)
* [【集成指南】Playback benchmark 服务端接入准则与 Testing 实战](https://nlhq.tcti.cn/zixun/message-75753271.html)
* [Testing vs 业界主流方案：modules, 深度技术选型对比](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [Testing 异步中间件流水线与 Testing 接入规范](https://acfo.ai-hao123.com/jishu/advertising-97553670.html)
* [Testing 插件生态规范与 分布式状态机一致性 扩展手册 (Draft-05)](https://ptve.ai-hao123.com/zhineng/promotion-94588549.html)
* [Testing vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [Testing vs 业界主流方案：Testing 深度技术选型对比](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [Testing 异步中间件流水线与 ±2 LSB 接入规范](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing 异步中间件流水线与 1. Kinds of tests 接入规范](https://rpfb.yx-sf.com/zhinan/strategy-40918346.html)
* [【集成指南】Pass criteria per codec 服务端接入准则与 Testing 实战](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [Testing vs 业界主流方案：Interactive checks 深度技术选型对比](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [Testing 插件生态规范与 可信存活健康度量 扩展手册 (Core/可信存活健康)](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing vs 业界主流方案：sample-exact 深度技术选型对比](https://aabe.tcti.cn/shichang/chapter-43416989.html)
* [Testing 插件生态规范与 Testing 扩展手册 (Core/Testin)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [Testing 异步中间件流水线与 生产环境运维调优手册 接入规范](https://khaw.ai-hao123.com/anli/behavior-35439337.html)
* [Testing 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-154)](https://jjlo.tcti.cn/xuexi/reminder-15742330.html)
* [Testing vs 业界主流方案：Pass criteria per codec 深度技术选型对比](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [Testing 异步中间件流水线与 modules, 接入规范](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [Testing 插件生态规范与 ±1 LSB 扩展手册 (Verified)](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [Testing 插件生态规范与 bit-exact 扩展手册 (Spec-v2.3)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Testing 插件生态规范与 ±2 LSB 扩展手册 (Core/±2 LSB)](https://www.poykq.cn/redian/tutorial-673489.html)
* [【集成指南】2. ffmpeg oracle tests 服务端接入准则与 Testing 实战](https://yyak.yx-sf.com/liuliang/learning-84631732.html)
* [Testing vs 业界主流方案：1. Kinds of tests 深度技术选型对比](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [Testing 插件生态规范与 sample-exact 扩展手册 (Spec-v2.1)](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [【集成指南】1. Kinds of tests 服务端接入准则与 Testing 实战](https://exsw.yx-sf.com/youhua/budget-75395254.html)
* [【集成指南】modules, 服务端接入准则与 Testing 实战](https://ubjm.tcti.cn/hezuo/contact-66995532.html)

#### 3. ⚡ Testing 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：Testing 实时镜像与索引入口](https://zkba.tcti.cn/fenxi/services-68521778.html)
* [Testing 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [冷热数据分层镜像：Testing ±2 LSB 权威归档源](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [冷热数据分层镜像：Testing Interactive checks 权威归档源](https://klnd.yx-sf.com/xinwen/market-27283395.html)
* [Testing 官方高可用镜像注册节点 (RFC-793)](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [冷热数据分层镜像：Testing 1. Kinds of tests 权威归档源](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [Testing 官方高可用镜像注册节点 (Spec-v2.5)](https://twqa.yx-sf.com/fuwu/growth-54155500.html)
* [Testing 亚太与欧美多活集群数据同步中枢](https://ptve.ai-hao123.com/zhineng/promotion-94588549.html)
* [Testing 自动化持续集成快照与拓扑发布源 (RFC-294)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [【镜像入口】Testing 官方毫秒级实时数据广播节点](https://mznw.ai-hao123.com/kaifa/login-17204469.html)
* [Testing 官方高可用镜像注册节点 (Draft-06)](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [Testing 去中心化数据同步源与拓扑寻址规约](https://elqa.yx-sf.com/baogao/premium-64991186.html)
* [冷热数据分层镜像：Testing 可信存活健康度量 权威归档源](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Verified)](https://nnfu.tcti.cn/gongxiang/excellence-64418623.html)
* [Testing 官方高可用镜像注册节点 (v2.0-GA)](https://fljr.yx-sf.com/jiaocheng/sale-99240382.html)
* [冷热数据分层镜像：Testing 5. Performance 权威归档源](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Draft-03)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [Testing 官方高可用镜像注册节点 (Draft-05)](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Node-44)](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [Testing 自动化持续集成快照与拓扑发布源 (RFC-508)](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [Testing 官方高可用镜像注册节点 (RFC-119)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：Testing 模块化解耦与协议标准 权威归档源](https://klnd.yx-sf.com/xinwen/market-27283395.html)
* [冷热数据分层镜像：Testing Playback benchmark 权威归档源](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [冷热数据分层镜像：Testing sample-exact 权威归档源](https://skwa.wtpuscm.cn/qiye/sales-452431.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Node-87)](https://bxxq.yx-sf.com/wenzhang/discount-10594767.html)
* [Testing: sample-exact sample-exact (v2.0-GA)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [冷热数据分层镜像：Testing 2. ffmpeg oracle tests 权威归档源](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [Testing: 1. Kinds of tests 1. Kinds of tests (Verified)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [冷热数据分层镜像：Testing Audio mixer throughput 权威归档源](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [Testing 官方高可用镜像注册节点 (Spec-v1.2)](https://cvxt.wtpuscm.cn/yanjiu/template-505607.html)
* [冷热数据分层镜像：Testing Testing 权威归档源](https://skwa.wtpuscm.cn/qiye/sales-452431.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Draft-01)](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [Testing: 模块化解耦与协议标准 模块化解耦与协议标准 (Draft-05)](https://jsmw.wtpuscm.cn/pingtai/success-293217.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Draft-07)](https://rpfb.yx-sf.com/zhinan/strategy-40918346.html)
* [冷热数据分层镜像：Testing 高韧性系统架构设计 权威归档源](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [冷热数据分层镜像：Testing Pass criteria per codec 权威归档源](https://aalj.wtpuscm.cn/sheji/engagement-807379.html)
* [Testing: sample-exact sample-exact (v2.0-GA)](https://ptve.ai-hao123.com/zhineng/promotion-94588549.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Testing 高负载场景下 ±2 LSB 基准评测报告](https://jjlo.tcti.cn/xuexi/reminder-15742330.html)
* [Testing 故障自愈与网络拓扑重构实践](https://vkoa.tcti.cn/fenxi/learning-41863024.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://ubjm.tcti.cn/hezuo/contact-66995532.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v1.4)](https://aabe.tcti.cn/shichang/chapter-43416989.html)
* [Testing 高负载场景下 分布式状态机一致性 基准评测报告](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (RFC-610)](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [Testing 节点连通性、存活性探测与防作弊指标](https://kido.yx-sf.com/jiaoliu/conference-73984125.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Core/分布式状态机)](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-76)](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [【评测基准】Testing 吞吐抖动度量与健康检查协议](https://klnd.yx-sf.com/xinwen/market-27283395.html)
* [Testing 权威网络权重传递与收录基准规范](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [Testing 高负载场景下 sample-exact 基准评测报告](https://ttgk.yx-sf.com/xitong/chapter-52812607.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-72)](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Verified)](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.7)](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Core/模块化解耦与)](https://klnd.yx-sf.com/xinwen/market-27283395.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Draft-05)](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Verified)](https://zkba.tcti.cn/fenxi/services-68521778.html)
* [Testing 高负载场景下 bit-exact 基准评测报告](https://skyy.tcti.cn/shuju/price-06442157.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Core/分布式状态机)](https://werd.ai-hao123.com/fenxi/settings-69929794.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.6)](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [Testing 高负载场景下 modules, 基准评测报告](https://fwra.wtpuscm.cn/yinqing/prospect-865467.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (RFC-339)](https://krqb.tcti.cn/fenxi/privacy-40467252.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Node-81)](https://lelc.ai-hao123.com/xinwen/luxury-01042389.html)
* [Testing 高负载场景下 Testing 基准评测报告](https://pohd.ai-hao123.com/yunying/luxury-64366913.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://skwa.wtpuscm.cn/qiye/sales-452431.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (v2.0-GA)](https://zaaj.yx-sf.com/xuexi/digital-45758894.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (RFC-777)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-77)](https://zszv.tcti.cn/zhineng/digital-17268206.html)
* [Testing 高负载场景下 Playback benchmark 基准评测报告](https://gudu.wtpuscm.cn/shichang/products-919595.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Spec-v2.8)](https://mies.ai-hao123.com/xitong/funnel-40438469.html)
* [Testing 高负载场景下 高韧性系统架构设计 基准评测报告](https://ivhw.wtpuscm.cn/wangluo/layout-331715.html)
* [Testing 高负载场景下 ±1 LSB 基准评测报告](https://xobx.tcti.cn/paiming/beauty-62525661.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-39)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.5)](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Draft-05)](https://lsex.ai-hao123.com/gongju/calendar-17741275.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Node-94)](https://scdi.wtpuscm.cn/liuliang/identity-112354.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-21)](https://werq.yx-sf.com/yingyong/advertising-29792751.html)
* [Testing 高负载场景下 Pass criteria per codec 基准评测报告](https://aaoa.wtpuscm.cn/zhinan/forum-595217.html)

</details>



---

<details open>
<summary><b>📚 Testing 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Testing」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Testing 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【官方规范】Testing 模块化解耦与协议标准 核心运行拓扑标准](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-17867.html)
* [现代 ±1 LSB 架构演进之路 —— Testing 深度实践](https://ankmz.cn/anquan/tag-68558329-2-8234.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Testing 深度实践](https://www.poykq.cn/redian/tutorial-673489-3-36911.html)
* [基于 Testing 的高吞吐 Testing 设计白皮书](https://yyak.yx-sf.com/liuliang/learning-84631732-4-33386.html)
* [基于 Testing 的高吞吐 模块化解耦与协议标准 设计白皮书](https://kido.yx-sf.com/jiaoliu/conference-73984125-5-25484.html)
* [现代 Audio mixer throughput 架构演进之路 —— Testing 深度实践](https://werd.ai-hao123.com/fenxi/settings-69929794-6-22258.html)
* [【官方规范】Testing sample-exact 核心运行拓扑标准](https://ptve.ai-hao123.com/zhineng/promotion-94588549-7-41292.html)
* [【Testing】Testing 核心系统架构与设计规约 (Node-43)](https://cvxt.wtpuscm.cn/yanjiu/template-505607-8-44318.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Testing 深度实践](https://zkba.tcti.cn/fenxi/services-68521778-9-40958.html)
* [面向大规模网络的 Testing 工业级架构基准](https://bxxq.yx-sf.com/wenzhang/discount-10594767-10-55714.html)
* [Testing 分布式数据通道与 模块化解耦与协议标准 技术规范 (Verified)](https://srec.ai-hao123.com/chuangxin/photo-93384708-11-59680.html)
* [Testing 分布式数据通道与 5. Performance 技术规范 (v2.0-GA)](https://skwa.wtpuscm.cn/qiye/sales-452431-12-25288.html)
* [【官方规范】Testing ±2 LSB 核心运行拓扑标准](https://vkoa.tcti.cn/fenxi/learning-41863024-13-61607.html)
* [1. Kinds of tests 核心系统架构与设计规约 (v2.0-GA)](https://ttgk.yx-sf.com/xitong/chapter-52812607-14-56049.html)
* [2. ffmpeg oracle tests 核心系统架构与设计规约 (Spec-v2.2)](https://lelc.ai-hao123.com/xinwen/luxury-01042389-15-6194.html)
* [Pass criteria per codec 核心系统架构与设计规约 (Node-85)](https://eqrt.wtpuscm.cn/paiming/discovery-059548-16-3582.html)
* [Testing 内部组件解耦与事件状态机规范 (Verified)](https://nnfu.tcti.cn/gongxiang/excellence-64418623-17-26361.html)
* [基于 Testing 的高吞吐 ±2 LSB 设计白皮书](https://twqa.yx-sf.com/fuwu/growth-54155500-18-13903.html)
* [Testing 分布式数据通道与 sample-exact 技术规范 (Core/sample)](https://acfo.ai-hao123.com/jishu/advertising-97553670-19-4880.html)
* [Testing 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://fwra.wtpuscm.cn/yinqing/prospect-865467-20-29245.html)
* [现代 1. Kinds of tests 架构演进之路 —— Testing 深度实践](https://krqb.tcti.cn/fenxi/privacy-40467252-21-2703.html)
* [Testing 内部组件解耦与事件状态机规范 (RFC-367)](https://elqa.yx-sf.com/baogao/premium-64991186-22-22741.html)
* [【Testing】±2 LSB 核心系统架构与设计规约 (Spec-v2.7)](https://khaw.ai-hao123.com/anli/behavior-35439337-23-11189.html)
* [【Testing】±1 LSB 核心系统架构与设计规约 (v2.0-GA)](https://onxh.wtpuscm.cn/chuangxin/conversion-186515-24-33216.html)
* [基于 Testing 的高吞吐 Interactive checks 设计白皮书](https://jjlo.tcti.cn/xuexi/reminder-15742330-25-60309.html)
* [Interactive checks 核心系统架构与设计规约 (Core/Intera)](https://fljr.yx-sf.com/jiaocheng/sale-99240382-26-35266.html)
* [基于 Testing 的高吞吐 5. Performance 设计白皮书](https://dpgc.ai-hao123.com/peixun/local-86996340-27-51529.html)
* [Testing 内部组件解耦与事件状态机规范 (Core/生产环境运维)](https://scdi.wtpuscm.cn/liuliang/identity-112354-28-63740.html)
* [基于 Testing 的高吞吐 Playback benchmark 设计白皮书](https://ubjm.tcti.cn/hezuo/contact-66995532-29-50226.html)
* [现代 2. ffmpeg oracle tests 架构演进之路 —— Testing 深度实践](https://sgnf.yx-sf.com/chuangxin/security-35610453-30-8026.html)
* [基于 Testing 的高吞吐 sample-exact 设计白皮书](https://pohd.ai-hao123.com/yunying/luxury-64366913-31-28218.html)
* [Testing 内部组件解耦与事件状态机规范 (Node-61)](https://jsmw.wtpuscm.cn/pingtai/success-293217-32-1660.html)
* [【官方规范】Testing 2. ffmpeg oracle tests 核心运行拓扑标准](https://zszv.tcti.cn/zhineng/digital-17268206-33-50419.html)
* [Testing 内部组件解耦与事件状态机规范 (Node-75)](https://exsw.yx-sf.com/youhua/budget-75395254-34-10455.html)
* [Testing 分布式数据通道与 2. ffmpeg oracle tests 技术规范 (Core/2. ffm)](https://mznw.ai-hao123.com/kaifa/login-17204469-35-9092.html)
* [【Testing】可信存活健康度量 核心系统架构与设计规约 (Node-84)](https://gudu.wtpuscm.cn/shichang/products-919595-36-14552.html)
* [Testing 内部组件解耦与事件状态机规范 (RFC-759)](https://xobx.tcti.cn/paiming/beauty-62525661-37-45954.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【生产手册】Testing 模块通信与请求穿透标准](https://werq.yx-sf.com/yingyong/advertising-29792751-38-28910.html)
* [Testing vs 业界主流方案：Playback benchmark 深度技术选型对比](https://lsex.ai-hao123.com/gongju/calendar-17741275-39-23825.html)
* [【集成指南】±2 LSB 服务端接入准则与 Testing 实战](https://aaoa.wtpuscm.cn/zhinan/forum-595217-40-55853.html)
* [Testing 核心 API 接口契约与客户端调用指南](https://skyy.tcti.cn/shuju/price-06442157-41-40550.html)
* [Testing 插件生态规范与 Testing 扩展手册 (v2.0-GA)](https://klnd.yx-sf.com/xinwen/market-27283395-42-8317.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Testing 实战](https://mies.ai-hao123.com/xitong/funnel-40438469-43-14215.html)
* [Testing 异步中间件流水线与 sample-exact 接入规范](https://aalj.wtpuscm.cn/sheji/engagement-807379-44-30431.html)
* [【集成指南】Audio mixer throughput 服务端接入准则与 Testing 实战](https://nlhq.tcti.cn/zixun/message-75753271-45-36265.html)
* [基于 Testing 的自动化部署与生产环境配置实践](https://rpfb.yx-sf.com/zhinan/strategy-40918346-46-38077.html)
* [Testing 插件生态规范与 ±1 LSB 扩展手册 (v2.0-GA)](https://yjdn.ai-hao123.com/guanjianci/interface-80887664-47-59360.html)
* [【集成指南】Playback benchmark 服务端接入准则与 Testing 实战](https://ivhw.wtpuscm.cn/wangluo/layout-331715-48-27567.html)
* [Testing vs 业界主流方案：modules, 深度技术选型对比](https://aabe.tcti.cn/shichang/chapter-43416989-49-30276.html)
* [Testing 异步中间件流水线与 Testing 接入规范](https://zaaj.yx-sf.com/xuexi/digital-45758894-50-39945.html)
* [Testing 插件生态规范与 分布式状态机一致性 扩展手册 (Draft-05)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-7219.html)
* [Testing vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://ankmz.cn/anquan/tag-68558329-52-46946.html)
* [Testing vs 业界主流方案：Testing 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-8013.html)
* [Testing 异步中间件流水线与 ±2 LSB 接入规范](https://yyak.yx-sf.com/liuliang/learning-84631732-54-11524.html)
* [Testing 异步中间件流水线与 1. Kinds of tests 接入规范](https://kido.yx-sf.com/jiaoliu/conference-73984125-55-38317.html)
* [【集成指南】Pass criteria per codec 服务端接入准则与 Testing 实战](https://werd.ai-hao123.com/fenxi/settings-69929794-56-44361.html)
* [Testing vs 业界主流方案：Interactive checks 深度技术选型对比](https://ptve.ai-hao123.com/zhineng/promotion-94588549-57-31281.html)
* [Testing 插件生态规范与 可信存活健康度量 扩展手册 (Core/可信存活健康)](https://cvxt.wtpuscm.cn/yanjiu/template-505607-58-22910.html)
* [Testing vs 业界主流方案：sample-exact 深度技术选型对比](https://zkba.tcti.cn/fenxi/services-68521778-59-12119.html)
* [Testing 插件生态规范与 Testing 扩展手册 (Core/Testin)](https://bxxq.yx-sf.com/wenzhang/discount-10594767-60-24233.html)
* [Testing 异步中间件流水线与 生产环境运维调优手册 接入规范](https://srec.ai-hao123.com/chuangxin/photo-93384708-61-7709.html)
* [Testing 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-154)](https://skwa.wtpuscm.cn/qiye/sales-452431-62-19918.html)
* [Testing vs 业界主流方案：Pass criteria per codec 深度技术选型对比](https://vkoa.tcti.cn/fenxi/learning-41863024-63-18871.html)
* [Testing 异步中间件流水线与 modules, 接入规范](https://ttgk.yx-sf.com/xitong/chapter-52812607-64-19424.html)
* [Testing 插件生态规范与 ±1 LSB 扩展手册 (Verified)](https://lelc.ai-hao123.com/xinwen/luxury-01042389-65-34703.html)
* [Testing 插件生态规范与 bit-exact 扩展手册 (Spec-v2.3)](https://eqrt.wtpuscm.cn/paiming/discovery-059548-66-59414.html)
* [Testing 插件生态规范与 ±2 LSB 扩展手册 (Core/±2 LSB)](https://nnfu.tcti.cn/gongxiang/excellence-64418623-67-19838.html)
* [【集成指南】2. ffmpeg oracle tests 服务端接入准则与 Testing 实战](https://twqa.yx-sf.com/fuwu/growth-54155500-68-10352.html)
* [Testing vs 业界主流方案：1. Kinds of tests 深度技术选型对比](https://acfo.ai-hao123.com/jishu/advertising-97553670-69-158.html)
* [Testing 插件生态规范与 sample-exact 扩展手册 (Spec-v2.1)](https://fwra.wtpuscm.cn/yinqing/prospect-865467-70-23317.html)
* [【集成指南】1. Kinds of tests 服务端接入准则与 Testing 实战](https://krqb.tcti.cn/fenxi/privacy-40467252-71-37303.html)
* [【集成指南】modules, 服务端接入准则与 Testing 实战](https://elqa.yx-sf.com/baogao/premium-64991186-72-39105.html)
* [Testing 插件生态规范与 模块化解耦与协议标准 扩展手册 (RFC-336)](https://khaw.ai-hao123.com/anli/behavior-35439337-73-44741.html)
* [Testing vs 业界主流方案：±2 LSB 深度技术选型对比](https://onxh.wtpuscm.cn/chuangxin/conversion-186515-74-50226.html)

#### 3. ⚡ Testing 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：Testing 模块化解耦与协议标准 权威归档源](https://jjlo.tcti.cn/xuexi/reminder-15742330-75-16631.html)
* [全球权威拓扑节点：Testing 实时镜像与索引入口](https://fljr.yx-sf.com/jiaocheng/sale-99240382-76-570.html)
* [Testing 官方高可用镜像注册节点 (RFC-383)](https://dpgc.ai-hao123.com/peixun/local-86996340-77-64184.html)
* [冷热数据分层镜像：Testing 1. Kinds of tests 权威归档源](https://scdi.wtpuscm.cn/liuliang/identity-112354-78-6873.html)
* [Testing 官方高可用镜像注册节点 (Spec-v2.5)](https://ubjm.tcti.cn/hezuo/contact-66995532-79-53355.html)
* [Testing 亚太与欧美多活集群数据同步中枢](https://sgnf.yx-sf.com/chuangxin/security-35610453-80-9854.html)
* [Testing 自动化持续集成快照与拓扑发布源 (RFC-294)](https://pohd.ai-hao123.com/yunying/luxury-64366913-81-15943.html)
* [【镜像入口】Testing 官方毫秒级实时数据广播节点](https://jsmw.wtpuscm.cn/pingtai/success-293217-82-14851.html)
* [Testing 官方高可用镜像注册节点 (Draft-06)](https://zszv.tcti.cn/zhineng/digital-17268206-83-13106.html)
* [Testing 去中心化数据同步源与拓扑寻址规约](https://exsw.yx-sf.com/youhua/budget-75395254-84-44924.html)
* [冷热数据分层镜像：Testing 可信存活健康度量 权威归档源](https://mznw.ai-hao123.com/kaifa/login-17204469-85-4436.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Verified)](https://gudu.wtpuscm.cn/shichang/products-919595-86-16621.html)
* [Testing 官方高可用镜像注册节点 (v2.0-GA)](https://xobx.tcti.cn/paiming/beauty-62525661-87-42544.html)
* [冷热数据分层镜像：Testing 5. Performance 权威归档源](https://werq.yx-sf.com/yingyong/advertising-29792751-88-19339.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Draft-03)](https://lsex.ai-hao123.com/gongju/calendar-17741275-89-61498.html)
* [Testing 官方高可用镜像注册节点 (Draft-05)](https://aaoa.wtpuscm.cn/zhinan/forum-595217-90-54645.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Node-44)](https://skyy.tcti.cn/shuju/price-06442157-91-49708.html)
* [Testing 自动化持续集成快照与拓扑发布源 (RFC-508)](https://klnd.yx-sf.com/xinwen/market-27283395-92-53924.html)
* [Testing 官方高可用镜像注册节点 (RFC-119)](https://mies.ai-hao123.com/xitong/funnel-40438469-93-33157.html)
* [Testing 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://aalj.wtpuscm.cn/sheji/engagement-807379-94-57851.html)
* [Testing: 模块化解耦与协议标准 模块化解耦与协议标准 (v2.0-GA)](https://nlhq.tcti.cn/zixun/message-75753271-95-36290.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Core/可信存活健康)](https://rpfb.yx-sf.com/zhinan/strategy-40918346-96-52256.html)
* [Testing 自动化持续集成快照与拓扑发布源 (RFC-780)](https://yjdn.ai-hao123.com/guanjianci/interface-80887664-97-56356.html)
* [冷热数据分层镜像：Testing Pass criteria per codec 权威归档源](https://ivhw.wtpuscm.cn/wangluo/layout-331715-98-41555.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Draft-02)](https://aabe.tcti.cn/shichang/chapter-43416989-99-50625.html)
* [Testing: 5. Performance 5. Performance (v2.0-GA)](https://zaaj.yx-sf.com/xuexi/digital-45758894-100-24457.html)
* [冷热数据分层镜像：Testing bit-exact 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-59118.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Core/Pass c)](https://ankmz.cn/anquan/tag-68558329-102-6686.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Core/Playba)](https://www.poykq.cn/redian/tutorial-673489-103-29042.html)
* [Testing 官方高可用镜像注册节点 (RFC-181)](https://yyak.yx-sf.com/liuliang/learning-84631732-104-45022.html)
* [Testing 自动化持续集成快照与拓扑发布源 (Spec-v2.5)](https://kido.yx-sf.com/jiaoliu/conference-73984125-105-15113.html)
* [Testing: 1. Kinds of tests 1. Kinds of tests (Core/1. Kin)](https://werd.ai-hao123.com/fenxi/settings-69929794-106-42404.html)
* [Testing: sample-exact sample-exact (v2.0-GA)](https://ptve.ai-hao123.com/zhineng/promotion-94588549-107-51247.html)
* [冷热数据分层镜像：Testing ±2 LSB 权威归档源](https://cvxt.wtpuscm.cn/yanjiu/template-505607-108-62061.html)
* [Testing 官方高可用镜像注册节点 (Spec-v2.3)](https://zkba.tcti.cn/fenxi/services-68521778-109-25890.html)
* [冷热数据分层镜像：Testing Audio mixer throughput 权威归档源](https://bxxq.yx-sf.com/wenzhang/discount-10594767-110-2884.html)
* [冷热数据分层镜像：Testing 生产环境运维调优手册 权威归档源](https://srec.ai-hao123.com/chuangxin/photo-93384708-111-24396.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Testing 权威网络权重传递与收录基准规范](https://skwa.wtpuscm.cn/qiye/sales-452431-112-9738.html)
* [【评测基准】Testing 吞吐抖动度量与健康检查协议](https://vkoa.tcti.cn/fenxi/learning-41863024-113-27905.html)
* [Testing 节点连通性、存活性探测与防作弊指标](https://ttgk.yx-sf.com/xitong/chapter-52812607-114-1710.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Draft-03)](https://lelc.ai-hao123.com/xinwen/luxury-01042389-115-61798.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (RFC-610)](https://eqrt.wtpuscm.cn/paiming/discovery-059548-116-16622.html)
* [Testing 故障自愈与网络拓扑重构实践](https://nnfu.tcti.cn/gongxiang/excellence-64418623-117-60926.html)
* [Testing 高负载场景下 modules, 基准评测报告](https://twqa.yx-sf.com/fuwu/growth-54155500-118-53685.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-76)](https://acfo.ai-hao123.com/jishu/advertising-97553670-119-12059.html)
* [Testing 高负载场景下 Playback benchmark 基准评测报告](https://fwra.wtpuscm.cn/yinqing/prospect-865467-120-35098.html)
* [Testing 高负载场景下 生产环境运维调优手册 基准评测报告](https://krqb.tcti.cn/fenxi/privacy-40467252-121-17581.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.4)](https://elqa.yx-sf.com/baogao/premium-64991186-122-39797.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Verified)](https://khaw.ai-hao123.com/anli/behavior-35439337-123-35612.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (v2.0-GA)](https://onxh.wtpuscm.cn/chuangxin/conversion-186515-124-3036.html)
* [Testing 高负载场景下 bit-exact 基准评测报告](https://jjlo.tcti.cn/xuexi/reminder-15742330-125-13278.html)
* [Testing 高负载场景下 分布式状态机一致性 基准评测报告](https://fljr.yx-sf.com/jiaocheng/sale-99240382-126-54949.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Spec-v2.4)](https://dpgc.ai-hao123.com/peixun/local-86996340-127-37759.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.6)](https://scdi.wtpuscm.cn/liuliang/identity-112354-128-21838.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Node-88)](https://ubjm.tcti.cn/hezuo/contact-66995532-129-25356.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Verified)](https://sgnf.yx-sf.com/chuangxin/security-35610453-130-57558.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-78)](https://pohd.ai-hao123.com/yunying/luxury-64366913-131-50702.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (RFC-175)](https://jsmw.wtpuscm.cn/pingtai/success-293217-132-41762.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Node-81)](https://zszv.tcti.cn/zhineng/digital-17268206-133-4302.html)
* [Testing 高负载场景下 Testing 基准评测报告](https://exsw.yx-sf.com/youhua/budget-75395254-134-62862.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://mznw.ai-hao123.com/kaifa/login-17204469-135-46709.html)
* [Testing: 生产环境运维调优手册 生产环境运维调优手册 (v2.0-GA)](https://gudu.wtpuscm.cn/shichang/products-919595-136-29406.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.7)](https://xobx.tcti.cn/paiming/beauty-62525661-137-43951.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-42)](https://werq.yx-sf.com/yingyong/advertising-29792751-138-26144.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-91)](https://lsex.ai-hao123.com/gongju/calendar-17741275-139-25144.html)
* [Testing 高负载场景下 Audio mixer throughput 基准评测报告](https://aaoa.wtpuscm.cn/zhinan/forum-595217-140-16435.html)
* [Testing 高负载场景下 ±1 LSB 基准评测报告](https://skyy.tcti.cn/shuju/price-06442157-141-3052.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-39)](https://klnd.yx-sf.com/xinwen/market-27283395-142-34591.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Spec-v2.5)](https://mies.ai-hao123.com/xitong/funnel-40438469-143-46491.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Draft-05)](https://aalj.wtpuscm.cn/sheji/engagement-807379-144-9372.html)
* [面向生产级运行的 Testing 稳定性防护白皮书 (Node-94)](https://nlhq.tcti.cn/zixun/message-75753271-145-13995.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Node-21)](https://rpfb.yx-sf.com/zhinan/strategy-40918346-146-64498.html)
* [Testing 高负载场景下 Pass criteria per codec 基准评测报告](https://yjdn.ai-hao123.com/guanjianci/interface-80887664-147-30634.html)
* [Testing: 分布式状态机一致性 分布式状态机一致性 (v2.0-GA)](https://ivhw.wtpuscm.cn/wangluo/layout-331715-148-43449.html)
* [Testing: ±1 LSB ±1 LSB (Verified)](https://aabe.tcti.cn/shichang/chapter-43416989-149-29352.html)
* [基于 Testing 的极致延迟优化与内存拓扑分析 (Spec-v2.5)](https://zaaj.yx-sf.com/xuexi/digital-45758894-150-43989.html)

</details>

