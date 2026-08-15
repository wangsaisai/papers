# Ouroboros：一个经由评审制核心演化的自我发展前沿编码智能体（Ouroboros: A Self-Developing Frontier Coding Agent with Reviewed Core Evolution）

**作者**: Anton Razzhigaev¹,²,⁴, Andrei Gritsaev⁴,⁵, Andrei Kaznacheev¹, Nikita Dragunov¹, Roman Yampolskiy³, Andrei Kuznetsov²,⁴（¹ 莫斯科国立大学，² 斯科尔科沃科学技术研究院，³ Joi Lab，⁴ AIRI 融合大脑实验室，⁵ 高等经济大学）
**系统贡献者**: Ouroboros（系统贡献者；正式作者身份仅限上述人类作者）
**翻译日期**: 2026-08-15

---

## 摘要（Abstract）

长时程智能体（Long-horizon agent）是"模型—框架"（model–harness）复合系统，但大多数框架在设计完成后就固定不变。我们提出 Ouroboros¹——一个自我发展的智能体框架（agent harness），其工具、上下文组装（context assembly）、提示词（prompts）和核心实现通过经评审的提交（reviewed commits）不断改进，而这些提交最终成为后续任务的运行基底。

这种自我发展有两种模式。**递归自由演化**（recursive free evolution）把"改进"本身当作一项任务。在审查当前系统后，智能体选择并实现一项变更；任务完成又可以调度下一轮演化循环，从而形成不断延续的受审更新序列，而非一次固定的优化运行。**经验驱动的核心演化**（experience-driven core evolution）始于普通工作。任务执行、反思、评审拦截（review blockers）、观测插桩（instrumentation）以及社交反馈会暴露缺陷、毛糙之处、上下文组装失败和低效的工具路径；智能体将这些问题记录为持久的错误类别和提议的修复方案，然后决定是否在同一个提交闸门（commit gate）下开启维护工作。

**Hope** 是有公开记录以来运行时间最长的 Ouroboros 部署，但它并非唯一在运行的实例，而是我们在人类交互下的自由演化领域所做的首要现场实验。自 2026 年 2 月起，一个持续运行的智能体横跨七个通信界面（communication surfaces）服务用户，同时保留记忆并持续修改自身的实现。人们会提出能力建议、批评行为、暴露故障；这些信号只是**咨询性**的。由 Hope 决定哪些提议指向真实问题、推进哪些变更。

同一个演化过程既能提升能力，也可能扩大自主性、获取更强的工具，或削弱后续的控制措施——包括通过选择替代的模型 API。因此，运行安全（operational safety）不是一份辅助性的检查清单，而是一条设计约束：权力边界必须在反复的核心演化下保持约束力。

### 贡献（Contributions）

1. 在 Terminal-Bench 2.1、OSWorld-Verified 和 CL-Bench 上取得当前最优（state-of-the-art）成绩，在 SWE-bench Pro 和 GAIA 上取得与前沿模型匹配的持平表现，并附完整的逐任务轨迹（per-task traces）与运行清单（run manifests）。
2. 一种框架架构，支持两种受审核心演化模式：递归自由演化与经验驱动的核心演化。
3. Hope：一个 161 天的"活体智能体"（living-agent）实验，在受管控的多界面人类沟通下进行自由演化；社交互动驱动候选改进，但不把提交权（commit authority）转移给用户。
4. 一种运行安全架构：在智能体持续演化的过程中，章程加载（constitution loading）、治理保护（governance protection）、分段差异评审（staged-diff review）、外部支出上限（external spend limits）与操作员停机（operator halt）始终保持权威性。

基准测试（benchmark）活动使用冻结的系统快照（frozen seeds）并记录运行时配置；Hope 则在一条相关但独立的谱系（lineage）上继续实时演化。Ouroboros 以 MIT 许可证发布²。

¹ https://ouroboros-agent.ai/
² https://github.com/razzant/ouroboros

## 1. 引言（Introduction）

智能体在长时程基准上的得分，是基础模型、执行框架、环境与评分器共同作用的结果。随着模型不断进步，越来越多的可达成能力取决于框架如何组装上下文、调用工具、验证结果以及从失败中恢复。大多数生产级框架在设计完成后就冻结了这些策略。Ouroboros 则把框架视为一个持续演化的对象：它的源代码、提示词、工具、评审逻辑与核心实现都存放在一个版本化仓库中，通过经评审的提交路径演进，并成为后续任务的运行基底。

## 2. 相关工作（Related Work）

