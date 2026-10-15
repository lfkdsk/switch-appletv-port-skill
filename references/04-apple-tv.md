# 04 · tvOS 专题：和 iOS/macOS 不一样的地方

Apple TV 是这套移植里约束最紧的目标。每条都给实测数字。

## 4.1 内存上限：2.05 GiB 硬顶

| 项 | 数字 | 说明 |
|---|---|---|
| 单 App jetsam 上限 | ≈ 2.05 GiB | 无 entitlement 时；两次 touch 运行都在累计分配 2,048 MB 后下一步被杀，可复现 |
| entitlement | 提不了 | IML（提物理上限）不支持 tvOS；EVA 只扩虚拟地址空间，不动物理上限 |
| 运行时查询 | `os_proc_available_memory()` | tvOS 27 可信：启动报 2.041 GiB，被杀前一步报 37.8 MiB |
| 虚拟地址（无 EVA） | 单块 mmap 上限 6 GB，累计 7 GB | 7.3 GiB 纹理包不能整包 mmap，改只读索引 + pread |
| Switch 侧对照 | 动森应用内存池 ≈ 3.2 GB | 差距约 1 GB，必须靠压缩/流式化压进 2 GB |

实测占用（Apple TV 4K 三代，A15）：

| 场景 | 峰值 | 备注 |
|---|---|---|
| 首 present（NVN 计数桩，无真实显存） | 710 MiB | 下界，不能推最终可行性 |
| 室内（BC 运行时解码路径） | 1,377–1,494 MiB | first-present 后增量几乎全是 Metal |
| 30 分钟实玩（纹理包启用） | 1,744 MiB | 上岛后平稳无增长，最少剩 353 MiB |

内存优化的两个大头：内存池按需驻留（省约 0.5 GB，见 03）和 BC→ASTC/EAC 纹理包（同一驻留工作集 43.96 GiB → 7.31 GiB，见 4.5）。

## 4.2 W^X 与签名代码页

tvOS 不能把匿名数据页改成可执行页。两条路线的做法：

- 改写后的四个 NSO 模块打进签名 Guest.framework（Mach-O section + vm_remap 映射已签名 RX 页）。
- 或把改写模块嵌进可执行文件的 `__TEXT`，运行时 vm_remap 共享已签名页（这条路线不需要 Guest.framework）。

ctr_el0 的改写必须在签名前离线完成，tvOS 上不能运行时改这条指令。

## 4.3 看门狗与可写目录

- **scene 看门狗**：主线程堵约 20 s 就被 0x8BADF00D 杀。`didFinishLaunching` 里只做立即返回，解包与客体宿主全放后台队列（修后解包 80 s 存活）。
- **只有 Caches/tmp 可写**：App 容器的 Documents 不可写（实测写 `/var/mobile/containers/...` 报错）。输出一律写 `Library/Caches`（NSCachesDirectory）或 tmp。
- **Caches 会被系统清**：Caches 属系统可清理缓存。App 启动做 preflight，关键数据缺失时明确报错，别假设还在。
- **env 覆盖顺序**：`Caches/env.txt` 先读、同名键先到先得，会整体盖掉包内 env。多人共用一台设备时，别人留在 Caches 的 env.txt 能让你的按键脚本/截图配置全部失效——排查「配置没生效」先查这个。

## 4.4 RomFS 放哪

动森 RomFS：103,400 个文件、9.78 GiB。三个方案：

| 方案 | 结论 | 数字 |
|---|---|---|
| `Library/Caches/romfs` | 开发期采用 | 21 片×≤500 MiB + 就绪标记；推送 461–616 s；设备侧展开 78.7–80.3 s；staging 核对文件数/字节数后原子发布 |
| App bundle 内 | 可安装、可读、不被 Caches 清理；代价是包体与安装时间 | bundle 10.7 GB；构建 79 s；安装 593 s |
| 局域网 HTTP | 未实现 | 前两个都通，不需要 |

直接逐文件 copy 不可行：CoreDevice 小文件事务慢，relay 断线还留残文件。

## 4.5 A15 无 BC 纹理解码 → ASTC/EAC 包

A15 不支持 BC 纹理，不替换就只能运行时解成未压缩像素，显存直接爆。

