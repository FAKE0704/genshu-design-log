# 02 · 内核补丁链：9 环台账与每环解决的问题

> fork 仓 [FAKE0704/deepseek-harness](https://github.com/FAKE0704/deepseek-harness)，默认分支 `patch-chain` = 完整补丁链。本仓不发布 fork 源码，逐环 diff 请到 fork 仓按分支核对。

## 为什么修在内核

把 DSH 当内核嵌入生产（而非 CLI 交互使用）后撞到的缺口分两类：

- **缺陷类**：CLI 用户同样会撞上——挂起审批失聪（132号：停泊即僵死 ×3 + SDK approvalRespond 无超时放大）、审批通过仍被沙箱拒绝、模型补丁写盘无校验（157号）
- **接口类**：嵌入方通用刚需——外部审批应答、跨进程会话接管、回合中止、实时流帧

原则（120号「插件宇宙北极星」）：**语义随本质走**——纯通用语义修内核走上游 PR，宿主私有语义进插件不进内核。台账三态就是把这条原则机械化的（K3号）。

## 三态台账制度（K3号）

`patches.json` 为机械真源，每环登记 id / 分支 / sha / 标题 / 设计文档 / 三态：

| 状态 | 含义 | 纪律 |
|---|---|---|
| `upstreamed` | 上游已吸收同义变更 | upstreamRef 必填；跟版 rebase 时该环可摘 |
| `upstream-pending` | 语义纯通用、待提上游 PR | 本链现状（9/9） |
| `host-private` | 掺宿主私有语义、无上游化计划 | note 必填理由，永久 fork 负债需显式接受 |

对账机制：尾环 sha 必等于宿主运行时钉子的 `forkSha`（哨兵对账，前移漏记账即 CI 红）；新环落地纪律 = K 文档 + 台账追加 + forkSha 前移，三件同 PR。

## 补丁链 9 环（旧 → 新）

| # | 环 | 分支 | 每环解决的宿主问题 |
|---|---|---|---|
| 1 | node-executable | `sdk/node-executable` | 非 Node 宿主（Bun）拉起内核：nodeExecutable 启动选项 |
| 2 | approval-respond | `sdk/approval-respond` | 挂起的审批请求没有外部通道可应答、会话永久卡死：session/approval-respond RPC + 外部应答器停泊 |
| 3 | session-resume | `sdk/session-resume` | 会话随创建进程而亡、服务重启现场冻结：adopt persisted sessions on prompt（跨进程续跑） |
| 4 | sandbox-extra-roots | `sdk/sandbox-extra-roots` | 「审批已通过、写入仍被沙箱拒绝」死循环：extraWritableRoots 多目录可写边界，与授权根并集 |
| 5 | session-interrupt | `session-interrupt` | 嵌入方无法程序化中止跑偏回合：host-requested turn abort without process teardown |
| 6 | session-stream | `session-interrupt`（同分支第 6 环） | 整回合结束才拿到输出：SDK 通知面活流帧，真流式 |
| 7 | eof-suicide | `sdk/eof-suicide` | 宿主意外退出后孤儿内核数小时持锁（dsh_lock_wedged 根因第三层）：stdin EOF 有界自尽 |
| 8 | kernel-flight-recorder | `genshu/flight-recorder-uplift` | 运行态不落盘、线上故障只能靠猜：runtime flight recorder「黑匣子」（132号 §4 kernel-side ask） |
| 9 | ask-user-id-tolerant | `genshu/ask-user-id-tolerant` | ask 工具漏 id 整批失败（~6%）：内核 execute 与宿主解析两侧同规则按位补位 |

每环带自动化测试；缺陷类环先复现故障再修复（复现测试与修复同 PR）。示例：`genshu/parked-deafness-repro` 分支顶部提交 = parked-approval deafness repro（4 项进程内证据 + 双进程 deaf 态 e2e）。

## K 系列设计文档（docs/kernel，宿主仓）

| 号 | 标题 | 状态 |
|---|---|---|
| K1 | session/interrupt 会话级中断 RPC（共享代停止的真停语义） | 已实施（2026-10-20） |
| K2 | session/stream SDK 通知面活流帧 | 已实施（2026-11） |
| K3 | 补丁面三态台账与跟版 diff | 已实施 |
| K4 | 父进程死亡 EOF 有界自杀（+ 双路并发 spec） | 已实施（2026-10-29） |
| K5 | ask 工具 id 宽容化（K5 为契约源） | 已实施（2026-11-04） |

## 与上游的关系（如实）

- 9 环全部 `upstream-pending`：语义判定为纯通用、目标形态是上游 PR，本地补丁链是过渡载体——**尚未提交上游就是尚未提交**，不冒充
- 拍板依据（122号 §④）：四部署补丁全部走上游 PR、零 Genshu 专有语义
- master 分支保持上游镜像不动，补丁链独立成支——跟版 rebase 时 upstreamed 环可摘，负债显式在账