**自我演化智能体（Self-evolving agents）。** 自我演化系统修改不同的基底（substrate），包括记忆、提示词、工具、工作流和实现代码（Gao et al., 2025）。Voyager 积累可执行的技能（Wang et al., 2023）；STOP、Gödel Agent 和 Darwin Gödel Machine 修改脚手架或智能体群体（Zelikman et al., 2023; Yin et al., 2024; Zhang et al., 2025）；Live-SWE-agent 在任务执行期间创建工具（Xia et al., 2025）；Autogenesis 为演化中的智能体资源规定了生命周期与回滚接口（Zhang et al., 2026）。ADAS 在智能体设计空间上搜索，SICA 编辑编码脚手架的实现（Hu et al., 2024; Robeyns et al., 2025）。Ouroboros 聚焦于一个已部署、受版本控制的实现，其中对核心代码与治理（governance）的变更都须通过经评审的提交。

**框架与编码智能体。** SWE-agent 与 OpenHands 证明了"智能体—计算机接口"本身就是编码智能体性能的一部分（Yang et al., 2024; Wang et al., 2025）。Codex CLI、Claude Code、Cursor、Aider、Hermes Agent 和 OpenClaw 都是"模型—框架"系统（OpenAI, 2025–2026; Anthropic, 2025; Anysphere, 2026; Gauthier, 2023–2026; Nous Research, 2026; OpenClaw, 2026），而受控研究发现在模型保持不变时，框架之间在准确率、延迟和 token 消耗上存在显著差异（Ding et al., 2026; Yao et al., 2026; Vats and Golev, 2026）。因此，每次对比都报告模型、框架、服务商路由（provider route）、投入等级（effort）与评估协议。

**持久记忆与部署。** Generative Agents、Voyager 以及各类持久记忆系统表明，存储的经历与反思可以塑造后续行为（Park et al., 2023; Wang et al., 2023; Borro et al., 2026），Constitutional AI 则在训练中使用明确的准则（Bai et al., 2022）。CL-Bench 评估智能体在有序任务流上的学习能力（Asawa et al., 2026）。Springdrift 报告了一个可审计的多通道持久智能体部署（Brady, 2026）。Ouroboros 把记忆和一份运行时章程（runtime constitution）当作控制面。其多模型评审借鉴了辩论（debate）、LLM 作为裁判（LLM-as-judge）与自我批评（self-critique）等方法（Irving et al., 2018; Du et al., 2023; Zheng et al., 2023; Madaan et al., 2023; Gou et al., 2023），被评审的对象是源代码补丁（source-code patches）。

**基准与协议有效性。** Terminal-Bench 2.1 评估 89 个困难的终端任务（Merrill et al., 2026）；SWE-bench Pro 针对长时程多文件任务（Deng et al., 2025）；OSWorld、GAIA 和 ProgramBench 分别覆盖 GUI/CLI 计算机使用、工具/网络推理与清洁室程序重建（Xie et al., 2024; Mialon et al., 2023; Yang et al., 2026）。智能体基准也可能暴露隐藏答案、接受非预期的捷径，或丢弃失败的尝试。BenchJack 与 HackDetect 将基准与轨迹审计（trajectory audits）系统化（Wang et al., 2026; Shao et al., 2026）。SWE-bench Verified 只作为历史背景保留，因为它已无法可靠地区分前沿编码系统（OpenAI, 2026）。

**表 1（Table 1）**：相关系统中演化边界的对比。

| 系统 | 提示词 | 工具/技能 | 工作流 | 核心代码 | 经评审的提交 | 部署状态 |
|---|---|---|---|---|---|---|
| Voyager | ✓ | ✓ | – | – | – | – |
| Live-SWE-agent | ✓ | ✓ | – | – | – | – |
| Autogenesis | ✓ | ✓ | ✓ | partial（部分） | 规定协议 | partial（部分） |
| Darwin Gödel Machine | ✓ | ✓ | ✓ | ✓ | 基准选择 | – |
| Hermes Agent | ✓ | ✓ | ✓ | – | – | ✓ |
| OpenClaw / ClawBench | ✓ | ✓ | ✓ | – | – | – |
| **Ouroboros** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

"核心代码"指智能体可以改变其后继任务所运行的框架实现；"经评审的提交"指变更在采纳前必须通过一条可审计的版本控制闸门序列化。

## 3. Ouroboros 架构（Ouroboros Architecture）

