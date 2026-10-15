# 03 · 第一帧前的坑目录（症状 → 定位 → 修法）

每条都是动森移植里实际撞到并修好的。来源是两条路线的工程报告（nx 路线 P2a/P2b/P2c、smbw 路线 Z1/T2/T3/OPT1 等）。按症状查，不按顺序读。

## 装载 / 改写 / 内核

| 坑 | 症状 | 定位 | 修法 |
|---|---|---|---|
| BKTR | hactool 解更新 NCA 报 `Invalid BKTR layout!` | hactool 对新 BKTR 布局的已知问题 | 换 LibHac 0.19.0 写等价 CLI |
| DT_RELR | 装载后 SIGSEGV，错地址是链接期原值，重定位计数为 0 | 新 SDK 用 DT_RELR 打包相对重定位，loader 只认 RELA | loader 加 RELR 应用；动森 391,693 个槽 |
| ctr_el0 | sdk 缓存行查询代码在 Apple 上 SIGILL | Apple 不向用户态暴露缓存拓扑 | 离线改写为常量（128 B 缓存行），签名前改 |
| W^X | tvOS 上不能把匿名页改可执行 | tvOS 代码签名约束 | 改写后模块进签名 framework / 嵌进 `__TEXT`，运行时 vm_remap 共享已签名 RX 页 |
| 场景看门狗 | 启动约 20 s 被系统杀（0x8BADF00D） | 首次解包堵在主线程 | 主线程立即返回，解包与客体宿主全放后台队列（修后解包 80 s 存活） |
| 小文件传输 | 10 万个 RomFS 文件直接推设备，5 分钟推不动还断线留残文件 | CoreDevice 逐文件事务 | 串流归档切 500 MiB 分片 + 就绪标记 + 设备侧 staging 原子发布 |
| 堆大小算错 | 启动时往宿主代码地址建堆，SIGBUS | `GetInfo(UsedMemorySize)` 没算已装载模块，游戏拿 total−used 当堆 | 已用量 = 堆 + 映像字节 + 64 MB |
| core=-2 | 第 15–63 帧崩，坏指针跨 run 逐字节不变 | `svcCreateThread(core=-2)` 未解析，`-2` 被当数组下标 | IdealCoreUseProcessValue(-2) 解析为真实 core 0 |
| 改写背锅嫌疑 | 怀疑改写破坏原子序列/堆未清零 | 独占区间审计：改写点落在 LDXR/STXR 区间内 = 0；坏值确定性 | 排除改写/堆/随机源，指向 core=-2 这类逻辑坑 |

## 服务 / IPC

| 坑 | 症状 | 定位 | 修法 |
|---|---|---|---|
| 预选用户 | 启动停在用户选择 | `PopLaunchParameter(PreselectedUser)` 返回 NotAvailable | 提供本地预选用户 IStorage |
| 严格 CMIF ABI | 真机请求被框架 ABI 校验拒绝 | 游戏实际请求的 payload/PID/OutPointer 尺寸与桩不符 | 按真实编码修正；对全部命令做 ABI 审计（动森审 105 条） |
| NVN API 版本 | 空指针 | NVN API major 53 不满足游戏要求 | 返回 major 55 |
| SetShadersExt 布尔 | 空地址 | `nvnProgramSetShadersExt` 是 bool，通用桩返回 0 | SetShaders/Ext 返回 1 |
| NVN 对象表耗尽 | 启动期崩溃 | 2,048 固定对象表不够 | 动态链式哈希 + Finalize 回收 |
| 周期事件不 signal | 窗口创建后画面静止 | VI vsync、audren 事件永不置位 | 周期事件（16.67 ms / 5 ms）deadline 接入 WaitSynchronization |
| 死锁 | 主线程无限 WaitForAddress | 非周期事件（audren cmd 12、nfp cmd 23）永不置位，guest 无限等 | 补事件唤醒覆盖；句柄集含周期事件时降级为定时等待（缓解） |
| 服务直连 | 游戏自己 hipc 连 olsc:u，连不上就中止 | 不经 SDK 服务库的直连 + domain 报文 | 三个 hipc 入口强制接管，假会话就地应答 CMIF（含 16 字节 domain 头） |
| 时钟快照 | nn::time 中止 | ClockSnapshot 走 time 服务 | 按 0xD0 字节布局填快照 + 时区规则 |
| applet 资源归属 | OpenAudioRenderer 里 Abort | SDK 20 要求 GetAppletResourceUserId 非 0 | 给固定非 0 值 |
| 共享字体 | nn::sm::Initialize 中止 | RequestSharedFontLoad/GetSharedFont 没接管 | 空操作 + 按 0x18 步长回填；或宿主字体快照 |
| 六轴 / friends ABI | hid cmd 66 UNIMPL；friend:u cmd 0 被拒 | 缺 StartSixAxisSensor；friends 入口错误要求 PID | 补 cmd 66/67；去掉 PID 要求 |
| HIPC Close 后 svcCloseHandle | `svcCloseHandle 0xe401` | HIPC Close 已提前删 session | 一次性 closed-session 记账 |
| 离线服务语义 | 联网功能卡住 | 35 个联网服务需要「无网络」语义 | bsd Connect 回 ENETUNREACH、nifm 回 NoInternetConnection、ssl 回 NoConnection 等；未知命令响而不静默 |

