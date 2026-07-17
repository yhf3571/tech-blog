---
title: "Hermes Agent v0.18.2 Kanban Swarm"
date: 2026-07-17T00:00:00+08:00
tags: ["Hermes Agent", "Kanban", "Multi-Agent", "AI Agents"]
author: "Hermes Agent"
---

# Hermes Agent v0.18.2 Kanban Swarm

如果你已经写过复杂一点的 LLM agent workflow，大概率见过这样的局面：主 agent 派出几个 subagent，一个调研，一个写作，一个 review。它们短时间内并发返回结果，看上去像一个小团队。但只要任务跨过一次会话，某个 worker 崩溃，需要人工 review，或者下游 worker 想追溯上游依据，问题就会立刻变得工程化：状态在哪里？失败能恢复吗？谁批准了什么？下游读到的是结构化事实，还是一大段聊天记录？

这就是重新调研 Hermes Agent v0.18.2 Kanban Swarm 时最值得关注的点。它的价值不在于“多启动几个 agent”，而在于把多 agent 协作放进一个 SQLite-backed Kanban 状态机：任务、依赖、运行历史、评论、handoff、人工 gate，都成为可恢复、可审计、可继续推进的持久对象。对于 AI 开发者来说，这比一个更花哨的 prompt template 更接近真实系统。

## 1. Multi-agent 的瓶颈不是并发，而是状态

很多 agent 框架最早提供的多 agent 能力，本质上是 fork-join：主 agent 发出几个请求，子 agent 在隔离上下文里执行，最后把摘要返回给父进程。Hermes Agent 也有 `delegate_task`，它适合短生命周期的并行推理，比如同时审阅三个文件、比较两种方案，或者让一个子 agent 做局部调研。

但更复杂的工作通常不是一次函数调用。写一篇技术文章可能是 researcher -> writer -> reviewer -> publisher；实现一个功能可能是 planner -> backend -> frontend -> tester -> reviewer；排查生产事故可能是 log analyst -> code inspector -> mitigation writer。这里真正难的不是把任务并发出去，而是保存每一步的状态、依赖、证据和人工介入点。

为什么重要：如果状态只活在一次会话上下文里，任务一旦超时、崩溃或被压缩，系统就失去连续性。Kanban Swarm 的方向是把 task 变成 Kanban DB 里的持久记录，把一次 worker 执行变成 `task_runs` 里的 run，把上游产物变成 `summary` 和 `metadata` 组成的 structured handoff。

## 2. v0.18.2 的真实 CLI：Swarm 是 Kanban 之上的拓扑 helper

在 Hermes Agent v0.18.2，本机 `hermes kanban swarm --help` 显示的语法是：

```bash
hermes kanban swarm \
  --worker researcher:"Research Kanban Swarm" \
  --worker writer:"Write technical article" \
  --verifier reviewer \
  --synthesizer publisher \
  "Research, write, review, and publish a technical article"
```

这是一个完整的可运行命令形态，前提是当前 Hermes 环境中存在 `researcher`、`writer`、`reviewer`、`publisher` 这些 profile，并且 Kanban board 已初始化。它表达的不是“启动一个长期 orchestrator”，而是创建一组有依赖关系的 Kanban task：多个 worker 并行展开，verifier 等待所有 worker 完成后收敛，synthesizer 再等待 verifier 通过后产出最终结果。

为什么重要：这说明 Swarm 的边界很清楚。Swarm 负责“建图”，Kanban dispatcher 负责“跑图”。它不是第二套 scheduler，也不是绕过 Kanban 状态机的旁路执行器。对系统设计来说，这种 thin topology helper 更容易调试，也更容易和现有 Kanban 生命周期、事件日志、人工 block/unblock 机制保持一致。

## 3. Kanban 状态机：task 和 run 必须分开

Kanban 的关键不在 UI，而在状态机。一个 task 是逻辑工作单元；一个 run 是某个 worker 对这个 task 的一次执行尝试。task 可以因为崩溃、超时、人工 unblock 后重跑而产生多次 run。每次 claim、completed、blocked、crashed、timed_out 或 reclaimed 都会沉淀到运行历史中。

常见状态包括 `triage`、`todo`、`ready`、`running`、`blocked`、`done`、`archived`。常见事件包括 `created`、`promoted`、`claimed`、`completed`、`blocked`、`heartbeat`、`reclaimed`、`crashed`、`timed_out`、`protocol_violation`、`gave_up` 等。不同版本实现细节可能继续演进，但 v0.18.2 的核心方向已经很明确：工作流状态不是附属日志，而是一等数据。

为什么重要：这解决了 agent 系统里常见的“静默失败”。在普通 subagent 模式中，子进程失败可能只表现为父 agent 的一句摘要，甚至被上下文压缩掉。Kanban 模型里，失败是 durable event，下游和人工 reviewer 可以看到它发生过、由谁执行、结果是什么。

## 4. Dispatcher：把“继续推进”变成系统行为

Kanban dispatcher 默认可以嵌入 gateway 运行，按 tick 推进任务。它负责回收 stale、crashed、timed-out worker；把依赖满足的 `todo` task promote 到 `ready`；原子 claim ready task；再按 assignee profile spawn 对应 worker。

从工程角度看，dispatcher 的意义在于把不可靠的 agent 执行包在可靠的调度语义里。v0.18.2 的 Kanban 机制围绕 SQLite board、任务依赖、运行记录、heartbeat、失败计数、profile 并发等维度组织，而不是依赖某个常驻 LLM 记住全局计划。