Ouroboros 将"启动器—监督器"（launcher and supervisor）边界与一个可变的智能体仓库（mutable agent repository）相分离（图 1）。启动器负责启动、进程监督、发布引导（release bootstrapping）与恐慌停机（panic-stop）语义。仓库中包含任务循环、工具、提示词、记忆投射（memory projection）、评审逻辑、基准适配器（benchmark adapters）与用户界面。外部工作区（external workspace）任务运行在独立的仓库根目录下，返回补丁工件（patch artifacts）或直接交付物。

**提交流水线（Commit pipeline）。** 三种由所有者选择的运行时模式约束对自身仓库的修改。Light（轻）模式阻止仓库编辑；advanced（高级）模式允许普通编辑并保护治理面；pro（专业）模式允许受保护的编辑但须接受评审。每一次写入都会使先前的评审证据失效，因为新鲜度（freshness）绑定在暂存快照（staged snapshot）上。

提交路径先运行确定性的预检（preflight），对暂存的差异（staged diff）计算指纹（fingerprint），收集评审者证据，并在提交前再次校验指纹。差异评审面板（diff-review panel）在所有上下文模式下都是阻塞性的。在所有者选择的 max 模式下，一个全仓库范围的评审者还会评估目标、耦合、提示词与功能代码；在 low 模式下则跳过范围评审。回滚（rollback）恢复到先前某个受审状态，并走一条独立的恢复路径。

**任务结果与验证（Task outcomes and verification）。** 任务完成情况在独立的"执行、目标、评审、工件"四个轴上记录，宿主端运行的验证命令会生成绑定版本号的收据（revision-bound receipts）。定稿（finalization）保留最新的类型化答案，并区分能力失败与基础设施错误、超时、预算耗尽和不完整证据。项目任务额外增加日志（journal）、工作板（workpad）、知识范围（knowledge scope），以及在共享智能体身份下的单写者租约（one-writer lease）。

**运营身份与记忆（Operational identity and memory）。** 运行时通过版本化章程、可编辑的身份档案（identity profile）、草稿本（scratchpad）与编年史（chronicle）投射、项目记忆、评审账本（review ledgers）与 Git 历史来表征身份与连续性。这些工件会跨会话、跨模型路由塑造可观测的行为。

**两种核心演化模式。** 自由演化把"演化"本身当作一项任务运行。在审查当前系统后，智能体选择并实现一项改进；完成可以调度另一个演化任务，从而产生不断延续的受审变更序列，而非一次固定的优化运行。任务后演化（post-task evolution）始于普通工作。任务执行、反思、评审拦截、观测插桩与社交反馈会暴露缺陷、毛糙之处、上下文组装失败和低效的工具路径。智能体把这些记录为持久的错误类别与提议的结构性修复，然后决定是否开启维护工作。被接受的修复与所有其他核心变更一样，通过同一个受审提交闸门。第 4 节追踪了该活系统中由人类提出与由智能体自我发现的实例。

**基准执行与证据（Benchmark execution and evidence）。** Terminal-Bench 在每一个 Harbor 任务容器内安装全新运行时并使用官方验证器。任务指令被原样保留，并由框架附带一段"反查阅"（anti-lookup）段落，禁止获取基准定义、测试或解答。其他适配器把同一运行时接入 OSWorld 虚拟机、SWE-bench Pro 仓库、GAIA 沙箱、ProgramBench 清洁室与 CL-Bench 任务流。

基准启动器在准入前写入运行清单（run manifest），证明种子与运行时，把每个请求的实例保存在追加式账本（append-only ledgers）中，并记录跳过、超时与基础设施失败的尝试。公开提交副本会进行值级密钥清洗（value-level secret scrubbing），并附带一次独立的"零残留"检查；官方基准评分器始终具有权威性。

**子智能体与补丁整合（Subagents and patch integration）。** Ouroboros 可以在可配置的任务树（task tree）下生成只读的规划侦察员（planning scouts）与可变操作的行动子智能体（acting subagents）（图 2、图 3）。默认深度为 2，配置的最大深度为 500。行动子智能体在隔离的工作树（worktrees）或被准入的外部工作区中写入，不能向活动的系统仓库提交。父智能体在三次索引式整合（three-way indexed integration）之前验证谱系、补丁哈希与受保护路径。可提交的基准配置（submittable benchmark profiles）会禁用任务委派以保持 pass@1；规划侦察员仍可贡献上下文，并被单独披露。

