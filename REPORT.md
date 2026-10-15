# SK1 报告：Switch 游戏原生移植 skill 仓库

一句结论：skill 仓库已建成，SKILL.md + 8 份 references + README + LICENSE，全部素材来自动森两条路线的工程报告与四个社区项目的公开内容，不含任何游戏代码/资源/keys。

## 交付物

| 文件 | 行数 | 内容 |
|---|---|---|
| `SKILL.md` | 83 | 入口：触发场景、6 条心智模型、工作流阶段表、症状分诊表、硬规矩 |
| `references/01-route.md` | 94 | 路线选型 |
| `references/02-bringup-checklist.md` | 118 | NSP+keys → 第一帧检查单 |
| `references/03-first-frame-pitfalls.md` | 86 | 坑目录（35 条，症状→定位→修法） |
| `references/04-apple-tv.md` | 110 | tvOS 专题 |
| `references/05-perf.md` | 87 | 性能 |
| `references/06-audio.md` | 74 | 音频 |
| `references/07-agent-workflow.md` | 82 | 多 agent 推进方式 |
| `references/08-community.md` | 94 | 社区项目 |
| `README.md` | 62 | 仓库首页：截图、安装、免责、致谢 |
| `LICENSE` | 21 | MIT |

每个文件写完即提交，共 11 个提交，无 Co-authored-by 尾注。

## 各文件素材来源

| 文件 | 主要素材 |
|---|---|
| SKILL.md | 全部素材的蒸馏；CONTEXT.md 的硬规矩 |
| 01-route.md | acnh-tv findings：S0、L1、X1、P2a、G0–G3、M0；acnh-smbw findings：Z1；社区：d2r-mobile README/docs、nxnative docs/agents、nx-port README |
| 02-bringup-checklist.md | acnh-tv findings：X1、L1、B1、H1、SV1–SV3b、AU1/AU2、G0–G3、M0、X2、I1、T1、P2c；acnh-smbw findings：Z1、T2、PW1、TV1 |
| 03-first-frame-pitfalls.md | acnh-tv findings：P2a、P2b、P2c、L1、T1、I1、SV3b、G1、G2、X2、AU2b；acnh-smbw findings：Z1、T2、T3、OPT1、AUD1；smbw-mobile 提交 7058f43/9a9d1eb/a37f2da/eca497a（经 RZ1/RT3 佐证）；nxnative docs/agents/03、04 |
| 04-apple-tv.md | acnh-tv findings：M0、T1、B1、X2、IN2；acnh-smbw findings：T2、TV1、PW1、OPT1、AUD1；规格 PERF1（进行中数字）；CONTEXT.md |
| 05-perf.md | acnh-smbw findings：PW1、TV1、Z1、OPT1、T2；acnh-tv findings：G1、G2、G3、X2；规格 PERF1-cpu-submit（进行中）；上游 d2r-mobile 提交 e2a0c9a/b94670c |
| 06-audio.md | acnh-tv findings：AU1、AU2、AU2b、A0、I1、P2b、SV3；acnh-smbw findings：Z1、AUD1、TV1；规格 MOV1（未完成） |
| 07-agent-workflow.md | 主会话记忆 acnh-appletv-research.md、fleet-review-of-needs-explicit-cwd.md；acnh-tv findings：R1、RSV3(b)、RAU2(b)、RG3、RZ1、RT3 等全部复审报告；CONTEXT.md |
| 08-community.md | d2r-mobile README/docs/git log（e2a0c9a、44a6770、b94670c、#36）；nxnative README/AGENTS/docs/agents/01–05；GitHub Johnoooob/nx-port README（gh api）；bellpad README/AGENTS/docs |

## 刻意没写的内容及原因

| 没写 | 原因 |
|---|---|
| 反汇编、shader 源码/MSL/二进制 | 硬规矩 1：不包含游戏代码或资源 |
| 游戏函数偏移（`main+0x...`）、build ID | 同上；d2r 仓库里有按 build ID 注册的地址表，只写了「按 build ID 键控」这个机制，不写具体地址 |
| keys、游戏文件内容、解包产物字节 | 同上；keys 只写「以路径传给解包工具」这个操作原则 |
| 游戏截图（除 assets/screens 已给的 7 张） | 硬规矩：截图只用已提供的缩略图 |
| 具体 NVN 结构体字段偏移 | 只写字段的公开含义与坑（如 TextureView 16 字节），不写布局表 |
| PERF1 户外优化的最终数字 | 任务进行中，只写了现状（约 11–12 fps）并标注「进行中」 |
| MOV1 影片音频的做法 | 规格已派但无 findings 报告，标注「未完成」 |
| nxnative 的 GitHub URL 之外的臆测 | 仓库内未写明的（如 nx-port 的目标游戏名）标「未确认」，不编 |
| d2r-mobile「800 个固定地址」的具体形式 | 属游戏特定偏移，只保留「按 build ID 注册、换版本要重验」的经验 |

## 数字一致性

关键数字与报告原文核对：首帧 236,672 ms→1,709 ms（PW1）、室内 p50 25.5–26.1 ms / 户外 p50 84.4 ms（TV1）、2.05 GiB 上限（M0）、峰值 1,744 MiB（TV1）、shader 6,141/6,141（G1–G3）与 9,738/9,742（Z1/T2）、纹理包 7.315 GiB / 116,751 张（X2）、RomFS 103,400 文件（Z1/T1）、改写 52,095 x18（L1）、REV15 voice 0x188/SNDD 0xA8（AU2b）、DeviceSink 0x14（AUD1）、1,800 s 稳定（TV1）。

## 自查结果

- 禁词扫描（首先/其次/最后/值得注意的是/总的来说/不仅/判定/非空）：0 命中；「即」字逐处复核，仅剩「立即/即使/即时」等正常用法。
- 敏感内容扫描（`main+0x`、build_id、32 位十六进制）：0 命中。
- 大于 5 MB 的文件：无（最大为截图 117 KB）。
- SKILL.md 83 行，限额 250。
- git 状态：11 个提交，工作树干净。

## 子代理使用情况

4 个前台只读子代理（general-purpose），符合 Subagent 规则（≤4 个、只读、前台同步）：

| 代理 | 任务 | 产出 |
|---|---|---|
| A | 蒸馏 acnh-tv/findings（nx 路线） | 路线/检查单/坑/性能/音频事实表 |
| B | 蒸馏 acnh-smbw findings + worktree 4846/4849（tvOS/音频/性能） | tvOS/坑/性能/音频事实表 |
| C | 调研 d2r-mobile、nxnative、nx-port、bellpad | 社区项目事实表 |
| D | 蒸馏多 agent 工作流（记忆文件 + 复审报告） | 工作流事实表 |

主代理负责全部写作、提交、自查、claim 与 artifact 登记。子代理报告中与规格不一致的数字（如 236 s→1.7 s 实际出自 smbw 路线而非 nx 路线）已在写作时按真实出处修正。

VERDICT: DONE
