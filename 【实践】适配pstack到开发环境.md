# 帖子【实践】适配pstack到开发环境

## 什么是pstack

cursor版本：[https://github.com/cursor/plugins/tree/main/pstack](https://github.com/cursor/plugins/tree/main/pstack) 

for codex版本：https://github.com/michael-denyer/pstack-claude/tree/main

### 适合的场景

它适合质量要求高、值得投入时间和 Token 的任务，不必每个改动都全套使用。重要功能、复杂代码库、错误代价高且需要深入验证时，可以考虑这套流程；小的设计调整或普通前端 UI 改动，通常不值得启动全部并行审查与验证步骤。

不需要使用复杂的RAG，使用LLM维护的索引文件，通过渐进式加载，在一定的规模上表现更好。

### Pstack 怎样组织 Agent 工作

只有能够信任单个代理产出可验证的代码，才能放心地并行工作。pstack 因此提供统一的工程方法，并允许按不同模型的长处安排任务。

**Pstack 分为三层：**

- **工程原则：**规定代码和验证的标准
- **playbook：**给 Agent 提供处理任务的步骤
- **skills 和斜杠命令**执行具体工作

它包含 **27 个功能技能、24 条工程原则、23 个任务流程和 2 个子代理**。

#### 主要的技能

| 方向 | 可用能力 | 适合我们的场景 |
| :---: | :---: | :---: |
| 理解系统 | [how](https://github.com/cursor/plugins/blob/main/pstack/skills/how/SKILL.md)、[why](https://github.com/cursor/plugins/blob/main/pstack/skills/why/SKILL.md)、teach | 沿代码调用链解释“怎么运行”，结合提交、文档和运行证据追查“为什么这样设计” |
| 变更前评估 | [blast-radius](https://github.com/cursor/plugins/blob/main/pstack/skills/blast-radius/SKILL.md)、architect | 查小改动的跨模块影响，先确定接口、类型和归属 |
| 质量验证 | [interrogate](https://github.com/cursor/plugins/blob/main/pstack/skills/interrogate/SKILL.md)、benchmark-checklist、create-verification-skill | 挑战代码改动、审查性能数字、建立可重复的真实行为验证 |
| 持续改进 | [correct](https://github.com/cursor/plugins/blob/main/pstack/skills/correct/SKILL.md)、show-me-your-work | 把反复出现的错误固化为检查，并给长任务留下可审阅的决策记录 |
| 工作流入口 | [poteto-mode](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/SKILL.md) | 按调查、修 bug、性能、功能、PR 跟进等场景选择流程，它会根据请求选择合适的 playbook。 |

Pstack 的取向是**在已有代码库里边探索边实现**。Lauren 认为“最好的规格是代码”，所以没有把详细的预先规划做成核心技能。

**一些具体的例子**

- **Arena：让多个 Agent 解同一个问题。** 不同模型并行提出设计和实现方案，最后选择或融合较好的部分。这适合关键功能、且 Token 成本可以接受的场景。
- **Swarm：把任务切成不同部分并行完成。** 它与 Arena 的区别是分工处理不同切片，再汇总结果。Pstack 会协调分支或 worktree 中的工作。
- **TDD：先写测试，再实现并让测试通过。** 
- **Why：追查“为什么这样做”。** 它会结合项目可用的 MCP 和 CLI，查产品信息、讨论记录、日志、架构决策记录等，帮助还原某项实现背后的原因。这比只翻当前对话记录更有用。
- **Recall：恢复项目上下文。** 隔了一段时间重返项目时，它会读此前的会话，并借助 Why 查询外部项目资料，整理当前进度与下一步。
- **Interrogate：多模型复核代码。** 两三个模型分别按标准检查实现，汇总意见；代价是明显增加 Token 消耗。
- **Create / Maintain Verification Skill：把验证变成可重复执行的脚本。** 前者为应用行为建立检查和交叉核对，后者在应用变化后更新这些验证。这一点很重要，因为应用可能“看起来能运行”，实际却返回错误数据。
- **Show Me Your Work：回溯工作过程。** 用它查看这次单条请求背后的步骤：先做可行性探针，明确需求和约束，搭建基础结构，并行尝试方案，随后测试、核验和审计。视频称最后的核验发现并纠正了 Agent 的三处错误说法。

#### 工程原则（我认为比较重要的）

1. **尽量用小改动解决问题。** 重构时先看能否删除代码、减少复杂度，而不是继续叠加实现。
2. **从第一性原理重新看设计。** 加新功能时，思考“如果它从项目第一天就存在，结构会怎样设计”，避免一直把新逻辑补丁式地接上去。
3. **降低代码审阅负担。** 控制 PR 的规模、分散程度和抽象数量，让后来读代码的人容易理解。
4. **探索多个可行设计。** 对重要问题，可以比较不同方案，再吸收各自有价值的部分。
5. **为重复劳动造工具。** 如果 Agent 反复手动做同一检查，就把它沉淀为 CLI 或脚本。
6. **用真实行为证明结果。** 编译成功、单测通过还不等于功能确实可用；必要时要检查实际产物、端到端流程或界面行为。
7. **管理主会话的上下文。** 把适合并行的子任务交给各自的 Agent，再将结果汇回主线。

#### 任务流程：可能最常用的playbooks

| 你要做什么 | playbook |
| :---: | :---: |
| 搞清系统如何工作、为何这样设计 | **Investigation**：只读调查并给出证据 |
| 修复故障 | **Bug fix**：复现、定位根因、修复、在实际使用路径上验证 |
| 实现新需求 | **Feature**：先明确数据形态，再实现和验证新行为 |
| 调整代码结构但保持行为 | **Refactoring** |
| 解决已测得的性能问题 | **Perf issue**：建立基线、定位瓶颈、对照测量 |
| 持续提升一个指标 | **Hillclimb**：反复提出假设、测量并保留有效改动 |
| 检查 PR 的评审、冲突和 CI | **Babysit**：推进到可合并状态 |
| 合并已核验的 PR 或 PR 栈 | **Shipping**：逐个独立核验后再落地 |

#### 2个子Agent是啥

| 子代理 | 作用 |
| :---: | :---: |
| [poteto-agent](https://github.com/cursor/plugins/blob/main/pstack/agents/poteto-agent.md) | 按 poteto-mode 的完整工程流程承担一个具体任务，产出实现和验证证据。 |
| [Comment Sicko](https://github.com/cursor/plugins/blob/main/pstack/agents/comment-sicko.md) | **只读审查代码注释**，找出重复解释代码、没有提供有效背景的注释；通常由 /no-comments 调用。 |

## 我最想要的功能

### 验证模块

pstack 把“验证”当作**完成任务的证据**：改动要在用户实际使用的入口产生预期结果，副作用也要可检查。编译成功、单测通过、CI 变绿分别提供一部分证据，但不能单独证明整条用户路径正常。

#### 每次任务都验证真实行为

修 Bug 时，先在出现问题的界面、CLI 或 API 上复现；修复后用**同一路径**重跑，并配合针对性的回归测试。

做新功能时，则从用户或调用方入口执行，检查返回结果和实际副作用。例如接口返回成功，还要确认预期的数据确实写入。playbooks/bug-fix.md也明确要求重跑原始复现路径。

#### 给项目建立可重复的“验证技能”

pstack会在项目里生成一个 `verify-<app>` 技能。它记录五件事：

| 部分 | 要回答的问题 |
| :---: | :---: |
| **Launch** | 如何启动待验证的应用，怎样判断它已就绪？ |
| **Doctor** | 当前实例、版本、认证和环境是否适合继续验证？ |
| **Drive** | 怎样按真实用户路径操作某项功能？ |
| **Evidence** | 要保存什么结果，例如响应、截图、日志、退出码或数据库状态？ |
| **Cleanup** | 如何清理本次启动的进程和临时数据，同时保留证据？ |

它还维护一份 **feature map**：逐项记录用户能做什么、怎样到达该功能、怎样操作、什么可观察结果算通过。生成技能后必须实际跑通至少一项功能；这证明验证方法能用，**不代表整张功能表都已验证**。

#### 随产品变化维护验证能力

`/maintain-verification-skill` 会逐项核对 feature map：先从源码检查每项功能的入口和行为，再实际驱动**每一项**。结果要区分：验证通过、因具体前置条件无法到达、文档或驱动脚本漂移，以及产品行为本身出错。维护流程只修验证技能里的已证实问题；发现产品回归则报告，不靠修改说明把它掩盖。最终结果是 `clean`、`changed` 或 `blocked`。

### 研发经验和流程可以复制和移植

**插件化解决的是分发和管理问题。**

最大的好处是**可以把它作为一套完整能力安装、使用和维护**，而不只是把技能文件散放在全局目录里。

- **安装和启用有明确状态**：可以查询、更新或移除 。
- **入口统一**：所有技能使用 统一的命名空间，避免与其他技能重名；总入口负责选择任务流程。
- **版本和来源清楚**：技能、流程、角色说明和适配文档放在同一个包里，便于审计和迭代。
- **迁移更方便**：以后换机器或给其他人使用，可以从插件源安装，不必逐个复制全局技能。

## 变成适合自己工作环境的pstack

**fork之后，在 pstack 的工作流里增加一个面向公司研发任务的工具入口。**

| 我的工作 | 组内插件的默认入口 |
| :---: | :---: |
| 查 LogID、服务日志 | [内部 CLI] log，必要时用 [内部MCP]-log 或 argos 补充诊断 |
| 发 HTTP/RPC 测试请求 | [内部 CLI] api-test，先核对接口契约、环境和测试流程要求 |
| 看 QPS、SLA、依赖 | `[内部 CLI] apm` |
| 排查 BOE、PPE | 按场景选 开发工具、env、容器管理、发布流水线，需要调试会话时再用专门 Skill |
| 查文档、IDL、代码与发布任务 | 分别路由到 [飞书文档]、【内部搜索】、[编译]、[代码管理] |
| 维护仓库 Wiki | 交给项目的 maintain-repo-wiki Skill 和治理规则 |

完整的功能路由在/skills/rd/SKILL.md。`poteto-mode` 遇到公司内部任务会读取它；较大的代码任务仍沿用原来的开发流程，只在查日志、测接口等步骤使用公司工具。/hooks/session-start-context.md则负责提示自动路由。你可以直接说“查这个 LogID 的请求链”，无需先指定 CLI 或 MCP。

还补了两类组内使用规则：缺少 [内部 CLI]、领域 Skill 或 MCP 时，指出对应的安装或连接入口；同学自己的流程可先写在项目 `AGENTS.md`、`CLAUDE.md` 或独立 Skill 中，成熟后再加进组内插件。日志结论要标明检索范围，测试请求要先确认目标环境和契约。这些规则在 skills/rd/SKILL.md。