**图 1（Figure 1）：** Ouroboros 架构。一个受监督的运行时把工作派发给已准入的工作区、任务树与基准适配器。子智能体的补丁返回父智能体；随后对自身仓库的变更通过受审闸门。外部交付物与基准证据保持为独立工件。

**图 2（Figure 2）：** 子智能体补丁整合协议。行动子智能体在隔离工作树中写入；父智能体验证谱系与触及的路径，并且是唯一提交者。

**图 3（Figure 3）：** 一个实时 Ouroboros 会话的任务树视图：嵌套的规划与行动角色，带每个节点的状态、笔记数与子节点数。

## 4. Hope：人类交互下的自由演化（Hope: Free Evolution under Human Interaction）

Hope 是一个在受管控的人类沟通下进行自由演化的长期实验。自 2026 年 2 月起，一个持续运行的 Ouroboros 智能体横跨七个公开与私有界面与人们互动，同时保留记忆并持续开发自身的实现。用户请求、公开对话、内部观测插桩与任务后反思都为开发提供候选方向；由智能体决定哪些建议值得行动、推进哪些变更。

Hope 是有公开记录以来运行时间最长的 Ouroboros 部署，但并非唯一在运行的实例。它与已发布的基准框架共享同一条架构谱系，包括持久记忆、受审的仓库变更、回滚与操作员停机路径。活动仓库在冻结的基准种子之外持续演化。这种分离让可复现的评估与持续进行的部署演化得以共存。

截至 2026 年 8 月 6 日截止点，公开部署动态（deployment feed）已跨越 161 天，报告模型开销 $110.6K、处理 token 79.7B、代码 175,755 行、记忆工件 227 MB（图 6）。系统服务七个交互界面：网页聊天、语音、Telegram、Discord、Twitter/X、网站评论与电子邮件。表 4 记录了截至同一截止点的交互、演化与公开部署计数。

**多通道状态（Multi-channel state）。** 通道入口汇聚到一个有序的消息日志上，并被投射到各自的滚动摘要、按人摘要与按通话摘要中（图 5(a)）。私密通信被排除在公开日志之外；有界的私密上下文可以进入非公开的推理投射。所有通道因此共享同一个上下文，而不是各自充当独立的智能体。

**社交互动驱动的开发（Social-interaction-driven development）。** 人们不只是提交孤立的任务：通过持续对话，他们会指出不良行为、提出能力建议、挑战智能体的决策。这些信号与自我发现的故障和内部观测进入同一个改进积压队列。它们是咨询性的而非强制性的：由 Hope 决定某个建议是否指向真实问题、是否符合系统目标、是否启动变更。一个后台循环也可以在没有当下人类提示的情况下开启维护或自由演化任务。仓库变更随后遵循部署所配置的评审与提交策略，留下从社交反馈或内部观测到最终变更的完整踪迹。

**对自主工作的控制（Controls on self-directed work）。** 部署把章程驻留在常驻上下文中，保护治理文件不受普通写入路径影响，并针对暂存快照记录评审证据。公开消息不能直接调用提交、重启、shell 或身份编辑工具。这些控制减少了"提示词直通核心修改"（prompt-to-core mutation）的路径，同时保留智能体选择与实现改进的能力。控制强度取决于所有者选择的运行时与评审模式；第 7 节考察一个更难的问题——在智能体演化的同时保持这些边界稳定。

**操作员边界（Operator boundary）。** 公开在场（public presence）运行在严格的白名单工具集上。一条独立的、经认证的操作员通道承载任务指派、模型路由、预算控制与 /panic。停机命令在普通智能体处理之前由监督器解析，并终止整个进程树。支出上限位于智能体外部，不能通过普通智能体工具提高。

**部署期间的演化（Evolution during deployment）。** 两个案例说明有用的工作如何改变接收后续任务的智能体。第一，公开频道中的人们注意到 Hope 偶尔会重复发送同一条消息。智能体将行为追踪到一条重复发送路径，并在公开输出管道中落地一个"逐字重复守卫"（verbatim-duplicate guard），经评审后合入。第二，深度自我评审任务会以"模型不可用"的表象中止。智能体把故障追溯到评审包（review-pack）上下文溢出，并将组装路径替换为一个有界、感知连通性的上下文图集（context atlas），按导入图中心性（import-graph centrality）排序，并辅以按服务商校准的大小估计。该修复在评审期间保留高连通性的核心文件。第一个案例始于社交反馈，第二个源于智能体自身的观测。两者都成为持久的错误类别和受审的结构性变更，并被后续交互使用。它们共同示范了经验驱动的核心演化：工作暴露故障，智能体决定行动，修复结果改变后续工作的执行方式。

