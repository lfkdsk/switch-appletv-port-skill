# 06 · 音频：audren 两条路与「没声音」的排查

## 6.1 audren 两条路，都走通过

游戏的音频走 Horizon 的 AudioRenderer（audren）：游戏每帧生成一段命令缓冲（voice 状态、mix、effect），宿主执行它并把 PCM 喂给 AudioOut。宿主实现有两条路：

| | 自写 audren（nx 路线） | 跑 SDK 自带渲染器（smbw 路线） |
|---|---|---|
| 做法 | 按 Ryujinx（MIT，固定 commit）实现命令解析与混音，纯 C11 | 不实现，进程内跑游戏 SDK 自带的渲染器（CPU 渲染 + 手动执行），宿主从命令表找 DeviceSink 取混音缓冲夹成立体声 |
| 优点 | 行为自己掌控，可 fuzz 验证 | 不用逆向混音语义，SDK 行为保真 |
| 代价 | 要实现 REV1–15 的线格式与语义 | SDK 20 有两处适配（见 6.4、6.5）；渲染线程竞态要处理 |
| 验证 | 语义测试 + fuzz 58,362,601 次 update 无崩 | 真机量 RMS（见 6.3） |

选型参考：第一方游戏用新 SDK、命令格式新（动森用 REV15），自写要补的语义多；第三方游戏 SDK 旧，自写更省事。不管哪条路，下面的坑都适用。

## 6.2 自写 audren 的要点

- 调度：48 kHz、240 帧、5 ms 量子；CLOCK_MONOTONIC absolute deadline pump；渲染帧/量子/event/underrun/overrun 计数。
- 能力面：PCM16 + GC-DSP ADPCM（predictor/scale/history/loop context，与独立 golden 逐样本 bit-exact）；4 个 wave buffer 环、有限/无限 loop、EOS、pause/resume/flush、starvation 补零；voice→submix DAG→final mix；逐帧 volume ramp；9 种 effect 全解析但按 processing_order 做 dry bypass。
- REV15 新增：splitter（destination + bias ramp，上限 256/1024）、mix buffer 全局预算 102（动森实际值）、voice SRC/pitch（线性插值，spec 允许）、depop。
- REV14/15 的 revision 号只是能力位，不改线格式——别被 revision 号吓到，先量实际段大小。

## 6.3 「回调在跑 ≠ 有声音」：要量 RMS

动森移植在 Apple TV 上踩过的最典型音频坑：声卡约 358,000 次回调、欠载 0、补零 0 ms，看起来一切正常，但日志里「已渲 200 帧，其中有声 0 帧」——每帧的 PCM 在命令校验处被丢弃，送进声卡的全是零。

取证方法（修后实测）：

| 指标 | 数字 |
|---|---|
| 有声帧（302 s 开场） | 58,586 / 60,000 |
| RMS | 最小 −44.2 / 中位 −28.9 / 最大 −23.5 dBFS |
| 频谱平坦度 | 0.002（白噪声约 0.56，乐音远小于 0.1） |
| 左右相关 | 0.94–0.96 |
| 240 采样帧边界跳变比 | 约 1.0（拼帧无接缝） |

结论：回调次数、欠载计数都不证明有声；要量 PCM 的 RMS，再看频谱形状确认是音乐不是噪声，再人耳听一遍。修前修后用同一套指标对比。

复审（RAUD1）又抓出两条比 bug 本身更通用的：

- **实时回调里不能打日志。** AudioUnit 的 render callback 里调 `LOG`（`fprintf` 加 stdio 锁）会让实时线程可能阻塞在锁上，日志频率低也不改变这一点。修法：电平统计改双桶原子——回调里只累加平方和/峰值/样本数，普通线程切桶、等旧桶写者退出后再计算再打日志。
- **RMS 数字要能由仓库脚本复算。** 复审用保留的 PCM 独立复算，发现报告里一个「全程 RMS −17.7 dBFS」和原始 PCM 对不上（实为 −27.028 dBFS）——从日志的 2 s 行拼全程时拼错了一行。修法：分析脚本进仓库（读 s16le 交错 PCM，按 `audioop.rms` 的整数口径换算，不依赖已从新版 Python 移除的 `audioop`），报告给完整命令，复审照命令重跑。

## 6.4 SDK 20 的 DeviceSink 命令号：0x13 → 0x14

症状：Apple TV 上没声音，声卡回调正常跑、欠载 0（接上一条）。

