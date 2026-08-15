# Ouroboros 论文架构图

```mermaid
flowchart TD
    subgraph "交互层（Interfaces）"
        UI["七个交互面<br/>网页 · 语音 · Telegram · Discord<br/>Twitter/X · 网站评论 · 邮箱"]
        LOG["有序消息日志 + 有界摘要<br/>滚动 · 按人 · 按通话"]
    end

    subgraph "启动器与监督器（Launcher + Supervisor）"
        LS["启动器<br/>启动 · 进程监督 · 发布引导"]
        OP["认证操作员通道<br/>任务指派 · 模型路由 · 预算"]
        PANIC["/panic 紧急停机<br/>监督器先于智能体处理解析"]
    end

    subgraph "任务运行时（Task Runtime）"
        RT["任务循环<br/>上下文组装 · 模型路由 · 工具循环"]
        SC["规划侦察员<br/>只读 · 贡献上下文"]
        AC["行动子智能体<br/>隔离 worktree · 不能提交主仓库"]
        EW["外部工作区 / 项目任务<br/>任务内 Git · 补丁工件"]
    end

    subgraph "核心演化（Core Evolution）"
        FE["递归自由演化<br/>改进本身即任务 · 完成可调度下一轮"]
        EXP["经验驱动的核心演化<br/>错误类别 · 模式登记簿 · 结构性修复"]
        SOCIAL["社交反馈（咨询性）<br/>用户指错 · 提需求 · 质疑决策"]
    end

    subgraph "受审提交闸门（Reviewed Commit Gate）"
        PRE["确定性预检<br/>版本 · 数据边界 · 体积健康"]
        FP1["暂存 diff 指纹化"]
        REVIEW["多模型 diff 评审<br/>法定多数 · 全仓库范围评审（max 模式）"]
        FP2["重新指纹校验<br/>不匹配即中止"]
        COMMIT["经评审的提交<br/>成为后续任务运行基底"]
    end

    subgraph "权威与治理（Authority & Governance）"
        CON["章程（版本化）<br/>原则 P0–P4 受保护核心<br/>常驻上下文 · 不可整体改写"]
        REPO["版本化系统仓库<br/>代码 · 提示词 · 工具 · 记忆"]
        LEDGER["评审账本 · Git 历史<br/>回滚 · 不可抵赖"]
    end

    subgraph "评估（Evaluation）"
        BENCH["基准适配器<br/>Terminal-Bench 2.1 · OSWorld-Verified<br/>CL-Bench · SWE-bench Pro · GAIA"]
        FROZEN["冻结快照评测<br/>官方验证器 · 反查阅条款<br/>运行清单 · 轨迹审计"]
        EVID["证据产物<br/>逐任务轨迹 · 清单 · 审计调整分数"]
    end

    UI --> LOG --> RT
    OP --> LS
    PANIC --> LS
    LS --> RT
    RT --> SC
    RT --> AC
    AC --> EW
    AC -->|"补丁返回父智能体验证整合"| RT

    RT -->|"业务执行 · 反思 · 插桩"| EXP
    RT --> SOCIAL
    SOCIAL --> EXP
    EXP -->|"提出结构性修复"| PRE
    FE -->|"评审当前系统后选择改进点"| PRE

    CON --> REVIEW
    REVIEW -->|"依据章程标准评审"| CON
    PRE --> FP1 --> REVIEW --> FP2 --> COMMIT
    COMMIT --> REPO
    REPO -->|"新版本成为之后的运行时"| RT
    LEDGER --> REPO

    RT --> BENCH
    BENCH -->|"冻结的种子与运行时"| FROZEN
    FROZEN --> EVID
    REPO -->|"冻结快照供基准评测；Hope 独立谱系实时演化"| FROZEN

    style COMMIT fill:#fdf2f8,stroke:#db2777,stroke-width:2px
    style CON fill:#eff6ff,stroke:#2563eb,stroke-width:2px