| 项 | 数字 |
|---|---|
| 源规模 | 116,751 张 BC 实例；内容去重后 81,741 张、11.41 GiB |
| 产物 | balanced 包 7.315 GiB，81,741 条，0 失败；编码 376 s（15 并行） |
| 键 | 原纹理内容哈希（NVN 层看不到 BNTX/路径，只看得到池里字节）；离线工具与运行时实现逐字节一致 |
| 显存收益 | 同一驻留工作集：运行时解 BC 43.96 GiB → 包 7.31 GiB（6.0 倍），比原始 BC 还省 1.56 倍 |

格式策略：

| 源格式 | 目标 | 原因 |
|---|---|---|
| BC1 | ASTC 8x8 | PSNR p50 42.52 dB |
| BC3/BC7 | ASTC 5x5 | PSNR p50 44.40 dB |
| BC5_SNORM（法线图，占 BC 字节 41%） | EAC_RG11Snorm | ASTC LDR 无有符号模式，转 ASTC 会把 [-1,1] 压成 [0,1]；EAC 块字节相同、语义精确、不改着色器 |
| BC4 | EAC_R11 | PSNR p50 52.79 dB |

两个易漏点：视图里重指定过格式的纹理要按底图格式建视图（见 03）；整包 mmap 放不下，改只读索引 + pread。

## 4.6 CAMetalDisplayLink 不回调

症状：框架缺省第 3 帧把出图切到 CAMetalDisplayLink，Apple TV 上回调从不来，游戏在兜底时钟空跑，实际出图停在第 3 次，画面冻在黑帧。

修法：tvOS 缺省让出图线程自取 drawable（env 可改回）。这类「Apple TV 上特有、模拟器和 Mac 都正常」的坑只能真机发现。

## 4.7 手柄

- **按位置映射**：Apple 的 GCExtendedGamepad 面键属性是固定几何位置，不是键帽字母。映射：`buttonB`→A、`buttonA`→B、`buttonY`→X、`buttonX`→Y、`buttonMenu`→+、`buttonOptions`→−。PS5 ○=A、×=B；Switch Pro 按位置（与标签一致）。
- **Siri Remote 兜底**：只在没接全键手柄时接入——触控板→左摇杆，按下→A，Menu→B，播放/暂停→+。
- **震动**：HD Rumble 数值清洗（振幅 [0,1]，低频/高频范围）→ intensity/sharpness，经 Core Haptics 按左右握把分侧。
- **验收状态**：动森移植的手柄验收在 Apple TV 上用真手柄做过（PS5 DualSense 与 Switch Pro），配对路径：设置 → 遥控器与设备 → 蓝牙。只在模拟器上验过不算数。

## 4.8 真机截图回传

Apple TV 没有现成截屏命令，流程：

1. App 自己把 Metal 最新一帧读回，写 PNG + JSON 到 `Library/Caches`（可用 env 配置截图帧号/间隔/输出尺寸）。
2. `devicectl device copy from` 取回（单文件约 1.4 s，App 不受影响）。
3. **取回的 PNG 必须自己打开看**，并做语义像素断言（某区域应是某颜色/某形状），不能只比文件哈希。

辅助：App 写完截图或到时间后自己退出，避免运行时 devicectl 文件服务被拖慢——游戏跑着时 copy from 会越来越慢直至卡死（CPU 满载 + 刚写约 18 GB Caches）。

## 4.9 多任务抢同一台设备

只有一台 Apple TV 时：

- **lease 串行**：派构建/真机任务带独占租约，同一时刻只有一个任务用 Mac 构建/真机。
- **同包名互顶**：各任务包名相同时，`devicectl --terminate-existing` 会把对方正在跑的 App 顺手干掉；并行必须给不同包名。动森项目里 TV1 的跑局被 OPT1 同包名安装顶掉过。
- **共享构建目录互顶**：多任务共用一个 build 目录会互相覆盖出包（未签名包顶掉签名包，被 codesign 拦下）；按工作树分目录。
- **装包前互检**：部署脚本装包前检查设备上有没有别的「我们的」App 在跑，有就等（缺省 600 s）或显式不等。
- **设备状态**：`devicectl list devices` 在 `available (paired)` 与 `connected` 间跳，两种状态下 install/copy/launch 都正常，闸门只拦 unavailable。

## 4.10 模拟器跑不了图形层

tvOS 模拟器 GPU 无 Metal 4、非统一内存、参数缓冲仅 tier-1，NVN→Metal 层起不来。图形相关验收只能真机；模拟器只适合验装载/服务/逻辑。