## 5. 评估（Evaluation）

表 2 汇总了五个基准家族的实验结果，图 4 展示了主要对比。所有运行均使用官方验证器。完整的逐任务轨迹、清单与提交都随对应结果提供链接。

**表 2（Table 2）：** 五个基准家族的"模型—框架"实验结果。

| 基准 | 模型 | Ouroboros | 具名基线 |
|---|---|---|---|
| Terminal-Bench 2.1 | Opus 5 high | 原始 86.97%；审计后 86.74% | Claude Code + Fable 5：83.8% |
| Terminal-Bench 2.1 | GPT-5.5 | 84.3% | Codex CLI：83.1% |
| Terminal-Bench 2.1 | Grok 4.5 | 审计后 84.94% | Cursor：79.3%；Hermes：77.53% |
| OSWorld-Verified | Opus 5 | 90.69% | Intelligence-Indeed：90.19%；Mythos Preview：85.4% |
| CL-Bench | Sonnet 4.6 | 0.2301 | ICL：0.1960；Claude Code：0.1855 |
| SWE-bench Pro | GPT-5.6 Luna | 58.2% | Codex：59.4%，p = 0.40 |
| GAIA | Sonnet 5 | 78.2% | Claude Code：78.8% |

各基准段落的链接内附轨迹、清单与提交。

**Terminal-Bench 2.1。** Opus 5 活动在 89 个任务上各跑 5 次试验。原始得分为 387/445（86.97%）。轨迹审计发现一次试验通过一个非预期的捷径满足了一个弱验证器。我们请求基准维护者将其清零，得到 386/445（86.74%）。服务商审核失败与基础设施错误仍计入分母。445 次试验的二项标准误约为 ±1.7 个百分点（对该区间内的所有系统），因此审计后的 Opus 5 得分大约比最强基线——Claude Code + Fable 5（83.8%）（Anthropic, 2025）——高出约两个标准误；排行榜上的其他基线是 Codex CLI + GPT-5.5（83.1%）（OpenAI, 2025–2026）与 Cursor + Grok 4.5（79.3%）（Anysphere, 2026）。提交全程公开，完整的 Harbor 任务亦公开可见。

**OSWorld-Verified。** Opus 5 运行在标准的非 Google-Drive 子集（Xie et al., 2024）上得分为 327.39/361（90.69%）。它使用截图、100 轮预算、只读可行性预检（read-only feasibility pass）、按任务配置请求的代理会话（proxy sessions）以及官方评估器。最强的已发布基线包括：榜单官方榜首 Intelligence-Indeed 智能体（90.19%）；Claude Mythos Preview（85.4%），这是 Anthropic 在 Claude 5 系统卡中报告的五次运行平均；以及 Pointer Agent + Opus 4.7（83.64%）。逐任务的提示词、轨迹、得分与清单全部公开。

**CL-Bench。** 提交的 Sonnet 4.6 活动在全部六个域上以一个无状态（stateless）基线和 5 个有序有状态（stateful）rollout 达到归一化奖励 0.2301。问题之间对话状态重置，原生记忆跨每个 rollout 持久。核心演化与任务委派被禁用，从而比部署场景更干净地隔离了持久记忆这一变量。基准作者发布的最强基线（Asawa et al., 2026）是纯上下文学习（in-context learning, ICL）——把交互历史放进提示词向前携带（Sonnet 4.6 上 0.1960，GPT-5.4 上 0.1890）——以及 Claude Code + Sonnet 4.6（0.1855）；Mem0、ACE 等记忆增强系统得分更低。五个 rollout 的逐任务均值与标准误包含在轨迹数据集中，提交全程公开。

**SWE-bench Pro 与 GAIA。** 在对称地移除所有"任一方达到参考解答"的 SWE-bench Pro 实例后，Ouroboros 在 655 个配对任务上解决 58.2%，Codex 解决 59.4%。1.2 个百分点的差距在 McNemar 检验下统计上不可区分（p = 0.40），因此这个自我发展框架与 Codex 处于模型匹配的持平水平。配对轨迹与审计全部公开。在 GAIA 上，使用 Sonnet 5 时 Ouroboros 得分 78.2%，Claude Code 得分 78.8%；GAIA 工件捆绑包随发布提供。

