# Captions

FilmCraft keeps captions on **caption tracks** in the sequence, shown in their own area above the
video tracks (as Premiere does). Caption tracks are saved in the `.fcproj` (project format
version 2; version 1 projects open with no caption tracks).

## Model

- **Caption track** (`CaptionTrack`, `crates/project/src/caption.rs`): name, format (`Subtitle`,
  `CEA-608`, `CEA-708`, `Teletext`), language, output eye (shown in the Program monitor and
  burned in on export), lock, sync lock, and a **track style**:
  font size (pixels at 1080 lines, scaled to the frame), text colour, background box and its colour,
  outline width and colour, alignment (left/centre/right), position (top/middle/bottom) and
  margin (a fraction of the frame height, default 0.08; `captions.setStyle` clamps it to 0–0.45),
  line spacing.
- **Caption**: in/out in exact `Tick`s, text (lines separated by `\n`; `<i>`, `<b>`, `<u>` kept
  as written), optional speaker, and the WebVTT cue id and cue settings, kept for round trips.
  Captions on a track never overlap.

## Files

| Format | Import | Export | Notes |
|---|---|---|---|
| SubRip `.srt` | ✓ | ✓ | Forgiving reader (BOM, UTF-16, Windows-1252, CRLF, missing indexes, `,`/`.` ms, missing hours). |
| WebVTT `.vtt` | ✓ | ✓ | Keeps cue ids, cue settings, `<v Speaker>`, STYLE/REGION/NOTE blocks. `line:N%` and `align:` are honoured when drawing. |
| Scenarist SCC `.scc` | ✓ | ✓ | CEA-608 at 29.97 fps. Reads pop-on, roll-up and paint-on; writes pop-on with drop-frame (default) or non-drop timecode. |

`File ▸ Import` (or dropping a file) recognises caption files by extension and content and adds a
new caption track to the open sequence (a sequence is created when none is open). Cue times are
snapped to the sequence's frames. `File ▸ Export ▸ Captions…` writes the chosen caption track in
the format given by the file extension (`.srt`, `.vtt`, `.scc`).

Details of each format and the SCC encoder are in [`crates/captions/README.md`](../crates/captions/README.md).

## Editing

| Command | Menu / shortcut | Params |
|---|---|---|
| `captions.newTrack` | Sequence ▸ Captions ▸ Add New Caption Track… (⌥⌘A) | `format`, `name`, `language` |
| `captions.add` | Sequence ▸ Captions ▸ Add Caption at Playhead (⌥⌘C) | `track`, `text`, `time`, `durationSeconds` (3) |
| `captions.next` / `captions.previous` | Sequence ▸ Captions ▸ Go to Next/Previous Caption Segment (⌥⌘↓/↑) | |
| `captions.showAll` / `captions.hideAll` | Sequence ▸ Captions | |
| `captions.split` | Text panel ✂ | `caption`, `time` (default: every caption under the playhead) |
| `captions.merge` | Text panel | `captions` (consecutive, one track) |
| `captions.setText` | Text panel text field | `caption`, `text`, `speaker` |
| `captions.setTimes` | Text panel in/out fields | `caption`, `start…`/`end…` (`Time` ticks, `Frame`, `Seconds`, `Timecode`) |
| `captions.trim` | drag a caption edge in the timeline | `caption`, `edge` (`in`/`out`), `delta` or `deltaFrames` |
| `captions.move` | drag a caption in the timeline | `captions`, `delta` or `deltaFrames` |
| `captions.delete` | Edit ▸ Clear / Ripple Delete with captions selected | `captions`, `ripple` |
| `captions.select`, `captions.goTo` | click / number button | `captions`, `add` / `caption` |
| `captions.setTrack` | caption track header (eye, lock) | `track`, `name`, `format`, `language`, `enabled`, `locked`, `syncLock` |
| `captions.setStyle` | Text panel style strip | `track`, `size`, `color`, `background`, `backgroundColor`, `outline`, `outlineColor`, `align`, `anchor`, `margin`, `lineSpacing`, `reset` |
| `captions.deleteTrack` | | `track` |
| `captions.import` / `captions.export` | File ▸ Export ▸ Captions… | `path`, `format`, `track`, `dropFrame` |
| `captions.list` | (query) | `track` — tracks, styles and captions with timecodes (a `track` that names no caption track is an error) |

