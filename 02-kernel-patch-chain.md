# 02 · 内核补丁链：9 条补丁的台账与各自解决的问题

> fork 仓 [FAKE0704/deepseek-harness](https://github.com/FAKE0704/deepseek-harness)，默认分支 `patch-chain` = 完整补丁链（每条补丁是链上的一个提交，下文称「一环」）。本仓不发布 fork 源码，逐环 diff 请到 fork 仓按分支核对。

## 为什么修在内核

把 DSH 当内核嵌入生产（而非在终端里交互使用）后撞到的缺口分两类：

- **缺陷类**：终端用户同样会撞上——挂起审批无响应（132号：审批挂起后连续三例会话永久卡死，SDK 的审批应答接口又没有超时）、审批通过了写入仍被沙箱拒绝、模型补丁写盘无校验（157号）
- **接口类**：嵌入方的通用刚需——外部审批应答、跨进程会话接管、回合中止、实时流式输出

原则（120号，插件体系的方向总纲）：**语义随本质走**——纯通用语义修内核、目标是进上游；宿主私有语义进插件、不进内核。台账三态就是把这条原则机械化的（K3号）。

## 三态台账制度（K3号）

`patches.json` 是唯一登记源，每环登记 id / 分支 / commit sha / 标题 / 设计文档 / 三态：

| 状态 | 含义 | 纪律 |
|---|---|---|
| `upstreamed` | 上游已吸收同义变更 | 必填上游版本号或链接；同步上游版本时该环可摘除 |
| `upstream-pending` | 语义纯通用、计划提上游 PR | 本链现状（9/9） |
| `host-private` | 掺宿主私有语义、不计划上游化 | 必填理由，永久维护负担需显式接受 |

对账机制：补丁链最新一环的 commit sha 必须等于宿主运行时配置里锁定的 `forkSha`——升级内核忘了登记台账，CI 立刻变红。新环落地纪律 = 设计文档 + 台账登记 + 运行时锁前移，三件事同一个 PR。

## 补丁链 9 环（旧 → 新）

| # | 环 | 分支 | 解决的问题 |
|---|---|---|---|
| 1 | node-executable | `sdk/node-executable` | 非 Node 宿主（Bun）拉起内核：nodeExecutable 启动选项 |
| 2 | approval-respond | `sdk/approval-respond` | 挂起的审批请求没有外部通道可应答、会话永久卡死：session/approval-respond RPC + 外部应答器挂起等待 |
| 3 | session-resume | `sdk/session-resume` | 会话随创建进程而亡、服务重启后现场冻结：新进程收养已落盘会话继续跑（跨进程续话） |
| 4 | sandbox-extra-roots | `sdk/sandbox-extra-roots` | 「审批已通过、写入仍被沙箱拒绝」死循环：多目录可写边界，与授权目录并集 |
| 5 | session-interrupt | `session-interrupt` | 嵌入方无法程序化中止跑偏的回合：宿主请求的回合中止，不拆进程 |
| 6 | session-stream | `session-interrupt`（同分支第 6 环） | 要等整回合结束才拿到输出：SDK 通知面实时流帧，真流式 |
| 7 | eof-suicide | `sdk/eof-suicide` | 宿主意外退出后内核成孤儿进程、数小时占着会话锁：stdin 关闭后有界自动退出 |
| 8 | kernel-flight-recorder | `genshu/flight-recorder-uplift` | 运行状态不落盘、线上故障只能靠猜：运行时「黑匣子」全量记录（132号 §4 内核侧诉求） |
| 9 | ask-user-id-tolerant | `genshu/ask-user-id-tolerant` | ask 工具漏传 id 导致整批失败（约 6%）：内核执行与宿主解析两侧同规则按位补齐 |

每环带自动化测试；缺陷类环先写复现测试再修复（复现与修复同一个 PR）。例：`genshu/parked-deafness-repro` 分支顶部提交 = 挂起审批无响应故障的复现（4 项进程内证据 + 双进程无响应态端到端用例）。

## K 系列设计文档（内核侧，宿主仓 docs/kernel）

| 号 | 标题 | 状态 |
|---|---|---|
| K1 | session/interrupt 会话级中断 RPC（共享引擎实例的「真停」语义） | 已实施（2026-10-20） |
| K2 | session/stream SDK 通知面实时流帧 | 已实施（2026-11） |
| K3 | 补丁面三态台账与版本同步 diff | 已实施 |
| K4 | 父进程死亡后 stdin EOF 有界自动退出（含双路并发测试） | 已实施（2026-10-29） |
| K5 | ask 工具 id 宽容化（K5 为契约源） | 已实施（2026-11-04） |

## 与上游的关系（如实）

- 9 环全部 `upstream-pending`：语义判定为纯通用、目标形态是上游 PR，本地补丁链是过渡载体——**尚未提交上游就是尚未提交**，不冒充
- 拍板依据（122号 §④）：内核补丁全部按可进上游的标准写，不含宿主专有语义
- master 分支保持与上游一致，补丁放独立分支——将来同步上游版本时，已被上游吸收的环可摘除，未吸收的在台账里显式记为维护负担