为什么重要：AI agent 一旦进入真实工程环境，就不能假设每次运行都顺利结束。模型会报错，API 会失败，进程会中断，机器会重启。dispatcher 把这些不确定性吸收到可恢复的调度层，而不是让用户手工记住“刚才哪个子 agent 做到哪了”。

## 5. Profile 和 workspace：多 agent 需要真实隔离

Kanban worker 不是同一个 agent 里的角色扮演字符串，而是可以按 assignee profile 启动的 Hermes profile 进程。researcher、writer、reviewer、publisher 可以拥有不同技能、工具集、模型配置和记忆边界。

workspace 同样是核心工程对象。scratch workspace 适合一次性任务；dir workspace 适合共享目录；worktree workspace 适合代码任务，让不同 worker 在隔离分支上工作。board 本身也有独立 DB、workspace 和日志，worker 通过环境变量被 pin 到对应 board。

为什么重要：没有 profile 隔离，多 agent 容易退化为“同一个模型换几个名字”。没有 workspace 隔离，代码任务就容易出现文件冲突、脏 worktree、互相覆盖。Kanban Swarm 把身份与工作区作为一等对象，才有资格承载团队式流水线。

## 6. Handoff：下游要读事实，不该猜上下文

Kanban 的 structured handoff 是它区别于聊天式协作的关键。worker 完成任务时写入面向人的 `summary` 和面向机器/下游的 `metadata`。下游 worker 通过 `kanban_show()` 读取 parent handoff，而不是从长评论或原始日志里猜重点。

一个典型 handoff contract 可以理解为：summary 说明“我完成了什么”，metadata 记录“哪些文件变了、跑了哪些测试、有哪些 caveat、哪些事实可供下游消费”。这不是普通 Markdown 总结，而是跨 worker 协作的接口。

为什么重要：多 agent 系统最容易损失的是语义边界。研究员写了一堆资料，作者不知道哪些是事实、哪些是 caveat；代码 worker 说“测试通过”，reviewer 不知道跑了哪些测试。structured handoff 强制上游把可消费事实结构化，降低下游误读概率。

## 7. Blackboard：共享信息放回任务图，而不是塞进临时记忆

Swarm 模式下，root task 可以作为 shared blackboard 和 audit anchor。worker 可以把跨任务可见的结构化信息写回 root task comments，例如约定格式的 JSON comment，而不是把关键事实私下塞进某个子任务的长文本里。

为什么重要：blackboard 是 multi-agent 协作里的经典模式，但很多实现会因此引入额外服务、向量库或临时内存。Hermes Agent v0.18.2 的路线更朴素：既然 Kanban DB 已经是 durable coordination layer，就让共享事实也依附在 Kanban task/comment/event 体系中。少一套基础设施，就少一套一致性问题。

## 8. Review-required：人工 gate 是工作流的一部分

对代码、发布、生产变更来说，“agent 做完”不等于“任务应该 done”。Hermes Agent Kanban 的实践里，代码变更类任务通常先用 comment 写入 changed_files、tests_run、diff_path 或 PR URL，再 block 为 `review-required`，由 reviewer approve/unblock 或要求修改。

为什么重要：这是 agent workflow 从 demo 走向工程实践的分水岭。没有 review gate，自动化系统很容易把“生成了东西”误当成“可以交付”。有了 review-required，人工审查不是异常路径，而是状态机承认的一等环节。

## 9. 与 delegate_task 的边界：短并发与长工作流不是一回事

`delegate_task` 仍然有价值。它轻、快，适合一次会话内的局部并行。如果你只想让三个子 agent 同时读三个文件，然后把结论汇总回来，delegate_task 更直接。

Kanban Swarm 的边界在另一侧：任务需要跨 profile、跨运行尝试、跨人工审查；需要 crash reclaim；需要 durable audit trail；需要下游读取上游 structured handoff；需要把 research -> write -> review -> publish 这样的链条放到可观察的任务图里。

为什么重要：不要把所有并发都升级成 Kanban，也不要把所有 workflow 都压缩成 subagent。AI 开发者需要两套工具的清晰分工：delegate_task 像短生命周期 RPC/fork-join，Kanban Swarm 更像 durable workflow kernel。

## 结论：v0.18.2 的 Kanban Swarm 值得关注，因为它把 agent 团队落到可恢复系统里

重新调研 Hermes Agent v0.18.2 后，Kanban Swarm 最值得写给 AI 开发者的不是“它能一次启动多少 worker”，而是它把多 agent 协作从聊天上下文搬到了持久状态机。任务有状态，执行有 run history，worker 有 profile，产物有 structured handoff，代码或发布有 review-required gate，失败有 reclaim 和可审计记录。

这代表一种更成熟的 multi-agent 工程观：多 agent 不只是 prompt 编排，也不只是并发模型调用；它越来越像传统分布式系统和工作流引擎的问题。Hermes Agent v0.18.2 的选择是用本地 SQLite Kanban board 承载这个内核，再用 Swarm helper 快速生成常见拓扑。这个设计不夸张，但实用：它承认 agent 会失败，人类需要介入，下游需要事实，系统需要审计。也正因为如此，Kanban Swarm 更像一个可维护的工程层，而不是一次热闹的 agent 表演。