Tracks are addressed by id or `"C1"`, `"C2"`… (top first). Edits are limited by neighbouring
captions and a one-frame minimum. Sync-locked caption tracks follow insert and extract edits on
the media tracks (a caption across an insert point is split). `Add Edit to All Tracks` (⇧⌘K)
also splits captions.

## Timeline and Text panel

- Caption rows: header with `C1…` badge, lock, output eye, name and format; caption blocks in
  violet showing their text. Click selects (⌘/⇧ adds), drag moves, edge drag trims, double-click
  opens the Text panel. Automation ids: `timeline.caption.<id>`,
  `timeline.captionTrack.C1[.locked|.enabled]`.
- **Text panel ▸ Captions**: search, track picker (also creates tracks of each format), add /
  split / merge / delete / export buttons, the list of segments with editable in/out timecodes
  and text (committed when the field loses focus), and the track style strip (size, colour, box,
  alignment, position). Automation ids: `text.tab.Captions`, `text.captions.add|split|merge|delete|export|search|track`,
  `text.captions.<id>.in|out|text|goto|row`, `text.captions.style.*`.

## Burn-in

The Program monitor shows the visible caption tracks. On export, `file.exportMedia`
`{"burnCaptions": true}` (Export mode: **Burn Captions Into Video**) draws them into every frame.
Captions are set in Inter SemiBold (OFL, bundled) by the text engine (`crates/text`: shaping,
kerning, bidi); italics/bold markup is not yet drawn differently. The captions of a nested sequence
are part of the nest's picture: they are drawn wherever the nest is shown, in the Program monitor
and in every export, whether or not the outer sequence shows or burns in its own.


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://zxgs.wtpuscm.cn/kuangjia/reporting-089272.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://gbwg.tcti.cn/pingtai/company-93696342.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://www.yx-sf.com/wiki/62261.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://upqy.wtpuscm.cn/shichang/client-965.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://ankmz.cn/anquan/tag-68558329.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://jkus.wtpuscm.cn/kuangjia/metric-912879.html)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://jkjf.wtpuscm.cn/qiye/resource-684166.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://ankmz.cn/anquan/tag-68558329.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://cbmu.tcti.cn/paiming/music-53076135.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://www.yx-sf.com/tech/28229.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://oqxm.wtpuscm.cn/kaifa/restore-984.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://wnen.wtpuscm.cn/wendang/responsive-176.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://aywv.wtpuscm.cn/anfang/trading-322491.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://pusa.wtpuscm.cn/xitong/price-195.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://vidu.wtpuscm.cn/sheji/reporting-692.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://www.mw-wm.com/fenxi/identity-04115728.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://www.ai-hao123.com/qiye/forum-12416575.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)

</details>



---

