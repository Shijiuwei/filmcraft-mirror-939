# Roadmap detail

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (new file: milestones moved from ROADMAP.md, Current focus rewritten from the 2026-10-10 re-measure, beta milestones added) · **Target:** Adobe Premiere Pro 2026 (26.5.2)

Forward-looking plan: milestones with status, the current focus and what's next, with estimates in
Opus 5.5 agent-hours (calibration in [target-app-parity.md](target-app-parity.md#calibration)).
The one-page summary is [ROADMAP.md](../ROADMAP.md); the ranked work list is [gaps.md](gaps.md).
There is no `cargo xtask scorecard` or `cargo xtask parity` yet ([G7](gaps.md#g7-no-measured-parity-tool-in-ci)).

## Current focus

In order. Each item links to its gap entry, which says what "done" means.

1. **Stop launch crashes and add pull-request CI** ([G4](gaps.md#g4-crashes-at-start-up-and-no-test-ci-on-pull-requests)): `cargo xtask ci` on macOS, Windows and Linux for every PR; old-macOS and Intel-GPU start-up. 40–80 h.
2. **Open `.prproj`** ([G2](gaps.md#g2-premieres-prproj-cannot-be-opened)): the beta blocker. 60–120 h.
3. **10-bit / 4:2:2 camera media in software** ([G3](gaps.md#g3-10-bit-and-422-camera-media-needs-hardware)). 30–50 h.
4. **Fidelity harness against Premiere** ([G1](gaps.md#g1-nothing-compares-our-output-with-premieres)) and **`cargo xtask parity`** ([G7](gaps.md#g7-no-measured-parity-tool-in-ci)): turns estimates into measurements. 50–90 h.
5. **Real-world media corpus and workflow tests** ([G6](gaps.md#g6-no-real-world-media-corpus-or-workflow-acceptance-tests)). 40–70 h.
6. **Hardware acceleration, continued** (#30, [G5](gaps.md#g5-hardware-acceleration-is-partial)): zero-copy upload, Lumetri curves / wheels / HSL and keys on the GPU, VideoToolbox Main 10, Quick Sync / AMF encoders. 120–200 h.
7. **UI fidelity issues** ([G8](gaps.md#g8-ui-fidelity-monitor-handles-effect-controls-docking)): Program monitor transform handles, Effect Controls, docking. 120–200 h.

## Alpha gate

Premiere's core daily workflows, checked end to end on the main platform (macOS, Apple silicon),
including saving the work as `.fcproj` and reopening it (atomic saves, schema migrations,
auto-save and crash recovery; migration tests). Strict reading: a "partial" that stops the
workflow would make FilmCraft pre-alpha.

| Workflow | Works end to end? | Evidence | Hours to pass |
|---|---|---|---|
| Ingest and organize media (import camera / phone / screen files, bins, metadata, proxies) | partial, not blocking | H.264 / HEVC / ProRes / DNx / MXF / AVCHD / MKV import with bit-exact decoders, Media Browser, bins, proxies, Link Media. Gaps: H.264 10-bit / 4:2:2 (Sony XAVC S-I, Canon XF-AVC) opens nowhere and HEVC 4:2:2 only through VideoToolbox (#626, [G3](gaps.md#g3-10-bit-and-422-camera-media-needs-hardware)); E-AC-3 (#647), AVI (#598), camera RAW | 0 (G3 is a beta item, 30–50 h) |
| Assemble and trim a sequence (insert / overwrite, ripple / roll / slip / slide, razor, J/K/L, multicam) | yes | Edit algebra with undo tests, Trim Monitor, Premiere keyboard preset, multicam with audio sync; UX issues (#683, #648, #668) slow it down but don't stop it | 0 |
| Colour correct and grade (Lumetri, LUTs, scopes, colour management) | yes | Lumetri complete with CPU / WGSL parity tests, all Lumetri Scopes, LUT import, HDR export signalling checked with ffprobe | 0 |
| Mix audio (levels, automation, Essential Sound, loudness) | yes on macOS | Mixer graph, automation, 53 effects, BS.1770 loudness matching ffmpeg; device reports are Windows / Linux (#605, #617) | 0 |
| Titles and captions | yes | Text engine, graphic clips, templates, SRT / VTT / SCC / MCC / STL / TTML import-export, burn-in; Arabic titles broken (#395) | 0 |
| Deliver (export H.264 / HEVC / ProRes / DNxHR / MXF with presets, queue) | yes | Own H.264 + AAC encoder, ProRes, DNxHR, MXF, hardware HEVC on macOS, ffprobe-checked exports; bitrate target miss (#371) | 0 |

**Passes:** every core workflow runs end to end on macOS and the project saves and reopens, so
FilmCraft stays **alpha**; the one partial (10-bit / 4:2:2 camera codecs) blocks some cameras'
originals, not the workflow, and is on the beta list. Interchange with Premiere's own `.prproj`
is a beta requirement, not an alpha one (sequences come over as FCP7 XML, AAF or OTIO).

## Beta milestones

Beta = ready for real work ≥ ~75% and `.prproj` opens reliably ([target-app-parity.md](target-app-parity.md#stage)).

| # | Milestone | Status | Est. |
|---|---|---|---|
| B1 | PR CI on three OSs + launch-crash fixes | ⬜ | 40–80 |
| B2 | `.prproj` import (sequences, clips, transitions, markers, bins, common effect parameters) | ⬜ | 60–120 |
| B3 | Camera media: H.264 High 10 / 4:2:2, HEVC 4:2:2, E-AC-3, AVI, multi-track audio confirmed | 🟡 (multi-stream audio #361) | 50–90 |
| B4 | Fidelity harness + `cargo xtask parity` + media corpus + workflow tests | ⬜ | 90–160 |
| B5 | Hardware: zero-copy, more GPU effects, Windows vendor encoders, VideoToolbox Main 10 | 🟡 | 60–100 |
| B6 | UI fidelity top issues and Windows / Linux runtime | 🟡 | 90–160 |
| B7 | AAF / OMF validated in Avid and Pro Tools; XML import speed | ⬜ | 20–40 |

## Milestones (history and remaining)

Status: ✅ done · 🟡 in progress · ⬜ not started. Estimates are remaining agent-hours.

| # | Milestone | Status | Done | Remaining | Est. |
|---|---|---|---|---|---|
| M0 | Skeleton + visual shell | ✅ | Workspace, 20 crates, dock/workspaces, Premiere 26 look, native menus, control channel, MCP, xtask gates (layers, wasm) | — | — |
| M1 | Media I/O | ✅ | MP4/MOV demux+mux, WAV, stills, MJPEG, symphonia audio (MP3/FLAC/ALAC/Vorbis), GOP seek + frame cache, import | Media Browser polish | 2 |
| M2 | H.264 decoder | ✅ | Own decoder, bit-exact on 37+ streams, 500–600 fps 1080p | — | — |
| M3 | Editing core | 🟡 | Edit algebra (insert/overwrite/razor/lift/extract/ripple/roll/slip/slide/rate-stretch/nest/paste), tools, markers, trim mode + Trim Monitor + dynamic J/K/L trimming, Keyboard Shortcuts editor with FilmCraft/Premiere/FCP/Avid presets; Edit/Clip/File menu commands (Label colours and Select Label Group, Paste/Remove Attributes, Select All Matching, Remove Unused, Consolidate Duplicates, Sequence From Clip, Bin From Selection, Offline File, Close Project, Make/Edit Subclip, Modify Audio Channels/Timecode, Frame Hold Options/Add Frame Hold/Insert Frame Hold Segment, Time Interpolation with frame blending, Fit/Fill frame, Breakout to Mono, Extract Audio, Replace With Clip; M3.11: Scene Edit Detection (pure-Rust cut detection, background job), Normalize Mix Track, Simplify Sequence, Transcribe Sequence, Find/Find Next and search bins, Automate to Sequence, Edit Original, Edit Offline, Source Settings, Update Metadata (XMP), Generate Audio Waveform, Project Settings General/Scratch Disks, Get Media File Properties, Save as Template, Selection as FilmCraft Project, Avid Log Exchange export, Flash Cue markers, Dynamic Audio Waveforms, Reveal Log Files, System Compatibility Report); M3.12 keyboard parity: Premiere's default keyboard (~115 keyboard-only commands: edit-point navigation on targeted/any track, select clip at playhead/next/previous, extend edit to playhead, nudge/slip/slide selection, target and source-patch toggles, clip volume ±1 dB/many, frame maximize/full screen/panel cycling, monitor zoom, track heights, Project and Text panel keyboard navigation, text size/leading/alignment, Export Frame, poster frames; see docs/keyboard.md); multicam (Create Multi-Camera Source Sequence, Multi-Camera view with live switching on 1–9, angle switching, Enable/Flatten, Edit Cameras) and sync (Synchronize, Merge Clips; In/Out/timecode/marker/audio — GCC-PHAT, sample-accurate) | Multicam paging >16 angles and grid thumbnails, optical flow (renders as frame blending), 39 Premiere default shortcuts skipped with reasons in docs/keyboard.md (Productions, AI/cloud tools, work area bar, Production/Search panels) | 6–10 |
| M4 | Playback | 🟡 | Audio-clock master, prefetch with cancellation, J/K/L, correct dropped-frame stats, playback resolution, render bar + content-hashed render previews, App Nap opt-out; 1080p H.264 and 3 stacked 1080p streams play with 0 dropped frames | 4K under load, 8K, frame-threaded AV1 decode, reduced-resolution decode for multicam grids | 6–10 |
| M5 | Effects, keyframes, GPU | 🟡 | All 93 Premiere 26 video effects and 84 transitions (CPU), blend modes and 33 common effects on the GPU (#32, GPU2, GPU4, GPU5), keyframes + value/velocity graphs, wgpu compositor, effect + opacity masks (ellipse/polygon/pen, feather, expansion, modes; CPU/WGSL parity), mask tracking (Lucas–Kanade + RANSAC), adjustment layers, effect presets (built-in + user, JSON import/export) | More effects on the GPU (34 ids done incl. Lumetri basic / creative / vignette; GPU export landed #421), GPU masks in the live path, exact Warp Stabilizer (3-D) and Morph Cut | 40–70 |
| M6 | Export | ✅ | Own H.264 encoder (High/Main/Baseline, B-frames, VBR/CBR/2-pass) → MP4 or QuickTime + own AAC; ProRes, DNxHR, MJPEG, PNG/TIFF/BMP sequences (Premiere-style numbering), GIF, WAV, AIFF; background jobs; M6.5: Export mode parity (frame size / rate / scaling / pixel aspect, profile / level, CBR / VBR 1- and 2-pass, keyframe distance, audio codec / rate / channels / bitrate / sample size, multiplexer, burn-in or sidecar captions, image / name / timecode overlays, video limiter, loudness normalization with true-peak limiter, metadata, ranges, estimated size, summary), 24 built-in presets + user presets with favourites and import/export (Preset Manager), export queue (reorder, cancel, retry, several sequences / ranges), Quick Export; `export.*` commands and `filmcraft-cli export --preset` | Interlaced encoding, 5.1 audio, smart render, publishing destinations | 6–8 |
| M7 | Audio | 🟡 | Mixer graph (tracks → submixes → Mix, pre/post-fader inserts and sends, latency-compensated, sample-accurate, ~6× realtime for 24 tracks × 3 effects on one core), Audio Track Mixer + Audio Clip Mixer panels, track automation (Off/Read/Latch/Touch/Write, recorded live while playing, thinned to keyframes, timeline lanes with pen editing), solo/solo-safe, channel mapping basics, peak + BS.1770 loudness meters (match ffmpeg), DSP crate with 16 clip/track effects, Audio Gain (set/adjust/normalize), Constant Power / Constant Gain / Exponential Fade, Essential Sound (types, Loudness auto-match, Repair incl. DeEss/DeReverb, Clarity, Creative, Ducking keyframes, presets; full Dialogue chain 22× realtime) | Music duration remix, ML speech enhancement, 5.1 panner and multichannel buses, voice-over record, effect editor windows (EQ curve), remaining effects (multiband, convolution reverb), clip-mixer automation recording | 6–9 |
| M8 | Colour | 🟡 | Lumetri: basic, creative + looks, RGB & hue curves, wheels, HSL secondary, vignette, section bypass; Input LUT / Look LUT (.cube 1D/3D/shaper, .3dl; tetrahedral CPU + WGSL; project LUT library; built-in camera conversions); Colour Match (Oklab tonal-range statistics, skin protection, solved in Lumetri wheels); colour management: Rec. 709 / Rec. 2100 PQ / HLG working spaces + wide gamut, Interpret Footage colour space (S-Log3, V-Log, Canon Log 2/3, LogC3/4, Apple Log, D-Log from published specs), metadata auto-detect, BT.2390 tone mapping, gamut mapping, HDR export signalling (VUI/colr/mdcv/clli/SEI, ffprobe-verified); Lumetri Scopes: Vectorscope YUV (75/100 % targets, skin-tone line) and HLS, Histogram, Parade (RGB/YUV/RGB-White), Waveform (RGB/Luma/YC/YC no Chroma), presets, Rec. 601/709/2100, 8-bit/float/HDR (cd/m²) scales, multi-scope grid, `scopes.read` | HDR-aware Lumetri maths, macOS EDR monitors, mastering metadata → tone-map peak, D-Log M (no published formula), HSL Secondary refine | 3–5 |
| M9 | More codecs | 🟡 | ProRes decode+encode, AAC decode+encode, HEVC Main/Main 10 decoder (bit-exact on 41 fixtures, ~225 fps 1080p), VP9 decoder (profiles 0–3, 8/10/12-bit, bit-exact on 50+ fixtures; WebM/MKV `V_VP9` and MP4 `vp09` import with key-frame-checked seeking), Matroska/WebM import (H.264/HEVC/VP9/ProRes/MJPEG + AAC/Opus/FLAC/MP3/Vorbis/PCM), Opus decoder (SILK/CELT/hybrid, 5.1/7.1 multistream; all RFC 8251 vectors range-exact; WebM/MKV/MP4), DNxHD/DNxHR decoder (all SMPTE ST 2019-1 CIDs, 8/10/12-bit, 4:2:2/4:4:4, interlaced, alpha; within IDCT precision of ffmpeg on 22 fixtures) + DNxHR LB/SQ/HQ/HQX/444 encoder and MOV `AVdh` export, AV1 decoder (Main profile, 8/10-bit, all intra/inter tools, loop filters, superres, film grain, intra BC, spatial layers; bit-exact vs libdav1d on 19 SVT fixtures + 22 libaom vectors; MP4 `av01` / WebM `V_AV1` import), MXF import (OP1a/OP-Atom; AVC with POC-ordered B pictures bit-exact, DNxHD/DNxHR, ProRes, PCM/AES3; MPEG-2 reported unsupported; index-table seeking, material package timecode), Ogg Opus/Vorbis (granule seeking, pre-skip, end trimming), image sequences, Broadcast WAV timecode | VP9 frame threading, AV1 threading/SIMD (~4 fps 1080p today), AV1 High/Professional profiles, H.264 High 10 / 4:2:2 and HEVC 4:2:2 in software (G3), E-AC-3, AVI (MPEG-2 decoder and hardware decode have landed) | 30–50 |
| M10 | Graphics & captions | 🟡 | Caption tracks (Subtitle/CEA-608/708/Teletext formats, track style), SRT/WebVTT/SCC import+export (frame-exact, property-tested), caption editing (add/split/merge/trim/move, sync-locked insert/extract), Text panel Captions tab, burn-in in Program monitor and export; text engine (`crates/text`: bundled + system fonts, harfrust shaping, bidi, line breaking, paragraph layout, glyph cache; 3-line 1080p title ≈ 0.15 ms warm); graphic clips with text + shape layers (fill, 2 strokes, background, shadow, keyframable transform), Type tool with on-monitor editing, shape/pen tools, Properties/Essential Graphics editor, align/distribute; graphics templates (own `.fcgt` format, 8 original built-ins, Browse tab with engine-rendered thumbnails, export / install / apply / editable properties; never reads .mogrt), rolls/crawls with ease and pre/postroll, responsive pins and intro/outro, per-character styles, Upgrade Caption to Graphic, source graphics, Replace Fonts; MCC (608+708), EBU STL, TTML/IMSC1, DFXP (frame-exact, property-tested) | 608/708 embedding in video streams, speech-to-text | 12–18 |
| M11 | Interchange & project management | ✅ | `.fcproj` schema versions + migrations, atomic saves, Save a Copy/Revert, auto-save ring + crash-recovery journal (Preferences ▸ Auto Save, recovery prompt), FCP7 XML, FCPXML, EDL, OTIO; offline media (own slate) + Link Media (fingerprint-checked relink, folder remap, search, Align Timecode, Make Offline); proxies (ProRes Proxy/LT, H.264 ¼/½ background jobs, attach/detach/reconnect, monitor toggle, export full-res) + ingest (copy/transcode/proxies); Project Manager (collect, consolidate + transcode with handles, size estimate) | Rename media to clip names, image-sequence conversion, Media Browser-driven relink, smart (cross-drive) path tracking | 1–2 |
| M12–M16 | Multicam, web (WASM), platform, long tail | 🟡 | Multicam: audio sync (GCC-PHAT, sub-sample), Merge Clips, multi-camera source sequences, Multi-Camera view with live switching (1–9); L0–L4 crates compile to wasm32 | Web app shell (file access, WebCodecs, audio), scene detection, auto reframe, ~800 remaining commands and dialogs, performance hardening | 35–55 |
| HW | Hardware acceleration (#30) | 🟡 | Blend modes and 34 effect ids on the GPU incl. Lumetri basic / creative / vignette (#32, GPU2–GPU7); GPU export rendering (#421); decode: VideoToolbox (#33, #41), Media Foundation / D3D11 H.264 / HEVC / VP9 / AV1 (HW2, HW3), VA-API H.264 / HEVC (#358), NVDEC H.264 / HEVC (#549); encode: VideoToolbox H.264 / HEVC, NVENC H.264 (Windows, Linux) and HEVC incl. Main 10 HDR (#328, #360) | Zero-copy, remaining GPU effects, VideoToolbox Main 10, Quick Sync / AMF, VA-API VP9 / AV1 and encode | 120–200 |
| L10N | Localization | 🟡 | Spanish, Japanese, Simplified Chinese whole interface; Portuguese (Brazil), Ukrainian menus | Hindi, Arabic (RTL), French, Indonesian, German, Korean, Vietnamese, Italian, Russian; native review | 150–230 |

## After beta

Plugin hosting (VST3 / AU, then OpenFX; owner decision, [G11](gaps.md#g11-no-plugin-hosting)),
AI features with openly licensed models ([G12](gaps.md#g12-ai-features)), delivery formats
([G13](gaps.md#g13-delivery-formats)), localization ([G14](gaps.md#g14-localization-8-of-the-12-key-languages-missing)),
the web app shell ([G20](gaps.md#g20-the-web-app-shell)).

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | minor | Alpha gate table added (craftrules core-workflow gate): six core workflows, all pass on macOS; stage stays alpha |
| 2026-10-10 | major | Created: milestone table moved from ROADMAP.md (M5 and M9 remaining work updated), HW and L10N rows, beta milestones B1–B7, Current focus from the re-measure |


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://uiwf.wtpuscm.cn/gongxiang/personalization-985435.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://fruh.wtpuscm.cn/yingyong/topic-615082.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://mrix.wtpuscm.cn/pingce/automation-787361.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://csjo.wtpuscm.cn/shuju/lead-946756.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://ntmq.wtpuscm.cn/chanpin/ai-085429.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://rvxn.wtpuscm.cn/jiaoliu/behavior-628183.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://txpp.wtpuscm.cn/fuwu/hosting-235712.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://hugl.wtpuscm.cn/pingtai/calendar-809.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://wark.wtpuscm.cn/liuliang/widget-670332.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://ufen.wtpuscm.cn/anli/settings-515457.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://vvmk.wtpuscm.cn/shichang/subscribe-671654.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://wptz.wtpuscm.cn/anli/unsubscribe-088301.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://bliu.wtpuscm.cn/yinqing/collaborate-931462.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://gzzw.wtpuscm.cn/jishu/growth-478119.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://ouzd.wtpuscm.cn/huodong/vendor-995353.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://yqvf.wtpuscm.cn/zixun/section-897013.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://ilta.wtpuscm.cn/wendang/tracking-841746.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://octg.wtpuscm.cn/yinqing/kpi-685026.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://kzgn.wtpuscm.cn/jiaoliu/comment-448643.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://erok.wtpuscm.cn/baogao/device-869654.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://vwpc.wtpuscm.cn/suanfa/plugin-608152.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://vaji.wtpuscm.cn/hezuo/video-316120.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://ywez.wtpuscm.cn/tuiguang/vendor-697980.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://hxif.tcti.cn/peixun/traffic-04330381.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://xrud.tcti.cn/yanjiu/behavior-68941829.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://mpfu.tcti.cn/anli/roi-53968663.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://nbyt.tcti.cn/yunsuan/goal-18300526.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://ziya.tcti.cn/keji/lesson-15948659.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://bzra.tcti.cn/pingtai/careers-67437830.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://cqcm.tcti.cn/jianzhan/integration-35965250.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://pktm.tcti.cn/yingyong/study-26532658.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://dlyb.tcti.cn/zhizhu/coupon-68864482.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://knzf.tcti.cn/kuangjia/folder-60390607.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://ofxk.tcti.cn/xitong/education-65507598.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://iwmf.tcti.cn/kaifa/products-40543782.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://ydsl.tcti.cn/yingxiao/team-97354074.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://znzn.tcti.cn/pingce/tag-98057550.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://knat.tcti.cn/ziyuan/recommendation-57137310.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://kaxr.tcti.cn/wenzhang/quality-66554998.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://gegw.tcti.cn/youhua/value-08020039.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://inhz.wtpuscm.cn/wendang/workshop-673766.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://www.mw-wm.com/peixun/search-67121874.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://www.yx-sf.com/news/82034)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://www.ai-hao123.com/kaifa/expense-97629262.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://kngv.tcti.cn/qiye/productivity-38108045.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://vtsm.tcti.cn/shichang/investment-85978175.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://bqys.wtpuscm.cn/xuexi/design-016050.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://cfxo.wtpuscm.cn/jishu/revenue-634009.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://ywtd.wtpuscm.cn/jianzhan/accessibility-020163.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://ruhq.wtpuscm.cn/anli/training-142796.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://bvpr.wtpuscm.cn/guanjianci/business-562651.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://hmei.wtpuscm.cn/shuju/customer-774416.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://tenk.wtpuscm.cn/suanfa/server-517348.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://aquz.wtpuscm.cn/yingxiao/analysis-552.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://cgky.wtpuscm.cn/kuangjia/success-630375.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://nyxa.wtpuscm.cn/yingxiao/content-871662.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://ewde.wtpuscm.cn/yingxiao/dashboard-075730.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://nxci.wtpuscm.cn/ziyuan/software-330086.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://vqwh.wtpuscm.cn/xitong/integration-144974.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://blos.wtpuscm.cn/chuangxin/satisfaction-276308.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://qaiz.wtpuscm.cn/zixun/module-672515.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://zmmr.wtpuscm.cn/shuju/success-621208.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://ozee.wtpuscm.cn/jishu/health-625255.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://qros.wtpuscm.cn/fenxi/vendor-872379.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://dtmv.wtpuscm.cn/wangluo/research-472597.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://xjvy.wtpuscm.cn/gongsi/excellence-369373.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://udzt.wtpuscm.cn/chuangxin/automation-590415.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://uwgu.wtpuscm.cn/yingyong/server-837736.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://uywl.wtpuscm.cn/xitong/version-814788.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://snhj.tcti.cn/chanpin/budget-22299762.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://wqmn.tcti.cn/zhineng/income-08015578.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://flym.tcti.cn/shuju/shopping-32886970.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://pujl.tcti.cn/zixun/recommendation-33057674.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://cgpr.tcti.cn/shuju/about-78530033.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://zfsb.tcti.cn/pingtai/lesson-74440956.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://wkgb.tcti.cn/gongsi/expense-69580681.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://hxmq.tcti.cn/wangluo/whitepaper-26599911.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://plxw.tcti.cn/anli/campaign-17573553.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://dchf.tcti.cn/shuju/marketing-48520319.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://exzx.tcti.cn/paiming/calculator-53552587.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://qwmy.tcti.cn/fuwu/management-98678949.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://tsfm.tcti.cn/jishu/api-01155825.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://dzhk.tcti.cn/suanfa/kpi-55035630.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://civg.tcti.cn/xinwen/home-47503952.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://kzsn.tcti.cn/suanfa/story-29773555.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://clqa.tcti.cn/ziyuan/blog-85716001.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://aknm.wtpuscm.cn/youhua/promotion-834884.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://www.mw-wm.com/xuexi/experience-74035782.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://www.yx-sf.com/tech/33325)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://www.ai-hao123.com/fenxi/behavior-67156800.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://baiw.tcti.cn/qiye/resource-18437994.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://poyz.tcti.cn/pingce/integration-21298847.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ytjr.wtpuscm.cn/zhizhu/document-338594.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://fglz.wtpuscm.cn/yingxiao/digital-382739.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://gbzs.wtpuscm.cn/xuexi/hotel-269684.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://crzk.wtpuscm.cn/zhinan/efficiency-470018.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://bdrz.wtpuscm.cn/anli/lead-053406.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://gzgb.wtpuscm.cn/keji/blog-621570.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://huri.wtpuscm.cn/youhua/social-801325.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://rqjb.wtpuscm.cn/tuiguang/efficiency-147.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://ayff.wtpuscm.cn/gongxiang/traffic-967706.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://uoga.wtpuscm.cn/wangluo/objective-520799.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://cybu.wtpuscm.cn/youhua/finance-360692.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://wfla.wtpuscm.cn/sheji/team-085150.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://dlvv.wtpuscm.cn/wenzhang/quality-871511.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://oyzk.wtpuscm.cn/jishu/data-386043.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://rrlx.wtpuscm.cn/jiaocheng/device-292268.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://cxxn.wtpuscm.cn/pingtai/recipe-053858.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://qvqf.wtpuscm.cn/youhua/backup-281703.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://sybg.wtpuscm.cn/paiming/finance-999891.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://pcoo.wtpuscm.cn/chanpin/software-962105.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://hnqs.wtpuscm.cn/yingxiao/share-985557.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://tulk.wtpuscm.cn/jishu/file-035056.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://yfrs.wtpuscm.cn/zhineng/collaborate-676125.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://davq.wtpuscm.cn/zixun/article-691062.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://hrgt.tcti.cn/suanfa/comment-29428813.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://ysij.tcti.cn/kuangjia/event-04049930.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://nyqd.tcti.cn/zhineng/story-44678279.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://bxor.tcti.cn/xinwen/section-58244723.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://zqgr.tcti.cn/shangye/calculator-86566240.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://znqg.tcti.cn/anfang/lesson-89607218.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://sjek.tcti.cn/jianzhan/identity-18216758.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://gddu.tcti.cn/kaifa/advertising-14373567.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://unas.tcti.cn/sheji/target-81302204.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://fvdj.tcti.cn/pingce/luxury-19593599.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://rlbs.tcti.cn/anli/reminder-11976691.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://qieq.tcti.cn/zhineng/goal-35563289.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://rwnv.tcti.cn/tuiguang/network-85183649.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://qkap.tcti.cn/xitong/brand-21788734.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://sspn.tcti.cn/peixun/shopping-64066307.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://eion.tcti.cn/anli/strategy-65836774.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://vcck.tcti.cn/zhizhu/company-98534353.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://riyb.wtpuscm.cn/kaifa/course-966820.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://www.mw-wm.com/baogao/deadline-53692822.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://www.yx-sf.com/wiki/8047)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://www.ai-hao123.com/zhinan/investment-98681964.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://byxd.tcti.cn/baogao/funnel-19983134.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://ykab.tcti.cn/huodong/solution-43904246.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://zrwd.wtpuscm.cn/wenzhang/article-934676.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://cxxt.wtpuscm.cn/chanpin/travel-750517.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://lvhe.wtpuscm.cn/huodong/machine-718948.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://hqhl.wtpuscm.cn/paiming/plugin-268515.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://ncjl.wtpuscm.cn/keji/income-546452.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://vcei.wtpuscm.cn/hezuo/business-657682.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://ugfq.wtpuscm.cn/yingyong/settings-015905.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://pcoa.wtpuscm.cn/jishu/network-026.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://cklk.wtpuscm.cn/paiming/milestone-977527.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://ikzd.wtpuscm.cn/jiaoliu/folder-451317.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://qbyc.wtpuscm.cn/xuexi/terms-526527.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://liuc.wtpuscm.cn/gongxiang/tutorial-637221.html)

</details>