根因：SDK 20 把 DeviceSink 命令号从 0x13 改成 0x14。宿主写死 0x13，取到的是另一条命令（0x13 一帧几十条），字段校验不过就整帧丢弃。保护性校验挡住了越界读，所以只是没声音，没有崩。

修法：命令号做成游戏档案项（动森档案 0x14，缺省 0x13 不影响旧游戏）。命令表实测 74 条，DeviceSink 是末尾一条（0x148 字节），表头布局与 SDK 16 相同，变的只有命令号。

另一个排查项：Apple TV 连着蓝牙音频设备时声音只从蓝牙出、电视喇叭不响——「没声音」先查 `AVAudioSession` 的 `currentRoute.outputs`，别只盯宿主。

## 6.5 SDK 20 音频渲染线程竞态

症状：岛上走几分钟必崩在音频渲染线程，栈里带命令头魔数。

根因：SDK 20 下宿主的追渲（手动执行渲染）调用公开的 `ExecuteAudioRendererRendering`，不经宿主接管层——不持锁、不计渲染次数，与渲染线程并发写同一命令缓冲。

修法：SDK 20 路径渲染线程持锁、计渲染次数、Update 互斥；追渲从每次更新降到全程 3 次。修后连续 2,632 s 无崩溃。

## 6.6 REV15 线格式与测试陷阱

动森用 REV15，自写路线的真 REV15 线格式与旧版不同：voice 段 0x188（旧版 0x170）、SNDD 0xA8（旧版 0x90），各差 0x18。实现按旧版解，合法 REV15 包被拒绝，游戏走 SetTerminateResult + svcBreak，首帧前崩。

修法：按 update/behavior revision 选 v1/v2 布局；splitter 段从「SNDD」魔数反推 stride；过渡期 host 侧保留带日志的静音 fallback（每 1000 次打一行计数）。

测试陷阱（这条比 bug 本身更值钱）：REV15 的测试包内容实际是 REV14 结构，fixture 与生产解析器共享「voice 永远 0x170」的假设，测试自洽通过，fuzz 也变异不出合法 0x188 包。修法：

- 测试布局表与生产常量物理隔离；
- 字节级断言尺寸/偏移/stride；
- 状态字段至少两次 update 测试（跨 update 保留语义）；
- 反向变异自检：改坏一处生产代码，确认测试变红。

## 6.7 影片音频：游戏自己开 AudioOut，宿主只解 PCM

动森开场影片是 WebM（VP9 + Vorbis）。关键事实：**游戏不指望系统替它播**——它把音轨当成第二条 MediaCodec 流交给宿主，拿回 PCM16 后自己开一路 `nn::audio::AudioOut` 追加，画面也按宿主回的时间戳对齐。所以宿主要做的不是「播声音」，而是拆封装多报一条音轨、多一个 PCM 解码器。

实现要点：

- WebM 两轨按文件顺序交错进一张样本表，如实报每个样本属于哪一轨；游标不能停在没选的轨道上，否则游戏的取帧线程拿到不认识的轨道号会卡住。
- Vorbis 用 vendored libvorbis（BSD）解成 PCM16，Configure 时从轨道配置取三个设置头。
- 安全开关：编码不是 A_VORBIS、没有 CodecPrivate、声道数/采样率不对，任一条都退回「只报视频轨」（影片照放、无声）——报了却解不出，游戏会一直等，宁可不报。
- 影片播放器每帧调 `nn::audio::SetAudioOutVolume`，不接管必崩（宿主的 AudioOut 是不透明句柄，SDK 自己的实现一读就是空指针）；接管后音量乘在送声卡的路上。

踩过的坑：解码线程无条件按 VP9 解所有流，音轨 203 包进去、0 帧出来，影片 3 s 后被游戏放弃——按流类型分流到 Vorbis 解码器才好。

实测（Mac，整段 65.47 s）：RMS −23.3 dBFS、峰值 −8.6 dBFS、谱平坦度 0.000（是音乐不是噪声）；视频 1959 帧、音频 4564 包，一包不差；影片期间游戏自己停掉 audren 混音（过场声音由影片自带，和原机一样），影片前后照常。tvOS 包能编出来；真机听感未验——真机上唯一 Mac 验不到的点是两路 AudioOut 并存时的欠载与蓝牙路由，留待真机复测。

## 6.8 hwopus

动森导入 8 个 `nn::codec` Hardware 变体（语音聊天用）。两个做法：改写层把构造函数替换成 SDK 自带软件解码器（同虚接口、对象放得下），或宿主 libopus HLE。离线语义阶段保持 UNIMPL 即可。