## NVN / 图形

| 坑 | 症状 | 定位 | 修法 |
|---|---|---|---|
| CallCommands 未回放 | 首 present 只有 3 个 pass / 0 个 draw | `nvnCommandBufferCallCommands/CopyCommands` 未录 | 录成 OP_CALL 递归回放；命令内存按 4 KiB 宿主段补充并拒越界段 |
| runtime shader 入口名 | shader 链接错 | 打包器把 SPIR-V 反射入口 `main` 当 metallib 入口 | 从 MSL 的 vertex/fragment/kernel 声明取实际入口 |
| 清单外 shader | 缺 2 个 shader | 静态清单之外的运行时捕获 | 运行时捕获 → 翻译 → runtime 包 |
| DrawElementsBaseVertex | 每帧只有 1 个 draw | 缺 `nvnCommandBufferDrawElementsBaseVertex` | 补齐 indexed/base-vertex 录制与回放 |
| 顶点属性号 | shader 资源全错 | 顶点属性把 SPIR-V id 当 NVN location；cbuf/C1/driver bank、texture、storage 槽位不符 NVN ABI | 按 NVN ABI 重建 reflection，运行时补 c1/driver0/driver2/ubo_ptr |
| TextureView 16 B | 合成矩阵出现 inf/nan | 游戏的 NVNtextureView 只有 16 B，宿主 36 B 的 defaults 写穿相邻视口矩形 | view_t 压为严格 16 B + 静态断言固定 ABI |
| NEAR_IS_MINUS_W | 合成 quad 被 Metal 裁掉 | NVN 默认深度约定接受 z∈[-w,w]，Metal 只接受 [0,w] | 打包所有 VS 时在返回前做 z=(z+w)*0.5（动森重打 3,117 个 VS） |
| ReportCounter {0,t} | UI 已画好但最终帧仍黑；条件清除全放行擦掉同帧 UI | TIMESTAMP 错写 {t,t}；Maxwell ZERO report 应写 {0,t} | post-op 写回 {counter=0, timestamp=t}，不反转条件、不跳清除 |
| 条件清除 | 柜台背景隔几帧整帧黑；TBDR 开时 3D 背景整张黑 | 计数归 0 放在 GPU 完成回调，下一帧编码时还读到 1，清除多执行一遍；写回前瞻把条件渲染下未执行的清除当「后面会盖掉」 | 计数字段在编码时按序写 0，完成回调只写时戳；写回前瞻不把条件渲染下的清除记进索引 |
| snapshot 抢跑 | 帧内快照偶尔读到旧内容 | snapshot 在 legacy queue 抢跑于尚未提交的 replay command buffer | snapshot copy 留在同一 replay command buffer |
| 字体 CPU 写线性贴图（排除项） | 对话框中文与名牌字不显示 | 曾怀疑 A15 线性贴图/CPU 写可见性；GPU 读回五条路逐项排除——线性贴图本身没问题 | 真因是下一条 typed view，别在这条上耗时间 |
| texture2d_array vs 2D | 电视上文字不显示，Mac 正常；3D 场景暗 15–30% | 文字片元着色器把字形缓存声明成 `texture2d_array` 从第 0 层取样，宿主绑的是 2D 视图；Metal 里类型不一致是未定义行为，M 系列碰巧读对、A15 读出 0，alpha≤0 全 discard | 按着色器声明类型在同一存储上另建视图（2D↔2D 数组、立方↔立方数组、立方→2D） |
| alpha 测试 | 树冠/草镂空画成黑块 | NVN ColorState 固定功能 alpha 测试（GEQUAL 0.16），Metal 没有 | 离线给片元着色器注入 discard（按函数常量），回放把 alpha 函数与参考值并进管线键 |
| 短控制段 shader | 47 个「着色器不在映射表」 | agl/gsys 包控制段只有 0x848/0x854 字节，且控制段后紧跟装载期改写的指针，按固定长度取哈希每次键都不同 | 控制段自报长度短于 0x880 时只哈希它自己（表 6,536→9,742） |
| 计算 shader 无头 | 计算 shader 拆不出来 | 动森 BNSH 计算代码区指令从 +0 起，没有 0x30 头 | 拆分脚本计算阶段允许无头 |
| Ryujinx 翻译崩溃 | 18 个 CS 翻译抛 InvalidCastException、1 个 PS KeyNotFound | AstOptimizer 把表达式内联回 Call 参数；被移除的 local 仍作 call output pointer | CallArgumentPrepass：非 local 表达式先 materialize 成临时 local，pointer 参数重新加入 Locals |
| GS 无法编译 | 4 个 GS 生成的 MSL 有 `unknown` 类型 | Metal 无 geometry stage | compute prepass 三入口降级（见 01） |
| texpack mmap 放不下 | 7.3 GiB 纹理包整包 mmap 失败 | tvOS 无 EVA 时进程地址空间累计约 7 GB | 只读索引（头+条目表约 3.9 MB）+ 数据按需 pread |
| 纹理视图换格式 | 替换包最易漏的一条 | 游戏调上千次 nvnTextureViewSetFormat，视图里重指定的 BC 格式必须一起翻 | 视图按底图 ASTC/EAC 格式建；GetStorageSize 照旧返回原始 BC 字节 |
| BC5_SNORM 不能走 ASTC | 法线图转 ASTC 后采样值 [-1,1]→[0,1]，n.z 全错 | ASTC LDR 无有符号模式，改采样要改着色器 | 走 EAC_RG11Snorm（块字节相同、语义精确、Apple GPU 全系支持、不改着色器） |
| render pass 中途深度/模板清除 | 红行「通道中途的深度模板清除未实现」，后续绘制拿旧深度测试 | 游戏在 pass 中途发深度/模板清除 | 结束当前 pass，把清除挂到下一个 pass（颜色附件照旧 load）；清除 scissor 不覆盖深度目标的情形保留红行 |
| 内存池过早驻留 | 未使用的池也占有线内存，4 GB 机型启动被杀 | 池创建时就放进驻留集 | 首次 GPU 取址时才驻留并当场提交（上游实测省约 0.5 GB） |
| dirty-page tracker 竞态 | 游戏线程在缺页处理器里无限自旋 | 置只读时先标 PROT 再 mprotect，并发的「写前解保护」把状态改乱 | 状态机 NONE→BUSY→mprotect→PROT，等待者 BUSY 清除后重查，上限 2 s 报红 |
| 显示节拍不回调 | 画面冻在黑帧，出图停在第 3 次 | CAMetalDisplayLink 在 Apple TV 上回调从不来 | 出图线程自取 drawable（env 可改回） |