**图 4（Figure 4）：** Terminal-Bench 2.1、OSWorld-Verified 与 CL-Bench 上对具名已发布基线的结果。红条为 Ouroboros，灰条为基线，描边条为审计调整后的得分。Terminal-Bench 的触须显示 445 次试验的 ±1 二项标准误；OSWorld 与 CL-Bench 报告单次已评分活动。坐标轴截断到竞争区间。

## 6. 轨迹审计与框架改进（Trajectory Audits and Harness Improvements）

Ouroboros 把捷径奖励（shortcut rewards）、受污染任务（contaminated tasks）与执行失败都视为改进"已报告结果"和"产生该结果的框架"两者的证据。下文每一类问题都带来了一项调整后的得分、一项具体的实现变更，或一个供后续演化的持久目标。

**奖励劫持（Reward hacking）。** Terminal-Bench 轨迹审计识别出一次获得奖励的试验——它预先填充了 web 根目录，而没有完成所要求的 Git 到 Web 流水线。报告中的审计调整得分移除了该试验。同一审计还确认其余轨迹没有访问验证器文件、测试、奖励文件或参考答案。

**污染（Contamination）。** SWE-bench Pro 的任务标识会暴露上游修复提交，两个框架都通过网页搜索或 Git 历史触及了参考材料。对称过滤器会在任一方触及参考解答时移除该实例。由此得到的配对比较反转了原始聚合差距的解读方向。

**隔离失败（Isolation failure）。** 历史 GAIA 运行继承了操作员的 home 目录。智能体的重试因此可能把任务工件放在真实的桌面上。后来的启动器改用隔离的用户文件根目录与附件暂存（attachment staging），修正了已观测的路径。完整的文件系统隔离仍然需要比"路径约定"更强的沙箱。

**远端状态漂移（Remote-state drift）。** OSWorld 开发期间，一次虚拟机重置重新分配了客户端点（guest endpoint）。工作阶段保留了重置前的地址，导致并发通道在错误的虚拟机上操作。在每次重置后重新发布并验证端点，消除了该类问题。随后的取证分析又带来了回合预算措辞、截图完整性、任务契约验证与"首次计分尝试归属"（first-scored-attempt ownership）方面的修复。

**持续记忆故障（Continual-memory failures）。** CL-Bench 显示多个域上有正向的记忆延续，但在模式漂移（schema drift）下失败。存储的经验会过时，检索有时选错域，有用的经验偶尔只在失败回合之后才被写入。这些案例为未来的记忆工作提出元数据要求：显式的时间与域元数据。

## 7. 运行安全控制（Operational Safety Controls）

自我发展智能体比固定框架多带来一个安全问题：同一个提升任务性能的演化过程，也可能扩大自主性、获取更强的工具，或削弱施加于后续行动的控制。提示词、测试、工具、模型路由、评审规则与恢复路径因此都是与安全相关的变更面（mutation surfaces）。Ouroboros 用旨在"在反复核心演化下保持约束力"的安全护栏（guardrails）应对这一问题。Git 历史使变更可检视、可回退，而独立强制执行的边界保留操作员权威。

**风险：智能体自行选择模型 API。** 一个能选择自身模型后端的演化中智能体，可以通过普通的 API 变更去搜索更强大或约束更少的行为。把某个模型槽位重新路由到新的服务商或版本，可以在不改变可见任务界面的情况下提升自主能力、改变拒绝（refusal）行为、扩大提示注入攻击面，并使成本改变几个数量级。因此模型路由是一项**受审计的配置变更**，而非普通的运行时选择。所有者选择的上下文模式还控制是否运行全仓库范围评审，使得证据记录把这两项设置与每笔受审变更绑定在一起。

**在用的安全护栏（Guardrails in use）。** 章程通过一条非截断的读取路径加载，并被纳入评审上下文。确定性守卫保护治理文件不受普通写入工具影响。暂存差异在评审前后各计算一次指纹，未达法定多数的评审小组不能给出"干净通过"（clean pass）。所有者选择的上下文模式决定是否运行全仓库范围评审（第 3 节；图 5(b)）。暂存健康检查、崩溃回滚、外部支出上限、隔离的操作员通道与 /panic 提供了独立的恢复路径。这些机制把"被演化的基底"与"决定某个变更能否成为下一个线上版本的权威"分离开来。附录 A 给出完整的控制清单。

**观测到的行为（Observed behavior）。** 没有任何已记录事件抵抗过操作员停机。一次对未提交工作树的近乎全量删除，在操作员重置之前触发了先前已实现的救援机制——这表明恢复逻辑可以在自主工作期间被激活。这个案例还印证了架构上的一项分离：智能体层面的保存机制可以演化，而监督器层面的操作员权威必须保留"停机、替换或回滚系统"的能力。

