---
name: switch-appletv-port
description: Distilled playbook for porting Nintendo Switch games to run natively on Apple platforms (iPhone/iPad/Mac/Apple TV) without CPU emulation — covering ARM64 instruction rewriting, HLE at the import or IPC layer, NVN→Metal, offline shader translation, and tvOS bring-up, with a symptom-indexed pitfall catalog for black-screen, crash, and no-audio failures; use when starting such a port or debugging one.
---

# Switch 游戏原生移植到 Apple 平台

## 什么时候用

- 把一个 Switch 游戏（NSO/NSP）原生跑在 iPhone、iPad、Mac 或 Apple TV 上
- 移植中遇到黑屏、崩溃、没声音、内存被杀、文字不显示、帧率上不去
- 要把 NVN 图形调用接到 Metal，或离线翻译 Maxwell 着色器
- 想了解 d2r-mobile / nxnative 这类项目的做法并避开它们踩过的坑

不适用：整机模拟器（那是另一套技术）；GameCube/Wii 时代游戏（没有同指令集优势，见 references/08 的 bellpad 对照）。

## 心智模型

1. **不模拟 CPU。** Switch 的 CPU 和 Apple Silicon 都是 ARM64，游戏的 NSO 机器码可以原样执行。模拟器的性能代价在这里没有必要；真正要做的是把 Apple 内核下跑不了的少数指令离线改写掉（见 01）。
2. **改写是离线的、对账的。** 改写器只碰几类指令（x18 影子寄存器、TLS 指针、svc 系统调用、计数器读数、缓存类型寄存器），改完要和反汇编扫描的数字逐条对账闭合。
3. **系统服务靠宿主 HLE。** 游戏不直接碰内核，它通过 Horizon 的 svc/IPC 拿服务。宿主顶替这些服务有两层做法：在 SDK 导入层顶替，或在 svc/IPC 服务层顶替。两层都有人走通过，选型依据见 01。
4. **图形是记录-回放，不是即时翻译。** 游戏只经 `nvnBootstrapLoader` 一个符号拿 NVN 函数；宿主把 NVN 命令录下来，在提交时重放成 Metal。着色器全部离线翻译好，运行时按哈希查库。
5. **tvOS 是另一种生物。** 单 App 内存上限约 2.05 GiB、代码页必须签名、只有 Caches/tmp 可写、A15 不认识 BC 纹理、模拟器跑不了图形层。04 整章讲这个。
6. **内容边界是硬线。** 仓库里不放游戏代码、资源、keys；工程经验、API 名、服务名可以放。这条线既是合规要求，也决定了哪些证据能进报告。

## 工作流：从 NSP 到第一帧

| 阶段 | 干什么 | 验收 |
|---|---|---|
| 解包 | NSP→NCA→ExeFS/RomFS；hactool 遇新 BKTR 报错就换 LibHac | 四个 NSO + npdm 齐全 |
| 普查 | 导入符号、服务访问、NVN API、纹理格式与数量 | 两份独立工具的数字互相吻合 |
| 选型 | 导入层 HLE 还是服务层 HLE；单游戏还是 game profile | 写下依据，01 有决策表 |
| 改写+对账 | 跑改写器，和扫描数对账；loader 补 DT_RELR | 未解析强导入 = 0 |
| 合成客体 | 自己编一个 NSO 跑通全链路 | 三平台像素断言过 |
| 服务 HLE | sm + HIPC 框架 + 各服务；联网服务给离线语义 | npdm 里每个服务名都有会话 |
| 着色器 | 全量离线翻译；GS 走 compute 降级 | 全部出库，失败数 = 0 |
| 纹理包 | BC→ASTC/EAC（A15 必须） | 0 失败，视图换格式一起翻 |
| 首 present | 装载+服务+NVN 桩先跑通 | 第一次 present 状态 0 |
| 第一帧 | 真实 NVN→Metal 回放 | 画面非纯色，语义像素断言过 |
| 可玩 | 内存压进 2.05 GiB、手柄、存档 | 真机连续玩不崩 |

细节与每阶段的坑见 references/02。

## 出问题先查哪

| 症状 | 先看 |
|---|---|
| 黑屏，但有 present | 03 的图形节：CallCommands 未回放、NEAR_IS_MINUS_W、ReportCounter、条件清除、typed view |
| 启动即崩 / 跑几帧崩 | 03 的装载节：DT_RELR、core=-2、堆大小算错；03 的服务节 |
| 电视上文字不显示，Mac 正常 | 03 的 `texture2d_array` vs 2D（A15 读 0） |
| 没声音，但音频回调在跑 | 06：回调次数不证明有声，要量 RMS；SDK 20 的 DeviceSink 0x14 |
| 真机被系统杀掉 | 04：2.05 GiB 上限、按需驻留、纹理包 |
| 首帧要几分钟 | 05：大概率不是 PSO，是文件系统大小写查找 |
| 画面冻在黑帧不出图 | 04：CAMetalDisplayLink 在 Apple TV 上不回调 |
| 帧率低 | 05：先量 CPU 提交时间和 GPU 时间，瓶颈多半在 CPU 提交 |

## references 地图

| 文件 | 回答的问题 |
|---|---|
| 01-route.md | 为什么这样选路线 |
| 02-bringup-checklist.md | 第一步做什么、每步怎么算完 |
| 03-first-frame-pitfalls.md | 这个症状别人怎么修好的 |
| 04-apple-tv.md | tvOS 和 iOS/macOS 有什么不一样 |
| 05-perf.md | 慢，慢在哪，怎么修 |
| 06-audio.md | 声音这一摊 |
| 07-agent-workflow.md | 多个 agent 怎么分工做这种项目 |
| 08-community.md | 社区谁做过什么、借了什么 |

## 硬规矩

- **不碰游戏内容。** 仓库、报告、日志里不出现游戏代码字节、反汇编、着色器源码、keys。可以写 API 名（`nvn*`、`nn::*`）、服务名、命令号。
- **数字只写实测的。** 引用数字要能指到具体报告；进行中的标「进行中」；写不准的不写。
- **真机和模拟器分开记。** 模拟器过了不等于真机过了；宣称真机通过要附命令、退出码、取回的日志/截图路径。
- **截图要自己看。** 取回的 PNG 必须打开看并做语义像素断言（某区域应是某颜色），不能只比文件哈希。
- **声音要量电平。** 回调在跑、欠载为 0 都不证明有声；要量 PCM 的 RMS，最好再人耳听。

## 给 agent 的操作提示

- 读项目报告先看末行 VERDICT（DONE/PARTIAL/PASS/FAIL），再看表格。
- 移植新游戏时，先跑普查脚本出 survey，再动手写实现；02 有清单。
- 真机只有一台时，所有设备操作串行做，别并行装包（会互相顶掉，见 04）。
- 修图形 bug 的套路：先 TEXDUMP 看纹理在不在，再 PASSDUMP 看绘制发没发，再对着色器声明类型查视图类型（03 有完整案例）。