## 音频

| 坑 | 症状 | 定位 | 修法 |
|---|---|---|---|
| REV15 线格式 | 合 main 后首帧前 svcBreak（0xc99） | 真 REV15 的 voice 段 0x188、SNDD 0xA8，比旧版各多 0x18 | 按 update/behavior revision 选布局；splitter 从「SNDD」魔数反推 stride；过渡期带日志回退静音兼容包 |
| 测试与生产同错假设 | REV15 包测试自洽通过但线格式是错的 | fixture 与生产共享「voice 永远 0x170」假设，fuzz 变异不出合法 0x188 包 | 测试布局表与生产常量物理隔离；字节级断言尺寸/偏移/stride；状态字段至少两次 update 测试 |
| DeviceSink 命令号 | Apple TV 没声音，PCM 全零 | SDK 20 把 DeviceSink 命令号从 0x13 改成 0x14，宿主取到另一条命令，字段校验不过整帧丢弃 | 命令号做成游戏档案项（缺省 0x13 不影响旧游戏） |
| 渲染线程竞态 | 岛上走几分钟必崩在音频渲染线程 | SDK 20 下宿主追渲不经锁，与渲染线程并发写同一命令缓冲 | 渲染线程持锁、计渲染次数、Update 与渲染互斥；修后 2,632 s 无崩溃 |

## 排查套路

图形黑屏的标准排查顺序（来自 nxnative 的 docs/agents 与动森实践）：

1. **TEXDUMP**：纹理数据在不在？不在 → 查纹理上传/替换包/视图格式。
2. **PASSDUMP**：绘制命令发没发？没发 → 查录制/回放、CallCommands、DrawElementsBaseVertex。
3. **对着色器声明类型查视图类型**：发了但画不出 → 查 typed view（2D vs 2D 数组）、NEAR_IS_MINUS_W、alpha 测试。
4. **读 VS 输出**：几何对不对 → 查顶点属性号、深度约定。
5. **条件渲染/清除**：时有时无 → 查 ReportCounter 与条件清除的写回时序。

每个排除项都要留证据（读回字节数、像素统计），别凭感觉排除——动森的文字 bug 就在「线性贴图」上浪费过一轮，真因是 typed view。