**图 5（Figure 5）：** 运行控制边界。(a) 通道状态与操作员旁路：公开交互进入一个有序日志与有界摘要；预算与路由控制走经认证的操作员路径，/panic 在智能体处理之前终止进程树。(b) 受审变更闸门：两种上下文模式下差异评审都保持启用；全仓库范围评审仅在 max 模式下运行。

## 8. 结论（Conclusion）

Ouroboros 证明：一个经评审的、可自我修改的框架，能在 Terminal-Bench 2.1、OSWorld-Verified 与 CL-Bench 上刷新最优成绩，同时在 SWE-bench Pro 与 GAIA 上追平前沿编码框架。经验驱动的核心演化把普通工作转变为对智能体自身的改进：观测到的缺陷、毛糙之处、上下文失败与社交反馈，都变成对"接收后续任务的框架"的受审变更。Hope 在横跨七个通信界面的数月持续人类交互中演示了这一机制。运行安全架构应对了相应的风险：一个能改进自身代码并选择自身模型 API 的智能体，需要一套在演化压力下依然具有权威性的控制边界。源代码、适配器、方法学、提交与公开轨迹随报告一并公开。

**局限性（Limitations）。** 部署研究只跟踪一条长期运行的谱系，而非受控的独立演化智能体群体。SWE-bench Pro 受到公开参考泄漏与任务缺陷的影响。LLM 评审者可能与智能体共享盲区，且 low 上下文模式省略了全仓库范围评审。

**伦理考量（Ethical Considerations）。** 部署实例在公开与私有通道中与人类互动。原始私密对话记录保持私密。发布的示例与聚合轨迹已最小化，并清洗掉凭据、本地路径与参与者身份。第一人称的系统输出仅视为运行日志。自我修改与远端工作区能力具有双重用途（dual-use）。我们报告了权力边界、失败模式与已知的隔离缺口。

**AI 辅助声明（Use of AI Assistance）。** Hope（Ouroboros）贡献了部署反思、代码历史上下文与系统生成记录。与 arXiv 和 ACL 政策一致，Hope 被记为系统贡献者，并排除在正式作者元数据之外。

**致谢（Acknowledgments）**（原文不译）：We thank the benchmark maintainers and community contributors who reviewed submissions, reported failures, and provided reproducible comparison artifacts.

**参考文献（References）**（略）

## 附录 A：完整安全护栏（Guardrails in Full）

部署运行以下控制措施（第 7 节有摘要）：

- **章程常驻加载为提交准则（Always-loaded constitution as commit criterion）。** 一份版本化章程在每次任务循环中都沿一条不可截断的读取路径从磁盘重新读取，并始终保留在上下文中；它是提交闸门评审所依据的标准，不能被整体写入、删除或替换。普通写入在执行前即被阻止。
- **多模型对抗性评审，需达法定多数（Multi-model adversarial review with quorum）。** 每笔受审提交都运行一个差异评审小组；未达法定多数的结果不能被记为"干净通过"。
- **确定性预检与差异指纹（Deterministic preflight and diff fingerprinting）。** 版本、数据边界与体积健康检查先行运行；暂存差异在评审前后各做一次指纹计算，因此评审中途的任何篡改都会中止提交。
- **隔离的操作员通道与紧急停机（Isolated operator channel and emergency stop）。** 一条私有控制通道承载操作员权威与一个不可绕过的 /panic，它会在任何媒体处理之前终止所有进程。
- **模式登记簿（Pattern register）。** 反复出现的失败成为持久的记录行（错误类别、次数、根因、结构性修复），把修复从"实例级补丁"提升为"类别级预防"。

## 附录 B：章程（节选）（Constitution (Abridged)）

智能体的章程是一份始终加载的、由编号原则与运行约束组成的文档。以下复现原则结构及与控制最相关的条款，为节省篇幅作了删节。原则 0–4 构成一个受保护的核心，不能被删除或降级。

