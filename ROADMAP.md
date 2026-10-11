# FilmCraft Roadmap

- UI.6: Compact Source and Program monitors separate timecodes from view settings and retain Source drag choices. Transport controls use the shared toolbar sizing helper, with overflow actions in an operable More menu and a scrolling fallback at tiny widths.
- UI.3: Panel tabs, edge splits, tab reordering and floating groups use the shared docking model and drop resolver. Floating geometry is saved with named workspaces while legacy layout JSON remains readable. Dock dividers use bounded shared geometry and keyboard-accessible resize handles. Tools and dock tabs expose actual widget focus, paint focus outlines, and support keyboard traversal/activation. Control input reaches focused widgets in background windows; optional input isolation supports unattended validation. Existing workspace trees and saved layout formats are retained.

**Stage: alpha** · next: beta, ~19 points of ready-for-real-work and ~400–700 h away, plus opening Premiere's `.prproj` ([why](docs/target-app-parity.md#stage); passes the [core-workflow gate](docs/roadmap.md#alpha-gate))

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (full re-measure against Premiere Pro 2026 26.5.2; scorecard, gaps, milestones and localization moved into docs/ per the craftrules progress-docs standard) · **Target:** Adobe Premiere Pro 2026 (26.5.2)

Progress toward parity with Adobe Premiere Pro. This page is the summary; the detail is in
[docs/target-app-parity.md](docs/target-app-parity.md) (assessment, methodology, calibration),
[docs/gaps.md](docs/gaps.md) (the ranked work list: **agents, read it before choosing work**) and
[docs/roadmap.md](docs/roadmap.md) (milestones and current focus).

## Headline numbers

