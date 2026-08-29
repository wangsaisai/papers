# Prime Agent 架构图

```mermaid
flowchart TD
    subgraph "人类交互层"
        Human["人类操作员"]
        AgentsView["智能体视图（Agents View）"]
    end

    subgraph "核心运行环境层"
        RootSession["根会话（Root Session）"]
        ContinualHarness["持续运行环境（Continual Harness）"]
        Daemon["守护进程（Daemon）"]
    end

    subgraph "子智能体层"
        Subagent1["子智能体 1"]
        Subagent2["子智能体 2"]
        SubagentN["子智能体 N"]
    end

    subgraph "状态层次"
        L0["L0: 模型权重"]
        L1["L1: 活跃上下文"]
        L2["L2: 持久 REPL 和子智能体"]
        L3["L3: 磁盘支持的历史、记忆、技能"]
    end

    subgraph "执行环境"
        IPythonREPL["IPython REPL"]
        Tools["工具调用"]
        CodeExec["代码执行"]
    end

    Human -->|"检查/干预"| AgentsView
    AgentsView -->|"管理会话"| Daemon
    
    Daemon -->|"启动/管理"| RootSession
    RootSession -->|"递归调用"| Subagent1
    RootSession -->|"递归调用"| Subagent2
    RootSession -->|"递归调用"| SubagentN
    
    RootSession -->|"使用"| ContinualHarness
    Subagent1 -->|"使用"| ContinualHarness
    Subagent2 -->|"使用"| ContinualHarness
    SubagentN -->|"使用"| ContinualHarness
    
    RootSession -->|"执行"| IPythonREPL
    Subagent1 -->|"执行"| IPythonREPL
    Subagent2 -->|"执行"| IPythonREPL
    SubagentN -->|"执行"| IPythonREPL
    
    IPythonREPL -->|"调用"| Tools
    IPythonREPL -->|"运行"| CodeExec
    
    RootSession -->|"读写"| L2
    Subagent1 -->|"读写"| L2
    Subagent2 -->|"读写"| L2
    SubagentN -->|"读写"| L2
    
    L2 -->|"压缩/精炼"| L3
    L3 -->|"检索"| L2
    L1 -->|"模型推理"| L0
    
    RootSession -->|"直接通信"| Subagent1
    RootSession -->|"直接通信"| Subagent2
    RootSession -->|"直接通信"| SubagentN
    Subagent1 -->|"直接通信"| Subagent2
    Subagent1 -->|"直接通信"| SubagentN
    Subagent2 -->|"直接通信"| SubagentN
    
    ContinualHarness -->|"版本化状态"| L3
    ContinualHarness -->|"补充提示"| L1
    
    Daemon -->|"持久化"| L3
    Daemon -->|"恢复"| RootSession
    Daemon -->|"恢复"| Subagent1
    Daemon -->|"恢复"| Subagent2
    Daemon -->|"恢复"| SubagentN
```

## 架构说明

Prime Agent 架构包含以下关键组件：

1. **人类交互层**：通过智能体视图（Agents View）让人类检查、附加到和管理智能体会话
2. **核心运行环境层**：根会话协调所有子智能体，持续运行环境管理跨轨迹的状态，守护进程负责会话生命周期
3. **子智能体层**：支持递归创建的子智能体，通过直接智能体间通信协调
4. **状态层次**：四层状态管理（L0-L3），从模型权重到磁盘存储
5. **执行环境**：持久 IPython REPL 支持代码执行和工具调用

这种架构实现了：
- **信息管理**：跨 L1-L3 的状态移动和持久化
- **计算管理**：测试时计算分配给程序、工具和并行子智能体
- **在线自我改进**：通过精炼机制将执行证据转化为持久状态