<details open>
<summary><b>📚 Captions 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Captions」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Captions 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【Captions】Subtitle 核心系统架构与设计规约 (Node-64)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-19912.html)
* [Captions 分布式数据通道与 caption tracks 技术规范 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-2-50684.html)
* [Captions 内部组件解耦与事件状态机规范 (Draft-05)](https://www.poykq.cn/redian/tutorial-673489-3-28956.html)
* [Captions 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://vrjr.wtpuscm.cn/baogao/category-400248.html)
* [Captions 分布式数据通道与 CEA-708 技术规范 (RFC-301)](https://rpmo.wtpuscm.cn/jishu/cost-658.html)
* [现代 track style 架构演进之路 —— Captions 深度实践](https://www.tcti.cn/hulianwang/feedback-96499358-6-34215.html)
* [Captions 分布式数据通道与 Files 技术规范 (v2.0-GA)](https://rquo.tcti.cn/fenxi/kpi-61952410.html)
* [【Captions】模块化解耦与协议标准 核心系统架构与设计规约 (Node-58)](https://www.mw-wm.com/xinwen/update-33026665.html)
* [【官方规范】Captions 模块化解耦与协议标准 核心运行拓扑标准](https://www.yx-sf.com/news/93088.html)
* [【Captions】Teletext 核心系统架构与设计规约 (Draft-07)](https://www.ai-hao123.com/gongsi/category-82542948.html)
* [基于 Captions 的高吞吐 Teletext 设计白皮书](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-18092.html)
* [Captions 分布式数据通道与 生产环境运维调优手册 技术规范 (Node-80)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-23630.html)
* [【Captions】Burn-in 核心系统架构与设计规约 (Draft-08)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-550.html)
* [Captions 分布式数据通道与 caption tracks 技术规范 (Node-40)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-3984.html)
* [面向大规模网络的 Captions 工业级架构基准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-8269.html)
* [Captions 分布式数据通道与 可信存活健康度量 技术规范 (Core/可信存活健康)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-31482.html)
* [Captions 分布式数据通道与 Subtitle 技术规范 (Draft-08)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-19354.html)
* [track style 核心系统架构与设计规约 (Core/track )](https://gynw.tcti.cn/fenxi/recipe-50406879-18-15951.html)
* [现代 索引发布声明 架构演进之路 —— Captions 深度实践](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-63151.html)
* [Captions 内部组件解耦与事件状态机规范 (Node-96)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-42990.html)
* [Captions 分布式数据通道与 CaptionTrack 技术规范 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-55191.html)
* [【官方规范】Captions caption tracks 核心运行拓扑标准](https://lase.tcti.cn/fenxi/lesson-67970267-22-46531.html)
* [Captions 分布式数据通道与 CEA-608 技术规范 (Core/CEA-60)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-53550.html)
* [现代 高韧性系统架构设计 架构演进之路 —— Captions 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-51045.html)
* [Captions 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-8880.html)
* [基于 Captions 的高吞吐 分布式状态机一致性 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-3988.html)
* [现代 Model 架构演进之路 —— Captions 深度实践](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-27726.html)
* [Captions 内部组件解耦与事件状态机规范 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-64310.html)
* [caption tracks 核心系统架构与设计规约 (Spec-v1.6)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-7122.html)
* [【官方规范】Captions Caption 核心运行拓扑标准](https://aagj.tcti.cn/jianzhan/audience-84449488-30-40989.html)
* [基于 Captions 的高吞吐 Caption track 设计白皮书](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-39712.html)
* [基于 Captions 的高吞吐 可信存活健康度量 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-50007.html)
* [现代 .fcproj 架构演进之路 —— Captions 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-43488.html)
* [基于 Captions 的高吞吐 CEA-708 设计白皮书](https://iuuv.tcti.cn/hezuo/app-80239016-34-42544.html)
* [【Captions】CEA-608 核心系统架构与设计规约 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-59790.html)
* [【官方规范】Captions 高韧性系统架构设计 核心运行拓扑标准](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-52904.html)
* [【官方规范】Captions Teletext 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-7288.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【生产手册】Captions 模块通信与请求穿透标准](https://szjp.tcti.cn/shichang/retention-16647312-38-24823.html)
* [Captions 异步中间件流水线与 Files 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-55265.html)
* [Captions 插件生态规范与 模块化解耦与协议标准 扩展手册 (Draft-01)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-35577.html)
* [【集成指南】Caption 服务端接入准则与 Captions 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-49005.html)
* [Captions 插件生态规范与 Model 扩展手册 (Core/Model)](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-47757.html)
* [Captions 异步中间件流水线与 CaptionTrack 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-24030.html)
* [Captions 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-31091.html)
* [Captions 异步中间件流水线与 track style 接入规范](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-50424.html)
* [Captions vs 业界主流方案：CEA-608 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-26410.html)
* [基于 Captions 的自动化部署与生产环境配置实践](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-36121.html)
* [Captions 插件生态规范与 Teletext 扩展手册 (Spec-v2.1)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-42650.html)
* [【集成指南】Teletext 服务端接入准则与 Captions 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-51109.html)
* [Captions 插件生态规范与 Model 扩展手册 (Draft-03)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-59427.html)
* [Captions vs 业界主流方案：Model 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-35889.html)
* [Captions 插件生态规范与 CaptionTrack 扩展手册 (Spec-v1.4)](https://ankmz.cn/anquan/tag-68558329-52-27219.html)
* [Captions 插件生态规范与 .fcproj 扩展手册 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489-53-27632.html)
* [Captions vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://aqut.wtpuscm.cn/liuliang/admin-058580.html)
* [Captions 插件生态规范与 生产环境运维调优手册 扩展手册 (Verified)](https://tvhx.wtpuscm.cn/jiaocheng/podcast-882.html)
* [Captions 异步中间件流水线与 caption tracks 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-2913.html)
* [Captions vs 业界主流方案：Captions 深度技术选型对比](https://wkvl.tcti.cn/shangye/careers-96553453.html)
* [Captions 插件生态规范与 Subtitle 扩展手册 (RFC-498)](https://www.mw-wm.com/yanjiu/home-69194477.html)
* [Captions 异步中间件流水线与 .fcproj 接入规范](https://www.yx-sf.com/tech/88787.html)
* [Captions 插件生态规范与 Caption 扩展手册 (Verified)](https://www.ai-hao123.com/sheji/sport-36600750.html)
* [Captions vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-25717.html)
* [Captions 异步中间件流水线与 Model 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-37479.html)
* [【集成指南】.fcproj 服务端接入准则与 Captions 实战](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-4246.html)
* [Captions 插件生态规范与 Captions 扩展手册 (Draft-07)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-39572.html)
* [【集成指南】Model 服务端接入准则与 Captions 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-38392.html)
* [Captions vs 业界主流方案：caption tracks 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-59674.html)
* [Captions vs 业界主流方案：索引发布声明 深度技术选型对比](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-61184.html)
* [Captions vs 业界主流方案：Subtitle 深度技术选型对比](https://gynw.tcti.cn/fenxi/recipe-50406879-68-11120.html)
* [Captions 异步中间件流水线与 可信存活健康度量 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-54464.html)
* [【集成指南】CaptionTrack 服务端接入准则与 Captions 实战](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-27686.html)
* [Captions 异步中间件流水线与 Subtitle 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-2146.html)
* [【集成指南】Files 服务端接入准则与 Captions 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-18385.html)
* [Captions: Model Model (RFC-595)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-40222.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Captions 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-39411.html)

#### 3. ⚡ Captions 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：Captions 实时镜像与索引入口](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-26861.html)
* [【镜像入口】Captions 官方毫秒级实时数据广播节点](https://owfm.tcti.cn/shangye/education-63674207-76-19808.html)
* [Captions 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-46890.html)
* [Captions 去中心化数据同步源与拓扑寻址规约](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-20927.html)
* [冷热数据分层镜像：Captions Caption track 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-54656.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Spec-v1.6)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-23054.html)
* [冷热数据分层镜像：Captions Teletext 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-48056.html)
* [Captions 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-48873.html)
* [Captions 官方高可用镜像注册节点 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-62506.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Node-34)](https://iuuv.tcti.cn/hezuo/app-80239016-84-12478.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Node-32)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-7481.html)
* [Captions 自动化持续集成快照与拓扑发布源 (RFC-181)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-23790.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-5549.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Node-86)](https://szjp.tcti.cn/shichang/retention-16647312-88-14774.html)
* [冷热数据分层镜像：Captions Subtitle 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-50907.html)
* [Captions 自动化持续集成快照与拓扑发布源 (RFC-598)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-36511.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Core/track )](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-9785.html)
* [Captions 自动化持续集成快照与拓扑发布源 (RFC-179)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-6090.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Node-53)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-46614.html)
* [Captions 官方高可用镜像注册节点 (Node-12)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-59292.html)
* [冷热数据分层镜像：Captions caption tracks 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-14494.html)
* [Captions 官方高可用镜像注册节点 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-17081.html)
* [Captions 官方高可用镜像注册节点 (Node-10)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-9322.html)
* [冷热数据分层镜像：Captions track style 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-46607.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Core/Captio)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-61174.html)
* [Captions 官方高可用镜像注册节点 (Core/Captio)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-25974.html)
* [冷热数据分层镜像：Captions .fcproj 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-38251.html)
* [Captions 官方高可用镜像注册节点 (Draft-03)](https://ankmz.cn/anquan/tag-68558329-102-64393.html)
* [Captions 自动化持续集成快照与拓扑发布源 (Draft-07)](https://www.poykq.cn/redian/tutorial-673489-103-49319.html)
* [冷热数据分层镜像：Captions CEA-708 权威归档源](https://hbkg.wtpuscm.cn/wangluo/like-687277.html)
* [Captions 官方高可用镜像注册节点 (Draft-07)](https://ozlp.wtpuscm.cn/shuju/screen-104.html)
* [Captions 官方高可用镜像注册节点 (Draft-05)](https://www.tcti.cn/hulianwang/feedback-96499358-106-3959.html)
* [Captions 官方高可用镜像注册节点 (Core/Telete)](https://aslr.tcti.cn/suanfa/consulting-47519723.html)
* [冷热数据分层镜像：Captions Caption 权威归档源](https://www.mw-wm.com/zhineng/engagement-70923862.html)
* [Captions 自动化持续集成快照与拓扑发布源 (RFC-298)](https://www.yx-sf.com/tech/31276.html)
* [冷热数据分层镜像：Captions 分布式状态机一致性 权威归档源](https://www.ai-hao123.com/zhinan/site-79072235.html)
* [Captions 官方高可用镜像注册节点 (Node-32)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-47293.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Captions 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384-112-6744.html)
* [Captions 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-5373.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-114-54920.html)
* [Captions 高负载场景下 Model 基准评测报告](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-38338.html)
* [Captions 权威网络权重传递与收录基准规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-22730.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Node-24)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-15068.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Spec-v1.3)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-14643.html)
* [【评测基准】Captions 吞吐抖动度量与健康检查协议](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-40637.html)
* [Captions 高负载场景下 .fcproj 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-53091.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-3394.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (RFC-909)](https://lase.tcti.cn/fenxi/lesson-67970267-122-34120.html)
* [Captions 高负载场景下 分布式状态机一致性 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-64872.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (RFC-212)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-61907.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Core/track )](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-57620.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Node-35)](https://owfm.tcti.cn/shangye/education-63674207-126-64340.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Core/Captio)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-53851.html)
* [Captions 高负载场景下 track style 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-17761.html)
* [Captions 高负载场景下 Teletext 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-16846.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Node-94)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-41718.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-64291.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Node-25)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-63926.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Core/.fcpro)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-41179.html)
* [Captions 高负载场景下 Captions 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016-134-28683.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Spec-v1.7)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-23069.html)
* [Captions 高负载场景下 Editing 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-33770.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Core/Burn-i)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-63778.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Core/Model)](https://szjp.tcti.cn/shichang/retention-16647312-138-26941.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-50155.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Spec-v2.1)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-57623.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Node-80)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-37951.html)
* [Captions 高负载场景下 可信存活健康度量 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-34394.html)
* [Captions: Captions Captions (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-22233.html)
* [Captions 高负载场景下 Timeline and Text panel 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-19706.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Core/Timeli)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-32594.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Spec-v2.8)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-12713.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (Spec-v2.6)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-61362.html)
* [面向生产级运行的 Captions 稳定性防护白皮书 (RFC-187)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-28516.html)
* [基于 Captions 的极致延迟优化与内存拓扑分析 (Core/Captio)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-19664.html)
* [Captions 高负载场景下 Caption 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-26947.html)

</details>