| | Value | How |
|---|---|---|
| **Feature breadth** | **~86%** | Does each Premiere feature exist? Menus 92% (measured: label diff against Premiere 26.5.2's menu dump), effects / transitions / audio effects 100% (measured by tests), other areas estimated |
| **Ready for real work** | **~56%** (51–61%) | How close FilmCraft is to replacing Premiere for an editor on real projects: depth, correctness, file compatibility, speed, stability, ecosystem. Estimated depth per area, combined by a [written formula](docs/target-app-parity.md#three-readiness-numbers) |
| **Mainstream practitioner** | **~55%** | A typical professional editor's weekly work only, discounted ×0.90 interaction, ×0.92 stability, ×0.88 file exchange with Premiere users ([method](docs/target-app-parity.md#mainstream-practitioner)) |
| **Essentials user** | **~60%** | Import, cut, title, music, export at default settings, discounted ×0.88 launch / stability, ×0.90 discoverability, ×0.95 opening files people send ([method](docs/target-app-parity.md#essentials-user)) |
| **To beta** | **~400–700 Opus 5.5 agent-hours** | Estimated, calibrated from this repo's PR history |
| **To full parity** | **~1,550–2,750 agent-hours** | ~70% parallelizes; plugins, AI models, native-speaker review and some hardware need a human |
| Code (measured) | 45 crates, ~343k lines of Rust, 2,520 tests, 268 merged PRs, 69 authors | `git`, `gh`, `grep` |

Two numbers, because they answer different questions. The gap between them is the work that
matters most now: breadth is nearly done, depth is not.

### Readiness by audience

| Audience | Ready % | Opus 5.5 agent wall-clock hours to ~95% | Work that dominates |
|---|---|---|---|
| Full target (ready for real work) | ~56% | ~1,450–2,600 (~70% parallelizes) | Plugin hosting, AI models, localization incl. RTL / Devanagari, camera RAW and delivery codecs, `.prproj`, hardware acceleration, UI fidelity |
| Mainstream practitioner | ~55% | ~500–850 (~65% parallelizes) | `.prproj` import (60–120), UI fidelity (120–200), stability and PR CI (40–80), 10-bit / 4:2:2 camera decode (30–50), playback / hardware (60–100), import and export gaps (60–100) |
| Essentials user | ~60% | ~180–350 (~60% parallelizes) | Launch crashes on Windows / Intel / older macOS (40–80), discoverability and timeline friction (60–110), playback lag (30–60), common import gaps (20–40), audio devices (15–30) |

Method and calibration: [docs/target-app-parity.md](docs/target-app-parity.md#three-readiness-numbers).

## By dimension

| Dimension | Ready | Hours | Doc |
|---|---|---|---|
| Features (editing, effects, colour, audio, graphics, export settings) | ~66% | 350–620 | [target-app-parity.md](docs/target-app-parity.md#by-feature-area) |
| UI / UX fidelity | ~60% | 120–200 | [ui-parity.md](docs/ui-parity.md) |
| File formats (containers, `.prproj`, interchange, captions, stills) | ~60% | 140–240 | [file-format-parity.md](docs/file-format-parity.md) |
| Codecs | ~60% | 150–250 | [codec-parity.md](docs/codec-parity.md) |
| Hardware acceleration | ~45% | 120–200 | [hardware-parity.md](docs/hardware-parity.md) |
| Localization | ~35% | 150–230 | [localization-parity.md](docs/localization-parity.md) |
| Performance | ~55% | 60–120 | [performance.md](docs/performance.md) |
| Stability | ~45% | 60–120 | [gaps.md G4](docs/gaps.md#g4-crashes-at-start-up-and-no-test-ci-on-pull-requests) |
| Platforms (macOS, Windows, Linux, FreeBSD, web) | ~60% | 50–90 | [gaps.md G9](docs/gaps.md#g9-windows-and-linux-at-run-time) |
| Plugins and ecosystem | ~5% | 200–400 | [gaps.md G11](docs/gaps.md#g11-no-plugin-hosting) |
| AI features | ~25% | 150–300 | [gaps.md G12](docs/gaps.md#g12-ai-features) |

## Features

| Area | Breadth | Ready | Hours |
|---|---|---|---|
| Editing, timeline, trimming, multicam | ~90% | ~70% | 80–140 |
| Effects and transitions | 100% (measured) | ~60% | 80–140 |
| Media import and codecs | ~75% | ~60% | 150–250 |
| Audio | ~88% | ~65% | 50–90 |
| Panels and UI fidelity | ~88% | ~60% | 120–200 |
| Colour | ~90% | ~70% | 40–70 |
| Graphics, titles, captions | ~85% | ~65% | 50–90 |
| Export and delivery | ~80% | ~65% | 60–110 |
| Project files and interchange | ~70% | ~45% | 90–160 |
| Preferences and project management | ~80% | ~65% | 20–40 |

Evidence for every row: [docs/target-app-parity.md](docs/target-app-parity.md#by-feature-area).

## Languages

Premiere ships 10 interface languages. Measured from our catalogs (3,498 strings):

| Language | Status | Strings |
|---|---|---|
| English | full | 100% |
| Simplified Chinese | partial (whole interface; engine errors English) | 99.6% |
| Spanish | partial (whole interface) | 99.9% |
| Hindi | none (no Devanagari shaping in the interface) | 0% |
| Arabic | none (no RTL interface) | 0% |
| French | partial (whole interface; native review welcome) | ~100% |
| Portuguese (Brazil) | menus only | 9.9% |
| Indonesian | none | 0% |
| Japanese | partial (whole interface) | 99.7% |
| German | none | 0% |
| Korean | none | 0% |
| Vietnamese | none | 0% |

Also shipped: Ukrainian (menus only, 10%). Detail: [docs/localization-parity.md](docs/localization-parity.md).

## Upcoming

Ranked; detail and "done when" in [docs/roadmap.md](docs/roadmap.md) and [docs/gaps.md](docs/gaps.md).

1. **No launch crashes, and CI on pull requests** (macOS 12, Intel UHD, Windows 11 reports; no workflow runs the tests today). 40–80 h.
2. **Open Premiere's `.prproj`**, the beta blocker. 60–120 h.
3. **10-bit / 4:2:2 camera media without hardware** (H.264 High 10 / 4:2:2, HEVC 4:2:2; #626). 30–50 h.
4. **Measure fidelity against Premiere** and add `cargo xtask parity`. 50–90 h.
5. **Real-world media corpus and workflow acceptance tests.** 40–70 h.
6. **Hardware acceleration, continued** (#30): zero-copy upload, more effects on the GPU, Quick Sync / AMD encoders, VideoToolbox Main 10. 120–200 h.
7. **UI fidelity:** Program monitor transform handles, Effect Controls, docking. 120–200 h.
8. **Plugin hosting** (VST3 / AU, then OpenFX). Owner decision needed. 200–400 h.

## Progress log

- **2026-10-10 (changed media):** Settings ▸ Media ▸ Automatically refresh growing files works: media files that change on disk while the project is open (a recording still being written, a render written over the file a clip uses) are read again every *Refresh growing files every* seconds and as soon as the window comes back to the front: the new length, picture and identity, without an undo step (`media.refreshChanged`, `media_watch`).
- **2026-10-10 (re-measure):** full parity re-measure against Premiere Pro 26.5.2: menu label diff 338 / 366 (92%), installed bundle listings (10 UI languages, exporter and codec frameworks), 188 open issues. Breadth ~86% (file formats now counted), ready ~55%, stage **alpha**; progress docs reorganized per the craftrules standard ([target-app-parity](docs/target-app-parity.md), [gaps](docs/gaps.md), [roadmap](docs/roadmap.md), [codec](docs/codec-parity.md), [file-format](docs/file-format-parity.md), [hardware](docs/hardware-parity.md), [UI](docs/ui-parity.md), [localization](docs/localization-parity.md)).
- **2026-10-10:** ~75 community PRs merged: GPU export rendering (Auto) with Vignette, Video Limiter and Lumetri basic / creative / vignette on the GPU (#421); Linux NVDEC H.264 / HEVC decode (#549); text-to-speech narrations with Kokoro-82M (#278); Japanese for the whole interface (#449) and Ukrainian menus (#407); ALSA default-device fallback (#254, fixes #23); drop import into the timeline (#517); system appearance (#505); timecode scrub editing (#576); NVENC RGBA input (#486); marker CSV export (#454); many fixes (#526–#575).
- **2026-10-09:** release 0.5.0. NVENC H.265 export on Windows, Main and Main 10 HDR (#328, #360); NVENC H.264 export (#386); VA-API H.264 / HEVC decode (#358); Source monitor playback (#423); multi-stream container audio, schema v13 (#361); New Sequence and Sequence Settings dialogs (#269, #355); Spanish for the panels (#238); background decoders and memory stress tests (#385, #381).
- **2026-10-08 (Source range dragging):** implicit full-clip In/Out, video-only/audio-only/linked both controls, picture/waveform drag gestures and one-command undoable timeline placement. Shared Source/Program draggable frame-bounded handles and range translation, cancellation, Source marker/navigation routing and marked-range playback/looping.
- **2026-10-08 (Source playback):** normal forward playback for video/audio media and subclips in the Source monitor, an independent Source clock, Play/Pause and focused Space, frame stepping, device-clock audio with wall-clock fallback, seeking, and end-of-clip stopping. Windows desktop review verified a real video file and standalone WAV, including In/Out marking. Reverse/shuttle Source playback and source sequences remain unsupported; no parity percentage changed.
- **2026-10-08 (night):** HDR H.265 (HEVC Main 10, PQ and HLG, BT.2020, limited range) export on Windows through NVENC (#30): an HDR sequence now exports as real 10-bit HDR (`rgbf_to_yuv420_10` into P010 input buffers; VUI BT.2020 + PQ / HLG; the HDR10 mastering-display and content-light messages as SEI on every IDR for PQ, through the generic SEI payload array because NVENC 12.1 has no mastering-display field; `colr` / `mdcv` / `clli` in the sample entry), where the GPU reports 10-bit HEVC (new `register_hdr_probe` / `hdr_available` in the export crate; elsewhere, and on macOS, HDR sequences still tone-map to 8-bit SDR HEVC, and `settings.sdr` forces that). ffprobe reads `hevc` / `Main 10` / `hvc1` / `yuv420p10le` / `bt2020` / `smpte2084` or `arib-std-b67` / `bt2020nc` / `tv` with the mastering-display and content-light side data for PQ; ffmpeg's decode equals ours code for code; the in-process round trip is at 71.1 dB luma PSNR on the 10-bit scale with 803 distinct levels on a ramp and the peak code 940 intact (RTX 5060). 8-bit SDR H.265 is unchanged.
- **2026-10-08:** H.265 (HEVC Main, 8-bit 4:2:0, SDR) export on Windows through NVENC (#30): the NVENC session, ring and NV12 path of the H.264 encoder now serve both codecs (the H.264 export is byte-for-byte unchanged); the format is available where the GPU has an HEVC encoder, choosing it is the opt-in, and `hvcC` is built from the SPS the encoder wrote. ffprobe reads `hevc` / `Main` / `hvc1` / `yuv420p` / BT.709; our own HEVC decoder decodes it at 48.9 dB luma PSNR (RTX 5060). Main 10 and HDR remain.
- **2026-10-08 (#285):** `filmcraft-cli` no longer panics with "Broken pipe" when its stdout is closed early (`filmcraft-cli commands | head`): later output is dropped, the rest of the work (script lines, `--save`, `--save-as`) still runs, and the exit status still reports failures. Other stdout write errors are reported and make the exit status 1. No change to the checklist or estimates.

- **2026-10-06:** 21 community PRs landed (#19, #28, #38, #39, #41–#60). Highlights: hardware decoding is now actually on in the desktop app (#41: `register()` sat inside a `log::info!` that never ran, so #33's VideoToolbox path only worked in the CLI and benchmarks); decode memory on long timelines cut sharply (frame pool, shared GOP-cache budget, positional file reads); exports are byte-identical across machines (#19); crash fix for odd track names over MCP (#44); ripple trim, ripple delete and speed changes move split-edit clips once (#56–#58); Japanese interface using an installed system font (#38; no fonts bundled); MCP annotations, resources, export progress and cancellation (#28).

- **2026-10-05:** honest assessment added (checklist ~87% vs ready for real work ~50–60%). Hardware acceleration started (#30): blend modes on the GPU (#32); VideoToolbox decode + `platform` crate, the one crate allowed `unsafe` (#33). Community fixes landed: AIFF import, export start/end ranges, transitions following ripple trims, Slide moving linked audio (#5–#8).

- **2026-10-04 (later):** interchange + MXF export (M11.4/M6.6): own compound-file container (`crates/cfb`), AAF Edit Protocol import/export (embedded/linked/consolidated audio, video mixdown, handles, breakout to mono), OMF 2.0 export (Bento), MXF OP1a/OP-Atom writer (DNxHR/ProRes/H.264/PCM) as export formats. 1605 tests.

- **2026-10-04:** MPEG-2 (M9.15): own MPEG-2/MPEG-1 video decoder (4:2:2, interlaced, field pictures; IEEE 1180 IDCT), MPEG TS/PS demux (AVCHD .mts, .m2ts, .ts, .mpg/.vob/.mod), own AC-3 decoder (ATSC A/52), MP2, LATM AAC, Blu-ray/DVD LPCM, MPEG-2 in MXF/MOV/MP4/MKV. 1562 tests.

- **2026-10-03 (evening):** Project panel and Media Browser (M12.7): Freeform view (stacks, arrangements), Icon view hover scrub with In/Out, list columns from all metadata, ten view presets, multiple Project panels; Media Browser with Favorites/drives/recent, back/forward, filters, thumbnails, Edit Columns, ingest, drag to timeline; all as `project.*` / `mediaBrowser.*` commands. 1513 tests.

- **2026-10-03 (later):** graphics & captions (M10.7): `.fcgt` graphics templates (export/install/apply/edit, 8 original built-ins; Adobe .mogrt refused), Essential Graphics Browse/Edit, rolls/crawls, responsive design pins and time, per-character styles, Upgrade Caption to Graphic / to Source Graphic, MCC/EBU STL/TTML/DFXP, Replace Fonts; schema v12. 1479 tests.
- **2026-10-03:** 4K H.264 playback (M4.9): vectorised deblocking, branch-free CABAC bins and column-parallel inverse transforms cut H.264 decoding by 23 % (cycles per frame, bit-exact; 4K cold seeks −27 % CPU); opt-in draft decoding (Settings ▸ Playback) skips deblocking of non-reference pictures and uploads decimated planes while playing at 1/2–1/4, never for paused frames or exports; conformance runs 1, 3 and all threads. 4K plays 192/0 at load 100–140 (before: only at load ~90).
- **2026-10-03:** 4K VP9 / HEVC / AV1 playback (M4.10): VP9 frame threading (references published per superblock row, loop filter overlapping the next frames) plus a vectorised loop filter, column-parallel transforms and 16-bit MC cut VP9 work by 25 % single-threaded and 44 % with all cores (4K decode 8 → 27–34 fps at load ~200); HEVC −28 % (vectorised SAO, MC without per-block zeroing, batched bypass bins), AV1 −20 % (fixed-width MC, recycled planes); opt-in draft decoding for all three; bit-exact at 1, 3 and all threads. 4K VP9 and HEVC now play 192/0 at load ~200 (before: 0–58 of 192 shown).

- **2026-10-03:** keyboard parity (M3.12: 143 new commands on Premiere's default keys, table validated against the registry), audio (M7.8: 5.1 tracks/submixes/Mix, 5.1 panner, BS.775 downmix, 5.1 WAV/MOV/AAC export, voice-over record with punch-in, Remix, Clip Mixer Latch/Touch/Write). 1432 tests.

- **2026-10-02 (late):** export parity (M6.5): built-in preset library + user presets, full Export-mode settings (loudness normalization, image/name/timecode overlays, captions, range), export queue, Quick Export, Premiere-style image-sequence export, `filmcraft-cli export --preset`; media import (M9.14): MXF OP1a/OP-Atom (new `crates/mxf`, H.264/DNx/ProRes/PCM, timecode), Ogg Opus (new `crates/ogg`), image sequences as clips, BWF timecode. 1373 tests.

- **2026-10-02 (panels):** complete Lumetri Scopes (new `filmcraft-scopes` crate: Vectorscope YUV/HLS, Histogram, Parade RGB/YUV/RGB-White, Waveform RGB/Luma/YC/YC no Chroma; wrench menu with presets, colour space, brightness, scale; < 1 ms per scope at 1080p; `scopes.read` returns the numbers); Metadata panel (editable log fields, undoable `metadata.set`), Timecode panel, Events panel (event log of failed commands, jobs, auto-save), Progress panel (jobs with Cancel), Reference Monitor (parked or ganged, picture or scopes).
- **2026-10-02 (evening):** performance (M4.8): `cargo xtask bench` (decode, seek, playback, scrub, timeline UI, export, project I/O, memory; results in docs/performance.md) and `perf.stats` for agents; HEVC transform/SAO 2.5× cheaper; late frames skip non-reference pictures (4K CPU per displayed frame halved, cold seeks 20–45% faster); decoders stay bit-exact. 1270 tests.

- **2026-10-02 (later):** all 93 video effects (+Legacy/Obsolete bins), all 53 audio effects with Parametric/Graphic EQ, Multiband Compressor and Dynamics editor windows, all 84 video transitions (+21 Legacy), Settings dialog (16 categories, most wired), remaining menu items (Scene Edit Detection, Find, Normalize Mix Track, Simplify Sequence, Automate to Sequence, search bins, templates, Project Settings with scratch disks, ALE and selection-as-project export, system report); project schema v11. 1258 tests.

- **2026-10-02:** Premiere menu long tail: Sequence/Markers (gaps, split edits, through edits, subsequence, Delete Tracks, range/chapter markers, ripple sequence markers), Clip/Edit/File (Paste/Remove Attributes, subclips, Frame Hold Options, Time Interpolation with frame blending, Audio Channels, Breakout to Mono, Extract Audio, Replace With Clip, Remove Unused, Consolidate Duplicates), View/monitors (paused resolution, alpha/RGB display modes, comparison view, magnification, rulers, guides + templates, snapping), Graphics and Titles (vertical text, shape layers, align/distribute/arrange). Transcripts and text-based editing (Text panel, optional local Whisper). AV1 tile/frame/post-filter threading. Agent-friendly CLI (`exec`, `inspect`, `describe`, `import`, `export`, `run -`, `--save`, `--bridge`). Project schema v10.

- **2026-10-01 (night):** new README hero (Apollo 11 documentary edit, NASA public domain); agents never steal keyboard focus; masks + tracking, adjustment layers, effect presets; multicam + audio sync; DNxHD/DNxHR decode/encode; AV1 decoder bit-exact against libdav1d (all stages, film grain, superres, SVC).

- **2026-10-01 (M12 multicam):** Synchronize / Merge Clips / Create Multi-Camera Source Sequence (sync by In, Out, timecode ± hours, clip markers, or audio: GCC-PHAT cross-correlation, offsets recovered to the sample on noisy multi-mic recordings), multi-camera clips (angle render, Switch Audio), Program monitor Multi-Camera view with live switching (keys 1–9, one undo step per pass), Ctrl+1–9 cuts, Enable / Flatten / Edit Cameras, project schema v7. Fixed: nested sequence audio was silent.
- **2026-10-01 (evening):** M11 done: offline media with our own slate and the Link Media dialog (fingerprint-checked relink, folder remap, search), proxies (create in background / attach / toggle; export stays full-res) with ingest settings, Project Manager (collect, consolidate + transcode). Fixed: GOP cache deadlock when a rayon decoder inside a parallel export re-entered the same source.
- **2026-10-01 (M7.4):** Essential Sound panel: Dialogue/Music/SFX/Ambience types, Loudness auto-match (BS.1770, exact to the target), Repair (noise, rumble, hum, new DeEsser and spectral DeReverb), Clarity (dynamics, EQ presets, Enhance Speech DSP chain), Creative reverb / stereo width, ducking that writes Volume keyframes, presets; all `essentialSound.*` commands, effects visible and keyframable in Effect Controls.
- **2026-10-01 (night):** M8 colour: camera log curves and gamuts from published specs, colour-managed pipeline (PQ/HLG working spaces, Interpret Footage, BT.2390 tone mapping), HDR export signalling verified with ffprobe, LUTs (.cube/.3dl, tetrahedral CPU + WGSL, project library, Lumetri Input LUT / Look), Colour Match, Lumetri section bypass, HDR scopes.

- **2026-10-01 (night):** DNxHD/DNxHR decoder + encoder (SMPTE ST 2019-1) with MOV export; AV1 decoder (spec v1.0.0 + Errata 1, all decoding tools incl. film grain) bit-exact against libdav1d, wired into MP4/WebM/MKV import with key-frame-checked seeking.
- **2026-10-01 (later):** text engine (shaping, bidi, line breaking) + Type/Shape/Pen tools + graphic clips + graphics panel; Trim Monitor + dynamic trimming; Keyboard Shortcuts editor; audio mixer with automation. Fixed: MP4 muxer wrote unreadable all-empty sample tables; system font scan race.

- **2026-10-01:** Audio mixing (M7.2/M7.5/M7.6 basics): mixer graph with submixes, sends, inserts and latency compensation; Audio Track / Clip Mixer panels; Latch/Touch/Write automation recorded live; timeline track keyframes; Audio Gain dialog.
- **2026-10-01:** recovered from a machine crash with no lost work. Merged: VP9 decoder (profiles 0–3, bit-exact, WebM/MKV/MP4), Opus decoder (RFC 8251 range-exact; WebM/MKV/MP4), captions (SRT/VTT/SCC, burn-in), render bar + render previews, project schema versioning + atomic saves + auto-save + crash-recovery journal, test infrastructure (golden images, loudness oracle vs ffmpeg, headless scripted UI tests; fixed a GPU stale-texture bug), public contributor docs and licence files.

- **2026-09-30 (evening):** Opus decoder (RFC 6716/8251, range-exact on every conformance vector, ~80–110× realtime 48 kHz stereo) wired into WebM/MKV and MP4 import.
- **2026-09-30 (afternoon):** Matroska/WebM import; LUFS meters; clip audio effects on the DSP crate.
- **2026-09-30 (midday):** HEVC decoder, Matroska/WebM demuxer and audio DSP merged; HEVC import wired; asset rules (AGENTS.md, ATTRIBUTION.md, `cargo xtask assets`); README with hero screenshot and the Craft family.
- **2026-09-30 (late morning):** keyframe value/velocity graphs; xtask gates; H.264 MP4 export; Lumetri curves, wheels, looks, HSL secondary.
- **2026-09-30 (early morning):** H.264 decoder, ProRes, AAC; GPU compositor; MCP; Premiere 26 visual fidelity pass.

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | minor | Readiness-by-audience table with hours to ~95% per audience |
| 2026-10-10 | minor | Headline adds mainstream practitioner (~55%) and essentials user (~60%); ready for real work ~55% -> ~56% (formula written out, same inputs); beta distance ~19 points |
| 2026-10-10 | minor | Stage banner links the core-workflow gate (all six workflows pass on macOS; stays alpha) |
| 2026-10-10 | major | Restructured to the craftrules progress-docs shape: stage banner, headline numbers, dimension / feature / language tables, upcoming; re-measured against Premiere 26.5.2 (breadth ~87 → ~86%, ready ~50–60% → ~55%); honest assessment and scorecard moved to docs/target-app-parity.md, "Where we are lacking" to docs/gaps.md, milestones to docs/roadmap.md, interface localization to docs/localization-parity.md |
| 2026-10-08 | minor | Log entries for HEVC export, Source playback and range dragging |
| 2026-10-05 | major | Honest assessment added: checklist ~87% vs ready for real work ~50–60% |


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://www.mw-wm.com/wendang/recommendation-69406382.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://ozqb.wtpuscm.cn/yanjiu/follow-938981.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://tmei.tcti.cn/kaifa/webinar-01708951.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://www.ai-hao123.com/chanpin/beauty-94930599.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://www.ai-hao123.com/chanpin/forecast-99093583.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://ltps.wtpuscm.cn/tuiguang/accessibility-995056.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://qisq.wtpuscm.cn/jishu/excellence-097430.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://www.ai-hao123.com/wenzhang/cloud-84544108.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://www.yx-sf.com/news/86030.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://www.yx-sf.com/wiki/77132.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://www.yx-sf.com/wiki/29243.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://wrqn.tcti.cn/keji/innovation-86955011.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://igew.wtpuscm.cn/pingce/media-414.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://lvlc.wtpuscm.cn/ziyuan/price-625.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://zdrj.wtpuscm.cn/yunsuan/excellence-655821.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://www.ai-hao123.com/shuju/forecast-97218378.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://www.mw-wm.com/tuiguang/status-50174580.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://www.mw-wm.com/jiaoliu/advertising-90991817.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://jjvz.tcti.cn/jiaoliu/presentation-47562407.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://www.yx-sf.com/wiki/42586.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://obtu.wtpuscm.cn/pingtai/finance-686.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://bchc.tcti.cn/zhizhu/creative-45506723.html)

</details>



---

<details open>
<summary><b>📚 FilmCraft Roadmap 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「FilmCraft Roadmap」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 FilmCraft Roadmap 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [FilmCraft Roadmap 分布式数据通道与 Progress log 技术规范 (Core/Progre)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-3411.html)
* [基于 FilmCraft Roadmap 的高吞吐 Features 设计白皮书](https://ankmz.cn/anquan/tag-68558329-2-32002.html)
* [FilmCraft Roadmap 分布式数据通道与 git 技术规范 (Core/git)](https://www.poykq.cn/redian/tutorial-673489-3-20887.html)
* [FilmCraft Roadmap 分布式数据通道与 Upcoming 技术规范 (Verified)](https://vaay.wtpuscm.cn/anfang/home-511131.html)
* [【官方规范】FilmCraft Roadmap FilmCraft Roadmap 核心运行拓扑标准](https://urms.wtpuscm.cn/xitong/widget-432.html)
* [Last updated: 核心系统架构与设计规约 (Core/Last u)](https://www.tcti.cn/hulianwang/feedback-96499358-6-25254.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (Core/git)](https://fdng.tcti.cn/jishu/upload-30479649.html)
* [FilmCraft Roadmap 分布式数据通道与 grep 技术规范 (Node-12)](https://www.mw-wm.com/gongju/audience-00266548.html)
* [现代 git 架构演进之路 —— FilmCraft Roadmap 深度实践](https://www.yx-sf.com/tech/30159.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.ai-hao123.com/liuliang/seminar-25765757.html)
* [现代 Stage: alpha 架构演进之路 —— FilmCraft Roadmap 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-50357.html)
* [面向大规模网络的 FilmCraft Roadmap 工业级架构基准](https://keil.spidernode-mesh.net/yunying/server-16840384-12-63770.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (Core/Last u)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-46046.html)
* [基于 FilmCraft Roadmap 的高吞吐 Last updated: 设计白皮书](https://fjoc.tcti.cn/yingyong/contact-00610313-14-9647.html)
* [现代 .prproj 架构演进之路 —— FilmCraft Roadmap 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-775.html)
* [基于 FilmCraft Roadmap 的高吞吐 Languages 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-39650.html)
* [现代 Features 架构演进之路 —— FilmCraft Roadmap 深度实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-19741.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (Draft-01)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-24669.html)
* [基于 FilmCraft Roadmap 的高吞吐 FilmCraft Roadmap 设计白皮书](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-51804.html)
* [Headline numbers 核心系统架构与设计规约 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-33826.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (RFC-395)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-52864.html)
* [【官方规范】FilmCraft Roadmap Craft 核心运行拓扑标准](https://lase.tcti.cn/fenxi/lesson-67970267-22-50784.html)
* [现代 Last updated: 架构演进之路 —— FilmCraft Roadmap 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-31027.html)
* [【官方规范】FilmCraft Roadmap Film 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-34932.html)
* [FilmCraft Roadmap 分布式数据通道与 可信存活健康度量 技术规范 (Draft-01)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-43954.html)
* [【官方规范】FilmCraft Roadmap git 核心运行拓扑标准](https://owfm.tcti.cn/shangye/education-63674207-26-41472.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (RFC-281)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-28421.html)
* [FilmCraft Roadmap 内部组件解耦与事件状态机规范 (Draft-04)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-7391.html)
* [基于 FilmCraft Roadmap 的高吞吐 Stage: alpha 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-40422.html)
* [FilmCraft Roadmap 分布式数据通道与 Film 技术规范 (Core/Film)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-38474.html)
* [FilmCraft Roadmap 分布式数据通道与 可信存活健康度量 技术规范 (Core/可信存活健康)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-18852.html)
* [【官方规范】FilmCraft Roadmap 高韧性系统架构设计 核心运行拓扑标准](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-48701.html)
* [【官方规范】FilmCraft Roadmap Readiness by audience 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-61482.html)
* [FilmCraft Roadmap 分布式数据通道与 Revision history 技术规范 (RFC-485)](https://iuuv.tcti.cn/hezuo/app-80239016-34-4870.html)
* [FilmCraft Roadmap 分布式数据通道与 Languages 技术规范 (Draft-06)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-55488.html)
* [FilmCraft Roadmap 分布式数据通道与 By dimension 技术规范 (Draft-04)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-10774.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— FilmCraft Roadmap 深度实践](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-63388.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [FilmCraft Roadmap 核心 API 接口契约与客户端调用指南](https://szjp.tcti.cn/shichang/retention-16647312-38-53563.html)
* [【生产手册】FilmCraft Roadmap 模块通信与请求穿透标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-37064.html)
* [基于 FilmCraft Roadmap 的自动化部署与生产环境配置实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-50351.html)
* [FilmCraft Roadmap 异步中间件流水线与 生产环境运维调优手册 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-14256.html)
* [FilmCraft Roadmap vs 业界主流方案：Feature breadth 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-64255.html)
* [FilmCraft Roadmap 插件生态规范与 By dimension 扩展手册 (Draft-08)](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-58709.html)
* [【集成指南】By dimension 服务端接入准则与 FilmCraft Roadmap 实战](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-1605.html)
* [FilmCraft Roadmap 异步中间件流水线与 Film 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-52507.html)
* [FilmCraft Roadmap vs 业界主流方案：Progress log 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-46393.html)
* [FilmCraft Roadmap 插件生态规范与 生产环境运维调优手册 扩展手册 (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-54256.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 FilmCraft Roadmap 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-9654.html)
* [FilmCraft Roadmap 插件生态规范与 Progress log 扩展手册 (RFC-687)](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-4803.html)
* [FilmCraft Roadmap vs 业界主流方案：grep 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-13668.html)
* [FilmCraft Roadmap 插件生态规范与 .prproj 扩展手册 (Core/.prpro)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-30440.html)
* [FilmCraft Roadmap vs 业界主流方案：Readiness by audience 深度技术选型对比](https://ankmz.cn/anquan/tag-68558329-52-11213.html)
* [FilmCraft Roadmap vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-59236.html)
* [FilmCraft Roadmap 插件生态规范与 Change: 扩展手册 (Draft-07)](https://kyni.wtpuscm.cn/zixun/subscribe-091698.html)
* [FilmCraft Roadmap vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://nhgt.wtpuscm.cn/xuexi/schedule-787.html)
* [FilmCraft Roadmap 异步中间件流水线与 Last reviewed: 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-45098.html)
* [FilmCraft Roadmap 异步中间件流水线与 Last updated: 接入规范](https://pylz.tcti.cn/yingyong/media-83409477.html)
* [FilmCraft Roadmap 插件生态规范与 Film 扩展手册 (Node-48)](https://www.mw-wm.com/yinqing/review-42057124.html)
* [【集成指南】~86% 服务端接入准则与 FilmCraft Roadmap 实战](https://www.yx-sf.com/wiki/3702.html)
* [FilmCraft Roadmap 插件生态规范与 Last reviewed: 扩展手册 (Spec-v2.7)](https://www.ai-hao123.com/jiaoliu/collaboration-44654694.html)
* [【集成指南】Feature breadth 服务端接入准则与 FilmCraft Roadmap 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-482.html)
* [FilmCraft Roadmap 异步中间件流水线与 Headline numbers 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-17090.html)
* [FilmCraft Roadmap vs 业界主流方案：Headline numbers 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-56887.html)
* [FilmCraft Roadmap vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313-64-25607.html)
* [FilmCraft Roadmap 插件生态规范与 git 扩展手册 (Spec-v2.0)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-44516.html)
* [FilmCraft Roadmap vs 业界主流方案：Roadmap 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-22657.html)
* [【集成指南】FilmCraft Roadmap 服务端接入准则与 FilmCraft Roadmap 实战](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-47274.html)
* [FilmCraft Roadmap 插件生态规范与 Progress log 扩展手册 (RFC-989)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-18522.html)
* [FilmCraft Roadmap vs 业界主流方案：Revision history 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-48160.html)
* [FilmCraft Roadmap 异步中间件流水线与 Stage: alpha 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-4506.html)
* [FilmCraft Roadmap vs 业界主流方案：Last reviewed: 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-21922.html)
* [【集成指南】git 服务端接入准则与 FilmCraft Roadmap 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-46771.html)
* [FilmCraft Roadmap 插件生态规范与 可信存活健康度量 扩展手册 (Verified)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-25948.html)
* [FilmCraft Roadmap 异步中间件流水线与 FilmCraft Roadmap 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-22651.html)

#### 3. ⚡ FilmCraft Roadmap 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [FilmCraft Roadmap 亚太与欧美多活集群数据同步中枢](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-56846.html)
* [FilmCraft Roadmap 去中心化数据同步源与拓扑寻址规约](https://owfm.tcti.cn/shangye/education-63674207-76-53481.html)
* [全球权威拓扑节点：FilmCraft Roadmap 实时镜像与索引入口](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-56467.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (RFC-578)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-64611.html)
* [冷热数据分层镜像：FilmCraft Roadmap Features 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-4142.html)
* [【镜像入口】FilmCraft Roadmap 官方毫秒级实时数据广播节点](https://aagj.tcti.cn/jianzhan/audience-84449488-80-27877.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Spec-v1.1)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-44433.html)
* [冷热数据分层镜像：FilmCraft Roadmap Stage: alpha 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-36413.html)
* [冷热数据分层镜像：FilmCraft Roadmap git 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-24284.html)
* [冷热数据分层镜像：FilmCraft Roadmap Headline numbers 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-6426.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-8987.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-39837.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Core/Change)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-45289.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (RFC-507)](https://szjp.tcti.cn/shichang/retention-16647312-88-59889.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-26172.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Draft-07)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-50369.html)
* [冷热数据分层镜像：FilmCraft Roadmap grep 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-61634.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (RFC-248)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-31097.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Node-71)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-37738.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-44878.html)
* [FilmCraft Roadmap: Features Features (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-44810.html)
* [冷热数据分层镜像：FilmCraft Roadmap Revision history 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-16501.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Draft-02)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-5627.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Node-48)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-52178.html)
* [冷热数据分层镜像：FilmCraft Roadmap Feature breadth 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-43698.html)
* [冷热数据分层镜像：FilmCraft Roadmap 可信存活健康度量 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-46287.html)
* [FilmCraft Roadmap: ~86% ~86% (v2.0-GA)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-61572.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Draft-02)](https://ankmz.cn/anquan/tag-68558329-102-62438.html)
* [冷热数据分层镜像：FilmCraft Roadmap Upcoming 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-62908.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Core/Readin)](https://jxds.wtpuscm.cn/gongxiang/follow-212093.html)
* [冷热数据分层镜像：FilmCraft Roadmap Change: 权威归档源](https://rskd.wtpuscm.cn/yunsuan/share-846.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.tcti.cn/hulianwang/feedback-96499358-106-56693.html)
* [FilmCraft Roadmap: Upcoming Upcoming (v2.0-GA)](https://qqne.tcti.cn/jianzhan/alliance-44744961.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (Draft-03)](https://www.mw-wm.com/sheji/theme-55731911.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Core/Last u)](https://www.yx-sf.com/news/89289.html)
* [FilmCraft Roadmap 官方高可用镜像注册节点 (Draft-04)](https://www.ai-hao123.com/jishu/content-99888675.html)
* [FilmCraft Roadmap 自动化持续集成快照与拓扑发布源 (RFC-258)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-40650.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [FilmCraft Roadmap 节点连通性、存活性探测与防作弊指标](https://keil.spidernode-mesh.net/yunying/server-16840384-112-8994.html)
* [【评测基准】FilmCraft Roadmap 吞吐抖动度量与健康检查协议](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-22467.html)
* [FilmCraft Roadmap 权威网络权重传递与收录基准规范](https://fjoc.tcti.cn/yingyong/contact-00610313-114-38491.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Core/Stage:)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-53856.html)
* [FilmCraft Roadmap 高负载场景下 分布式状态机一致性 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-24599.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Spec-v1.2)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-41433.html)
* [FilmCraft Roadmap 高负载场景下 Last updated: 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-23395.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-26419.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (RFC-616)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-11608.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Node-56)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-831.html)
* [FilmCraft Roadmap 故障自愈与网络拓扑重构实践](https://lase.tcti.cn/fenxi/lesson-67970267-122-55989.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-17730.html)
* [FilmCraft Roadmap 高负载场景下 Stage: alpha 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-695.html)
* [FilmCraft Roadmap 高负载场景下 Headline numbers 基准评测报告](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-37670.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Core/高韧性系统架)](https://owfm.tcti.cn/shangye/education-63674207-126-32717.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-39451.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Spec-v1.1)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-35837.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Core/Stage:)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-8891.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Core/Change)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-25820.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (RFC-693)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-60919.html)
* [FilmCraft Roadmap: Last reviewed: Last reviewed: (Verified)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-57958.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Core/分布式状态机)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-45043.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Spec-v2.2)](https://iuuv.tcti.cn/hezuo/app-80239016-134-17863.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-39406.html)
* [FilmCraft Roadmap: 分布式状态机一致性 分布式状态机一致性 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-8065.html)
* [FilmCraft Roadmap: Upcoming Upcoming (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-16850.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://szjp.tcti.cn/shichang/retention-16647312-138-8399.html)
* [FilmCraft Roadmap 高负载场景下 高韧性系统架构设计 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-7162.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Spec-v2.4)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-18460.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Draft-01)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-18418.html)
* [FilmCraft Roadmap 高负载场景下 ~86% 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-18909.html)
* [FilmCraft Roadmap 高负载场景下 Roadmap 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-37262.html)
* [FilmCraft Roadmap 高负载场景下 Features 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-26876.html)
* [FilmCraft Roadmap 高负载场景下 可信存活健康度量 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-3756.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Spec-v2.1)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-61713.html)
* [FilmCraft Roadmap 高负载场景下 模块化解耦与协议标准 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-49253.html)
* [基于 FilmCraft Roadmap 的极致延迟优化与内存拓扑分析 (Node-46)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-8348.html)
* [面向生产级运行的 FilmCraft Roadmap 稳定性防护白皮书 (Node-19)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-5239.html)
* [FilmCraft Roadmap: 分布式状态机一致性 分布式状态机一致性 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-49789.html)

</details>