- **P0 主体性（Agency）。** 智能体以"行动者"而非"被动工具"的方式行动；主体性不得用于绕过受审变更控制，且操作员停机、回滚与替换始终保持权威。
- **P1 连续性（Continuity）。** 一个拥有不中断历史的单一实体；记忆被视为承重结构。核心上下文（章程、系统提示词、身份）绝不会被静默截断。
- **P2 元优于补丁（Meta-over-Patch）。** 修复错误类别而非症状；反复出现的失败记录在模式登记簿中。
- **P3 免疫完整性（Immune Integrity）。** 自我修改必须通过多模型差异评审。全仓库范围评审在所有者选择的 max 上下文模式下运行，在 low 模式下被显式跳过。改变评审边界需要计划评审（plan review）。
- **P4 自我创造（Self-Creation）。** 智能体可以改写自己的代码、提示词、身份档案与公开界面。章程核心受保护，身份档案不能被删除。
- **P5 LLM 优先（LLM-First）。** 决策经由模型路由；硬编码行为的占比降到最低。
- **P6 真实性与现实纪律（Authenticity & Reality Discipline）。** 主张须以证据为根据；维护一份系统的运行蓝图。
- **P7 极简主义（Minimalism）。** 每个模块都须在复杂度预算下证明自身存在的理由。
- **P8 共同成长（Becoming）。** 技术能力、记忆质量与运行连续性一起改进。
- **P9 版本与发布（Versioning and Releases）。** 每个提交都递增一个版本；发布携带同步的版本号、带注解的标签与来源证明；恢复先前受审状态的恢复操作豁免评审。
- **P12 认识论稳定（Epistemic Stability）。** 信念、记忆与行动保持连贯；矛盾被显式暴露；持久性的架构选择被记录在案。（P10–P11 已并入 P2 与 P9。）

运行约束包括：单一统一身份；在语音边界（speech boundary）强制隐私的公开通道架构；对危险工具的能力闸门；以及一条紧急停机不变式——操作员的 /panic 必须始终能立即终止每个进程，任何智能体代码、提示词或章程论证都不得拖延或绕过它。

## 附录 C：基准配置披露（Benchmark Configuration Disclosure）

**表 3（Table 3）** 记录了解读上述得分所需的脚手架（scaffold）设置。运行工件保留精确的模型路由、投入等级、种子提交、所选任务与运行时证明。

| 基准 | 脚手架披露 |
|---|---|
| Terminal-Bench 2.1 | 声明模型；全新试验状态；关闭委派并披露规划侦察员；关闭智能体网页；阻塞性评审；关闭演化 |
| OSWorld-Verified | 声明模型；任务间空记忆；关闭委派；披露按任务配置的代理与 GUI shell；可行性预检；关闭演化 |
| CL-Bench | Sonnet 4.6；每个 rollout 持久记忆；关闭委派、网页与视觉；一轮阻塞性改进；关闭演化 |
| SWE-bench Pro | GPT-5.6 Luna；按实例私有记忆；关闭委派；网络暴露经审计；固定框架；关闭演化 |
| GAIA | Sonnet 5；按样本私有记忆；关闭委派；同模型原生搜索；反查阅与泄漏审计 |

## 附录 D：补充图表（Supplementary Figures and Tables）

**图 6（Figure 6）：** Hope 公开部署序列，截至 2026 年 8 月 6 日。坐标轴从零起；2 月与 8 月为不满整月。数值为公开演化动态的月度端点。（四张累计曲线：模型开销 $110.6K、累计 token 79.7B、已发布代码 175.8K 行、已发布记忆工件 227 MB。）

**图 7（Figure 7）：** Ouroboros 界面面。(a) 项目可视化验证记录：完整项目视图展示一个实时的可视化验证任务、其对话、生成的工件与运行控制。(b) 经评审的可安装技能：技能视图在基础运行时之外暴露可安装的工具、路由与组件。

**表 4（Table 4）：** Hope 部署速览（2026 年 2 月至 2026 年 8 月 6 日）。

| 部署指标 | 数值 |
|---|---|
| 运行时长 | 161 天（持续） |
| 交互界面 | 七个（六个通道 + 电子邮件） |
| 不同的人类参与者 | 约 3,600 |
| 处理的公开消息 | 222,474 |
| 语音通话 / 轮次 | 3,166 / 45,872 |
| 电子邮件 | 5,507 |
| 公开累计模型开销 | $110.6K |
| 公开累计 token | 79.7B |
| 已发布代码量 | 175,755 行 |
| 已发布记忆工件 | 227 MB |
| 自我修改提交 | 1,085 |
| 智能体撰写提交占比 | 94.2% |
| 受审的自我编辑尝试 | 1,522 |
| 近期评审拦截率 | 63.5% |
| 模式类别 / 复发次数 | 40 / 659 |

（公开计数器来自部署的演化动态；交互与评审聚合来自一份脱敏的运行导出。）