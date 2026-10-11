# Hardware parity

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (new file: hardware decode / encode per platform and codec, GPU effects, devices; re-measured after NVDEC #549 and GPU export #421) · **Target:** Adobe Premiere Pro 2026 (26.5.2)

Each hardware feature Premiere uses, per platform, against ours. Implementation detail lives in
[`crates/platform/README.md`](../crates/platform/README.md) (decode / encode backends, the one
crate allowed `unsafe`, [ADR 0001](adr/0001-platform-ffi.md)), the GPU compositor in
[architecture.md](architecture.md), and timings in [performance.md](performance.md).
Hardware paths are always optional: our software codecs are the tested fallback, and a hardware
decoder that fails mid-stream hands over to software (`HybridDecoder`).

Premiere's side is from Adobe's published GPU and hardware-acceleration documentation and
release notes (Premiere does not run on Linux).

## Hardware decode

| Codec / profile | Premiere macOS | **Ours macOS** (VideoToolbox) | Premiere Windows | **Ours Windows** (Media Foundation / D3D11) | **Ours Linux** (NVDEC, VA-API) |
|---|---|---|---|---|---|
| H.264 8-bit 4:2:0 | yes | **yes** | Intel, NVIDIA, AMD | **yes** (Baseline / Main / High) | **yes** (NVDEC; VA-API Intel / AMD / Mesa) |
| H.264 10-bit / 4:2:2 | Apple silicon | **yes** where the chip decodes it | Intel; NVIDIA Blackwell (MXF too) | no (declined; no software fallback, [G3](gaps.md#g3-10-bit-and-422-camera-media-needs-hardware)) | no |
| HEVC Main / Main 10 4:2:0 | yes | **yes** | yes | **yes** | **yes** (NVDEC 8/10; VA-API Main / Main 10) |
| HEVC 4:2:2 10-bit | Apple silicon | **yes** where supported | Intel; NVIDIA Blackwell | no | no |
| VP9 profile 0 / 2 | — | software | — | **yes** | software (VA-API VP9 not yet) |
| AV1 main 8 / 10 | — | software | yes | **yes** | software |
| ProRes | Apple silicon (M1 Pro / Max and later) | software (fast: 318 fps) | software | software | software |
| Field-coded H.264 | yes | no | yes | no | no |

Measured (`cargo xtask bench --hw off|auto`): M4 Pro, CPU per 2160p frame H.264 119 → 4.2 ms,
HEVC 86 → 3.7 ms, 4K plays with no dropped frames at Full; Intel Iris Xe VA-API H.264 2160p
417 → 11.4 ms, HEVC Main 10 2160p 212 → 18.4 ms. Every decoded picture is read back to the CPU
and re-uploaded to wgpu (**no zero-copy** yet), which Premiere avoids.

## Hardware encode

| Codec | Premiere macOS | **Ours macOS** | Premiere Windows | **Ours Windows** | **Ours Linux** |
|---|---|---|---|---|---|
| H.264 8-bit | yes | **yes** (VideoToolbox, opt-in, no B-frames) | Intel QSV, NVENC, AMD | **NVENC only** (opt-in) | **NVENC only** |
| H.264 10-bit / 4:2:2 | yes (Apple silicon) | no | Intel | no | no |
| HEVC Main 8-bit | yes | **yes** (hardware-only format) | QSV, NVENC, AMD | **NVENC only** | no |
| HEVC Main 10 / HDR (PQ, HLG) | yes | **no** (tone-mapped to 8-bit SDR) | yes | **NVENC yes** (HDR10 SEI, `mdcv` / `clli`) | no |
| ProRes | Apple silicon | software only | software | software | software |
| AV1 | — | no | via plug-ins / newer GPUs | no | no |

Measured: VideoToolbox H.264 1080p25 encode 200 fps on M1; a 9:29 timeline exported in 311 s
against 793 s with our software encoder (SSIM 0.990). NVENC HEVC round trip 48.9 dB luma PSNR
(8-bit), 71.1 dB (10-bit) on an RTX 5060.

## GPU rendering

| Feature | Premiere (Mercury Playback Engine: Metal, CUDA, OpenCL) | Ours (wgpu: Metal, D3D12, Vulkan) | % |
|---|---|---|---|
| Compositing, blend modes, transforms | GPU | **GPU** (#32), CPU compositor is the oracle | 90% |
| Accelerated video effects | ~82 of ~92 effects marked Accelerated (reference list `plan/premiere/10-effects-list.md`) | **34 effect ids** (`render::gpufx::GPU_EFFECTS`): colour, levels, blurs, sharpen, crop, transform, flips, vignette, video limiter, ASC CDL, channel mix, Lumetri basic / creative / vignette | ~40% |
| Lumetri curves, wheels, HSL secondary | GPU | CPU | 0% |
| Keys (Ultra Key), masks, distortions, transitions | GPU | CPU (masks have WGSL parity tests, not in the live path) | 10% |
| Export rendering on the GPU | yes | **yes** (#421; Export ▸ GPU Rendering, Auto): 2.4× (1080p) to 3.2× (4K) faster on effects-heavy timelines | 80% |
| Thumbnails on the GPU (26.0) | yes | CPU | 0% |
| Software adapter fallback | warns | silently renders everything on the CPU (#632) | — |
| Intel integrated graphics | supported | start-up crash on Intel UHD (#512) | — |

## Devices and I/O

| Hardware | Premiere | Ours | % |
|---|---|---|---|
| Audio interfaces | CoreAudio, ASIO, WASAPI; multichannel mapping | cpal (CoreAudio, WASAPI, ALSA); no ASIO; device reports #605, #602 | 60% |
| Voice-over microphone | yes | yes (punch-in); macOS privacy declaration missing (#659) | 80% |
| External video monitoring (Mercury Transmit, Blackmagic DeckLink, AJA) | yes | no | 0% |
| Second computer monitor full-screen playback | yes | full screen inside the app window; no separate-display output or floating windows on other screens (#466) | 40% |
| HDR / EDR display | macOS EDR, Windows HDR | no (SDR display of HDR sequences) | 0% |
| Control surfaces (EUCON, Mackie HUI / MCU) | yes | no | 0% |
| LTC timecode input | yes | no | 0% |
| Tape / deck control | legacy | no | n/a |

## Summary

**Hardware dimension: ~45%, 120–200 h.** Ahead of Premiere: hardware decode on Linux (VA-API,
NVDEC) and FreeBSD builds. Behind, by user impact: zero-copy upload; GPU effects beyond 34
(Lumetri curves / wheels / HSL, keys, masks, transitions); Quick Sync and AMD encoders on Windows;
VideoToolbox Main 10 HDR encode; 10-bit / 4:2:2 decode off macOS; HDR display; external video
I/O and control surfaces (need hardware we don't own). Calibration: one decode backend ~3–5 h,
one encoder ~4–8 h ([target-app-parity.md](target-app-parity.md#calibration)).

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | major | Created from ROADMAP.md's performance-and-hardware row and `crates/platform/README.md`; added Linux NVDEC (#549), GPU export and Lumetri on the GPU (#421), Premiere's per-platform hardware list, devices and I/O |


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://grvm.wtpuscm.cn/jiaoliu/demographic-569590.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://gbtx.wtpuscm.cn/pingtai/like-235052.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://reer.wtpuscm.cn/fuwu/premium-708422.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://snqp.wtpuscm.cn/yinqing/webinar-437874.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://bnbb.wtpuscm.cn/yanjiu/follow-859324.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://ppph.wtpuscm.cn/zhizhu/careers-515356.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://acnh.wtpuscm.cn/zixun/contact-101991.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://xpvn.wtpuscm.cn/paiming/integration-770.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://dmlz.wtpuscm.cn/gongsi/tool-809712.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://ixza.wtpuscm.cn/huodong/upload-036211.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://zqnp.wtpuscm.cn/qiye/solution-864288.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://incg.wtpuscm.cn/shangye/services-884817.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://vrau.wtpuscm.cn/pingce/help-799301.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://qyqp.wtpuscm.cn/ziyuan/excellence-568441.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://gxrp.wtpuscm.cn/anli/guide-318886.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://rlqa.wtpuscm.cn/kaifa/luxury-125420.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://jhdv.wtpuscm.cn/gongsi/movie-022806.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://esbt.wtpuscm.cn/ziyuan/server-146517.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://xfib.wtpuscm.cn/hezuo/design-062612.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://csnl.wtpuscm.cn/liuliang/system-051698.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://icbz.wtpuscm.cn/anli/prospect-988564.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://skew.wtpuscm.cn/pingtai/update-299487.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://erxk.wtpuscm.cn/shangye/calculator-760377.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ucne.tcti.cn/anli/travel-86192840.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://ivdn.tcti.cn/liuliang/online-72830267.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://mkmk.tcti.cn/kuangjia/satisfaction-37684274.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://nijz.tcti.cn/paiming/creative-15151935.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://osyd.tcti.cn/fuwu/data-04946302.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://allr.tcti.cn/kaifa/admin-65299788.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://iwuo.tcti.cn/jiaoliu/deal-33950559.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://uwlt.tcti.cn/chanpin/collaborate-77745744.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://cocb.tcti.cn/gongxiang/workshop-61444991.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://mduv.tcti.cn/fenxi/web-88772725.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://jfvp.tcti.cn/jishu/document-00854721.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://bdfy.tcti.cn/suanfa/content-47176544.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://zctj.tcti.cn/fuwu/document-83955526.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://dsja.tcti.cn/xitong/follow-40560783.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://lfbi.tcti.cn/guanjianci/forum-93642957.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://lbls.tcti.cn/chuangxin/restaurant-89027150.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://vboo.tcti.cn/xinwen/food-26089056.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://gezu.wtpuscm.cn/qiye/interface-739147.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://www.mw-wm.com/fuwu/study-27592117.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://www.yx-sf.com/news/15207)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://www.ai-hao123.com/jianzhan/help-06984660.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://qybf.tcti.cn/jishu/device-22089668.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://siap.tcti.cn/jiaocheng/partner-23112961.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://gvqd.wtpuscm.cn/jianzhan/social-122988.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://mlmd.wtpuscm.cn/jiaoliu/consulting-796997.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://gddh.wtpuscm.cn/wenzhang/domain-367702.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://ayfv.wtpuscm.cn/xuexi/identity-571613.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://wwbl.wtpuscm.cn/jiaocheng/expense-723280.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://ccbv.wtpuscm.cn/yanjiu/about-752922.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://ogte.wtpuscm.cn/baogao/movie-179227.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://odeo.wtpuscm.cn/jishu/shopping-152.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://crrt.wtpuscm.cn/chanpin/optimization-680476.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://updk.wtpuscm.cn/anfang/shopping-785951.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://xjot.wtpuscm.cn/liuliang/target-446382.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://spuw.wtpuscm.cn/anli/local-027322.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://srqg.wtpuscm.cn/wenzhang/photo-403735.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://ifla.wtpuscm.cn/jiaoliu/management-784563.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://pdow.wtpuscm.cn/shichang/premium-562486.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://mcyg.wtpuscm.cn/huodong/analysis-733219.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://hzex.wtpuscm.cn/youhua/traffic-728971.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://zfin.wtpuscm.cn/pingtai/status-792490.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://ydfg.wtpuscm.cn/yanjiu/retention-403988.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://pgpa.wtpuscm.cn/shuju/social-904740.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://cndt.wtpuscm.cn/fuwu/website-479332.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://hawq.wtpuscm.cn/yanjiu/movie-225217.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://ozto.wtpuscm.cn/shuju/networking-309067.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://sbeo.tcti.cn/pingce/music-79380301.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://tmrn.tcti.cn/peixun/planning-04037063.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://fsbq.tcti.cn/youhua/research-42832631.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://cjjq.tcti.cn/kuangjia/url-23308670.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://ufgo.tcti.cn/ziyuan/automation-47775360.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://tprs.tcti.cn/fuwu/internet-30137253.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://cgoi.tcti.cn/pingtai/like-68757808.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://pheq.tcti.cn/huodong/solution-61509781.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://qkzh.tcti.cn/wangluo/social-75838215.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://heto.tcti.cn/gongxiang/expensive-88057433.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://exwh.tcti.cn/zhizhu/analytics-39592384.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://ilvn.tcti.cn/chuangxin/page-30062577.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://aiip.tcti.cn/yingxiao/careers-34917086.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://aqbp.tcti.cn/anfang/saving-51263622.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://svpi.tcti.cn/xuexi/login-68230359.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://msdy.tcti.cn/yingyong/investment-14266651.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://fdxm.tcti.cn/zhizhu/lesson-32181727.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://tkee.wtpuscm.cn/guanjianci/revenue-865290.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://www.mw-wm.com/yanjiu/responsive-49849127.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://www.yx-sf.com/news/69056)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://www.ai-hao123.com/liuliang/app-73342759.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://kluv.tcti.cn/pingce/feedback-78949089.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://qths.tcti.cn/gongju/community-26992816.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://xzna.wtpuscm.cn/chuangxin/search-452119.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://ggto.wtpuscm.cn/jishu/tactic-890791.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://ugwr.wtpuscm.cn/yingyong/tag-346542.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://xxmm.wtpuscm.cn/pingtai/version-786026.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://ppwb.wtpuscm.cn/shuju/subject-953313.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://yrwq.wtpuscm.cn/chanpin/sport-885237.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://ijpw.wtpuscm.cn/qiye/subscribe-107734.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://dlsw.wtpuscm.cn/xuexi/growth-140.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://kkvy.wtpuscm.cn/fuwu/presentation-379289.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://wfvh.wtpuscm.cn/zixun/supplier-003286.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://qjgm.wtpuscm.cn/sheji/machine-516527.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://reyg.wtpuscm.cn/jiaocheng/recommendation-911518.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://atte.wtpuscm.cn/fuwu/research-934890.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://qwzg.wtpuscm.cn/zhinan/search-034395.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://vwbg.wtpuscm.cn/chanpin/deadline-706875.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://dtiv.wtpuscm.cn/fuwu/document-732008.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://eoko.wtpuscm.cn/yunsuan/investment-741816.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://imsr.wtpuscm.cn/yunsuan/change-129476.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://yrpp.wtpuscm.cn/zhizhu/profile-964109.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://jkmq.wtpuscm.cn/shangye/company-481763.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://dogp.wtpuscm.cn/xuexi/music-336723.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://kmee.wtpuscm.cn/jiaocheng/loyalty-902261.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://iqpk.wtpuscm.cn/yanjiu/customization-189346.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://gjxv.tcti.cn/ziyuan/audience-48917581.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://ybag.tcti.cn/anli/system-77125456.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://pcvl.tcti.cn/yunying/growth-26393376.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://pqzy.tcti.cn/suanfa/meeting-76648856.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://gfei.tcti.cn/jianzhan/home-84960994.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://tbpv.tcti.cn/tuiguang/goal-81851388.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://hjyw.tcti.cn/youhua/tool-91871697.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://avxz.tcti.cn/chuangxin/training-61456557.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://jytt.tcti.cn/guanjianci/game-13353357.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://qdbt.tcti.cn/guanjianci/chapter-75413507.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://ksan.tcti.cn/yanjiu/machine-51891320.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://dzle.tcti.cn/qiye/experience-57146025.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://bixi.tcti.cn/wenzhang/support-01975500.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://acyq.tcti.cn/wangluo/consulting-51848527.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://vzsv.tcti.cn/yingyong/screen-71774401.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://tyng.tcti.cn/xinwen/achievement-98611374.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://mmwd.tcti.cn/gongsi/navigation-58650200.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://yihd.wtpuscm.cn/yingxiao/sync-039031.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://www.mw-wm.com/shichang/technology-04032389.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://www.yx-sf.com/wiki/92597)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://www.ai-hao123.com/sheji/conversion-79439817.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://cuye.tcti.cn/suanfa/price-27363904.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://fsqj.tcti.cn/gongxiang/productivity-47423544.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://wndh.wtpuscm.cn/wangluo/economy-797299.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://ofge.wtpuscm.cn/xuexi/register-625352.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://deni.wtpuscm.cn/keji/device-539503.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://upqk.wtpuscm.cn/chanpin/conversion-367856.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://jtcr.wtpuscm.cn/liuliang/sales-518226.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://inre.wtpuscm.cn/qiye/feedback-484613.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://mftu.wtpuscm.cn/suanfa/widget-228095.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://rpkb.wtpuscm.cn/xuexi/automation-672.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://bnkl.wtpuscm.cn/zhinan/ai-705172.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://ynrk.wtpuscm.cn/yingxiao/community-580487.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://lanx.wtpuscm.cn/zhizhu/objective-527309.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://qmad.wtpuscm.cn/jiaoliu/revenue-630224.html)

</details>

