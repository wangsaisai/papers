# 时空可组合性的编程范式（A Programming Paradigm for Spatiotemporal Composability）

**作者**: Yifan Shi, Wei Zhang, Tianyi Cui（北京大学、DeepSeek-AI）
**翻译日期**: 2026-08-15

---

## 摘要（Abstract）

现代软件——从插件系统到自进化智能体框架（self-evolving agent harnesses）——越来越需要动态组合（dynamic composition），但其形式化基础仍不成熟。我们识别出该问题的两个正交维度：时间可组合性（temporal composability），即组件被移除时能够完整地撤销该组件副作用的能力；以及空间可组合性（spatial composability），即声明并响应式地管理组件间依赖的能力。我们通过将经典的效应（effect）与余效应（coeffect）概念提升为运行时机制来解决这两个维度。具体而言，我们形式化了可逆转效应（revertible effects），其中每个上下文变换都携带一个由运行时追踪的逆。我们形式化了响应式余效应（reactive coeffects），其中上下文的每次变化都会依据组件的余效应规约对其进行通知。我们将效应上下文与余效应上下文统一为单一上下文类型，这本身就构成了一种编程范式。之后，我们将这些机制组合为组件的概念，并给出一个动态组合演算（calculus of dynamic composition），其元理论将时空可组合性从单个组件推广到整个交错组件的系统。我们在 Cordis 中实现了这些思想——一个时空可组合性的元框架（meta-framework），它提供了一个具备效应追踪与余效应解析的核心库，以及一个具备配置调和（configuration reconciliation）与热模块替换（hot module replacement）能力的声明式组件加载器。

## 1. 引言（Introduction）

组合（Composition）——从较简单的部分组装出复杂系统——是软件工程的基础原则 [1]。传统上，组合是静态的：函数调用、模块导入和类继承在编译期解析，并在整个执行期间保持不变。然而，现代软件越来越需要动态组合：组件在运行时被加载、卸载和重新配置。插件架构 [2] 和自进化智能体框架都需要能够在运行中安全地添加和删除功能的系统，而当前的实践往往退而求其次，采用粗粒度的机制 [3]，只能通过重启（丢弃运行时状态）来重新配置。尽管动态组合的实际重要性日益增长，其理论基础仍发展不足——与静态组合可用的丰富形式框架相比尤其如此。

### 1.1 可组合性的维度（Dimensions of Composability）

为刻画动态组合的需求，我们识别出两个超出组合的代数方面（已被充分研究）的正交维度：

- **时间可组合性**（Temporal composability）处理时间维度：当组件被移除时，该组件对共享环境所做的修改必须被完整且安全地逆转。这要求追踪组件执行的每一次资源分配、事件注册和状态变更，并保证在移除时有序地回收它们。
- **空间可组合性**（Spatial composability）处理空间维度：组件必须能够以结构化、可验证的方式声明、发现和解析彼此之间的依赖。这要求管理依赖拓扑，并协调组件生命周期以响应依赖变化。

在静态设定下，时间可组合性退化为词法作用域（如 RAII [4]、bracket 模式 [5]），空间可组合性退化为模块导入解析 [6]。在动态设定下（组件在运行时到达和离开），两个维度都变得困难得多：时间可组合性必须处理生命周期长、有状态且作用域不受词法约束的效应；空间可组合性必须处理在执行期间出现、消失或改变身份的依赖。

### 1.2 动机示例（Motivating Examples）

#### 1.2.1 插件系统（Plugin Systems）

插件系统是动态组合的典型实例。我们以最广泛使用的可扩展 IDE 之一 Visual Studio Code（VSCode）作为代表性例子。

**时间层面的局限。** VSCode 在名为扩展宿主（extension host）的共享进程中运行所有扩展。虽然扩展可以动态安装，但该宿主不提供在运行时卸载单个扩展代码的机制。一旦扩展的 `activate` 函数执行完毕，禁用或卸载它就需要重启整个宿主，影响所有已加载的扩展。纯声明式扩展（如主题、键绑定和代码片段）不携带代码，可以自由移除。然而，在安装量排名前 100 的扩展中，有 87 个包含可执行代码¹，因此移除它们都需要这样的重启。虽然 VSCode 提供了 `deactivate` 钩子，但它仅仅是宿主进程终止时的优雅关闭回调，因此无法实现实时移除。此外，该钩子将效应清理与效应创建（在 `activate` 中）分离，违反了关注点局部性（locality of concern），使得完整清理难以验证。

**空间层面的局限。** VSCode 确实提供了 `extensionDependencies` 用于声明扩展之间的依赖，但它很少被使用：在安装量排名前 100 的扩展中，只有 7 个对非内置扩展声明了 `extensionDependencies`。¹ 这种稀缺反映了扩展 API 的形态，它暴露的是固定的、表面级的扩展点（如命令、视图和语言特性）。扩展通过这些点向宿主贡献功能，而不是彼此依赖，因此扩展间依赖很少出现。此外，VSCode 的扩展间交互机制不提供结构契约：它通过 `vscode.extensions.getExtension(...).exports` 向其他扩展暴露功能，但返回值是无类型的（默认是 `any`），因此依赖方无法依赖一个被检查过的接口。简言之，VSCode 将扩展引向一组固定的宿主提供扩展点，并且不提供安全、结构化的方式让它们相互依赖。

这两个局限并非 VSCode 独有；它们普遍存在于各种插件系统中 [2, 7]，只是程度不同。

¹ 数据取自 2026 年 6 月 9 日的 Visual Studio Code Marketplace。

#### 1.2.2 自进化智能体框架（Self-Evolving Agent Harnesses）

现代 AI 智能体依赖运行时智能体框架 [8–10]。这些系统可以组合多样的工具套件 [11] 和执行环境，管理权限与沙箱，维护会话状态与持久化，提供上下文管理与记忆系统 [12]，编排子智能体与多智能体工作流 [13]，并向用户和自动化暴露接口。未来的框架可能在其持续服务请求的同时，生成并部署对其自身组件的修改。模型合成的可复用工具是组件级自我修改的一个更狭窄的前兆 [14]。这样的每次修改本身就是动态组合的一次实例。

由于这些修改持续发生且很少或没有人工监督，动态可组合性变得不可或缺。没有时间可组合性，每次自我修改都会强制一次完整重启，丢弃所有进程内累积的状态；在如此高的频率下，累积的不可用时间会非常可观，进行中的任务会被反复打断；更糟的是，一次有缺陷的自我修改可能使用于恢复的进程本身失效。没有空间可组合性，每个模块都必须自行检测并适应其所依赖模块的变化（这些模块会出现、消失或改变身份），而只能通过临时手段做到；更糟的是，一个天真的代码替换策略可能静默地破坏依赖方，或引入只在重载时才暴露的循环依赖。

#### 1.2.3 粗粒度的工作区（The Coarse-Grained Workaround）

动态可组合性之所以得到的正式关注有限，一个原因是操作系统和容器编排器已经提供了粗粒度的替代品。操作系统在进程的粒度上提供时间可组合性；容器编排器 [3] 在服务的粒度上提供空间可组合性。实践中，大多数软件通过退回到这些粗粒度机制来容忍细粒度可组合性的缺失：一个行为不当的模块通过重启进程来处理，服务依赖由容器编排器管理。

然而，这种工作区付出了可观的代价。在时间上，每次重启丢弃所有进程内累积状态（如缓存、连接、部分计算），重建它们需要几秒到几分钟 [15]；期间维持可用性需要冗余副本，为弥补无法恢复单个组件而付出资源开销。在空间上，容器级编排无法表达共享同一地址空间的组件之间的依赖，并为本可以是本地函数调用的交互引入网络开销。这两种机制都作用于进程和容器边界，而现代系统越来越多地在更细的层级上进行组合。这种粒度不匹配要求一种组合抽象，在与组件本身相同的层级上管理效应和依赖。

### 1.3 贡献（Contributions）

动态可组合性的两个维度分别关注计算如何修改其环境，以及如何依赖其环境。这两个方向正是效应系统 [16, 17] 和余效应系统 [18, 19] 所形式化的：效应提供关于环境修改推理的形式词汇，余效应提供关于环境需求推理的形式词汇。然而，现有表述将推理限制在词法固定作用域上的编译期分析，并未扩展到组件在运行时到达和离开的动态场景。通过将效应提升为可逆转的运行时模型，将余效应提升为响应式依赖解析机制，我们获得了动态可组合性的统一形式基础——一个与语言无关、适用于任何需要动态组合的软件架构的基础。我们做出以下贡献：

1. 我们形式化了**可逆转效应**（第 3.1 节）：每个上下文变换都携带一个由运行时追踪的显式逆，且追踪与恢复都保持组合性质，因此在组件移除时上下文得以恢复。这建立了局部时间可组合性。
2. 我们形式化了**响应式余效应**（第 3.2 节）：组件将其所需的余效应声明为一个规约，上下文的每次变化都依据该规约将组件分类为激活（activating）、停用（deactivating）或中性（neutral）并通知它。这建立了局部空间可组合性。
3. 我们将效应上下文与余效应上下文统一为单一**上下文类型**（第 3.3 节），其中余效应上的一个观察等价关系（observational equivalence）为效应提供独立性，构成一种面向时空可组合性的编程范式。
4. 我们给出了一个**动态组合演算**（第 4 节），它将两种机制组合为组件的概念，并为组件生命周期配备操作语义。其元理论将时空可组合性从单个组件推广到整个交错组件的系统。
5. 我们在 **Cordis**（第 5 节）中实现了这些思想——一个时空可组合性的元框架，提供实现该形式模型（含效应追踪与余效应解析）的核心库，以及一个具备配置调和与热模块替换能力的声明式组件加载器。

## 2. 预备知识（Preliminaries）

本节简要概述效应与余效应系统——支撑我们工作的两大理论支柱。我们假定读者熟悉基本的类型论与范畴论；此处目标是固定记号并引入第 3 节将作为运行时机制进行操作化的关键抽象。

### 2.1 效应（Effects）

在简单类型 lambda 演算（simply typed lambda calculus, STLC）[20, 21] 中，类型判断 $\Gamma \vdash t : T$ 表示项 $t$ 在上下文 $\Gamma$ 下具有类型 $T$。效应系统提炼类型，以描述计算可能产生的副作用，得到如下形式的判断

$$\Gamma \vdash t : T^{\text{effect}} \qquad (1)$$

这里，结果类型被标注以效应代数（effect algebra）中的元素，描述计算可能产生的副作用，从而支持对有状态计算的组合推理。这一方法起源于 Lucassen 和 Gifford [22]，他们引入了一种分类型系统，区分类型、效应和区域，以发现并行程序中的调度约束。

**单子效应（Monadic effects）。** Moggi [16] 首先通过单子（monad）对计算效应进行范畴化建模；Wadler [23] 在 Haskell 中推广了该方法。范畴 𝒞 上的单子（$T$, $\eta$, $\mu$）将带效应的计算封装为类型 $T(A)$ 的值，其中 $\eta : A \to T(A)$ 提升纯值，$\mu : T(T(A)) \to T(A)$ 对嵌套计算排序。经典实例包括 Maybe 单子（用于偏序）、State 单子（用于可变状态）和 IO 单子（用于外部交互）。

**代数效应（Algebraic effects）。** Plotkin 和 Power [17, 24] 证明了代数运算决定单子，建立了一个框架，其中效应接口与其实现解耦。效应签名 Σ 声明一组运算（如状态的 get : () → S、put : S → ()）；程序自由地调用运算，而不承诺特定的解释。Plotkin 和 Pretnar [25] 随后引入效应处理器（effect handlers），通过提供续延语义来解释运算：

$$\text{handle } e \text{ with } \{ op(v, \kappa) \mapsto ... \} \qquad (2)$$

处理器接收运算参数 $v$ 和定界续延（delimited continuation）$\kappa$，可以零次、一次或多次调用它，从而在统一框架内支持异常、协程和非确定性 [26]。Koka [27, 28]、Eff [29] 和 OCaml 5 [30] 等语言以不同的设计权衡采用了代数效应。

### 2.2 余效应（Coeffects）

与效应相对偶，余效应系统 [18, 31] 丰富的是上下文而非类型，产生如下形式的判断

$$\Gamma^{\text{coeffect}} \vdash t : T \qquad (3)$$

这里，上下文被标注以余效应代数中的元素，描述计算对环境的需求，如需要访问的资源、需要持有的权限，或需要依赖的服务。效应建模程序对世界的影响，余效应则建模世界对程序的约束。

**共单子余效应（Comonadic coeffects）。** 使用共单子（comonad）来结构化上下文依赖计算的思想最早由 Uustalu 和 Vene [32] 提出，他们主张对称的（半）单式共单子是 Moggi 面向效应的单子框架的对偶，可捕捉数据流和属性求值等概念。Petricek 等人 [18] 在此基础上提出将余效应作为上下文依赖的统一静态分析。共单子（$D$, $\varepsilon$, $\delta$）捕捉上下文依赖计算：$\varepsilon : D(A) \to A$ 从上下文中提取当前值，$\delta : D(A) \to D(D(A))$ 为嵌套访问复制上下文。Environment 共单子 $D(X) = E \times X$ 建模对固定环境 $E$ 的依赖；Stream 共单子 $D(X) = \mathbb{N} \to X$ 建模对时序数据的依赖。

**分级余效应（Graded coeffects）。** 为进行更细粒度的追踪，分级余效应系统使用偏序半环（pre-ordered semiring）$\mathcal{S} = (S, \le, +, \times, 0, 1)$ 作为余效应代数 [33]，这一纪律后来被 Gaboardi 等人 [19] 与分级效应统一。$S$ 的元素标注每个变量绑定以量化其使用：0 表示未使用，1 表示线性使用，$n$ 表示有界使用，$\infty$ 表示不受限使用。半环运算按顺序（×）和并行（+）组合余效应，从而在统一的代数框架 [37] 内实现精确的资源追踪、敏感性分析 [34] 和信息流控制 [35, 36]。

### 2.3 与动态可组合性的关系（Relationship to Dynamic Composability）

效应与余效应系统沿两个互补方向组织关于计算的推理：效应描述计算如何修改其环境，余效应描述计算如何依赖其环境。这两个方向对应第 1 节识别的动态可组合性的两个维度：

- **时间可组合性**要求组件卸载时其对共享环境的修改可被逆转。相关的效应是有状态的那类，它们持久地变换环境；撤销这样的变换要求它承认一个逆。
- **空间可组合性**要求组件间依赖被声明并响应式地管理。这类依赖正是余效应所捕捉的，管理它们相当于将每个依赖对照环境所提供的内容进行解析。

然而，经典效应与余效应系统是静态工具：效应在词法固定的作用域内被追踪，并由编译期处理器释放；余效应注解在针对执行前确定的上下文验证。相比之下，动态组合要求这些保证对运行时到达和离开的组件成立，且上下文持续演化。没有固定的词法作用域可以界定部署后加载的插件；没有编译期上下文可以预见运行时配置中涌现的依赖。

这促使视角的转变：与其用更多注解扩展静态类型系统，我们将效应与余效应的概念结构实体化（reify），使运行时可以直接对它们进行操作，从而动态地建立这些系统静态提供的保证。

## 3. 可逆转效应与响应式余效应（Revertible Effects and Reactive Coeffects）

本节将第 2 节引入的效应与余效应概念提升为运行时机制，构建动态组合的理论。核心思想是：把携带效应与余效应的类型上下文变成上下文类型（context types）——将上下文实体化为头等实体、可在运行时操作的类型。对于效应类型，我们将其建模为成对携带逆（inverse）的上下文变换，从而获得局部时间可组合性。对于余效应上下文，我们将其建模为携带依赖信息的类型，从而获得局部空间可组合性。余效应上的观察等价关系随后为效应提供独立性。同时携带效应与余效应的统一上下文本身构成一种编程范式。

### 3.1 可逆转效应（Revertible Effects）

时间可组合性是指在运行时加载和卸载组件，使得卸载时共享环境恢复到组合前的状态。这要求组件对环境的每次修改既可追踪又可恢复。因此，我们将效应建模为类型 $\Gamma \to \Gamma \times (\Gamma \to \Gamma)$ 的函数：把它应用于当前上下文，它产出修改后的上下文以及一个显式的逆。提供该逆使效应可被逆转，将其返回给运行时使效应可被追踪。我们称这样的效应为可逆转的：通过在执行期间追踪和组合这些逆，完整的环境恢复成为一种结构性保证。

#### 3.1.1 效应上下文（Effect Context）

给定任意不纯函数 $f_{\text{impure}} : X \to Y$，我们将其变换为纯形式 $f : \Gamma \times X \to \Gamma \times Y$，其中 Γ 是上下文，所有可能的副作用都可以表示为对 Γ 的变换。对任意固定输入 $x : X$，诱导的映射 $\gamma \mapsto \text{pr}_1(f(\gamma, x))$ 独立于返回值捕捉 $f$ 的副作用。因此 Γ 上的效应活在复合运算 ∘ 下的变换幺半群（monoid）Γ → Γ 中，每个幺半群公理都有直接对应于效应性质的解读：

- **封闭性**：两个效应的顺序复合仍是效应；
- **结合性**：复合效应不依赖于如何加括号；
- **幺元**：Γ 上的恒等函数 idΓ 是复合的单位。

为建模可撤销的效应，我们将每个变换 $f$ 与另一个撤销 $f$ 的变换 $g$ 配对，称 $g$ 为 $f$ 的左逆（left inverse），本文中简称逆。撤销是单侧的：对逆的要求是 $g \circ f$ 而非 $f \circ g$。变换对携带自己的乘法：

**定义 1.** 定义上下文变换对的扭曲复合（twisted composition）为
$$(f_1, g_1) \circ (f_2, g_2) \coloneqq (f_1 \circ f_2, g_2 \circ g_1) \qquad (4)$$

与 ∘ 一样，左操作数在右操作数之后作用，逆按相反顺序累积。它使 $(Γ → Γ) × (Γ → Γ)$ 成为以 $(id_Γ, id_Γ)$ 为幺元的幺半群，即变换幺半群与其对偶之积，我们称之为 Γ 上的扭曲复合幺半群（twisted composition monoid）$\mathfrak{T}_Γ$。

为在上下文内部追踪效应，我们引入以下定义：

**定义 2.** 给定上下文 Γ，定义其效应上下文（effect context）为：
$$\partial Γ \coloneqq Γ \times (Γ \to Γ) \qquad (5)$$

它可以理解为一对 $(\gamma, \varphi)$，其中：

- $\gamma : Γ$ 是当前上下文状态；
- $\varphi : Γ \to Γ$ 是累加器（accumulator）——迄今为止所执行效应的逆的复合，也是将上下文恢复到初始状态的函数。

特别地，初始效应上下文可表示为 $(\gamma_0, id_Γ)$。

我们也记 $\partial^2 Γ = \partial Γ \times (\partial Γ \to \partial Γ)$，以此类推向上堆积。

由于累加器 $\varphi$ 的存在，对 $\partial Γ$ 执行的所有效应都可以被追踪和恢复。我们现在给出追踪与恢复的具体构造。

**定义 3.** 在上下文函数对上定义变换 $\text{track}_Γ$：
$$\text{track}_Γ : (Γ \to Γ) \times (Γ \to Γ) \to \partial Γ \to \partial Γ \qquad (6)$$
$$\text{track}_Γ = (f, g) \mapsto (\gamma, \varphi) \mapsto (f(\gamma), \varphi \circ g)$$

该变换将正向函数 $f$ 连同候选逆 $g$ 转换为效应上下文 $\partial Γ$ 上的变换。将 $\text{track}_Γ(f, g)$ 应用于状态 $(\gamma, \varphi)$ 会用 $f$ 变换 $\gamma$，并将逆 $g$ 复合到 $\varphi$ 上，从而在上下文中追踪 $f$ 的效应。

**定理 4.** 对每个 $(f, g) \in (Γ \to Γ) \times (Γ \to Γ)$，下图交换，即
$$\text{pr}_1 \circ \text{track}_Γ(f, g) = f \circ \text{pr}_1 \qquad (7)$$

证明. 对所有 $(\gamma, \varphi) \in \partial Γ$：
$$(\text{pr}_1 \circ \text{track}_Γ(f, g))(\gamma, \varphi) = \text{pr}_1(f(\gamma), \varphi \circ g) = f(\gamma) = (f \circ \text{pr}_1)(\gamma, \varphi) \quad \square$$

**定理 5.** trackΓ 是从 $\mathfrak{T}_Γ$ 到 $\partial Γ \to \partial Γ$ 的幺半群同态。即：
1. $\text{track}_Γ(id_Γ, id_Γ) = id_{\partial Γ}$；
2. 对所有 $(f_1, g_1), (f_2, g_2) \in \mathfrak{T}_Γ$，
$$\text{track}_Γ((f_1, g_1) \circ (f_2, g_2)) = \text{track}_Γ(f_1, g_1) \circ \text{track}_Γ(f_2, g_2) \qquad (8)$$

证明. 1. 幺元被映射到幺元，因为 $\text{track}_Γ(id_Γ, id_Γ)(\gamma, \varphi) = (\gamma, \varphi \circ id_Γ) = (\gamma, \varphi)$。
2. 对复合，取任意 $(\gamma, \varphi) \in \partial Γ$：
$$(\text{track}_Γ(f_1, g_1) \circ \text{track}_Γ(f_2, g_2))(\gamma, \varphi) = \text{track}_Γ(f_1, g_1)(f_2(\gamma), \varphi \circ g_2)$$
$$= (f_1(f_2(\gamma)), \varphi \circ g_2 \circ g_1) = \text{track}_Γ(f_1 \circ f_2, g_2 \circ g_1)(\gamma, \varphi) \quad \square$$

**定义 6.** 在 $\partial Γ$ 上定义变换 $\text{recover}_Γ$：
$$\text{recover}_Γ : \partial Γ \to \partial Γ \qquad (9)$$
$$\text{recover}_Γ = (\gamma, \varphi) \mapsto (\varphi(\gamma), id_Γ)$$

该变换将恢复函数 $\varphi$ 应用于当前状态 $\gamma$，并将 $\varphi$ 重置为单位。下面的图示说明在将一串效应 $\text{track}(f_1, g_1), \cdots, \text{track}(f_n, g_n)$ 应用于 $\partial Γ$ 后，recover 如何将上下文恢复到初始状态：

$$\Gamma \xrightarrow{f_1} \Gamma \xrightarrow{f_n} \Gamma \qquad \partial Γ \xrightarrow{\text{track}} \partial Γ \xrightarrow{\text{track}} \partial Γ \xrightarrow{\text{recover}} \partial Γ$$

图示表明：被追踪的效应后接 recover 将初始效应上下文带回自身。每个追踪步骤所保持的正是恢复（recover）本身的结果，无论从什么状态取它：

**定理 7.** 对每个 $(\gamma, \varphi) \in \partial Γ$ 和每个满足 $g(f(\gamma)) = \gamma$ 的对 $(f, g)$：
$$\text{recover}_Γ(\text{track}_Γ(f, g)(\gamma, \varphi)) = \text{recover}_Γ(\gamma, \varphi) \qquad (10)$$

证明.
$$\text{recover}_Γ(\text{track}_Γ(f, g)(\gamma, \varphi)) = \text{recover}_Γ(f(\gamma), \varphi \circ g)$$
$$= (\varphi(g(f(\gamma))), id_Γ) = (\varphi(\gamma), id_Γ) = \text{recover}_Γ(\gamma, \varphi) \quad \square$$

序列不需要单独的论证。设 $(f_1, g_1), \cdots, (f_n, g_n)$ 从 $(\gamma, \varphi)$ 按顺序应用，记 $\delta_0 = \gamma$ 且 $\delta_i = f_i(\delta_{i-1})$。由定理 5，复合 $\text{track}_Γ(f_n, g_n) \circ \cdots \circ \text{track}_Γ(f_1, g_1)$ 是扭曲复合 $(f_n \circ \cdots \circ f_1, g_1 \circ \cdots \circ g_n)$ 的 track，且若对每个 $i$ 有 $g_i(\delta_i) = \delta_{i-1}$，则 $(g_1 \circ \cdots \circ g_n)(\delta_n) = \delta_0 = \gamma$。该对因此在 $\gamma$ 处满足定理 7 的前提，应用一次该定理给出
$$\text{recover}_Γ((\text{track}_Γ(f_n, g_n) \circ \cdots \circ \text{track}_Γ(f_1, g_1))(\gamma, \varphi)) = \text{recover}_Γ(\gamma, \varphi) \qquad (11)$$

取 $(\gamma, \varphi) = (\gamma_0, id_Γ)$，恢复将每个以此方式到达的状态带回 $(\gamma_0, id_Γ)$。满足 $g \circ f = id_Γ$ 的对在每个状态满足前提。

恢复通过量 $\varphi(\gamma)$ 读取状态，我们将 $\varphi(\gamma) = \gamma_0$ 称为 $\partial Γ$ 中状态的健全性不变式（soundness invariant）。

#### 3.1.2 可逆转效应函数（Revertible Effect Functions）

前一节的 track/recover 模型将逆视为先验给定的：$\text{track}_Γ(f, g)$ 在看到任何上下文状态之前就固定了 $g$，所以一个 $g$ 要服务于效应被应用到的每个状态。然而在实践中，每个效应的逆并非先验已知：它必须由调用方在效应应用点提供。此外，recover 是全有或全无的：它不能选择性地撤销一个效应而保留其他效应。为解决这两个问题，我们在输入侧和输出侧同时增强模型：

1. 在**输入侧**，我们不仅变换 Γ，还同时返回逆函数，使逆在效应被应用的地方提供：Γ → Γ × (Γ → Γ)，即 Γ → ∂Γ；
2. 在**输出侧**，我们不仅变换 ∂Γ，还同时返回逆函数，使一个效应可以被撤销而其余效应被保留：∂Γ → ∂Γ × (∂Γ → ∂Γ)，即 ∂Γ → ∂²Γ。

这一增强保持了输入与输出之间的结构一致性，因此我们仍能定义保持 track 数学性质的理论。由此得到的类型是效应函数 $\mathfrak{E}_Γ$ 及其带见证的精细化（witnessed refinement）$\mathfrak{E}_Γ^*$：

**定义 8.** 定义效应函数 $\mathfrak{E}_Γ$ 和带见证效应函数 $\mathfrak{E}_Γ^*$：
$$\mathfrak{E}_Γ \coloneqq Γ \to Γ \times (Γ \to Γ)$$
$$\mathfrak{E}_Γ^* \coloneqq (e : Γ \to Γ \times (Γ \to Γ)) \times ((\gamma : Γ) \to ((\delta : Γ) \times (g : Γ \to Γ) \times ((\delta, g) = e(\gamma) \to g(\delta) = \gamma))) \qquad (12)$$

其中 $e(\gamma)$ 产出对 $(\delta, g)$，表示：

- $\delta : Γ$ 是新上下文；
- $g : Γ \to Γ$ 是当前效应的逆函数。

$\mathfrak{E}_Γ^*$ 的元素按状态选择其逆，约束 $g(\delta) = \gamma$ 将该选择限定为撤销效应在其应用点的作用，而 $g$ 在其他任何地方不受约束。一个满足 $g \circ f = id_Γ$ 的单一 $g$ 在每个状态同时满足该约束，并通过 $(f, g) \mapsto \gamma \mapsto (f(\gamma), g)$ 诱导出 $\mathfrak{E}_Γ^*$ 的元素，定理 11 证明这是一个同态。该约束可可视化为下面的交换图，确保逆 $e$ 确实在 $e$ 被应用的状态处逆转该变换：

$$\Gamma \xrightarrow[f]{g} \Gamma, \qquad \partial Γ \xleftarrow[e]{\text{pr}_2 / \text{pr}_1}$$

由于效应函数 $\mathfrak{E}_Γ$ 不再是上下文上的自同态，它们不能直接复合。因此我们定义效应复合的新运算：

**定义 9.** 给定函数 $f, g \in \mathfrak{E}_Γ$，定义它们的效应复合 $f \diamond g$：
$$f \diamond g : Γ \to \partial Γ$$
$$f \diamond g = \gamma \mapsto \textbf{let } (\delta, s) = g(\gamma) \textbf{ in } \textbf{let } (\varepsilon, t) = f(\delta) \textbf{ in } (\varepsilon, s \circ t) \qquad (13)$$

**定理 10.** 效应复合将 $\mathfrak{T}_Γ$ 的幺半群结构搬运到 $\mathfrak{E}_Γ$ 上。即：
1. $(\mathfrak{E}_Γ, \diamond)$ 是以 $\eta_Γ \coloneqq \gamma \mapsto (\gamma, id_Γ)$ 为幺元的幺半群；
2. 指派 $(f, g) \mapsto \gamma \mapsto (f(\gamma), g)$ 是从 $\mathfrak{T}_Γ$ 到 $\mathfrak{E}_Γ$ 的幺半群同态。

证明. 1. 结合性和幺元律按分量从 ∘ 得出。
2. 记 $e_i = \gamma \mapsto (f_i(\gamma), g_i)$；则 $(e_1 \diamond e_2)(\gamma) = (f_1(f_2(\gamma)), g_2 \circ g_1)$，正是 $(f_1, g_1) \circ (f_2, g_2)$ 的像，且 $(id_Γ, id_Γ)$ 映射到 $\eta_Γ$。□

**定理 11.** 见证在效应复合下成立，且一致逆在每个状态作见证。即：
1. $\mathfrak{E}_Γ^*$ 是 $\mathfrak{E}_Γ$ 的子幺半群；
2. 定理 10 中的同态将每个满足 $g \circ f = id_Γ$ 的对带入 $\mathfrak{E}_Γ^*$。

证明. 1. 幺元在 $\mathfrak{E}_Γ^*$ 中，因为 $id_Γ(\gamma) = \gamma$。对封闭性，取 $f, g \in \mathfrak{E}_Γ^*$ 和任意 $\gamma \in Γ$，并设 $(\delta, s) = g(\gamma)$、$(\varepsilon, t) = f(\delta)$，则 $(f \diamond g)(\gamma) = (\varepsilon, s \circ t)$。于是 $s(\delta) = \gamma$ 且 $t(\varepsilon) = \delta$，因此 $(s \circ t)(\varepsilon) = s(\delta) = \gamma$。
2. $g \circ f = id_Γ$ 在每个 $\gamma$ 给出 $g(f(\gamma)) = \gamma$，故此类对的像是每个状态都被见证的。□

正如 track 将 Γ 上的变换对提升到 ∂Γ，我们定义 effect 将 $\mathfrak{E}_Γ$ 提升到 $\mathfrak{E}_{\partial Γ}$：

**定义 12.** 定义效应函数变换 $\text{effect}_Γ$：
$$\text{effect}_Γ : \mathfrak{E}_Γ \to \partial Γ \to \partial^2 Γ \qquad (14)$$
$$\text{effect}_Γ = e \mapsto (\gamma, \varphi) \mapsto \textbf{let } (\delta, g) = e(\gamma) \textbf{ in } ((\delta, \varphi \circ g), \text{track}_Γ(g, \text{pr}_1 \circ e))$$

由于 $\text{effect}_Γ(e)$ 本身就是 $\mathfrak{E}_{\partial Γ}$，它返回的是一个按定义 8 高一层的意义上的逆。那个逆本身就是通过交换效应的两个方向得到的对的 track。普通追踪规则再次适用：撤销效应本身就是一个效应，用 $g$ 变换状态，而撤销它的方式就是再次执行该效应——这正是 $\text{pr}_1 \circ e$ 所做的。因此该逆像 track 所规定的那样复合到它所收到的累加器上。

我们现在可以证明 effect 与 track 类似的性质。

**定理 13.** effect 保持 ⋄ 运算。即，$\forall f, g \in \mathfrak{E}_Γ$：
$$\text{effect}_Γ(f) \diamond \text{effect}_Γ(g) = \text{effect}_Γ(f \diamond g) \qquad (15)$$

证明. 取任意 $(\gamma, \varphi) \in \partial Γ$，设 $(\delta, s) = g(\gamma)$ 且 $(\varepsilon, t) = f(\delta)$，则 $(f \diamond g)(\gamma) = (\varepsilon, s \circ t)$ 且 $\text{pr}_1 \circ (f \diamond g) = (\text{pr}_1 \circ f) \circ (\text{pr}_1 \circ g)$。于是
$$(\text{effect}_Γ(f) \diamond \text{effect}_Γ(g))(\gamma, \varphi) = ((\varepsilon, \varphi \circ s \circ t), \text{track}_Γ(s, \text{pr}_1 \circ g) \circ \text{track}_Γ(t, \text{pr}_1 \circ f))$$
$$= ((\varepsilon, \varphi \circ s \circ t), \text{track}_Γ(s \circ t, (\text{pr}_1 \circ f) \circ (\text{pr}_1 \circ g))) = \text{effect}_Γ(f \diamond g)(\gamma, \varphi)$$

其中第一步在 $(\gamma, \varphi)$ 和 $(\delta, \varphi \circ s)$ 处展开定义 12，第二步是定理 5，第三步折叠定义 12。□

两个层级如何关联，由下面的图示说明。其上部三角形是 $e$ 的见证条件（按定义 8），下部三角形考察 $e'$ 是否像 $e$ 一样被见证。

$$\Gamma \xrightarrow[f]{g} Γ, \quad \partial Γ \xrightarrow[f']{g'} \partial Γ, \quad \partial^2 Γ \xleftarrow[\text{pr}_2]{\text{pr}_1} e'$$

在层级之间，投影 pr1 将每个被提升的映射与它所提升的映射关联起来，正如它对 trackΓ 所做的那样（定理 4）。

**定理 14.** 设 $e \in \mathfrak{E}_Γ$，记 $f \coloneqq \text{pr}_1 \circ e$，并设 $e' \coloneqq \text{effect}_Γ(e)$ 具有正向映射 $f' \coloneqq \text{pr}_1 \circ e'$。则
1. $\text{pr}_1 \circ f' = f \circ \text{pr}_1$；
2. 对每个 $(\gamma, \varphi) \in \partial Γ$，提升的逆 $g' \coloneqq \text{pr}_2(e'(\gamma, \varphi))$ 与在此见证的逆 $g \coloneqq \text{pr}_2(e(\gamma))$ 满足 $\text{pr}_1 \circ g' = g \circ \text{pr}_1$。

证明. 1. 由定义 12，$f'(\gamma, \varphi) = (f(\gamma), \varphi \circ g)$，其状态为 $f(\gamma) = (f \circ \text{pr}_1)(\gamma, \varphi)$。
2. 这是将定理 4 应用于 $g' = \text{track}_Γ(g, f)$。□

下部三角形是否闭合，由计算提升的逆所返回的内容来决定：

**定理 15.** 设 $e \in \mathfrak{E}_Γ^*$ 且记 $f \coloneqq \text{pr}_1 \circ e$。固定 $(\gamma, \varphi) \in \partial Γ$，设 $(\delta, g) = e(\gamma)$，并记 $(\Delta, g')$ 为 $\text{effect}_Γ(e)$ 在 $(\gamma, \varphi)$ 处的值。则
$$g'(\Delta) = (\gamma, \varphi \circ g \circ f) \qquad (16)$$

状态被精确恢复。累加器也被恢复——等价地，$\text{effect}_Γ(e) \in \mathfrak{E}_{\partial Γ}^*$——当且仅当 $g \circ f = id_Γ$；且在每种情况下 $(\varphi \circ g \circ f)(\gamma) = \varphi(\gamma)$，因此健全性不变式被保持。

证明. 由定义 12，$\Delta = (\delta, \varphi \circ g)$ 且 $g' = \text{track}_Γ(g, f)$，故
$$g'(\Delta) = (g(\delta), \varphi \circ g \circ f) = (\gamma, \varphi \circ g \circ f)$$

这里用到了 $g(\delta) = \gamma$。属于 $\mathfrak{E}_{\partial Γ}^*$ 要求这在每个输入处等于 $(\gamma, \varphi)$；取 $\varphi = id_Γ$ 将累加器的相等变成 $g \circ f = id_Γ$，而该条件反过来在每个 $\varphi$ 处给出累加器的相等。最后 $(\varphi \circ g \circ f)(\gamma) = \varphi(g(\delta)) = \varphi(\gamma)$。□

因此下部三角形仅在 $\gamma$ 处见证的逆在每个状态都逆转 $f$ 时才闭合，所以 effectΓ 不把 $\mathfrak{E}_Γ^*$ 带入 $\mathfrak{E}_{\partial Γ}^*$。每种情况下都成立的是在 $\gamma$ 处的一致：$\text{recover}_Γ(g'(\Delta)) = \text{recover}_Γ(\gamma, \varphi)$，这正是定理 7 对累加器的全部假设，因此逆转不会扰动恢复目标。

按与效应应用相反的顺序逆转效应不需要更多条件，因为每个逆遇到的恰好是其自身应用所产生的状态：

**定理 16.** 设 $e_1, \cdots, e_n \in \mathfrak{E}_Γ^*$ 从 $(\gamma_0, id_Γ)$ 按顺序应用并按相反顺序逆转。则
1. 每次逆转恢复其应用所针对的上下文状态；
2. 每个中间状态满足健全性不变式。

证明. 每一步是一次应用或一次逆转。应用将 $(\gamma, \varphi)$ 带到 $(\delta, \varphi \circ g)$，其中 $g(\delta) = \gamma$，因此它由定理 7 保持 $\varphi(\gamma)$，其前提恰是 $\mathfrak{E}_Γ^*$ 的见证。按相反顺序逆转使每个逆遇到其自身应用所产生的状态，因此由定理 15，该逆转精确恢复前一状态并保持 $\varphi(\gamma)$；两个结论都不依赖于逆所收到的累加器。□

#### 3.1.3 效应的独立性（Independence of Effects）

定理 16 覆盖的是在效应自身应用所产生的状态处逆转效应；本小节覆盖的是在任何其他状态处逆转。后一种情况有两种情形。一个逆可能在后续效应仍然在位时运行——这相当于从运行中的系统撤回一个组件；一个序列还可能交错多个组件的效应，每个组件保存自己的逆的集合，使得一个组件的逆被另一个组件的应用所分隔。在这两种情形中，逆都会遇到被外来效应移动过的状态，而它是否仍能撤销它被构建来撤销的东西是一个交换（commutation）问题：需要交换的是某个效应能执行的每个变换与另一个效应能执行的每个变换，正向映射与产出的逆都一样。单一的累加器无法解决这两种情形，$\varphi$ 是一个复合，它按一种顺序同时运行它所持有的所有逆。

**定义 17.** 对效应函数 $e \in \mathfrak{E}_Γ$，变换幺半群 $\mathfrak{M}(e)$ 是由 $e$ 的正向映射连同 $e$ 产出的每个逆生成的 Γ → Γ 的子幺半群，$\mathfrak{M}(e)$ 的生成元即该生成集中的元素：
$$\mathfrak{M}(e) \coloneqq \langle \{\text{pr}_1 \circ e\} \cup \{\text{pr}_2(e(\gamma)) \mid \gamma \in Γ\} \rangle \qquad (17)$$

由对 $(f, g) \in \mathfrak{T}_Γ$ 诱导的效应以 $f$ 和 $g$ 为生成元，它在每个状态产出的逆都是 $g$。

**引理 18.** 交换在生成元上即可判定，且 ⋄ 不扩大任何变换幺半群。即：
1. 若 $\mathfrak{M}(e_1)$ 的每个生成元与 $\mathfrak{M}(e_2)$ 的每个生成元交换，则 $\mathfrak{M}(e_1)$ 的每个元素与 $\mathfrak{M}(e_2)$ 的每个元素交换；
2. $\mathfrak{M}(e_1 \diamond e_2) \subseteq \langle \mathfrak{M}(e_1) \cup \mathfrak{M}(e_2) \rangle$。

证明. 1. 与 $\mathfrak{M}(e_2)$ 的每个生成元交换的映射构成 Γ → Γ 的子幺半群，因为 $id_Γ$ 在其中，且 $f \circ f'$ 在 $f$ 与 $f'$ 都满足时也在其中。该子幺半群由假设包含 $\mathfrak{M}(e_1)$ 的生成元，因而包含 $\mathfrak{M}(e_1)$。固定 $f \in \mathfrak{M}(e_1)$，与 $f$ 交换的映射同样构成包含 $\mathfrak{M}(e_2)$ 的生成元、因而包含 $\mathfrak{M}(e_2)$ 的子幺半群。
2. 由定义 9，$e_1 \diamond e_2$ 的正向映射是 $(\text{pr}_1 \circ e_1) \circ (\text{pr}_1 \circ e_2)$，它在任何状态产出的逆是 $s \circ t$，其中 $s$ 由 $e_2$ 产出、$t$ 由 $e_1$ 产出。因此 $\mathfrak{M}(e_1 \diamond e_2)$ 的每个生成元都是两者的生成元的复合。□

**定义 19.** 当以下条件成立时，效应函数 $e_1, e_2 \in \mathfrak{E}_Γ$ 称为独立的（independent）：
1. 其中一个的每个变换与另一个的每个变换交换，
$$\forall f \in \mathfrak{M}(e_1), g \in \mathfrak{M}(e_2).\ f \circ g = g \circ f \qquad (18)$$
2. 其中一个的变换都不扰乱另一个产出的逆，
$$\forall g \in \mathfrak{M}(e_2), \gamma \in Γ.\ \text{pr}_2(e_1(g(\gamma))) = \text{pr}_2(e_1(\gamma)) \qquad (19)$$
且 $e_1$ 与 $e_2$ 互换后的相同条件也成立。

族 $(e_l)_{l \in L}$ 是两两独立的（pairwise independent），当对每个 $l \neq l'$，$e_l$ 与 $e_{l'}$ 独立。一个族可以重复同一个效应函数，使一个效应函数与其自身独立等价于要求 $\mathfrak{M}(e)$ 交换。

对于由对 $(f_1, g_1)$ 和 $(f_2, g_2)$ 诱导的效应，条款 (1) 由引理 18(1) 化为四对的交换：$f_1, f_2$；$g_1, g_2$；$f_1, g_2$；$g_1, f_2$，而条款 (2) 无条件成立，因为诱导效应在每个状态只产出一个逆。⋄ 下的交换是另一种性质。$e_1 \diamond e_2 = e_2 \diamond e_1$ 所等同的是两种顺序的复合正向映射和两种顺序的复合逆，每个逆在其自身应用所产生的状态进入复合；独立性则相反，将某个效应的每个变换与另一个效应的每个变换相关联，包括正向映射与外来逆的配对。

在独立性下，逆可以在后续效应已移动的状态处运行，且它在彼处撤回的是它自己的贡献而非其他：

**定理 20.** 设 $e_1, \cdots, e_n \in \mathfrak{E}_Γ^*$ 两两独立并从 $\gamma_0$ 按顺序应用。记 $f_i \coloneqq \text{pr}_1 \circ e_i$，设 $\delta_i \coloneqq f_i(\delta_{i-1})$ 且 $\delta_0 \coloneqq \gamma_0$，并设 $g_i \coloneqq \text{pr}_2(e_i(\delta_{i-1}))$ 为 $e_i$ 在其应用处产出的逆。固定 $j$，记 $\delta'_i \coloneqq (f_i \circ \cdots \circ f_{j+1})(\delta_{j-1})$ 为省略 $e_j$ 后的序列的状态，使得 $\delta'_j = \delta_{j-1}$。则对每个满足 $j \le u \le n$ 的 $u$：
1. $\delta_u = f_j(\delta'_u)$ 且 $g_j(\delta_u) = \delta'_u$；
2. 每个 $i > j$ 的 $e_i$ 在 $\delta'_{i-1}$ 产出与它在 $\delta_{i-1}$ 产出的相同逆 $g_i$。

证明. 1. 第一个等式是对 $u$ 的归纳。在 $u = j$ 处它读作 $\delta_j = f_j(\delta_{j-1})$，正是 $\delta_j$ 的定义。对归纳步骤，$\delta_{u+1} = f_{u+1}(\delta_u) = f_{u+1}(f_j(\delta'_u)) = f_j(f_{u+1}(\delta'_u)) = f_j(\delta'_{u+1})$，中间的等式是 $e_{u+1}$ 与 $e_j$ 的定义 19 条款 (1)，它们是族中不同的效应，因为 $u + 1 > j$。对第二个等式，条款 (1) 将 $g_j$ 穿过 $e_j$ 之后应用的正向映射，使 $e_j$ 的见证在它成立的唯一状态处使用：
$$g_j(\delta_u) = (g_j \circ f_u \circ \cdots \circ f_{j+1})(\delta_j) = (f_u \circ \cdots \circ f_{j+1})(g_j(f_j(\delta_{j-1}))) = \delta'_u$$

最后一个等式依赖于 $g_j(f_j(\delta_{j-1})) = \delta_{j-1}$，这是定义 8 要求 $e_j$ 在 $\delta_{j-1}$ 处的见证。

2. 由 (1)，状态 $\delta'_{i-1}$ 是 $f_j(\delta_{i-1})$，且 $f_j \in \mathfrak{M}(e_j)$，故对 $e_i$ 与 $e_j$ 的定义 19 条款 (2) 给出 $\text{pr}_2(e_i(f_j(\delta_{i-1}))) = \text{pr}_2(e_i(\delta_{i-1}))$。□

条款 (1) 定位逆所到达的状态：它是同一序列若从未应用该效应（无论其后应用了什么效应）本会到达的状态。条款 (2) 定位其他逆在彼处所持有的内容，两者一起使该定理可以再次应用于更短的序列：

**推论 21.** 设 $e_1, \cdots, e_n \in \mathfrak{E}_Γ^*$ 两两独立并从 $\gamma_0$ 按顺序应用，$g_1, \cdots, g_n$ 如上。按照 {1, ⋯, n} 的任意置换的次序在 $\delta_n$ 处应用这 $n$ 个逆会到达 $\gamma_0$。

证明. 对 $n$ 向下归纳。设置换以 $j$ 开头。由定理 20(1)，在 $\delta_n$ 应用 $g_j$ 到达 $\delta'_n$——省略 $e_j$ 的序列所到达的状态，且由定理 20(2)，其余效应在彼处产出的逆正是手中的 $g_i$。该序列两两独立，因为是子族，所以归纳假设适用于它和置换的其余部分；空序列到达 $\gamma_0$。□

LIFO 次序就是这样的一个置换，而定理 16 在没有任何假设的情况下按此次序逆转。独立性带来的是所有其他次序，以及随之而来的交错多个组件的序列，第 4.4.2 节将它带到整个系统的轨迹上。

这些构造合起来构成可逆转效应：$\mathfrak{E}_Γ^*$ 中的每个效应函数显式提供自己的逆，effect 在效应上下文 $\partial Γ$ 上追踪这些逆，⋄ 运算在保持可逆转性的同时复合它们。它们交付的是局部时间可组合性——"局部"在于该保证是对单个组件的效应本身读出的。我们将其定为如下准则：对组件应用的每个效应函数序列，累加器恢复它所开始的上下文（定理 7），且逆转该序列使每个逆遇到其自身应用所针对的状态（定理 16）。加载组件就是应用这样的序列并把逆累积进 $\varphi$；卸载它则是应用 $\varphi$。

该准则遗漏了两件事，两件都在多个组件参与时到来：偏离累加器强加次序的逆转，以及交错他人效应的序列。独立性交付它们（推论 21），且它是关于效应的条件而非构造的性质——第 3.3.2 节识别满足它的纪律，第 4.4.2 节将保证直接读自整个系统的轨迹。在独立性失败之处，次序必须由别处承载：在一个组件内部由累加器承载，它按 LIFO 次序任意地逆转效应（第 4.3.2 节）；跨组件则由声明的余效应承载，它使一次激活对另一次激活排序（第 4.3.1 节）。

### 3.2 响应式余效应（Reactive Coeffects）

空间可组合性是指组件能够声明对彼此的依赖，且系统在运行时解析、提供和撤回这些依赖的能力。这要求每当共享上下文改变时重新评估依赖满足状态，使组件在其依赖可用时激活、在依赖被撤回时停用。因此，我们将组件的依赖建模为一个规约（specification），并将上下文的每个变化对照该规约分类为激活、停用或中性。对照规约分类正是检测满足状态的变化；响应这一分类则驱动激活与停用。我们称这样的余效应为响应式的：通过分类上下文变化并从中驱动激活与停用，正确的余效应排序成为一种结构性保证。

#### 3.2.1 余效应上下文（Coeffect Context）

传统的控制反转（inversion-of-control, IoC）容器 [38] 通常将依赖建模为简单的键值映射。本节将 IoC 形式化为与可逆转效应协同的余效应上下文，为动态组合提供数学基础。

**定义 22.** 给定类型族 $\mathcal{V} : K \to \text{Type}$，定义余效应上下文为依赖部分函数类型：
$$\Sigma \coloneqq (k : K) \rightharpoonup \mathcal{V}_k \qquad (20)$$

其中 $\sigma : \Sigma$ 是有限部分函数，为每个 $k \in \text{dom}(\sigma) \subseteq K$ 指派 $\mathcal{V}_k$ 类型的值。我们记：

- $\sigma(k)$ 为应用（当 $k \in \text{dom}(\sigma)$ 时有定义）；
- $\sigma[k \mapsto v]$ 为在 $k$ 处绑定 $v$、其余与 $\sigma$ 一致的表；
- $\sigma \setminus k$ 为限制（当 $k \in \text{dom}(\sigma)$ 时有定义）；
- $k \in \text{dom}(\sigma)$ 为隶属。

使用类型族 $\mathcal{V}$ 确保每个依赖键 $k$ 关联特定的值类型 $\mathcal{V}_k$，为依赖访问提供静态类型安全。扩展与限制携带前置条件，由下面的运算施加：依赖不能被提供两次（扩展要求 $k \notin \text{dom}(\sigma)$），也不能在缺席时被撤销（限制要求 $k \in \text{dom}(\sigma)$）。违反前置条件会以错误形式被报告且不产生转移，因此描述实际发生的转移的效应代数对这些运算不加修改地适用。偏好将失败内化的读者可以把下面的每个 $\Sigma \rightharpoonup \Sigma$ 读作 $\Sigma \to \text{Maybe}(\Sigma)$ 并在 Maybe 单子中复合（第 2.1 节），代价是用运算定义域上的部分恒等替换每个恒等。基于此上下文结构，我们定义两个核心运算：

**定义 23.** 在 Σ 上定义 get 与 set 运算：
$$\text{get} : (k : K) \to \Sigma \rightharpoonup \mathcal{V}_k$$
$$\text{get} = k \mapsto \sigma \mapsto \sigma(k)$$
$$\text{set} : (k : K) \times \mathcal{V}_k \to \Sigma \rightharpoonup \Sigma \times (\Sigma \rightharpoonup \Sigma) \qquad (21)$$
$$\text{set} = (k, v) \mapsto \sigma \mapsto (\sigma[k \mapsto v], \lambda \sigma'. \sigma' \setminus k)$$

其中 get(k) 以 $k \in \text{dom}(\sigma)$ 为前置条件，set(k, v) 以 $k \notin \text{dom}(\sigma)$ 为前置条件。

值得注意的是，set(k, v) 的类型是 $\mathfrak{E}_\Sigma^*$，恰好是余效应上下文上的效应函数。因此我们可以直接应用第 3.1 节的效应机制：effectΣ 提供依赖注册的自动追踪与恢复。这就是响应式余效应与可逆转效应之间的协同：余效应运算就是效应，而效应是可逆转的。

get 交给组件的是一个值，组件能用该值做什么取决于该键处的余效应所提供的内容。因此一个键携带的比值类型更多：

**定义 24.** 键 $k$ 处的余效应是一个三元组 $(\mathcal{V}_k, \simeq_k, \mathcal{A}_k)$，其中 $\mathcal{V}_k$ 是定义 22 的值类型，$\simeq_k$ 是 $\mathcal{V}_k$ 上的等价关系（值在 $k$ 处按它比较，见第 3.3.2 节），$\mathcal{A}_k$ 是余效应运算的集合——绑定在 $k$ 处的值向持有它的组件提供的运算。运算 $a \in \mathcal{A}_k$ 携带参数类型 $X_a$ 和结果类型 $B_a$，并单独作用于该值：
$$a : X_a \to \mathcal{V}_k \rightharpoonup \mathcal{V}_k \times (\mathcal{V}_k \rightharpoonup \mathcal{V}_k) \times B_a \qquad (22)$$

其前两个成分构成 $\mathcal{V}_k$ 上的效应函数（按定义 8 的要求被见证），第三个是结果。每个运算都必须尊重 $\simeq_k$：在 $\simeq_k$ 相关的值处，它要么在两者都有定义、要么在两者都无定义；在有定义处，它产出 $\simeq_k$ 相关的后继，将 $\simeq_k$ 相关的值带到 $\simeq_k$ 相关值的逆，以及相等的结果。运算通过其提升作用于余效应上下文：

$$a_\Sigma(x)(\sigma) \coloneqq \textbf{let } (v, g, b) = a(x)(\sigma(k)) \textbf{ in } (\sigma[k \mapsto v], \lambda \sigma'. \sigma'[k \mapsto g(\sigma'(k))], b) \qquad (23)$$

当 $k \in \text{dom}(\sigma)$ 时有定义，其前两个成分是 Σ 上的效应函数。

将 $k$ 的运算类型化为 $\mathcal{V}_k$ 上的运算正是把运算限制在 $k$ 处的绑定：提升读和写该绑定，其余键保持原样，因此无需附带条件来说明这一点。在隔离（isolation）生效之处，它所到达的绑定是领域（realm）所解析到的绑定（定义 28），两个键共享一个领域即共享一个绑定。行为取决于其他键的运算将该键的值读入其参数 $X_a$，而下一小节的响应式纪律在读取它的组件运行期间保持该值固定（定理 63）。

#### 3.2.2 规约与通知（Specification and Notification）

上述定义描述单个依赖如何被注册和访问。然而，访问缺席的依赖是运行时失败。组件应该只在它声明的所有依赖都存在时才激活，而不是乐观地访问并在缺失时失败。这引出两个问题：组件声明的依赖是否被联合满足，以及系统应如何响应满足状态的变化。余效应上下文 Σ 携带一种自然的观察结构，使两个问题都可处理：对任意余效应规约 $d \subseteq K$，定义满足谓词（satisfaction predicate）：
$$\sigma \models d \coloneqq \forall k \in d.\ k \in \text{dom}(\sigma) \qquad (24)$$

该谓词是可判定的（因为 $\text{dom}(\sigma)$ 有限）。由于对 σ 的所有变更都经由效应函数（其逆恢复之前的定义域），满足状态的变化在每个效应边界处都是可检测的。这就是响应性的代数基础：效应系统保证每个余效应变化都被观察到。

**定义 25.** 余效应规约（coeffect specification）是：
$$\mathfrak{D}_\Sigma \coloneqq \text{Set}(K) \qquad (25)$$

表示组件从环境声明的依赖集合。

使这个规约具有响应性的，是它如何分类状态转移。任何将 σ 变换为 σ′ 的效应都可以由规约 $d \in \mathfrak{D}_\Sigma$ 按 $d$ 的满足状态是否被改变来分类：

**定义 26.** 给定余效应规约 $d \subseteq K$ 和状态 $\sigma, \sigma' \in \Sigma$，定义：
$$\text{notify}_d(\sigma, \sigma') \coloneqq \begin{cases} \text{activating} & \text{if } \sigma \nvDash d \wedge \sigma' \vDash d \\ \text{deactivating} & \text{if } \sigma \vDash d \wedge \sigma' \nvDash d \\ \text{neutral} & \text{otherwise} \end{cases} \qquad (26)$$

这是良定义的，因为 $\sigma \vDash d$ 可判定，且所有状态转移都由效应函数中介。响应式不变式是：激活转移触发组件效应的执行（带完整效应追踪），而停用转移通过应用累加器触发恢复。这些转移的精确操作语义取决于它们与控制流的交互，在第 4 节发展。

set 与 notify 共同交付局部空间可组合性——"局部"与之前含义相同，保证是对单个组件的余效应本身读出的。我们将其定为如下准则：组件只在满足其规约的状态激活，因此它从不读取缺席的绑定；上下文的每个变化都对照该规约被分类，因此满足状态的丧失在其发生处被检测并驱动停用。两半都从上述定义直接得出，满足是组件将激活处检查的前置条件，notify_d 在每个转移处有定义。

该准则覆盖余效应排序的一个方向而非另一个。如果组件 A 提供键 k，组件 B 声明 $k \in d_B$，那么 B 只能在 A 激活并提供 k 之后激活，因为 $\sigma \vDash d_B$ 要求 $k \in \text{dom}(\sigma)$。反过来不成立：卸载 A 将 k 从 $\text{dom}(\sigma)$ 中移除，从而破坏 B 的满足状态，但通知本身无法在 B 自己的拆除（teardown）需要期间保持 k 可读，也无法让 A 的恢复等待 B 完成。将撤回安排在它所导致的停用之后，是对其他组件而非行为者的条件，因此属于该保证的全局形式，第 4.3.1 节提供所需机制。

#### 3.2.3 隔离与拦截（Isolation and Interception）

基本余效应上下文 Σ 建模扁平的依赖表。然而在实践中，系统可能需要为不同组件将不同值绑定到同一逻辑依赖。本节用两种机制扩展余效应上下文：余效应隔离（coeffect isolation，同一键在不同上下文中解析不同）与余效应拦截（coeffect interception，依赖访问上的横切行为）。

**实现方式。** 两种机制与 get、set 的不同在于它们作用于什么。提供（provision）写入所有组件读取的共享表，所以它是该表上的效应并携带撤回它的逆。隔离与拦截则调整键在某个上下文下为组件解析的方式，表本身保持原样。将运算类型化为效应固定其指称（denotation）——后继状态配一个逆——但不固定其实现（realization），后者决定该逆如何被执行。

**定义 27.** 上下文上的效应函数承认两种实现：
- **原地实现**（In-place realization）修改上下文并返回非平凡逆；后继状态与输入互为别名，恢复运行该逆以撤销变更。
- **派生实现**（Derived realization）保持输入不变，返回从中派生出的新上下文，以恒等为逆；恢复丢弃派生的上下文。从一个上下文派生的上下文是定义 32 的递归结构所承载的。

在纯函数式设定下两者重合，命令式宿主可以对每个运算任选其一；第 5.1.2 节实现了两者。隔离与拦截被直接赋予派生实现：各自产生新上下文，其自身表与继承的表不同，因此两者在下面被类型化为从上下文到上下文的映射，而非效应函数。共享表没有任何变化，所以没有逆可追踪，也没有供定义 12 提升的东西，恢复将派生上下文连同其携带的调整一起丢弃。对派生表的赋值覆盖继承表在键处的内容，这就是为什么两个运算都不携带前置条件。

**余效应隔离（Coeffect Isolation）。** 通过引入隔离领域（isolation realms），余效应隔离允许同一依赖在不同上下文中绑定到不同值。这在多租户系统、测试环境和组件沙箱中有广泛的应用。

**定义 28.** 定义带隔离的余效应上下文：
$$\Sigma^{\text{iso}} \coloneqq (K \rightharpoonup R) \times ((r : R) \rightharpoonup \mathcal{V}_r) \qquad (27)$$

它可以表示为一对 $(\rho, \sigma)$，其中：

- $\rho : K \rightharpoonup R$ 是隔离领域表，为每个被隔离的键指派领域标识符；$\text{dom}(\rho)$ 之外的键解析到自己的领域，因此我们在彼处记 $\rho(k) = k$（$R \supseteq K$）；
- $\sigma : (r : R) \rightharpoonup \mathcal{V}_r$ 是依赖表，从领域标识符到类型化值的部分依赖函数。

两层映射结构将逻辑层与存储层解耦，使依赖访问具有上下文感知能力。访问键 k 时，系统先将 $\rho(k)$ 解析为领域标识符 r，然后访问 $\sigma(r)$ 获取实际值。

**定义 29.** 在 $\Sigma^{\text{iso}}$ 上定义 get、set 与 isolate 运算：
$$\text{get} : (k : K) \to \Sigma^{\text{iso}} \rightharpoonup \mathcal{V}_{\rho(k)}$$
$$\text{get} = k \mapsto (\rho, \sigma) \mapsto \sigma(\rho(k))$$
$$\text{set} : (k : K) \times \mathcal{V}_{\rho(k)} \to \Sigma^{\text{iso}} \rightharpoonup \Sigma^{\text{iso}} \times (\Sigma^{\text{iso}} \to \Sigma^{\text{iso}}) \qquad (28)$$
$$\text{set} = (k, v) \mapsto (\rho, \sigma) \mapsto ((\rho, \sigma[\rho(k) \mapsto v]), \lambda(\rho', \sigma'). (\rho', \sigma' \setminus \rho'(k)))$$
$$\text{isolate} : K \times R \to \Sigma^{\text{iso}} \to \Sigma^{\text{iso}}$$
$$\text{isolate} = (k, r) \mapsto (\rho, \sigma) \mapsto (\rho[k \mapsto r], \sigma)$$

其中 get 与 set 携带沿 ρ 搬运的定义 23 的前置条件，即 $\rho(k) \in \text{dom}(\sigma)$ 与 $\rho(k) \notin \text{dom}(\sigma)$。isolate(k, r) 派生的上下文将领域 r 指派给 k 并原样继承依赖表，因此已被隔离的键是被重新指派而非拒绝。

余效应隔离机制本质上实现了运行时特设多态（ad-hoc polymorphism）系统。通过隔离领域标识符，同一依赖键在不同上下文可以解析到完全不同的值，且这种多态可以在运行时动态调整。与传统依赖注入相比，余效应隔离提供更细粒度的控制，能够为特定组件定制隔离；set 仍是效应函数（$\mathfrak{E}_{\Sigma^{\text{iso}}}^*$），因此继承可逆转性，而 isolate 不需要逆，它派生上下文而非写入共享表。

**余效应拦截（Coeffect Interception）。** 第二种机制将横切元数据附加到依赖访问上，在不修改依赖值的情况下增加行为。这种元数据既可以由上下文携带，也可以由组件声明，因此我们同时扩展余效应上下文和余效应规约：

**定义 30.** 定义带拦截的余效应上下文与规约：
$$\Sigma^{\text{inter}} \coloneqq ((k : K) \to \mathcal{M}_k) \times ((k : K) \rightharpoonup (\mathcal{M}_k \to \mathcal{V}_k)) \qquad (29)$$
$$\mathfrak{D}^{\text{inter}} \coloneqq (k : K) \rightharpoonup \mathcal{M}_k$$

上下文 $\Sigma^{\text{inter}}$ 是一对 $(\iota, \sigma)$：$\iota$ 是安装在上下文本身上的上下文携带元数据，默认为空（$\epsilon_k$）；σ 将每个键映射到一个从元数据 $\mathcal{M}_k$ 到值 $\mathcal{V}_k$ 的提供者函数。规约 $d \in \mathfrak{D}^{\text{inter}}$ 携带组件声明的元数据，为每个键指派其元数据 $d(k)$，$\text{dom}(d)$ 充当依赖集。每个键为其元数据配备幺半群 $(\mathcal{M}_k, \oplus_k, \epsilon_k)$：合并 $\oplus_k$ 结合且以 $\epsilon_k$（空元数据）为幺元。

**定义 31.** 在 $\Sigma^{\text{inter}}$ 上定义 get、set 与 intercept 运算：
$$\text{get} : (k : K) \times \mathcal{M}_k \to \Sigma^{\text{inter}} \rightharpoonup \mathcal{V}_k$$
$$\text{get} = (k, \mu) \mapsto (\iota, \sigma) \mapsto \sigma(k)(\mu \oplus_k \iota(k))$$
$$\text{set} : (k : K) \times (\mathcal{M}_k \to \mathcal{V}_k) \to \Sigma^{\text{inter}} \rightharpoonup \Sigma^{\text{inter}} \times (\Sigma^{\text{inter}} \to \Sigma^{\text{inter}}) \qquad (30)$$
$$\text{set} = (k, \psi) \mapsto (\iota, \sigma) \mapsto ((\iota, \sigma[k \mapsto \psi]), \lambda(\iota', \sigma'). (\iota', \sigma' \setminus k))$$
$$\text{intercept} : (k : K) \times \mathcal{M}_k \to \Sigma^{\text{inter}} \to \Sigma^{\text{inter}}$$
$$\text{intercept} = (k, \nu) \mapsto (\iota, \sigma) \mapsto (\iota[k \mapsto \iota(k) \oplus_k \nu], \sigma)$$

其中 get 与 set 在提供者表上携带定义 23 的前置条件，即 $k \in \text{dom}(\sigma)$ 与 $k \notin \text{dom}(\sigma)$。intercept(k, ν) 派生的上下文将 ν 合并到 k 处继承的元数据上，原样继承提供者表。

当带规约 d 的组件访问键 k 时，系统求值 $\sigma(k)(d(k) \oplus_k \iota(k))$：组件声明的元数据与上下文携带的元数据 ι 合并，提供者函数应用于结果。该合并遵循每个键自身的语义（如标量字段被覆盖、集合字段取并集），且是右偏的，所以 $\iota(k)$ 优先并可覆盖组件的声明，使外层上下文可以不修改组件就约束组件如何使用余效应（如第 6.3 节）。

### 3.3 上下文范式（The Context Paradigm）

第 3.1 节与第 3.2 节各自作用于一个上下文，前者作为效应的载体，后者作为余效应的载体，但一个同时携带两者的单一上下文是什么样的仍是开放的。本节为该统一给出具体构造，从余效应组装出为第 3.1.3 节留空的效应独立性提供来源的观察等价关系，并论证由此得到的上下文类型本身构成一种编程范式。

#### 3.3.1 统一上下文（Unified Context）

对于上下文 Γ，效应上下文 $\partial Γ$（第 3.1 节）提供更高层的抽象，携带上一层的上下文和该层的累加器（定义 2）。使该结构递归并与余效应上下文 Σ 组合，得到如下类型：

**定义 32.** 上下文类型 Γ∞ 定义为：
$$\Gamma^\infty \coloneqq \mu Γ.\ Γ \times (Γ \to Γ) \times \Sigma \qquad (31)$$

其中三个投影是：

- Γ：当前上下文状态（递归的）；
- Γ → Γ：恢复本层效应的累加器；
- Σ：携带依赖信息的余效应上下文。

在此定义下，效应映射将 $\mathfrak{E}_{Γ^\infty}$ 映射到自身，把 $\partial$-塔统一为单一自相似类型。余效应上下文 Σ 被结构化地整合：依赖运算（set、get）作用于 Σ，累加器追踪其逆转。由于 Σ 底层的类型族 $\mathcal{V}$ 不受约束，系统需要在组件间共享的任何状态都可以编码为带适当值类型的依赖——Σ 包含所有共享可变状态，不仅是组件间依赖。组件与环境之间的每次交互都通过这一单一实体。

**层级组合（Hierarchical composition）。** Γ∞ 的递归结构支持层级控制：父上下文聚合多个子层级的效应，形成树状控制结构，在保持模块化的同时实现统一的跨层级管理。效应变换实现了字面的"插件"隐喻：

- **加载组件**对应于执行其效应（插入）；
- **卸载组件**对应于恢复其效应（拔出，不影响其他运行中的组件）；
- **层级不同层的组件**可独立加载和卸载；父上下文聚合和管理其所有子级的效应，支持任意嵌套的组合。

#### 3.3.2 观察等价（Observational Equivalence）

第 3.1 节的恢复保证断言状态相等（定理 7），这是一种理想化，因为物理状态无法恢复到原状。例如，free 将块释放给分配器，却不会恢复 malloc 之前堆的布局；生成性名字（generative name）也不会被丢弃它的逆所恢复，因为下一次创建会取出一个新名字 [39]。因此第 3 节的相等要把 ≃ 读到等价关系为止，而我们取 ≃ 为观察等价关系：当没有任何观察者能区分两个状态时，它们是相关的。比较行为而非表示是程序等价的标准路径 [40]，而这类比较所得的关系取决于观察者被给予什么 [41]。上下文观察者被给予的是它携带的余效应，每个余效应自带等价关系（定义 24），因此上下文上的关系由它们的关系组装而成。组装是本小节的事，除以它正是第 3.1.3 节所求独立性的代价。

**定义 33.** 两个余效应上下文在把相同键绑定到相关值时相关，上下文的两个状态在其余效应投影相关时相关：
$$\sigma \simeq \sigma' \coloneqq \text{dom}(\sigma) = \text{dom}(\sigma') \wedge \forall k \in \text{dom}(\sigma).\ \sigma(k) \simeq_k \sigma'(k) \qquad (32)$$
$$\gamma \simeq \gamma' \coloneqq \sigma_\gamma \simeq \sigma_{\gamma'}$$

其中 $\sigma_\gamma$ 记 γ 的余效应投影（定义 32）。

没有键绑定的那部分状态因此被遗忘，正是遗忘它使定理 7 能够被读到 ≃：在上述例子中，堆布局和生成性名字都位于关系之外，除非某个键绑定它们。第 3.2.2 节对 ≃ 的需要是其推导得出而非假设。相关的状态有相同定义域，因此在满足谓词 $\sigma \models d$ 和定义 26 的分类 notify_d 上一致，响应性是 Σ/≃ 的性质。

称此关系为观察性的，是对每个 $\simeq_k$ 的主张：它分离的不多于 k 的运算所能区分的。值的观察者运行这些运算并读取它们的结果。

**定义 34.** 设 V 依定义 24 的意义携带运算集 $\mathcal{A}$，记 $\mathfrak{M}(a)$ 为效应函数 $a(x)$ 在每个参数 $x : X_a$ 上的变换幺半群（定义 17）。$\mathcal{A}$ 上的测试（test）是幺半群 $\mathfrak{M}(a)$（$a \in \mathcal{A}$）生成元上的有限字，每个字母应用于此前字母留下的值；其结果是在途中的运算正向映射所产出的结果，在前置条件失败处无定义。当 $\mathcal{A}$ 上的每个测试在 $v$ 与 $v'$ 两处都有定义或都无定义且在两者产生相同结果时，值 $v, v' : V$ 是不可区分的（indistinguishable），记作 $v \approx_{\mathcal{A}} v'$。

**引理 35.** 不可区分性是运算所尊重的最粗关系。即：
1. $\mathcal{A}$ 的每个运算尊重定义 24 意义上的 ≈；
2. $\mathcal{A}$ 的每个运算所尊重的每个等价关系都包含于 ≈。

因此每个可接受的 ≃ 选择都包含于 $\approx_k$，且 $\approx_k$ 本身是可接受的。

证明. 1. 设 $v \approx_{\mathcal{A}} v'$ 且 $a \in \mathcal{A}$ 被应用于某参数。在测试前加一个字母仍是测试，所以正向映射到达的值不可区分，任一产出的逆从不可区分的参数所到达的值也不可区分；单字母测试给出定义性一致和结果的相等。
2. 设 R 是这样一个等价关系且 $vRv'$。测试的每个字母是运算的正向映射或产出的逆，尊重关系沿两者都传递，使到达的值保持相关、每个字母处的结果相等。因此每个测试在 $v$ 和 $v'$ 处一致。□

将 ≃ 替换 = 并不是全部，因为效应函数除了状态还返回逆，而 ≃ 所等同的两个状态必须产出 ≃ 所等同的逆。

**定义 36.** 当 $\forall \gamma, \gamma' \in Γ.\ \gamma \simeq \gamma' \Rightarrow f(\gamma) \simeq f(\gamma')$ 时，映射 $f : Γ \to Γ$ 尊重 ≃：
$$\forall \gamma, \gamma' \in Γ.\ \gamma \simeq \gamma' \Rightarrow f(\gamma) \simeq f(\gamma') \qquad (33)$$

两个映射在所有状态一致时相关，$\partial Γ$ 中的两个对在两个分量都相关时相关：
$$f \simeq g \coloneqq \forall \gamma \in Γ.\ f(\gamma) \simeq g(\gamma)$$
$$(\delta, g) \simeq (\delta', g') \coloneqq \delta \simeq \delta' \wedge g \simeq g' \qquad (34)$$

尊重 ≃ 的映射是下降到 Γ/≃ 的映射，由 ≃ 相关的映射是下降到同一映射的两个映射。效应函数两者都需要：前者使其计算的状态在商上确定，后者使其返回的逆也是。

**定义 37.** 将定义 8 读到 ≃：当 $e$ 作为映射 Γ → ∂Γ 尊重 ≃，且对每个 $\gamma \in Γ$、记 $(\delta, g) = e(\gamma)$ 时满足以下条件的 $e \in \mathfrak{E}_Γ$ 属于 $\mathfrak{E}_Γ^*$：
1. $g(\delta) \simeq \gamma$；
2. $g$ 尊重 ≃。

在 Γ 上取 ≃ 为相等即恢复定义 8。

**引理 38.** 在 $\mathfrak{E}_Γ^*$ 按定义 37 读取的情况下，第 3.1 节断言的每个状态相等在将 = 替换为 ≃ 后成立，且从 $(\gamma_0, id_Γ)$ 可到达的每个状态的累加器尊重 ≃。

证明. 累加器是逆的复合，每个逆按定义 37(2) 尊重 ≃，尊重 ≃ 的映射的复合尊重 ≃，基础情形是 $id_Γ$。第 3.1 节的证明随后原样通过，尊重关系正是把关系穿过逆的东西：从 $g_2(\delta_2) \simeq \delta_1$ 和 $g_1(\delta_1) \simeq \gamma$，尊重关系给出 $(g_1 \circ g_2)(\delta_2) \simeq \gamma$，这正是每个逆复合所走的步骤，而定理 7 的健全性不变式按该步骤读作 $\varphi(\gamma) \simeq \gamma_0$。□

定义 19 所要求的交换由同一引理读到 ≃，这样读才是可实现的：两个运算可以留下 ≃ 所等同的值仍算交换。对两个运算，它比对其提升所诱导的效应函数多要求一点——运算还产出结果。

**定义 39.** 当 $a$ 与 $a'$ 的提升作为效应函数（定义 19）在每个参数对处独立，且其中一个的变换不扰乱另一个产出的结果时，运算 $a$ 与 $a'$ 是独立的：
$$\forall x : X_a, g \in \mathfrak{M}(a'_\Sigma), \sigma \in \Sigma.\ \text{pr}_3(a_\Sigma(x)(g(\sigma))) = \text{pr}_3(a_\Sigma(x)(\sigma)) \qquad (35)$$

$a$ 与 $a'$ 互换后相同条件也成立，其中 $\mathfrak{M}(a_\Sigma)$ 记提升 $a_\Sigma(x)$ 在每个参数上的变换幺半群（定义 34 为运算本身写 $\mathfrak{M}(a)$）。当 $\mathcal{A}_k$ 中任意两个运算独立时，键 k 是交换的（commutative），运算与自身独立也算在内。

**跨不同键，条件无条件成立。**

**定理 40.** 不同键处的运算是独立的。

证明. 设 a 在 $\mathcal{A}_k$ 中、a′ 在 $\mathcal{A}_{k'}$ 中且 $k \neq k'$。由定义 24，$\mathfrak{M}(a_\Sigma)$ 的每个生成元形如 $\sigma \mapsto \sigma[k \mapsto u(\sigma(k))]$（u 是 $\mathcal{V}_k$ 上的映射），或是正向映射的提升或是产出逆的提升，a′ 在 k′ 处同理。两个这样的映射交换，它们各自单独读写一个键且两个键不同，引理 18(1) 将交换从生成元扩展到两个幺半群。对第二个条件，$a_\Sigma$ 在 σ 处产出的内容（逆与结果都一样）由 $\sigma(k)$ 决定，而 $\mathfrak{M}(a'_\Sigma)$ 的每个生成元保持 $\sigma(k)$ 不变。□

值是一张可独立增删条目的表的键是交换的，路由或事件监听器的注册是代表性情形：两种顺序的两次注册留下一个对每个测试都同样应答的表，且任一注册都能在另一注册仍在时被撤回。值是有序链的键不交换，因为插入在另一个中间件之前的中间件看到不同的请求，且任一顺序都不能在不扰乱另一个的情况下被撤回。开篇例子的分配器按它的接口所公布的内容划分。若其发出的句柄不被键的任何运算比较，$\simeq_k$ 可以按句柄的重命名关联两个堆——这正是 CompCert 关联程序和其翻译的内存状态的方式 [42]——分配是交换的；若地址是按相等比较的结果，则没有任何可接受的 $\simeq_k$ 使两种分配顺序一致，该键不交换。

组件执行的是一串运算，其中每个运算可能依赖前面的运算产出的内容，这种形状的效应函数正是下面的定理所谈论的。

**定义 41.** 余效应中介的效应函数（coeffect-mediated effect functions）构成最小的集合 $\mathfrak{E}_\Sigma^{\mathcal{A}} \subseteq \mathfrak{E}_\Sigma$，它包含幺元 $\eta_\Sigma$，并在以下条件下封闭：对键 k、运算 $a \in \mathcal{A}_k$、参数 $x : X_a$、以及成员族 $(e_b)_{b \in B}$，
$$\sigma \mapsto \textbf{let } (\delta, s, b) = a_\Sigma(x)(\sigma) \textbf{ in } \textbf{let } (\varepsilon, t) = e_b(\delta) \textbf{ in } (\varepsilon, s \circ t) \qquad (36)$$

又是一个成员。每个阶段执行一个运算并通过结果选择随后的内容，因此参数可以依赖已获得的结果。成员中出现的运算是其阶段所执行的运算，对每个结果选择都是如此。

**定理 42.** 设 $e_1, e_2 \in \mathfrak{E}_\Sigma^{\mathcal{A}}$，且两者运算出现的每个键都是交换的（定义 39）。则 $e_1$ 与 $e_2$ 独立（定义 19）。

证明. 对定义 41 的构造归纳，$\mathfrak{M}(e_i)$ 位于 $e_i$ 中出现的运算的生成元所生成的子幺半群内：幺元生成平凡幺半群，阶段是 $a_\Sigma(x)$ 与成员的 ⋄ 复合，引理 18(2) 适用。
对定义 19 的条款 (1)，由引理 18(1)，只需 $e_1$ 中出现的运算的生成元与 $e_2$ 中出现的运算的生成元交换。两者位于不同键处是定理 40；位于同一键处时该键携带两者的运算，按假设交换。
对条款 (2)，取 $g \in \mathfrak{M}(e_2)$，即 $e_2$ 中出现的运算生成元的复合，并对 $e_1$ 的构造归纳。幺元在每个状态产出 $id_\Sigma$。在阶段，设 $(\delta, s, b) = a_\Sigma(x)(\sigma)$ 且 $(\varepsilon, t) = e_b(\delta)$，则阶段在 σ 处产出 $s \circ t$。对 g 的生成元逐一应用运算的独立性，在 $g(\sigma)$ 处再次得到 s 和 b，因此选择相同的续延 $e_b$，条款 (1) 将其运行的状态放在 $g(\delta)$ 处，归纳假设在彼处再次给出 t。因此该阶段在 $g(\sigma)$ 处产出 $s \circ t$。□

组件与环境之间的每次交互都穿过上下文，且类型族 $\mathcal{V}$ 不受约束，因此系统可以在自己的键处绑定它在组件间共享的每个位置（第 3.3.1 节）。组件的效应函数则是沿余效应投影的提升的余效应中介函数，独立性转移到该提升，其变换只移动投影。第 3.1.3 节留出的假设由此得到满足，随之而来的是整个组件系统的时间可组合性。

该分解将计算的可交换部分与顺序敏感部分分开。可交换部分由效应承载：组件按其任务要求的任何顺序执行它们，推论 21 按系统认为方便的任何顺序逆转它们，两个组件互不约束。顺序敏感部分由余效应承载，因为运算不交换的键正是次序必须从效应外部强加的键，而有两个地方可以强加它。在一个组件内部由累加器强加，它按 LIFO 次序任意逆转效应（定理 16）。跨组件由声明的余效应强加，一个组件提供另一个所声明的，提供先于声明的满足（第 3.2.2 节）。可组合性因此以组件为粒度而非以单个效应为粒度获得，这正是第 4 节工作的尺度。

该定理的两个限制值得一提。把每个共享位置绑定到键是范式的纪律而非构造的性质，因此系统无法实体化为余效应的位置位于第 6.1 节的边界之外，也位于定理之外。键的交换性是该键公布的接口的性质，因此满足它是提供该键的组件的义务，而非消费它的组件的义务。

#### 3.3.3 上下文范式的定位（Situating the Context Paradigm）

编程范式在如何处理副作用上根本不同。两个既有端点定义了这个谱系：

**显式状态穿线（函数式）。** 为保持引用透明性，纯函数式语言将副作用建模为状态上的显式变换。State 单子 $S \to (A, S)$ [23] 将环境穿过每个计算。这一方法带来强大的组合保证：效应在类型中可见，可以等理推理。然而它付出显著的可用性代价：调用链中的每个函数都必须接受并返回状态参数，即使它只是原样穿过状态。随着效应维度的增长（日志、配置、I/O），单子堆叠或效应处理器样板泛滥。

**隐式变异（命令式/OOP）。** 主流命令式语言允许组件修改共享状态并在调用点不做显式声明地访问依赖。在效应侧，代表性例子是 React 的 useEffect 钩子：它在组件的内部 fiber 上注册持久副作用，但效应目标和注册机制都不作为显式参数出现——识别依赖隐藏运行时状态中的调用顺序位置。在余效应侧，Java 的服务定位器模式（如 Spring 的 `ApplicationContext.getBean(...)`）在运行时从进程级注册表检索依赖，要求在每个调用点做空检查和类型转换；依赖关系隐式且散落在代码库各处。更一般地，理解 f() 如何修改或依赖系统需要传递式地阅读其实现。重构变得脆弱，因为移动或移除调用可能静默破坏远处的不可变式。

**上下文范式**结合了函数式方法的可追踪性与命令式方法的可用性。效应与余效应都通过显式上下文参数中介。因此每个运算都可归因于调用它的那个特定上下文，进而归因于该上下文所属的组件。

除了结合两个端点的优势，上下文范式让开发者单独处理每个效应与依赖，并将它们自动组合进系统行为。对可逆转效应，开发者提供每个原子运算的逆，任意复合的逆由复合得出（第 3.1 节），因此组件的拆除（teardown）由加载推导而来，而非与加载并排编写。对响应式余效应，组件只声明它需要的依赖，运行时自动解析并重新接线（第 3.2 节），在提供者被添加、移除或替换时保持一致接线。在两个方向上，原本依赖开发者纪律的正确性成为范式的结构性质。

## 4. 动态组合演算（A Calculus of Dynamic Composition）

第 3 节单独建立了空间与时间可组合性的局部形式。把它们带到整个系统需要将系统分解为组件，每个组件配对一个余效应规约和一个带见证的效应函数，使与共享环境的每次交互都可归因于它们之一。以下小节为该分解提供操作语义，并建立空间与时间可组合性的全局形式。

第 4.1 节与第 4.2 节给出生命周期可以给出规则的最小演算，该演算将每个转移视为原子、即时、无误的；第 4.3 节放弃这三个假设——原子性按转移可能运行的每个方向各放弃一次，承认运行时在转移开始与结束之间插入的控制流形式——并得到真实运行时实现的演算；第 4.4 节建立该演算的元理论，即保持（preservation）、全局时间与空间可组合性、进展（progress）与会合（confluence）。

### 4.1 组件与纤维（Components and Fibers）

本节固定规则所作用的对象：组件；纤维（fiber），带自身生命周期状态的组件实例化；以及注册表（registry），持有状态携带的纤维，余效应上下文从中读出。

**组件。** 组件以三元组给出，其余效应侧被拆分为它从环境读取的内容与它向环境提供的内容。

**定义 43.** 组件是 Γ 上的组件（Γ 同时携带效应与余效应，定义 32）：
$$\mathfrak{C}_Γ \coloneqq \mathfrak{D}_Γ \times \mathfrak{P}_Γ \times \mathfrak{E}_Γ^* \qquad (37)$$

表示三元组 $(d, p, e)$，其中：

- $d : \mathfrak{D}_Γ$ 是定义 25 的余效应规约，声明从环境要求的依赖；
- $p : \mathfrak{P}_Γ \coloneqq \text{Set}(K)$ 是提供（provision），声明组件可能提供的余效应键，且 $p$ 之外的键不是其效应函数所写的；
- $e : \mathfrak{E}_Γ^*$ 是定义 8 的带见证效应函数，定义组件激活时贡献的效应以及撤回它们的逆。

两个声明是一个接口的两个方向，d 是组件从环境读取的内容，p 是组件写入环境的内容，而第 4.2 节不允许一个注册表内的两个纤维提供相交。下标全文取在 Γ 上，余效应上下文是它的投影之一（定义 32），因此定义 25 的 $\mathfrak{D}_\Sigma$ 此处写作 $\mathfrak{D}_Γ$。

提供的不相交是本章与第 3.2.3 节分道之处。定义 28 的隔离让一个键通过领域表解析，因此两个纤维可以在不同领域提供同一键；携带领域的演算会把不相交放宽为领域内的不相交，并按声明纤维的领域解析声明的键。我们不在此引入领域，而是在一个共享领域读取每个键，这使上面的不相交成为正确条件，并使每个键的提供者唯一（定义 45）。它限制的是组件可以被实例化的次数：提供非空内容的组件一次只有一个纤维，因此下面的多次实例化都是不提供任何内容（只消费，或注册其他组件的常见情形）的组件的实例化。

运行系统中实例化的组件随时间被激活和停用，因此它携带生命周期状态，转移将它从一个生命周期状态移动到另一个：激活执行 e，在上下文上累积副作用；停用应用累加器以恢复上下文。其最简单的形式是两个状态模型（图 1），第 4.2 节为它给出规则；第 4.3 节在承认每个控制流特性时细化它。

**图 1 | 基础组件生命周期**：Inactive ⇄ Active（L-Unload / L-Reload）

**纤维。** 一个组件可以被实例化多次，每次实例化携带自身的生命周期状态。我们称这样的实例化为纤维。纤维记录产生它的组件、它在其下实例化的纤维、它提供的余效应，以及它处于生命周期的何处。

**定义 44.** 固定纤维名集合 $\mathfrak{N}$。实例化组件 $(d, p, e) \in \mathfrak{C}_Γ$ 的纤维是元组 $\langle d, p, e, \pi, \sigma, \tau, \theta \rangle$，其中

- $d : \mathfrak{D}_Γ$、$p : \mathfrak{P}_Γ$、$e : \mathfrak{E}_Γ^*$ 是定义 43 的余效应规约、提供和效应函数；
- $\pi : \mathfrak{N} \cup \{\text{root}\}$ 是父（parent）——该纤维在其下实例化的纤维，或根标记 root；
- $\sigma : \Sigma$ 是纤维自身的余效应表（定义 22），激活前为空，由效应运行时写入；
- $\tau : \{\bot, \top\}$ 是退役标志（retirement flag），新纤维为 ⊥，编排器（orchestrator）退役该纤维后为 ⊤；
- $\theta : \Theta_Γ$ 是生命周期状态，在第 4.2 节的两个状态模型中是
$$\Theta_Γ \coloneqq \text{Inactive} \mid \text{Active}(g, \omega) \qquad (38)$$
其中 $g : Γ \to Γ$ 是累加器，$\omega : d \to \mathfrak{N}$ 是已承诺视图（committed view）。

已承诺视图 ω 将纤维声明的每个键发送到转移提交时提供该键的纤维名。第 4.3 节用进行中转移所需的扩展替换 ΘΓ；定义 44 的其余部分对两者只给出一次，只是 e 在第 4.3 节每一层引入的更丰富效应类型处读取。

**注册表。** 状态在名字下持有其纤维，纤维的身份与第 3.2 节的余效应上下文都从该安排读出。

**定义 45.** 记 $\mathfrak{F}_Γ$ 为 Γ 上的纤维集合。状态 $\gamma \in Γ$ 携带注册表
$$F_\gamma : \mathfrak{N} \rightharpoonup \mathfrak{F}_Γ \qquad (39)$$

一个有限部分函数，其父指针构成以 root 为根的树，连同 Γ 中没有纤维的 σ 命名的其余内容。我们记 $\gamma(n)$ 为 $F_\gamma(n)$，并在状态明确时用下标 n 缩写 $\gamma(n)$ 的字段，使 $d_n, p_n, e_n, \pi_n, \sigma_n, \tau_n, \theta_n$ 是定义 44 的字段，$g_n, \omega_n$ 是 θn 携带的累加器与已承诺视图；$\gamma[\theta_n \mapsto \theta']$、$\gamma[n \mapsto \langle\cdots\rangle]$、$\gamma \setminus n$ 分别是在一个字段、一个纤维、一个纤维的存在上不同于 γ 的状态。

纤维的名字赋予它一种在自身变异下存活的标识：下面的每个规则都重写一个纤维的生命周期状态并保持其他不变，所以规则必须说明是哪一个；两个字段引用纤维而非描述它们，即父 π 和已承诺视图 ω。名字是原子：没有规则计算一个、检查其结构或将两个按相等以外的关系关联，引入纤维只是抽取一个未使用过的。这是动态创建局部名字的纪律 [39]，此处用于纤维标识。

每个纤维持有表意味着余效应上下文是被推导而非存储的：它就是活跃纤维共同提供的内容。

$$\sigma_\gamma \coloneqq \bigcup\{\sigma_m \mid m \in \text{dom}(F_\gamma), \theta_m = \text{Active}(-, -)\} \qquad (40)$$

该并集良定义，因为纤维只写它声明的键，$\text{dom}(\sigma_n) \subseteq p_n$，且不同纤维的提供不相交（定义 43），所以每个 $k \in \text{dom}(\sigma_\gamma)$ 恰好位于一个 Active 纤维的表中，其名字我们记作 $\text{provider}_k(\gamma) \in \mathfrak{N}$ 并称为 k 的提供者。因此每个键只有一个可能的提供者，由提供而非状态固定。没有规则直接写 $\sigma_n$：纤维的提供是其自身效应函数执行的集合运算，落在 $\sigma_n$ 中，因此已经是状态 $e_n$ 返回的一部分，它们随累加器离开。效应的余效应部分只被这样记录，因为只有余效应部分是其他纤维要声明的；变更状态在 γ 其他部分的效应与其他效应一样由 g 追踪，但没有纤维能在规约中命名它们，因此它们不贡献排序约束。

第 3.2.2 节的满足关系随后不加修改地适用，$\gamma \models d$ 缩写 $\sigma_\gamma \models d$。当且仅当某个 Active 纤维安装了它时，键才位于 $\text{dom}(\sigma_\gamma)$，其提供是它可能安装的键而非已安装的键，因此 $\gamma \models d$ 已经要求每个声明的键有 Active 提供者。仅对 Active 纤维取并集，正是让纤维能在撤回任何内容之前停止提供的东西，第 4.3.1 节将其变成排序纪律。

### 4.2 基础演算（The Base Calculus）

本节给出图 1 的两状态生命周期的演算，且仅此而已：每个纤维与之比较的目标，以及移动它的五个规则。

**目标视图。** 规则将每个纤维与一个目标比较，即它是否应当运行以及按什么依赖解析。目标不是纤维自身的性质，因为纤维声明的键对整个状态解析，所以它是该状态上的谓词。

**定义 46.** n 在 γ 处的目标视图（target view）将每个声明的键映射到其提供者，所以它是全映射 $d_n \to \mathfrak{N}$，当 n 根本不应当运行时为 ⊥：
$$\text{target}_n(\gamma) \coloneqq \begin{cases} \bot & \text{if } \tau_n \vee \neg(\gamma \models d_n) \\ (k \in d_n) \mapsto \text{provider}_k(\gamma) & \text{otherwise} \end{cases} \qquad (41)$$

当每个纤维到达其目标视图时状态是静息的（quiescent）：
$$\text{quiet}(\gamma) \coloneqq \forall n \in \text{dom}(F_\gamma).\ \begin{cases} \text{target}_n(\gamma) = \bot & \text{if } \theta_n = \text{Inactive} \\ \text{target}_n(\gamma) = \omega_n & \text{if } \theta_n = \text{Active}(-, \omega_n) \end{cases} \qquad (42)$$

目标应答两件事且只应答两件事：通过 $\tau_n$ 应答退役，通过 $\gamma \models d_n$ 与 provider_k 应答余效应解析，每个声明的键在定义 43 的唯一共享领域上从 $\sigma_\gamma$ 读出。

定义 44 的已承诺视图与目标视图同类型，生命周期由比较它们驱动：$\omega_n$ 是 n 激活所针对的解析，$\text{target}_n(\gamma)$ 是它应当运行的解析，下面每个规则在两者一致或不一致时触发。记录提供者而非值使比较可用，因为提供相等值的不同纤维否则会比较相等。组件读取的值通过视图到达，因为提供者的表持有该值，而实现将该映射保存在 fiber.committed 中并保存其哈希在 fiber.target 中（第 5.1.3 节）。

**规则。** 基础演算将每个转移视为原子、即时、无误的：激活一步应用其效应函数，停用一步应用累加器，两者都能成功做到。第 4.3 节放弃全部三个假设。

五个规则生成两个关系。编排规则（orchestration rule，前缀 O-，写作 $\gamma \Rightarrow \delta$）是编排器可以执行的动作；其前提说明动作何时合法，而非何时发生。生命周期规则（lifecycle rule，前缀 L-，写作 $\gamma \longrightarrow \delta$）是系统在其前提成立时主动采取的步骤。步骤序列交错两者，下面的 ⟶ 仅指生命周期步骤。

$$\frac{n \notin \text{dom}(F_\gamma) \quad \pi \in \text{dom}(F_\gamma) \cup \{\text{root}\} \quad (d, p, e) \in \mathfrak{C}_Γ \quad \forall m \in \text{dom}(F_\gamma).\ p \cap p_m = \varnothing}{\gamma \Rightarrow \gamma[n \mapsto \langle d, p, e, \pi, \varnothing, \bot, \text{Inactive}\rangle]} \text{ O-Insert}$$

$$\frac{n \in \text{dom}(F_\gamma)}{\gamma \Rightarrow \gamma[\tau_n \mapsto \top]} \text{ O-Retire}$$

$$\frac{\tau_n = \top \quad \theta_n = \text{Inactive} \quad \forall m.\ \pi_m \neq n}{\gamma \Rightarrow \gamma \setminus n} \text{ O-Remove}$$

插入与退役是仅有的外部输入：编排器要求一个纤维存在或停止存在，绝不直接设置其生命周期状态。O-Retire 对纤维状态是无条件的，因为退役是请求，生命周期规则负责执行它。退役与移除分离也是出于同样原因：已退役但仍 Active 的纤维必须先被停用，更早移除它会丢弃累加器并造成泄漏。前提 $\forall m. \pi_m \neq n$ 通过在父之前移除子来保持树良构。O-Insert 最后一个前提是施加单一来源纪律之处：因为编排器不得允许第二个组件声明一个键，所以一个键只有一个可能的提供者。

$$\frac{\theta_n = \text{Inactive} \quad \omega = \text{target}_n(\gamma) \neq \bot \quad e_n(\gamma) = (\delta, g)}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Active}(g, \omega)]} \text{ L-Reload}$$

$$\frac{\theta_n = \text{Active}(g, \omega) \quad \text{target}_n(\gamma) \neq \omega \quad g(\gamma) = \delta}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Inactive}]} \text{ L-Unload}$$

L-Reload 将已承诺视图与逆一并安装；L-Unload 应用逆并丢弃已承诺视图。两者由同一比较驱动：L-Reload 在纤维不持有已承诺视图且其目标视图不为 ⊥ 时触发，L-Unload 在它所持有的已承诺视图不是其目标视图时触发。这是第 3.2 节的响应式纪律，从应答退役与余效应的目标读出：每当目标视图变化时就发起转移，无论两者中哪一个移动了它。

**实例化。** 组件可以在安装其效应时实例化另一个组件——插件宿主在插件加载自身插件时就是这样做的。目前规则把注册表留给编排规则，这样的实例化无处发生。一个原语给了它位置。

**定义 47.** 应用 $e_n$（或第 4.3.2 节适用时的某次迭代）可以注册组件 $(d, p, e) \in \mathfrak{C}_Γ$。它代替状态映射进行该组件以 $\pi = n$ 的 O-Insert，并产出该纤维的 O-Retire 作为其逆。规则抽取名字（服从 O-Insert 的新鲜性前提）并将其交给效应函数。

逆退役而非移除，原因是逆必须在任何到达的地方适用。O-Remove 携带前提，因此由它构建的逆可能失败：子仍 Active 的父无法运行其累加器，且没有规则会移动子，因为定义 46 不读纤维树。O-Retire 的唯一前提是 $n \in \text{dom}(F_\gamma)$。它留下的条目在注册被取的状态是退役、Inactive(⊥)、持空表的，即引理 57 的残迹条目（vestigial entry）：它与纤维的缺席仅在控制字段上有差异，没有规则能区分两者。

退役子设置 τ，从而将其目标视图变为 ⊥，此后普通规则将它带回到 Inactive。父不必等待，O-Retire 无条件，所以 L-Unload 无论子是否离开都适用于父。孙辈一次被到达一层，子的累加器退役子所注册的内容。定理 66 连同第 4.3.1 节沿余效应强加的级联一起覆盖该级联。

**约束（Confinement）。** 在唯一例外到手之后，效应函数被要求的纪律就可以给出。它限制应用所写的内容，使应用它的规则能交代其他每个变化；限制应用所读的内容，使纤维只看到它声明的余效应和注册表的其余部分。限制写的内容使第 4.4 节能把表 1 读作对它们的完整清点。

**定义 48.** 当对每个满足 $n \in \text{dom}(F_\gamma)$ 的 $\gamma \in Γ$、记 $\delta = f(\gamma)$ 时满足以下条件，映射 $f : Γ \to Γ$ 约束到 n（confined）：
1. （写。）$\text{dom}(F_\delta) = \text{dom}(F_\gamma)$，对每个满足 $m \neq n$ 的 $m \in \text{dom}(F_\gamma)$ 有 $\delta(m) = \gamma(m)$，且 $\delta(n)$ 与 $\gamma(n)$ 仅在 σ 上不同；
2. （读。）在 $\sigma_n$、对每个 $m \in \text{dom}(F_\gamma)$ 的限制 $\sigma_m|_{d_n}$、以及没有纤维表命名的状态部分上一致的两个状态，被 f 带到在这三样上一致的状态。

当它的每次应用（以及第 4.3.2 节适用时它的每次迭代）或者注册一个组件（定义 47），或者其状态映射 $\text{pr}_1 \circ e$ 与其产出的逆都约束到 n 时，效应函数 e 约束到 n。每个纤维的效应函数都被要求约束到该纤维。

注册写 O-Insert 所写的条目，在它抽取的唯一名字处，别无其他；它作为逆产出的 O-Retire 写该名字的 τ，别无其他。两种应用因此都不写已存在纤维的控制字段（除那一个 τ），且什么都不读。

条款 (2) 是组件为何能读取它声明的值：那些值位于其提供者的表中，所以不读 $\sigma_n$ 以外任何表的效应函数将无法使用自己的余效应。它不能读的是 $d_n$ 之外的任何表或任何控制字段，这使组件不能对未声明的纤维的生命周期状态分支。

规则是非确定性的：多个纤维可能持有不同于其目标视图的已承诺视图，关系不承诺它们之间的顺序。它们也只是响应式的，没有规则提到调度器；步骤是任意规则应用序列，因此对所有这样的序列证明的定理对运行时可采用的每个调度策略都成立。

### 4.3 进行中的转移（Transitions in Progress）

本节在四个设定下扩展基础演算。第一个提供第 3.2 节要求而第 4.2 节无法表达的东西：跨越一个其依赖方可占据的区间的停用；另外三个放弃转移是原子、即时、无误的理想化，而真实运行时中的转移三者都不是。被放弃的是整个转移是一步，而非一步是一次规则应用；而四个设定共享一个结构后果，此处一并处理：不是一步的转移需要一个状态供其在途期间占据，每个它可能运行的方向各一个。

**定义 49.** 本节的生命周期状态将 ΘΓ 替换为
$$\Theta_Γ \coloneqq \text{Inactive}(\zeta) \mid \text{Reloading}(i, g, \omega) \mid \text{Active}(g, \omega) \mid \text{Unloading}(g, \omega, \zeta) \qquad (43)$$

其中 $i : \mathfrak{E}_Γ^{\text{iter}*}$ 是剩余的效应迭代器（定义 51），$g : Γ \to Γ$ 是迄今构建的累加器，$\omega : d \to \mathfrak{N}$ 是已承诺视图，$\zeta : \{\bot\} \cup \Xi$ 是结果，由 Unloading 作为其停用所指向的结果携带、由 Inactive 作为所到达的结果携带，或者为 ⊥，或者是第 4.3.4 节提供的错误集 Ξ 中的错误。

当纤维处于三个携带累加器与已承诺视图的状态之一时它是已安装的（installed），当它携带错误结果时是失败的（failed）：
$$\text{installed}_n(\gamma) \coloneqq \theta_n \neq \text{Inactive}(-), \quad \text{failed}_n(\gamma) \coloneqq \exists \xi \in \Xi.\ \theta_n = \text{Inactive}(\xi) \qquad (44)$$

已安装的纤维 n 在 $\omega_n(k) = m$ 时将 k 解析到 m。定义 46 的静息在更宽的状态空间上读作

$$\text{quiet}(\gamma) \coloneqq \forall n \in \text{dom}(F_\gamma).\ \begin{cases} \zeta \neq \bot \vee \text{target}_n(\gamma) = \bot & \text{if } \theta_n = \text{Inactive}(\zeta) \\ \text{target}_n(\gamma) = \omega_n & \text{if } \theta_n = \text{Active}(-, \omega_n) \\ \bot & \text{otherwise} \end{cases} \qquad (45)$$

第 4.1 节的定义按此状态空间平移，有两个读法需要固定。首先，第 4.2 节的 Inactive 在 O-Insert 的结论中读作 Inactive(⊥)，在 O-Remove 的前提中读作 Inactive(−)。其次，$\sigma_\gamma$ 仍然只对 Active 纤维的表取并集，所以两个方向的转移在途中的纤维通过其持有的 ω 读其余效应，且不提供自己的任何内容；因此其转移已写下的键还不是依赖方可据以激活的键。在两状态演算中该区别是空的，那里的每个已安装纤维都是 Active。

图 2 画出这些状态形成的生命周期，以下四个小节提供其边上的规则。

**图 2 | 带进行中转移的生命周期**：Inactive ⇄ Reloading ⇄ Active ⇄ Unloading ⇄ Inactive（L-Begin、L-Iter、L-Finish、L-Divert、L-Raise、L-Leave、L-Unload）

#### 4.3.1 撤回（Withdrawal）

第 3.2 节要求依赖方在其依赖之后激活，依赖提供方只在依赖方停用后撤回其提供。前半在基础演算中已经成立：激活要求 $\gamma \models d_n$，所以声明 k 的纤维不可能在某个纤维积极提供 k 之前激活。后半是实质性的，它必须交付的不只是状态变化的排序。因为其提供者将消失而被拆除的组件正在运行自己的拆除代码，它可能需要恰恰正在被撤回的余效应；关闭连接池通常意味着把连接交还给提供它们的东西。后半必须交付的是：消费者在其自身停用期间仍能读取 k，且提供者对 k 的撤回只在之后生效。基础演算根本不能交付它：它的 L-Unload 将撤除提供与运行逆放在一起，不给消费者的拆除留下任何区间。

本层将该步骤一分为二，并用以下条件看守后半。

**定义 50.** 当某个其他已安装纤维将某键解析到它时，纤维 n 在 γ 处被依赖（relied）：
$$\text{relied}_n(\gamma) \coloneqq \exists m \in \text{dom}(F_\gamma), k \in d_m.\ m \neq n \wedge \text{installed}_m(\gamma) \wedge \omega_m(k) = n \qquad (46)$$

$$\frac{\theta_n = \text{Active}(g, \omega) \quad \text{target}_n(\gamma) \neq \omega}{\gamma \longrightarrow \gamma[\theta_n \mapsto \text{Unloading}(g, \omega, \bot)]} \text{ L-Leave}$$

$$\frac{\theta_n = \text{Unloading}(g, \omega, \zeta) \quad \neg \text{relied}_n(\gamma) \quad g(\gamma) = \delta}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Inactive}(\zeta)]} \text{ L-Unload}$$

L-Leave 记录停用决定但不执行它，这使纤维停止提供其余效应，同时保持自身已承诺视图和他人视图完好。L-Unload 应用累加器，丢弃已承诺视图，并使纤维以其携带的结果停在 Inactive；在第 4.3.4 节提供另一情形之前，结果都是 ⊥。它是演算中唯一应用累加器的规则。

排序的两个半部因此由形式的不同部分承载：可见性半部由已承诺视图承载，L-Unload 最后丢弃它；排序半部由前提 $\neg \text{relied}_n(\gamma)$ 承载，我们称之为守护（guard），它将 k 的撤回保持到每个把它解析到 n 的消费者都已离开。定理 63 建立两者。

守护按绑定而非按纤维施加：$\text{relied}_n(\gamma)$ 测试某个已承诺视图是否命名 n，所以没有声明 n 的任何键的纤维不构成障碍，在另一领域（第 3.2.3 节）解析了 n 的键的纤维也不构成障碍。在第 4.2 节的单一来源纪律下，按绑定的读法与更粗的测试 $\exists m \neq n, k \in d_m.\ \text{installed}_m(\gamma) \wedge k \in p_n$ 重合，一个键在那里只有一个可能的提供者。

这样的守护通常会死锁。使它不死锁的是 Unloading 以及 $\sigma_\gamma$ 只对 Active 纤维取并集：一旦 L-Leave 标记了 n，它的表离开 $\sigma_\gamma$，所以任何目标视图都不再命名 n，每个向 n 承诺的消费者自己也在离开的路上。定理 66 把这个变成守卫总会释放的主张。

守护沿余效应而非沿纤维树对停用排序：父可以在其子仍在 Unloading 时运行逆，因为 relied 只谈论已承诺视图。父与子因此被比定理 63 对提供者与消费者的排序更弱地排序，而效应在环境状态中相遇的父与子由定义 60 的独立性假设管辖。

#### 4.3.2 迭代（Iteration）

一次激活可能按序执行多个效应，停用必须恢复它们。我们用效应迭代器（effect iterator）建模这样的激活，每次迭代产出修改后的上下文、逆和续延（continuation）：

**定义 51.** 将效应迭代器 $\mathfrak{E}_Γ^{\text{iter}}$ 与带见证效应迭代器 $\mathfrak{E}_Γ^{\text{iter}*}$ 定义为如下递归类型：
$$\mathfrak{E}_Γ^{\text{iter}} \coloneqq \mu \mathfrak{I}.\ Γ \to Γ \times (Γ \to Γ) \times \text{Maybe}(\mathfrak{I})$$
$$\mathfrak{E}_Γ^{\text{iter}*} \coloneqq \mu \mathfrak{I}.\ (e : Γ \to Γ \times (Γ \to Γ) \times \text{Maybe}(\mathfrak{I})) \times ((\gamma : Γ) \to (\textbf{let } (\delta, g, o) = e(\gamma) \textbf{ in } g(\delta) \simeq \gamma)) \qquad (47)$$

其中 $e(\gamma)$ 产出三元组 $(\delta, g, o)$，表示：

- δ 是新上下文；
- g 是当前效应的逆函数；
- o 指示续延：
  - Nothing 表示迭代终止；
  - Just(i) 提供下一次迭代。

见证按定义 33 的 ≃ 读取，如定义 37 读取 $\mathfrak{E}_Γ^*$ 的那样：当 $i \in \mathfrak{E}_Γ^{\text{iter}}$ 尊重 ≃ 且它产出的每个 g 尊重 ≃ 并满足上述条款时，i 位于 $\mathfrak{E}_Γ^{\text{iter}*}$。三元组按分量比较，Nothing 只与 Nothing 比较、Just(i) 在 $i \simeq i'$ 时与 Just(i′) 比较，迭代器上的 ≃ 是满足这些条款的最大关系。在 Γ 上取 ≃ 为相等即恢复精确读取。

效应迭代器变换 $\text{effect}_Γ^{\text{iter}}$ 通过递归调用将 effectΓ 扩展到迭代器结构上：

**定义 52.** 定义效应迭代器变换 $\text{effect}_Γ^{\text{iter}}$ 为：
$$\text{effect}_Γ^{\text{iter}} : \mathfrak{E}_Γ^{\text{iter}} \to \partial Γ \to \partial^2 Γ$$
$$\text{effect}_Γ^{\text{iter}} = i \mapsto (\gamma, \varphi) \mapsto \textbf{let } (\delta, g, o) = i(\gamma) \textbf{ in } \textbf{let } t = \text{track}_Γ(g, \text{pr}_1 \circ i) \textbf{ in } \textbf{match } o$$
$$\mid \text{Nothing} \Rightarrow ((\delta, \varphi \circ g), t)$$
$$\mid \text{Just}(i') \Rightarrow \textbf{let } (s, r) = \text{effect}_Γ^{\text{iter}}(i')(\delta, \varphi \circ g) \textbf{ in } (s, t \circ r) \qquad (48)$$

在每次迭代，逆 g 按应用顺序复合到 φ 上，所以累加器 $\varphi \circ g_1 \circ \cdots \circ g_k$ 在应用时自然地按 LIFO 顺序恢复效应。因为 $\text{effect}_Γ^{\text{iter}}$ 落在与 effectΓ 相同的 $\partial Γ \to \partial^2 Γ$ 中，迭代器本身就是一个效应，可以在任何使用效应的地方使用。组件的整个激活就是这样一个使用，这正是本节其余部分形式化的内容，而实现承认每个变更点处的迭代器（第 5.1.1 节）。$\text{Maybe}(\mathfrak{E}^{\text{iter}})$ 续延使任意两个连续迭代之间有一个边界可用，边界处上下文是迭代至此所做成的，累加器恢复那些且仅那些。在这个意义上，效应迭代器是实体化的定界续延（reified delimited continuation）——主流语言通过 yield 运算符暴露的结构 [43]——因此模型直接映射到它们已经提供的生成器上。

在演算中，定义 44 的 $e_n$ 从此在 $\mathfrak{E}_Γ^{\text{iter}*}$ 处读取，用迭代器替换原子效应函数将基础 L-Reload 拆成轨迹经过的已开始状态，并给纤维第二个离开该状态的出口。

$$\frac{\theta_n = \text{Inactive}(\bot) \quad \omega = \text{target}_n(\gamma) \neq \bot}{\gamma \longrightarrow \gamma[\theta_n \mapsto \text{Reloading}(e_n, id_Γ, \omega)]} \text{ L-Begin}$$

$$\frac{\theta_n = \text{Reloading}(i, g, \omega) \quad \text{target}_n(\gamma) \neq \omega \quad (\delta, h) = (\gamma, id_Γ) \vee i(\gamma) = (\delta, h, -)}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Unloading}(g \circ h, \omega, \bot)]} \text{ L-Divert}$$

$$\frac{\theta_n = \text{Reloading}(i, g, \omega) \quad \text{target}_n(\gamma) = \omega \quad i(\gamma) = (\delta, h, \text{Just}(i'))}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Reloading}(i', g \circ h, \omega)]} \text{ L-Iter}$$

$$\frac{\theta_n = \text{Reloading}(i, g, \omega) \quad \text{target}_n(\gamma) = \omega \quad i(\gamma) = (\delta, h, \text{Nothing})}{\gamma \longrightarrow \delta[\theta_n \mapsto \text{Active}(g \circ h, \omega)]} \text{ L-Finish}$$

每次迭代将新产出的逆按 $g \circ h$ 复合到累加器上，遵循定义 52，使累加器按后进先出顺序应用逆。任意两个连续迭代之间，系统可以在目标视图变化时转移该转移，应用迄今累积的逆以恢复上下文。L-Divert 与每个其他停用一样经 Unloading 路由，而不是就地应用累加器，且它在那里遇到的守卫是空洞的：从未 Active 的纤维不提供任何内容、不出现在任何已承诺视图中。它的两个备选中的第一个中止纤维所持有的迭代，这只有迭代边界才使之可能，所以转移可能落下的粒度是迭代器的粒度；第二个让该迭代落地，第 4.3.3 节正是需要它的地方。

普通效应函数（$\mathfrak{E}_Γ$）是第一次迭代已产出 Nothing 的退化情形。这样的转移仍经过 Reloading，L-Divert 仍适用于彼处，但累加器是 idΓ 且没有迭代运行，所以没有恢复任何东西，转移要么安装其全部效应、要么全不安装。

#### 4.3.3 异步（Asynchrony）

此前的各层让环境在一次迭代与下一次之间移动，并假设每次迭代本身瞬时完成，其发起与落地是一步。我们将非即时性抽象地建模：迭代产出 $\text{Future}(A)$ 类型的值，其中 Future 是不透明类型构造子，其定义性质是提交与解析之间外部状态可能改变。

在此模型下一次迭代在一个状态发起、在另一个状态落地，纤维在其飞行期间处于 Reloading。该层增加的是惯性（inertia）：一旦发起，迭代就会落地，且落地不能拒绝。因此飞行期间转变的目标视图不能通过中止迭代来应答，只有 L-Divert 的落地备选可用：迭代落地，纤维随后停用。因此本层不增加规则，也不增加规则所匹配的类型；在 Γ 的粒度上，惯性是它的全部内容，它采取的形式是对宿主可取 L-Divert 哪个备选的限制。

那个备选是基础演算无法表达的。在彼处，目标视图已转变的转移在发现它的同一一步被撤销；此处飞行中的迭代必须先落地，所以纤维需要在其逆运行时的某个地方，唯一健全的地方是持有迭代所产生逆的 Unloading。改为经 Active 路由会让纤维在一步的长度内提供其余效应，并迫使依赖方针对已经离开的组件激活。这是实现中 reload 与 unload 的相互链式。

停用也可以直接链回激活，由复合而非规则完成。L-Unload 不携带目标视图前提，所以无论目标视图在纤维停用期间变成什么，累加器运行、纤维变为 Inactive，L-Begin 可以立即从那里开始新转移。

#### 4.3.4 失败（Failure）

此前的每个规则假设它运行的效应成功，运行时做不到。组件安装的效应达到追踪它们的上下文之外，所到达的东西可能拒绝：已被绑定的端口、不存在的文件、不应答的对端。失败的转移仍必须使纤维的效应被恢复而非搁浅。

设 Ξ 为错误集，精化定义 51 的效应迭代器，使迭代可以代替产出三元组而抛出：

$$\mathfrak{E}_Γ^{\text{fail}} \coloneqq \mu \mathfrak{I}.\ Γ \to \text{Either}(\Xi, Γ \times (Γ \to Γ) \times \text{Maybe}(\mathfrak{I}))$$
$$\mathfrak{E}_Γ^{\text{fail}*} \coloneqq \mu \mathfrak{I}.\ (e : Γ \to \text{Either}(\Xi, Γ \times (Γ \to Γ) \times \text{Maybe}(\mathfrak{I}))) \times ((\gamma : Γ) \to (\textbf{let } \text{Right}(\delta, g, o) = e(\gamma) \textbf{ in } g(\delta) \simeq \gamma)) \qquad (49)$$

见证只约束 Right 情形，在模式不匹配处是空洞的，抛出没有要撤销的东西，此后 Reloading 携带的 i 在 $\mathfrak{E}_Γ^{\text{fail}*}$ 处读取。定义 52 的提升按抛出代替三元组传播而平移，所以抛出迭代器像普通迭代器一样可以在任何使用效应的地方使用。该层增加一条规则，并利用定义 49 的第二个结果，O-Remove 无需扩宽即可接纳它。L-Iter、L-Finish 与 L-Divert 的前提在它们匹配的三元组周围读取 Right。抛出是迭代所做的事，所以规则是离开 Reloading 的出口。

$$\frac{\theta_n = \text{Reloading}(i, g, \omega) \quad i(\gamma) = \text{Left}(\xi)}{\gamma \longrightarrow \gamma[\theta_n \mapsto \text{Unloading}(g, \omega, \xi)]} \text{ L-Raise}$$

L-Raise 先恢复后记录。纤维带着错误作为其结果路由进 Unloading，累积到失败迭代为止的累加器在那里被应用，纤维带着什么都没安装到达 Inactive(ξ)，所处的状态与中止的 L-Divert 会产生的状态仅在纤维携带的结果上有差异。让失败像每个其他停用一样路由，正是使每个结果只能通过 L-Unload 到达，这是定理 59 所依赖的唯一事实。L-Begin 以 Inactive(⊥) 为前提，所以不会从错误结果重新进入生命周期；这是结果的本质，它扣留一个其效应函数已在所对状态显出不健全的纤维，而不是让它在未变化的环境中重试。失败的纤维也不阻塞任何东西：它是 Inactive，所以不携带已承诺视图，不能使 relied 成立。

失败记录在纤维上而非传播给其父，所以转移失败的组件让其兄弟保持运行，这是插件宿主想要的行为，也是结果按纤维而非整个状态的性质的原因。

### 4.4 元理论（Metatheory）

第 4.3 节提供了十条规则：第 4.2 节的三个编排规则；激活的 L-Begin、L-Iter、L-Finish；激活提早结束两种方式的 L-Divert 与 L-Raise；停用的 L-Leave 与 L-Unload。本节从这些规则读取两个维度的可组合性的全局形式——一个纤维的保证无论其他纤维在中间做什么都成立——并添加只有整个系统才能被要求的：它总是到达其目标所要求的配置，且该配置正是静态组装会产生的那个。下面的每个性质都是步骤序列的性质，因此我们对步骤编号并从该编号读取状态的字段。

两个约定把第 3.3.2 节带入本节。下面的每个状态相等都读到定义 33 的观察等价 ≃，如引理 38 读取第 3.1 节的相等；效应函数被要求的见证条件是定义 37 给出的，按定义 51 对迭代器读取，在注册迭代处按下面的 ≈ 读取。

**定义 53.** 用 t 对步骤编号，使 $\gamma^t$ 是前 t 个步骤到达的状态，并记
$$\text{step}^t \coloneqq r(n) \qquad (50)$$

为在 $\gamma^t$ 处采取的步骤：它应用的规则 r（十条之一）及其应用该规则的名字 $n \in \mathfrak{N}$。序列从 $\text{dom}(F^0) = \varnothing$ 的 $\gamma^0$ 开始，所以每个纤维都经 O-Insert 存在，无论是编排器的还是迭代取的（定义 47）。$\gamma^t$ 的字段带指数上标，使 $\theta_n^t, \omega_n^t, \sigma_n^t, g_n^t, i_n^t$ 是 n 在 $\gamma^t$ 的生命周期状态、已承诺视图、表、累加器和剩余迭代器，$F^t$ 与 $\sigma^t$ 是 $\gamma^t$ 本身的注册表与余效应上下文，即定义 45 的 $F_\gamma$ 与 $\sigma_\gamma$ 在彼处的读取。谓词以状态为参数、其余作下标，所以 $\text{installed}_n^t, \text{target}_n^t, \text{relied}_n^t, \text{quiet}^t$ 是定义 46、49、50 的谓词在 $\gamma^t$ 的值。n 的一次插曲（episode）是 indices 的最大区间 $[b, u]$，其间 installed_n 成立。它在 b 处开启，其中 $b > 0$ 且 $\neg \text{installed}_n^{b-1}$，空的 F 使开头没有任何纤维安装；它在 u 处闭合，此时 installed_n^u 且不成立 $\text{installed}_n^{u+1}$，最后的插曲不必闭合。

第 4.3 节的每条规则都以 $\gamma \longrightarrow \delta[\cdots]$ 的形状结尾，其中前提从 γ 计算 δ，在未计算处使之为 γ，括号编辑注册表的命名字段。两个半部被分别命名，都是全 Γ 上的映射。步骤在 $\gamma^t$ 处由作用于 n 的规则采取的状态映射为

$$\Psi^t \coloneqq \begin{cases} \text{pr}_1 \circ i & \text{at L-Iter, L-Finish, and a landing L-Divert} \\ g & \text{at L-Unload} \\ id_Γ & \text{at every other rule} \end{cases} \qquad (51)$$

其中 i 与 g 是 $\theta_n^t$ 携带的迭代器与累加器，编辑 $\text{edit}^t : Γ \to Γ$ 是作为函数的括号，指派前提在 $\gamma^t$ 计算的值给其命名的字段。两者因此由 step^t 连同 $\gamma^t$ 固定，并在每个状态有定义，这使定理 61 与引理 71 可以在 $\gamma^t$ 之外对它们求值。每步分解为

$$\gamma^{t+1} = \text{edit}^t(\Psi^t(\gamma^t)) \qquad (52)$$

例如在 L-Unload 处，$\text{edit}^t$ 是 $[\theta_n \mapsto \text{Inactive}(\zeta)]$，在 O-Remove 处是移除 $\setminus n$，这就是为什么第二个半部是编辑而非指派。字段沿同一条缝划分：表 $\sigma_m$，一旦创建 m 的 O-Insert 将它置空便没有 edit^t 写它；以及控制字段 $\theta_m, \tau_m, \pi_m, d_m, p_m, e_m$ 连同 $\text{dom}(F_\gamma)$，没有 Ψ^t 写它们（定义 47 的原语除外）。当两个状态在除控制字段外的一切上一致时，记 $\gamma \approx \delta$。

关系 ≈ 不是定义 33 的 ≃，两者互不细化，因为每个忘记了另一个必须保留的东西。恢复精确性是关于效应的主张，所以 ≈ 精确比较表与环境状态，只忘记注册表对哪个纤维安装它们的记录。规则读控制字段来决定是否适用，所以 ≃ 必须保留它们，本节将其读作定义 33 与注册表定义域及每个纤维每个控制字段的一致之合取：

$$\gamma \simeq \delta \coloneqq \sigma_\gamma \simeq \sigma_\delta \wedge \text{dom}(F_\gamma) = \text{dom}(F_\delta) \wedge \forall n, c \in \{\theta, \tau, \pi, d, p, e\}.\ c(\gamma(n)) \simeq c(\delta(n)) \qquad (53)$$

函数类型字段（如 $e_n$ 和 $\theta_n$ 内的 g）按定义 36 比较映射，迭代器按定义 51 比较，任何其他类型字段按相等比较。下面的结果对两个关系都成立，每个对应状态的一个半部，引理 55 对所有十条规则一次建立 ≃ 半部。

表 1 将第 4.3 节的十条规则读作这样的写。累加器、已承诺视图和剩余迭代器是 $\theta_n$ 的成分，所以第三列也记录对它们的写，第四列的 h 命名迭代产出的逆，L-Divert 中止该迭代处为 idΓ。由来自迭代器的 Ψ^t 注册纤维之处（定义 47），该注册在它抽取的名字处携带 O-Insert 行的写，其累加器退役某者的 L-Unload 携带 O-Retire 行的写。下面的每个情形分析都是表内查找，五个查找频繁出现而值得命名。

**表 1 | 规则作为其对作用纤维 n 的写**，其中 step^t 是在 n 处应用该规则。

| 规则 | $\theta_n^t$ | $\theta_n^{t+1}$ | Ψ^t | 编辑的控制字段 |
|---|---|---|---|---|
| O-Insert | 未定义 | Inactive(⊥) | idΓ | dom(Fγ) |
| O-Retire | 不受约束 | 不变 | idΓ | τn |
| O-Remove | Inactive(−) | 未定义 | idΓ | dom(Fγ) |
| L-Begin | Inactive(⊥) | Reloading(en, idΓ, ω) | idΓ | θn |
| L-Iter | Reloading(i, g, ω) | Reloading(i′, g∘h, ω) | pr1∘i | θn |
| L-Finish | Reloading(i, g, ω) | Active(g∘h, ω) | pr1∘i | θn |
| L-Divert | Reloading(i, g, ω) | Unloading(g∘h, ω, ⊥) | idΓ 或 pr1∘i | θn |
| L-Raise | Reloading(i, g, ω) | Unloading(g, ω, ξ) | idΓ | θn |
| L-Leave | Active(g, ω) | Unloading(g, ω, ⊥) | idΓ | θn |
| L-Unload | Unloading(g, ω, ζ) | Inactive(ζ) | g | θn |

**引理 54.** 读表 1 与定义 48，对在 $\gamma^t$ 存在的每步 t 和所有纤维 m, n：
1. 只有 step^t 作用于 m 时才成立 $\sigma_m^{t+1} \neq \sigma_m^t$，写位于 Ψ^t 内；
2. ωn 只出现在 step^t = L-Begin(n) 处、只消失于 step^t = L-Unload(n) 处，因此 $\omega_n^t$ 在 n 的插曲内为常数；
3. 只有 step^t = L-Unload(n) 处才有 Ψ^t = $g_n^t$，且没有其他步骤对状态应用 $g_n$；
4. $\neg \text{installed}_n^t \wedge \text{installed}_n^{t+1} \Rightarrow \text{step}^t = \text{L-Begin}(n)$，且 $\text{installed}_n^t \wedge \neg \text{installed}_n^{t+1} \Rightarrow \text{step}^t = \text{L-Unload}(n)$；
5. πn、dn、pn、en 随 n 的条目存在且此后永不被写，τn 是单调的，只被 O-Retire 写为 ⊤。

证明. 设 step^t 在 n 处应用 r。由定义 53 它分解为 $\text{edit}^t \circ \Psi^t$，其中 edit^t 写表 1 第五列命名的字段且只写那些，Ψ^t 是 idΓ、n 的一次迭代的应用或累加器 $g_n^t$（迭代产出逆的复合）。三者按定义 48 约束到 n，所以 Ψ^t 不写 $\gamma^t$ 已存在纤维的任何字段但 σn，连同注册添加的条目和其逆写的 τ。两个半部因此划分写，每个条款就是该划分在某字段处的读取。第二、三列的一个读法使用两次：Inactive 是唯一不携带已承诺视图的生命周期状态，L-Begin 是唯一引出它的规则，L-Unload 是唯一引入它的规则，而每个其他行将其前提的 ω 原样带入结论。
(1) edit^t 不写表，第五列不命名任何表，且对已存在的 $m \neq n$，Ψ^t 不写 $\sigma_m$。所以 σm 只能在 m = n 时且只在 Ψ^t 内移动。
(2) ωn 是 θn 的成分，只有作用于该纤维的 edit^t 写它，所以按上面的读法，ωn 出现在 n 的 L-Begin 处、消失于 n 的 L-Unload 处。n 的插曲是 installed_n 成立的区间，因此是 ωn 有定义的整个区间，两个规则都不落在其内部。
(3) 第四列，累加器只在 L-Unload 出现：其他规则取正向映射 $\text{pr}_1 \circ i$ 或 idΓ，且没有 edit^t 对状态应用映射。
(4) installed_n 是 $\theta_n \neq \text{Inactive}(-)$，按上面的读法，L-Begin 与 L-Unload 是前提与结论在 $\theta_n$ 是否为 Inactive 上不同的仅有的规则。作用于某 $m \neq n$ 的步骤不写 $\theta_n$，注册添加的条目在不曾在 $\gamma^t$ 的名字处。
(5) 第五列没有行命名 π、d、p 或 e；它们随 O-Insert 添加的条目存在，其结论写它们，注册取的 O-Insert 也写。只有 O-Retire 写 τ，写为 ⊤，无论编排器取还是作为注册的逆（定义 47）；O-Insert 在不曾存在的名字处设置 τ = ⊥，所以没有步骤将 τ 返回 ⊥。□

三个进一步的查找说明规则不能看到什么。第一个是它们只通过上面的观察读状态，所以整个演算下降到 Γ/≃。

**引理 55.（≃ 不变性。）** 设按上面的读法 $\gamma \simeq \gamma'$。则规则在 γ 处作用于 n 适用当且仅当它在 γ′ 处作用于 n 适用，且两次应用到达的状态再次由 ≃ 相关。

证明. 第 4.3 节的每个前提属于四种之一，每种读取关系保留的成分。匹配 θn 或 τn 于模式的前提，以及 O-Remove 的 $\forall m. \pi_m \neq n$ 读控制字段。前提 $(d, p, e) \in \mathfrak{C}_Γ$ 与 $\forall m. p \cap p_m = \varnothing$ 读 d、p、e。提及 target_n 或 relied_n 的前提读 τn、θm 内的已承诺视图与 $\text{dom}(\sigma_\gamma)$，定义 45 从 θm 与 $\text{dom}(\sigma_m)$ 计算它，定义 33 只在定义域一致时关联两个余效应上下文。其余前提读 $\text{dom}(F_\gamma)$。没有任何前提读 $\sigma_\gamma(k)$ 的值（除到 ≃ 为止），所以没有前提分离两个 ≃ 相关的状态。
对结论，由定义 53 有 $\gamma^{t+1} = \text{edit}^t(\Psi^t(\gamma^t))$。edit^t 指派的值是它所匹配前提的成分，由上一段与定义 51 在两个状态处相关，定义 51 关联迭代器在 ≃ 相关状态产出的三元组。而 Ψ^t 尊重 ≃：它是 idΓ，或 $e_n$ 的一次迭代（定义 51 要求尊重 ≃），或 θn 内的累加器（逆的复合，每个按同一定义尊重 ≃）。□

状态携带的名字被其中两个观察读取，$\text{dom}(F_\gamma)$ 与控制字段的索引，而抽取名字的规则抽取任何未用过的名字（定义 47）。因此把下面的结果读到 ≃ 也要求读到重命名，这正是第 4.1 节纪律的兑现。

**引理 56.（等变性。）** 设 $\chi : \mathfrak{N} \to \mathfrak{N}$ 是双射，$\chi \cdot \gamma$ 是携带注册表 $F_\gamma \circ \chi^{-1}$、每个出现在 $\pi_m$ 或 $\omega_m$ 中的名字被替换为其像的状态。则 χ·γ 是状态，γ 良构处它也良构，且 step^t = r(n) 携带 $\gamma^t$ 到 $\gamma^{t+1}$ 当且仅当 $r(\chi(n))$ 携带 $\chi \cdot \gamma^t$ 到 $\chi \cdot \gamma^{t+1}$。

证明. 前提只通过将名字与其他名字比较来读它，无论是直接的（如 O-Insert 的新鲜性 $n \notin \text{dom}(F_\gamma)$ 与 O-Remove 的 $\forall m. \pi_m \neq n$）还是通过名字的表（target_n 与 relied_n 读取 $\pi_m$ 与 $\omega_m$）。双射保持每个这样的比较。规则只写π（O-Insert 设置）与 ω（L-Begin 设置），两者取自其前提所读的内容，所以写与 χ 交换；效应函数不写任何名字，只通过定义 47 的原语抽一个，定义 48 将其约束到该原语添加的条目。良构性（定义 58）是比较名字与名字的四个条件。□

因此一个序列与其重命名按相同顺序取相同规则并到达仅差 χ 的状态。除注册抽取的名字外一致的两个序列因此被等同，下面的结果读到识别它们的重命名。

第二个查找是：剥离一切只留名字的条目对规则不可见，这使定义 47 能在其恢复的状态没有该纤维处退役一个纤维，引理 72 移除删除的插曲所作的注册。

**引理 57.（残迹条目。）** 当 $\tau_n = \top$、$\theta_n = \text{Inactive}(\bot)$、$\sigma_n = \varnothing$ 且没有 m 有 $\pi_m = n$ 时，称 n 在 γ 处是残迹的（vestigial）；残迹条目满足 $\gamma \approx \gamma \setminus n$。若 n 在 γ 处残迹，则对每条规则和每个 $m \neq n$：
1. 在 γ 处作用于 m 的规则在 $\gamma \setminus n$ 处作用于 m 也适用，两次到达的状态仅在 n 处的条目上不同，且该条目保持残迹；
2. 反之，在 $\gamma \setminus n$ 处作用于 m 的规则在 γ 处适用，除非它是抽取名字 n 或主张 $p_n$ 的键的 O-Insert。

证明. 残迹的 n 不贡献作用于 $m \neq n$ 的规则前提所读的任何观察。它不是 Active，所以 $\sigma_n$ 不进入任何 $\sigma_\gamma$、n 不是任何键的提供者，$\gamma \models d_m$ 与 target_m 不动；installed_n 失败，所以 n 不给 relied_m 贡献析取项；没有 $\pi_{m'}$ 命名 n，所以某 m 的 O-Remove 的前提 $\forall m'. \pi_{m'} \neq m$ 不动；而 $\theta_n, \tau_n, \pi_n$ 由作用于 n 的规则单独读取。条款 (2) 除外的两个前提是被移除所放宽的：缺席的名字是新鲜的，缺席的提供满足每个其他。由引理 54，作用于 $m \neq n$ 的规则不写 n 的字段，所以条目存活，且步骤的状态映射按定义 48 约束到 m，所以它使 σn 保持空。□

简化生命周期状态连同匹配它们的规则产生子演算，并非每个结果都在简化下存活。舍弃第 4.3.1 节是要紧的情形，这是从元理论侧面读第 4.3 节开头所作的划分：其守护建立定义 58 的条款 (3)、(4)，定理 63 依赖守护创造的区间，所以三个离开守护都不成立。其他三个小节所加的内容可以简化掉而不扰动下面的结果，它们各自只是向定义 49 固定的状态空间加规则。

#### 4.4.1 保持（Preservation）

定义 45 固定注册表的形状，在下面的结果可以为其添加内容之前必须对照核查规则。本小节识别规则保持的不变式，第一条条款是那个形状，其余是那些结果所假设的。

**定义 58.** 当对所有 $m, n \in \text{dom}(F_\gamma)$ 和所有 $k \in K$ 满足以下条件时，注册表 $F_\gamma$ 是良构的：
1. $\pi_n \in \text{dom}(F_\gamma) \cup \{\text{root}\}$；
2. $m \neq n \Rightarrow p_m \cap p_n = \varnothing$；
3. $\text{installed}_n(\gamma) \Rightarrow \omega_n$ 在 $d_n$ 上全、取值于 $\text{dom}(F_\gamma)$；
4. $\text{installed}_n(\gamma) \wedge k \in d_n \wedge \omega_n(k) = m \Rightarrow \text{installed}_m(\gamma)$。

条款 (1) 是一次读一条边的定义 45 的树，使父指针落在注册表中。该定义还要求的无环性不需要条款，因为指针命名的纤维先于命名它的纤维注册。

**定理 59.（保持。）** 若 $F^t$ 良构，则无论 step^t 应用哪条规则，$F^{t+1}$ 都良构。每个条款从 $\gamma^t$ 处的全部四条在 $\gamma^{t+1}$ 建立。

证明. 设 step^t 作用于 n。
(1) 由表 1，只有 O-Insert 与 O-Remove 写 π 或 $\text{dom}(F_\gamma)$。O-Insert 以 $\pi_n \in \text{dom}(F^t) \cup \{\text{root}\}$ 为前提，这对其添加的纤维就是该条款，并保持每个其他 π 不变，同时扩大 $\text{dom}(F_\gamma)$。O-Remove 有 $\forall m. \pi_m \neq n$，所以存活的 $\pi_m$ 都不命名它移除的纤维。
(2) O-Insert 的最后一个前提是 $\forall m. p_n \cap p_m = \varnothing$，这对其添加的纤维就是该条款，且由表 1 没有其他规则写 p 或扩大 $\text{dom}(F_\gamma)$。两个后果用于下文：由定义 43，$\text{dom}(\sigma_m) \subseteq p_m$，所以不同表不相交，σγ 是函数；且 $k \in p_m \cap p_{m'}$ 迫使 $m = m'$，所以 k 至多有一个可能提供者。
(3) 由引理 54(2)，唯一写 ωn 的规则是 L-Begin，其前提 $\omega = \text{target}_n^t \neq \bot$ 使它全于 $d_n$ 且取值于 $\text{dom}(F^t)$，目标命名提供者。由表 1，唯一缩小 $\text{dom}(F_\gamma)$ 的规则是 O-Remove，其前提 $\theta_n^t = \text{Inactive}(-)$ 给出 $\neg \text{installed}_n^t$，由 $\gamma^t$ 处的条款 (4) 没有 m 在 installed_m^t 时有 $\omega_m^t(k) = n$、$k \in d_m$；n 自身不携带 ω。
(4) 由引理 54(2)、(4)，条款只能在 $\gamma^{t+1}$ 的以下情况失败：某个已安装者倒下、某个 ω 被写、或某 ω 命名的纤维离开 $\text{dom}(F_\gamma)$。最后一种是 O-Remove，其移除的纤维未安装，因此由 $\gamma^t$ 处的条款 (4) 不被任何已安装 m 的 $\omega_m^t$ 命名。第一种是 n 的 L-Unload，其前提 $\neg \text{relied}_n^t$ 读
$$\forall m \neq n, k \in d_m.\ \text{installed}_m^t \Rightarrow \omega_m^t(k) \neq n$$

且它不写 $m \neq n$ 的 $\omega_m$、留下 $\neg \text{installed}_n^{t+1}$，所以条款对 n 也成立。第二种是 n 的 L-Begin，写 target_n^t，其值是 $d_n$ 的键的提供者、故在 $\gamma^t$ 处是 Active；该步骤不改变其他纤维的 θ，所以它们到 $\gamma^{t+1}$ 仍是已安装的。□

L-Unload 上的守护承载条款 (3)、(4)。O-Remove 的前提 $\forall m. \pi_m \neq n$ 只谈父指针；使已承诺视图不命名被移除纤维的是守护，提前若干步出于不同理由施加。因为失败经 Unloading 路由，错误结果的论证不必重复。基础演算不具备的两件事随之而来。O-Remove 释放的名字可以被 O-Insert 重新发放，因为没有过期的已承诺视图能命名它；纤维一旦 Inactive 即可被移除，无需单独检查没有人依赖它。

#### 4.4.2 时间可组合性（Temporal Composability）

局部时间可组合性用一个累加器恢复一个效应序列（第 3.1.3 节）。注册表每纤维持有一个累加器，纤维交错：在 n 将逆复合到 $g_n$ 的时刻与 $g_n$ 运行的时刻之间，其他纤维已移动状态。$g_n$ 在彼处是否仍撤销它被构建来撤销的东西，是保证的全局形式所断言的，其成立条件是中国步骤与 $g_n$ 交换。

**定义 60.** 对 $i \in \mathfrak{E}_Γ^{\text{iter}*}$，令 $\text{reach}(i)$ 为包含 i 并在续延下封闭的最小迭代器集，通过取其生成元为 reach(i) 中每个迭代器的正向映射与产出逆来按迭代器读定义 17 的变换幺半群：
$$\text{reach}(i) \coloneqq \bigcap\{S \mid i \in S \wedge \forall i' \in S, \gamma \in Γ.\ i'(\gamma) = (-, -, \text{Just}(i'')) \Rightarrow i'' \in S\} \qquad (54)$$
$$\mathfrak{M}(i) \coloneqq \langle\{\text{pr}_1 \circ i' \mid i' \in \text{reach}(i)\} \cup \{\text{pr}_2(i'(\gamma)) \mid i' \in \text{reach}(i), \gamma \in Γ\}\rangle$$

在第 4.3.4 节适用处读三元组周围的 Right，并记 $\text{len}(i)$ 为续延排序的链 $C \subseteq \text{reach}(i)$ 中 |C| 的上确界。当两个迭代器 i、j 在定义 19 的意义上独立（读这些变换幺半群，迭代的产出是其逆连同续延）时它们是独立的：
$$\forall f \in \mathfrak{M}(i), g \in \mathfrak{M}(j).\ f \circ g \simeq g \circ f$$
$$\forall i' \in \text{reach}(i), g \in \mathfrak{M}(j), \gamma \in Γ.\ \text{pr}_{2,3}(i'(\text{g}(\gamma))) \simeq \text{pr}_{2,3}(i'(\gamma)) \qquad (55)$$

在 j 中对称，按定义 36 读映射上的 ≃，按定义 51 读续延上的 ≃，按定义 47 以它命名的组件的一致读注册迭代上的 ≃。族 $(i_l)_{l \in L}$ 的迭代器两两独立，当每个 $l \neq l'$ 的 $i_l$ 与 $i_{l'}$ 独立；步骤序列两两独立，当 $(e_n)_{n \in N}$ 如此，其中 N 是序列曾持有的名字集，编排器插入的每个纤维和迭代注册的每个纤维各一个。

这个意义上的独立性正是迹理论（trace theory）取为原语的东西：交换动作生成序列上的等价，在其下交换两个相邻独立动作保持端点 [44]，引理 71 就是这些规则的该种交换。族而非集是让一个组件的两个名字保持在作用域中：该条件于是要求该组件的效应函数与自身独立，即要求 $\mathfrak{M}(i)$ 交换。第一个条件是定理 61 所用的，第二个是定理 73 额外需要的：交换两个纤维的步骤在另一纤维移动的状态处求值迭代器，仅交换映射本身并不说明迭代器在彼处产出相同逆与相同续延。检查第一个条件最多需要迭代本身，因为引理 18(1) 将交换从生成元带到它们生成的幺半群。

在这些条件下，定理 7 的单累加器不变式在交错下存活，形式给了时间可组合性其内容：运行逆撤回纤维的贡献且只撤回它。

**定理 61.（恢复精确性。）** 设步骤序列两两独立，n 的插曲在 b 打开，$u \geq b$ 位于其中，且 $t_1 < \cdots < t_l$ 是 $[b, u)$ 中作用纤维不是 n 的指标。则
$$g_n^u(\gamma^u) \approx (\Psi^{t_l} \circ \cdots \circ \Psi^{t_1})(\gamma^b) \qquad (56)$$

即，在 $\gamma^u$ 处应用 n 的累加器，到控制字段为止，给出那些相同步骤会从 $\gamma^b$ 产生的状态。把右边读作 n 从未开始的假设下本会到达的状态，这还要求没有纤维 n 的注册在 $[b, u)$ 内取步，因为纤维 n 的注册是不会在那里取步的。

证明. 对 u 归纳，u + 1 在插曲中的指标。在 u = b，第 b−1 步是 L-Begin（插曲按定义 53 打开），所以 $g_n^b = id_Γ$（表 1），指标集为空，主张是 $\gamma^b \approx \gamma^b$。每步使用两个事实。由于 edit^t 只写控制字段，
$$\gamma^{t+1} \approx \Psi^t(\gamma^t)$$

且由于 $\mathfrak{M}(e_n)$ 中的每个映射不写控制字段（除注册添加的以外）（定义 48 连同定义 47），每个这样的映射把 ≈ 相等的状态带到 ≈ 相等的状态。设 step^u 作用于 n。因为插曲在 u 与 u+1 打开，引理 54(4) 排除 n 的 L-Begin 与 L-Unload，O-Insert 与 O-Remove 读 installed_n^u 所否认的 θn，留两种情形。规则是 L-Iter、L-Finish 或落地 L-Divert 处，表 1 给出 $\Psi^u = \text{pr}_1 \circ i_n^u$ 且 $g_n^{u+1} = g_n^u \circ h$，h 为该迭代产出的逆。定义 51 的见证条件读 $h(\Psi^u(\gamma^u)) = \gamma^u$，到迭代注册纤维（引理 57）处的 ≈ 为止，且 $g_n^u$ 按上面的等式携带 ≈，所以
$$g_n^{u+1}(\gamma^{u+1}) \approx (g_n^u \circ h)(\Psi^u(\gamma^u)) = g_n^u(\gamma^u)$$

规则是 L-Leave、L-Raise、中止 L-Divert 或 n 的 O-Retire 处，表 1 给出 $\Psi^u = id_Γ$ 且 $g_n^{u+1} = g_n^u$，所以同一等式在 h = idΓ 下成立。任一种方式，归纳假设以指标集不变平移，这是定理 7 的计算一步一步进行。
设 step^u 作用于 $m \neq n$。则按表 1 有 $g_n^{u+1} = g_n^u$，且 $\Psi^u \in \mathfrak{M}(e_m)$（规则是编排规则处为 idΓ），所以独立性给出
$$g_n^u(\gamma^{u+1}) \approx g_n^u(\Psi^u(\gamma^u)) = \Psi^u(g_n^u(\gamma^u))$$

这是追加 Ψ^u 的归纳假设。□

**推论 62.（终局恢复。）** 设步骤序列两两独立，n 的插曲在 b 打开、在 u 闭合，无论 n 到达什么结果。则，$t_1 < \cdots < t_l$ 如定理 61：
$$\gamma^{u+1} \approx (\Psi^{t_l} \circ \cdots \circ \Psi^{t_1})(\gamma^b) \qquad (57)$$

被 O-Remove 移除的纤维也不留下任何东西，其前提只接纳 $\theta_n = \text{Inactive}(-)$。

证明. 由引理 54(4)，step^u 是 n 的 L-Unload，其 Ψ^u 按引理 54(3) 是 $g_n^u$，所以 $\gamma^{u+1} \approx g_n^u(\gamma^u)$ 且定理 61 适用。陈述与 ≈ 都不提及 ζ，按表 1 它是 L-Divert 与 L-Raise 导致不同的唯一字段。□

上面的结果假设组件两两独立，第 3.3.2 节就是放行它的地方：在组件执行的每个效应都是某键的运算且每个键都交换之处，任何两个由这些运算构建的效应函数都独立（定理 42）。将该结果从效应函数带到迭代器不需要新东西，余效应中介的效应函数（定义 41）已经按阶段产出的结果选择每个阶段之后的内容，这正是迭代器在其续延中携带的。第 3.2 节的余效应运算是不需要任何假设的情形：组件在彼处贡献的映射是 set 运算与相应限制的复合，两个这样的映射在触及不相交键时交换，定义 58 的条款 (2) 使不同纤维的提供不相交。

#### 4.4.3 空间可组合性（Spatial Composability）

局部空间可组合性把组件约束到它自己的规约：只在依赖被提供处激活它，并把每个上下文变化对照它们分类（第 3.2.2 节）。全局形式添加了对其他纤维量化的内容：提供方只在每个把它解析到的依赖方都已停用后才撤除绑定，且转移安装其效应所对照的解析在其下不发生变动。余效应一侧的两个性质交付这两样东西，它们一起证明，是同一个不变式的两半，即引理 54(2) 确立的 ωn 在插曲内的固定性。排序定理（ordering theorem）是固定性在插曲中 n 处于 Active 继而 Unloading 的部分所买到的东西，相干定理（coherence theorem）是它在 n 安装其效应的部分所买到的东西。

**定理 63.（排序。）** 纤维只在其依赖被提供处开始转移：
$$\text{step}^t = \text{L-Begin}(m) \Rightarrow \gamma^t \models d_m \qquad (58)$$

进一步设 $[b', u']$ 是 m 的一次插曲，$\omega_m^{b'}(k) = n$ 对某个 $m \neq n$ 与 $k \in d_m$ 成立，设 $[b, u]$ 是包含 b′ 的 n 的插曲，t 在 $[b', u']$ 上取值。则
1. $\omega_m^t(k) = n$；
2. $b < b'$，且若 $[b, u]$ 闭合则 $u' < u$；
3. $k \in \text{dom}(\sigma_n^t)$ 且 $\sigma_n^t(k) = \sigma_n^{b'}(k)$。

证明. 第一个主张是 L-Begin 的前提 $\text{target}_m^t \neq \bot$，由定义 46 它给出 $\gamma^t \models d_m$。
(1) 是引理 54(2)。
对 (2)，b′ − 1 处的 L-Begin 写 $\omega_m^{b'} = \text{target}_m^{b'-1}$，其值是提供方，所以 $\theta_n^{b'} = \text{Active}(-, -)$；b − 1 处的 L-Begin 使 $\theta_n^{b} = \text{Reloading}(-, -, -)$，所以 $b \neq b'$ 从而 $b < b'$，两个插曲都按定义 53 开启。设 $[b, u]$ 闭合且假设 $u \leq u'$。则 $u \in [b', u']$，所以 $\text{installed}_m^u$ 且按 (1) 有 $\omega_m^u(k) = n$；那便是 $\text{relied}_n^u$，为 u 处的 L-Unload 所否认。因此 $u' < u$。
对 (3)，n 是 k 在 $\gamma^{b'}$ 处的提供方，所以 $k \in \text{dom}(\sigma_n^{b'})$。没有 n 的 L-Unload 落在 $[b', u']$ 内：在 $[b, u]$ 闭合处它落在 u > u′ 处（按 (2)）；在不闭合处，引理 54(4) 使 n 根本没有 L-Unload。由于 $\theta_n^{b'} = \text{Active}(-, -)$，表 1 于是在 $[b', u']$ 内留下 L-Leave 作为唯一可作用于 n 的规则，它的 Ψt 是 idΓ；按引理 54(1)，σn 在彼处为常数。□

若非如此，跨步骤展开的转移可能安装针对已在其下变动的解析算出的效应，两个前提防止它。L-Iter 与 L-Finish 携带 $\text{target}_n(\gamma) = \omega$，所以转移只有在其已承诺视图仍是其目标视图时才推进，L-Divert 携带否定，所以任何对目标视图的改变都把纤维带出转移。L-Raise 根本不受目标视图约束——抛出是迭代所做的事而非环境所请求的事——且它无论如何都退出转移。两个变更方向不加区分：依赖已消失的组件与被替换依赖的组件走同一条路，因为已变为 ⊥ 的目标视图与已变为某个其他纤维的目标视图与 ω 同样地不相等。

惯性是使这不成其为关于每一步的保证的东西。目标视图转变时已在飞行中的迭代无论如何都要落地（经 L-Divert），而该落地安装了一个针对不再成立的解析算出的效应。因此规则交付的是一个析取，第二个分支正是使第一个分支安全的原因。

**定理 64.（解析相干性。）** 设 n 的一次插曲 $[b, u]$ 在 b 处打开，$\omega_n^b = \omega$。则在插曲的初始区间 $[b, r]$ 上 $\theta_n$ 是 $\text{Reloading}(-, -, -)$，且转移的每次迭代都对照同一个解析 ω 运行：
$$\forall t \in [b, r].\ \text{step}^t \in \{\text{L-Iter}(n), \text{L-Finish}(n)\} \Rightarrow \text{target}_n^t = \omega \qquad (59)$$

在纤维离开该区间处，即 r < u，以下恰好有一项成立：
1. $\text{step}^r = \text{L-Finish}(n)$ 且 $\theta_n^{r+1} = \text{Active}(-, \omega)$；
2. $\text{step}^r \in \{\text{L-Divert}(n), \text{L-Raise}(n)\}$，且插曲在某个 $u > r$ 处闭合，$\gamma^{u+1} \approx (\Psi^{t_l} \circ \cdots \circ \Psi^{t_1})(\gamma^b)$ 如推论 62。

证明. b − 1 处的 L-Begin 写 Reloading，按表 1 它是通往该生命周期状态的唯一规则；其前提 $\theta_n = \text{Inactive}(\bot)$ 与引理 54(4) 把它的任何第二次应用放到插曲之外。所以 Reloading 占据 $[b, u]$ 的初始区间 $[b, r]$ 且不被重入。第一个主张则是表 1 给 L-Iter 与 L-Finish 的前提 $\text{target}_n(\gamma) = \omega'$，连同引理 54(2) 的 $\omega' = \omega$。
对二分，stepᵣ 是前提有 $\theta_n = \text{Reloading}(-, -, -)$ 而结论没有的规则，表 1 提供 L-Finish、L-Divert、L-Raise；第一个落在 Active(−, ω)，另两个落在 Unloading(−, ω, −)，从那里引理 54(4) 使 L-Unload 成为唯一出口，推论 62 提供等式。落地 L-Divert 贡献的迭代是 n 自己的迭代之一，故在累加器所撤回的映射之中。在 r = u 处，序列以转移仍在飞行中结束，第一个主张就是全部断言。□

#### 4.4.4 进展（Progress）

把提供方的撤除推迟到其依赖方离开的守护，只有最终释放才交付定理 63。注册表纤维上的一个关系承载论证。

**定义 65.** 注册表名字上的先序关系（precedence relation）为
$$n \prec m \coloneqq p_n \cap d_m \neq \varnothing \qquad (60)$$

即 n 可能提供 m 声明的键。它只读 d 与 p，按引理 54(5) 它们随纤维的条目一起产生、此后永不被写。

定理 66 与定理 73 在 ≺ 无环的假设下建立，这是一个假设而非定义交付的东西，$n \prec n$ 对声明自己提供的键的组件成立。≺ 排序的是两个纤维的激活而非它们的生命期：$n \prec m$ 说 n 必须先于 m 成为 Active，而提供方活得比其消费者久是定理 63(2)，一个关于带守护演算的定理。
纤维的目标视图既要应答创建它的纤维也要应答其提供方。创建者写 τn，经定义 47 的原语，τ 按引理 54(5) 是单调的。因此创建者在其子的整个存在期间至多转动一次子的目标视图。
进展是关于某条规则适用的主张，因此它按宿主必须提供的规则表述：L-Begin、L-Leave、L-Unload、落地规则 L-Iter、L-Finish、L-Raise 以及 L-Divert。它无处诉诸 L-Divert 的中止备选，所以受第 4.3.3 节惯性约束的宿主也被覆盖。

**定理 66.（进展。）** 假设 ≺ 无环，每个 n 的 $\text{len}(e_n) \leq K$，定义 60 的名字集 N 有限；并设每步都应用生命周期规则。记 S(n) 为作用于 n 的步数，
$$V(n) \coloneqq |\{t : \text{target}_n^t \neq \text{target}_n^{t+1}\}| \qquad (61)$$

为其目标视图转动的次数。则
1. （无死锁。）$\neg \text{quiet}^t$ 蕴含某条生命周期规则在 $\gamma^t$ 适用；
2. （终止。）$S(n) \leq (K + 4)(V(n) + 1)$，且 V(n) 与 $\sum_n S(n)$ 都有限。

因此每个最大生命周期步序列都以静息状态结束。

证明. 无死锁。设 $\neg \text{quiet}^t$，则某个纤维 n 不满足定义 49 的静息的任何条款。按其可属的四种之一对表 1 读数：
- $\theta_n^t = \text{Inactive}(\bot)$ 且 $\text{target}_n^t \neq \bot$：L-Begin 适用；
- $\theta_n^t = \text{Reloading}(-, -, \omega_n)$ 且 $\text{target}_n^t = \omega_n$：$i_n^t(\gamma^t)$ 的值选择 L-Iter、L-Finish、L-Raise 之一适用；
- $\theta_n^t = \text{Reloading}(-, -, \omega_n)$ 且 $\text{target}_n^t \neq \omega_n$：若 $i_n^t(\gamma^t)$ 抛出则 L-Raise 适用，否则 L-Divert 适用，落地该迭代而非中止它；
- $\theta_n^t = \text{Active}(-, \omega_n)$ 且 $\text{target}_n^t \neq \omega_n$：L-Leave 适用。

设没有纤维属于任何这些类，留下某个 $\theta_{m_0}^t = \text{Unloading}(-, -, -)$ 的 $m_0$。如下构造 $m_0, m_1, \ldots$：给定处于 Unloading 的 $m_j$，要么 $\neg \text{relied}_{m_j}$，此时 L-Unload 适用于 $m_j$ 且构造停止；要么存在 $m_{j+1} \neq m_j$ 与 $k_j$ 使 $\text{installed}_{m_{j+1}}^t$ 且 $\omega_{m_{j+1}}^t(k_j) = m_j$。后一种情形有
$$k_j \in d_{m_{j+1}} \cap \text{dom}(\sigma_{m_j}^t) \subseteq d_{m_{j+1}} \cap p_{m_j}$$

其中第二个成员资格是定理 63(3) 在 $m_{j+1}$ 的 t 所落在的插曲处的读取，所以 $m_j \prec m_{j+1}$。且 $\text{target}_{m_{j+1}}^t \neq \omega_{m_{j+1}}^t$：Unloading 纤维在定义 σγ 的并集之外，所以 $k_j$ 在 γ^t 处未被提供或由 $m_j$ 之外的纤维提供。若 $m_{j+1}$ 处于 Active 或 Reloading，则它属于被排除的四种之一，所以它处于 Unloading，构造继续。$m_j$ 是 ≺ 递增的，故由无环性互异，而 $\text{dom}(F^t)$ 有限，故构造停止。
终止. 两个主张界定 S(n)。
(A) 在 $\text{target}_n^t$ 在 $\omega^*$ 处为常数的最大区间上，至多 K + 4 步作用于 n。读表 1 的 θn 列：从 $\text{Active}(-, \omega)$（$\omega \neq \omega^*$）起，纤维经 L-Leave 和 L-Unload，然后若 $\omega^* \neq \bot$ 再经 L-Begin 和至多 $\text{len}(e_n) \leq K$ 次落地，若最后一次落地是 L-Raise 还要第二次 L-Unload；从 $\omega \neq \omega^*$ 的 Reloading 起，它用 L-Divert 代替 L-Leave，从任何其他状态起则取该序列的后缀。区间内没有进一步的 L-Divert 或 L-Leave，L-Begin 所写的 ω 正是 $\text{target}_n^t = \omega^*$ 本身，而在 $\text{Active}(-, \omega^*)$、$\omega^* = \bot$ 的 $\text{Inactive}(\bot)$、以及 $\text{Inactive}(\xi)$ 处根本没有任何规则适用。
(B) 若 $\text{target}_n^t \neq \text{target}_n^{t+1}$ 且 step^t 作用于 m，则要么 $m \prec n$，要么 step^t 写 τn。由定义 46，target_n 的值是 τn 与 $d_n$ 的键的提供方的表的函数；提供方满足 $k \in \text{dom}(\sigma_m) \cap d_n$ 从而 $m \prec n$，表按引理 54(1) 只在作用于其自身纤维的步骤改变。无环性给第一个情形的 $m \neq n$，引理 54(5) 的单调性允许第二个情形每纤维一个 t。
按 (A)，区间计数把 S(n) 界定为 $S(n) \leq (K + 4)(V(n) + 1)$，按 (B)，target_n 的每次转动要么消耗严格 ≺ 低于 n 的纤维的一步，要么是 τn 提供的唯一一次转动，所以 $V(n) \leq 1 + \sum_{m \prec n} S(m)$。由于 ≺ 无环且 N 有限，递归
$$B(n) \coloneqq (K + 4)(2 + \sum_{m \prec n} B(m))$$

良基并定义满足 $S(n) \leq B(n)$ 的 B；从而 V(n) 有限且 $\sum_n S(n) \leq \sum_n B(n)$。按 (1)，不能扩展的序列是静息的。□

N 的有限性是假设而非推导，组件上的一个条件交付它。宿主持有的组件是运行前给出的有限多个程序，所以若没有组件能（无论多间接地）注册某组件自身的纤维之纤维，则注册形成有界深度的树，$\text{len}(e_n) \leq K$ 界定其分支。该假设排除的是无界注册自身实例的组件。

目标记录提供纤维而非布尔值，在第 4.2 节的单一来源纪律下一个键只有一个可能提供者，两者驱动相同的转移。视图买到的是上述结果的词汇，定理 63 与定理 64 都言说纤维所对照激活的解析，也正是它使这些结果在第 3.2.3 节的受限解析下存活——彼处一个键在不同领域解析到不同提供方，提供不再强制视图。实现承载该作用域并把视图放在 fiber.committed 中（第 5.1.3 节）。

#### 4.4.5 会合（Confluence）

至此的结果都是关于单个纤维的。把系统作为整体刻画的性质是它的动态历史不留痕迹：运行中的系统无论经历过什么激活与停用序列，它静息于的状态正是同样的插入与退役在下列假设下会产生的状态——每个最终处于激活的组件被装载一次、按依赖序、且从不卸载。生命周期关系是会合的，它收敛于的正规形是静态组装的那个。这是变更传播（change propagation）为增量计算确立的与从头求值的一致性，在动态组合中的对物 [45]。
主张只关于 ⟶。编排步骤是输入，被给予不同输入的两个序列落在不同地方没有值得注意的理由；问题是生命周期规则——它在哪个纤维下一步、Reloading 纤维走哪个出口上都是非确定性的——能否被弄得彼此分歧。
先需要三个引理。第一个在不参照任何步骤序列的情况下固定最终处于 Active 的纤维集，这使它成为输入而非调度的函数。

**定义 67.** 当纤维未被退役、注册它的纤维是被支撑的、且它声明的每个键都被某个被支撑的纤维提供时，称纤维在 γ 处被支撑（supported）。$\text{dom}(F_\gamma)$ 上的支撑关系是这些条款所读的两个关系的并，
$$m \lhd n \coloneqq m \prec n \vee \pi_n = m \qquad (62)$$

在其良基处（引理 68）记 A 为支撑集，即 γ 处被支撑的纤维：
$$n \in A \coloneqq \neg \tau_n \wedge (\pi_n = \text{root} \vee \pi_n \in A) \wedge \forall k \in d_n.\ \exists m \in A.\ k \in p_m \qquad (63)$$

其中 $\pi_n = \text{root}$ 标记编排器插入的纤维，πn 否则标记其激活注册 n 的纤维。条款不读除 τ、π、d、p 之外的任何字段。两个半部都把纤维关联到其正下方的一个纤维，父而非祖先、直接提供方而非传递提供方，因为条款所读的正是这些；下面的结果在要一个序处取传递闭包，其极小元、极大元与线性化就是 ⊲ 的。

条款引用 A 自身，所以定义是沿 ⊲ 的递归，使其有解的正是下面的引理。

**引理 68.（支撑是良基的。）** 设 ≺ 无环，γ 由步骤序列到达。则 ⊲ 良基，且 A 是定义 67 的唯一解，是 τ、π、d、p 的函数。

证明. 按注册每个名字的步骤的指标对 $\text{dom}(F_\gamma)$ 的名字排序，定义 53 以空注册表开头的序列提供该指标。⊲ 的父半部在该指标上下降：O-Insert 以 $\pi \in \text{dom}(F_\gamma)$ 为前提，所以父指针命名更早注册的纤维，迭代它在有限步到达一个名字的全部祖先。因此环必须使用 ≺，且由于 ≺ 无环必须混合两者，这需要某个 m 声明的键由 m 自己的子树的纤维提供。这样的纤维由 m 或 m 的后代的激活注册，故在 m 的 L-Begin 之后的步骤；该 L-Begin 以 $\gamma \models d_m$ 为前提，所以提供该键的纤维在它之前已经是 Active，定义 58 的条款 (2) 不给该键第二个可能提供方。因此闭合该环的纤维从不会被注册，边不在 $\text{dom}(F_\gamma)$ 中。良基递归有一个解，条款只读那四个字段。□

最后一个条款读 p——组件可能提供的键——而目标读 $\text{dom}(\sigma_\gamma)$——其纤维已安装的键，定义 43 用 $\text{dom}(\sigma_n) \subseteq p_n$ 单独关联两者。因此支撑集一般过近似 Active 纤维，弥合差距的条件如下。

**定义 69.** 当它的一次完成的激活已安装 p 的每个键（即每个实例化它的 Active 纤维处 $\text{dom}(\sigma_n) = p_n$）时，组件 $(d, p, e)$ 在其提供上是全的（total on its provision）。

像独立性（定义 60）一样这是仅关于组件的条件，不提及生命周期状态也不提及步骤，而独立性已经界定了它可能失败多远：若组件只在另一组件的效应所到达的上下文状态安装某键，其正向映射就不与该组件的交换，所以纤维安装的键由它的组件而非调度固定。全性添加的是：该固定集是全部 p 而非其真子集。

**引理 70.（静息处的支撑。）** 设 ≺ 无环、quiet(γ)、γ 没有失败的纤维、γ 的每个组件在其提供上都是全的（定义 69）。则支撑集就是 Active 纤维集：
$$A = \{n : \theta_n = \text{Active}(-, -)\} \qquad (64)$$

证明. 记 A′ 为右侧。没有失败的纤维，定义 49 的静息留下 Inactive(⊥) 与 Active 作为仅有的状态，读作
$$n \in A' \iff \text{target}_n(\gamma) \neq \bot$$

由定义 46，右侧恰在 $\neg \tau_n$ 且每个 $k \in d_n$ 都在 $\text{dom}(\sigma_\gamma)$ 中时成立，且由定义 69，$\text{dom}(\sigma_\gamma) = \bigcup_{m \in A'} p_m$。中间条款是目标不再携带的那个，注册提供它：$\pi_n \neq \text{root}$ 的纤维只被 $\pi_n$ 的激活注册，若 $\pi_n \notin A'$ 则 $\pi_n$ 不处于 Active，所以其累加器已运行并按定义 47 退役 n，给出 τn。因此 A′ 满足定义 67 的条款，引理 68 给它们唯一解，所以 A = A′。□

**引理 71.（换位。）** 设步骤两两独立且 $F^t$ 良构，设步骤 t 与 t + 1 作用于互异的纤维 m 与 n。
1. 若两者都应用激活规则，即 L-Begin、L-Iter 或 L-Finish，且 step^{t+1} 在 $\gamma^t$ 适用，则 step^t 在 step^{t+1} 从 $\gamma^t$ 产生的状态适用，且两种顺序到达相同的 $\gamma^{t+2}$。
2. 若 step^t 在 m 应用激活规则、step^{t+1} 在 n 应用编排规则、且 step^t 不注册 n，则对两者同样成立。

证明. 对 (1)，按表 1，m 的步骤写 θm，并在 $\Psi^t \in \mathfrak{M}(e_m)$ 内写表 σm 与效应部分。因此它不碰 θn 与 $i_n$，且由定义 60 的第二个条件也不碰 $i_n$ 所产出的逆与续延，所以只需检查 step^{t+1} 提及 target_n 的前提。其退役半部不会倒下，没有激活规则写 τ。其解析半部也不会移动：step^{t+1} 在 $\gamma^t$ 适用使每个 $k \in d_n$ 在 $\text{dom}(\sigma^t)$ 中，定义 58 的条款 (2) 使提供这样的 k 的纤维是唯一可能的提供者，所以 $k \notin p_m$ 且 σm 的任何写都达不到 $d_n$ 的键。同一论证反向进行使 step^t 仍适用。最后，Ψt ∈ 𝔐(e_m) 与 Ψ^{t+1} ∈ 𝔐(e_n) 按定义 60 的第一个条件交换，两个编辑写互异纤维的控制字段，所以复合在任一顺序相同。
对 (2)，编排步骤按表 1 有 $\Psi^{t+1} = id_Γ$，所以两个状态映射径直交换，且其 edit^{t+1} 只在 n 处写 τn 或 $\text{dom}(F_\gamma)$，激活步骤既不读也不写这些：后者的前提读 θm、$i_m$、τm 与 target_m，而新 n 的 O-Insert 不移动任何目标——新纤维不提供任何东西——n 的 O-Retire 或 O-Remove 使 σγ 留在原地——n 在一个情形是 Inactive，在另一个其表不受影响。所以 step^t 仍适用。反之，编排步骤的每个前提要么在 n 处读（step^t 不写），要么是 O-Insert 的两个前提之一（更小的注册表只放宽它们），于是它在 $\gamma^{t+1}$ 的适用性给出在 $\gamma^t$ 的适用性；这里 step^t 不注册 n 使 n 在 $\gamma^t$ 仍在场，而 O-Retire 与 O-Remove 要求它。□

**引理 72.（删除。）** 设步骤序列两两独立，每个组件在其提供上都是全的（定义 69），它到达没有纤维失败的静息 $\gamma^T$，设 $[b, u]$ 是 n 的闭合插曲，序列中没有 $n \prec m$ 的任何 m 的插曲闭合，且 n 在 $[b, u]$ 内注册的纤维没有插曲。记 R 为这些注册抽取的名字。则删除 $[b, u]$ 内作用于 n 的步骤连同作用于 R 的名字的每个步骤，留下一个达到与 $\gamma^T$ ≈ 相等、且在 R 外与它 ≃ 相等的步骤序列。

证明. 被删步骤把状态留在它们发现它的地方。设 $t_1 < \cdots < t_l$ 是 $[b, u]$ 内作用于 n 之外纤维的步骤。推论 62 读作
$$\gamma^{u+1} \approx (\Psi^{t_l} \circ \cdots \circ \Psi^{t_1})(\gamma^b)$$

其右侧正是 $[b, u]$ 的幸存步骤自己产生的，$\gamma^{b-1} \approx \gamma^b$ 且它们的编辑所写的控制字段是删除不碰的 n 之外纤维的。按表 1，n 的被删步骤除 θn 外不写任何字段，引理 54(4) 在 u 处把它恢复到 Inactive(⊥)——没有纤维失败——而它在 $\gamma^{b-1}$ 持有的正是它。
一个不变式携带后缀。记 $\gamma'^t$ 为幸存步骤在对应 t 的点到达的状态。断言：对每个 $t > u$，$\gamma^t \approx \gamma'^t$，R 的每个名字在 $\gamma^t$ 处是残迹的且在 $\gamma'^t$ 缺席，且两个状态在 R 外每个名字的每个字段一致。在 t = u + 1 处这是上段连同定义 47，后者使 R 的每个名字被 u 处运行的累加器退役、处于 Inactive(⊥) 且持有空表，R 的纤维按假设没有插曲。归纳步是引理 57(1) 依次在 R 的每个名字处应用：在 R 外作用的步骤在两个状态有相同前提，到达再次 ≈ 相等的状态，并留下 R 的条目残迹。作用于 R 的名字的步骤是被删者之一，引理 57(2) 正是它必须被删而非保留的原因——缺席名字的 O-Retire 或 O-Remove 没有纤维可作用；按 (1) 这样的步骤不移动 R 外的任何字段，所以丢弃它保持不变式。因此最终状态 ≈ 相等，且在 R 外相等。
没有幸存步骤丢失前提。作用于 $m \notin R \cup \{n\}$ 的步骤只经 target_m(γ) 或 relied_m(γ) 读 n。前者在 m 声明 n 提供的键——从而 $n \prec m$——时依赖 n，也在 n 注册了 m——使 m ∈ R——时依赖 n。第一个情形中 m 的插曲按假设不闭合，所以它在 $\gamma^T$ 开着，静息给出 $\omega_m = \text{target}_m^T$ 且引理 70 把其值放在 Active 纤维中，n 不是；由于一个键至多有一个可能提供者，n 在 m 的 L-Begin 处也没有提供 $d_m$ 的键。第二个只经 ωn 的值读 n，而删除插曲只能使 relied 为假，这放宽 L-Unload 上的守护而非阻塞它。这样的步骤所读的 R 的名字由不变式覆盖。两两独立性是效应函数的性质，所以删除步骤保持它。□

**定理 73.（会合。）** 设步骤序列到达没有纤维失败的静息 $\gamma^T$，步骤两两独立、每个组件在其提供上都是全的（定义 69），A 如定义 67。则
1. （正则形。）$\gamma^T$ 从 $\gamma^0$ 到达，到缩减所撤走名字的条目为止，其序列：按原顺序取相同的编排步骤，编排器插入的纤维处的编排步骤先于每个生命周期步骤，其余每个都跟在注册其所作用纤维的步骤之后；并按 A 的线性化 ⊲ 的枚举 $n_1, \ldots, n_k$，按该顺序取每个 $n_i$ 的一次插曲。
2. （会合。）从 $\gamma^0$ 取相同编排步骤的任意两个这样的序列，到达按引理 56 的重命名的 ≃ 相关且 ≈ 相关的状态。

证明. 对 (1)，序列的插曲有两种：闭合的与在 $\gamma^T$ 仍开着的，后者按 quiet^T 与引理 70 是 A 的每个纤维各一次插曲。
闭合插曲先走，按它们的数目归纳。每一阶段在插曲仍闭合的纤维中挑一个 ⊲ 极大的纤维 n 的闭合插曲；引理 68 与 N 的有限性保证存在。引理 72 的三个假设随即满足。没有 $n \prec m$ 的 m 有闭合插曲，按极大性。且 n 在 $[b, u]$ 内注册的纤维没有插曲：这样的纤维被 u 处运行的累加器退役（定义 47），按引理 54(5) 保持退役，所以其目标视图是 ⊥，引理 70 把它放到 A 之外，从而它在 $\gamma^T$ 没有开着的插曲；而 ⊲ 经其父指针把它关联到 n，所以按极大性它也没有闭合的。引理删去该插曲连同它所注册的名字的步骤，把 $\gamma^T$ 留在原地直到那些名字。度量减一，所以没有闭合插曲剩下。
A 之外的纤维不取生命周期步骤。它在 $\gamma^T$ 没有开着的插曲（引理 70 与 quiet^T），现在也没有闭合的，所以它根本没有插曲、全程是 Inactive(⊥)；L-Begin 是彼处唯一适用的规则，应用它就会开启插曲。
编排步骤随后走。编排器插入的纤维处的编排步骤借引理 71(2) 向前移动一步越过不同纤维的生命周期步骤，它适用因为 A 的纤维的步骤不注册这样的名字：注册抽取新名字，而这里的名字是原序列的 O-Insert 引入的。与同一纤维的生命周期步骤没有可交换的：n 的 O-Insert 已先于 n 的每步，n 的 O-Retire 或 O-Remove 只在 A 之外适用，而那不取生命周期步骤。逐个把每个移到前面保持它们的相对顺序。某激活注册的纤维处的编排步骤不能到前面，其前提要求该纤维在场，所以它留在注册放置它的地方；它按上段在 A 之外作用，因此借引理 71 的同一条款与它和注册之间的一切交换。
插曲被排序并做成连续块，按 |A| 归纳。设 $n_1$ 在 A 中 ⊲ 极小。则 $d_{n_1} = \varnothing$ 且 $\pi_{n_1} = \text{root}$，因为定义 67 把 $d_{n_1}$ 的键的提供方与注册 $n_1$ 的纤维都放进 A，而 ⊲ 把它们都放在 $n_1$ 之下。所以 target 不读另一个纤维的任何字段，且没有编排步骤剩下写 τ，没有 $n_1$ 之下的纤维剩下退役它，它是常数。作用于 $n_1$ 的每步都是激活步骤，没有插曲闭合，其剩余前提读 θ 与 $i_{n_1}$，按表 1 只有 $n_1$ 写；因此每一步在更早的状态也适用，引理 71 把它向前移一步而不移动端点。先于 $n_1$ 的一步的其他纤维步骤数每次应用减一，所以 $n_1$ 的插曲成为初始连续块。论证在 $A \setminus \{n_1\}$ 上于块之后的后缀重复，彼处 $n_1$ 全程 Active 且不再取步，所以它也贡献常数目标。这样产生的枚举按构造线性化 ⊲。
对 (2)，两个序列都按 (1) 归约为正则序列，两个归约在同一 A 上运行到重命名为止。定义 67 读 τ、π、d、p，其中后三个随纤维的条目写一次（引理 54(5)），所以要看的是同样的名字带着同样的 d、p、π 产生，同样的名字被退役。插入两序列按假设共享。注册也共享：A 的纤维的激活在其每次迭代注册迭代器在彼处命名的组件，定义 60 的第二个条件在交错下把它保持固定，所以 A 纤维之下的注册树是该纤维组件的函数；注册抽取的名字不共享，这里应用引理 56，用双射匹配两棵树。退役要么是编排步骤（共享），要么是累加器取的 O-Retire，后者恰好退役同一激活注册的名字。线性化 ⊲ 的两个枚举只差不可比插曲的转置，引理 71 再次使端点不变，所以两个正则序列一致。连同定理 66 的终止，生命周期关系因此有唯一正规形。□

失败被排除在陈述之外，因为它是真正的分歧来源，而演算不应被读作否认它：步骤是否抛出取决于它所对照运行的状态，所以一个调度可能使某个纤维失败而另一个完成它，两个静息态于是在该纤维的生命周期状态上不同。它们在任何其他东西上不不同，按推论 62，后者把失败纤维的贡献置零。

第 4.2 节的基础演算中同一定理成立，证明不需要超出删去一条条款的代入。彼处的 L-Unload 不携带守护，所以引理 72 的最后一段是空洞的；该引理其余部分只诉诸 quiet^T，基础演算原样提供它。
该定理为把 Cordis 应用当作静态组装来推理发放许可。增添组件、移除它、替换提供方、再撤销替换的编排器，保证到达它从一开始写下最终组合就会得到的状态；而推理哪些余效应在作用域中的组件作者可以只推理静息状态。它也界定保证：它说的是状态，不是系统沿途产生的发射（emissions），这正是第 6.1 节在边界内追踪的获取（acquisition）与跨过边界的发射之间所作的区分。

## 5. 实现与案例研究（Implementation and Case Study）

本节呈现 Cordis，它把第 3 节的形式模型实现为实用的编程抽象。Cordis 是时空可组合性的元框架（meta-framework）：不同于面向特定领域（如 web 路由、ORM、UI 渲染）的应用框架，它不规定具体场景；它唯一的职责是提供通用的动态组合语义。实现分为三层：(1) 核心库（第 5.1 节）直接实现效应系统与余效应系统；(2) 组件加载器（第 5.2 节）在核心之上扩展配置对账（configuration reconciliation）与热模块替换；(3) 应用框架如 Koishi（第 5.3 节）在前两层之上构建领域特定功能。

### 5.1 核心库（Core Library）

表 2 概括理论构造与其运行时对应物之间的对应。特别地，本节通篇使用下文引入的运行时名字，保留理论符号给形式对应。我们还写 @@name 表示框架内部符号键，因此 ctx[@@store] 中的方括号表示对上下文上不透明槽的符号键访问，而非对字符串键映射的下标。

**表 2 | 理论到实现的对应**

| 理论（第 3 节、第 4 节） | 实现 |
|---|---|
| Γ∞ | ctx，一等上下文 |
| 𝛾∈Γ | 上下文树连同运行中系统所触及的一切 |
| 𝔈Γ、𝔈Γ^iter | 返回/产出逆的效应回调 |
| effectΓ(𝑒) | ctx.effect(callback) |
| Σ、Σ^iso、Σ^inter | ctx[@@store]、ctx[@@isolate]、ctx[@@intercept] |
| get(𝑘)、set(𝑘, 𝑣) | ctx.get(key)、ctx.set(key, value) |
| isolate(𝑘, 𝑟) | ctx.isolate(key, realm) |
| intercept(𝑘, 𝜈) | ctx.intercept(key, metadata) |
| ⟨𝑑, 𝑝, 𝑒, 𝜋, 𝜎, 𝜏, 𝜃⟩ | fiber，ℭΓ 中组件的实例化 |
| dom(𝐹𝛾) | 经 ctx.registry 枚举 |
| 𝑛:𝔑 | fiber.uid |
| 𝑑 : 𝔇Γ | fiber.inject |
| 𝑝 : 𝔓Γ | 组件的 provide |
| 𝑒 : 𝔈Γ* | fiber.apply |
| 𝜋:𝔑 | fiber.parent.fiber.uid，拥有其被实例化所处的上下文的纤维 |
| 派生实现（定义 27） | fiber.ctx，纤维所运行的子上下文 |
| 𝜃（定义 44） | fiber.state，生命周期状态，其 LOADING 即 Reloading、FAILED 即 Inactive(ξ) |
| recover、累加器 g | fiber.dispose，累加器 |
| 𝜔（定义 44） | fiber.committed，已承诺视图 |
| provider_𝑘(𝛾) | 其提供纤维为 ACTIVE 的 Impl |
| target(𝛾, 𝑛) | fiber.target，由 refresh 重算（算法 5），其中 ⊥ 为 INACTIVE |
| Future、惯性（第 4.3.3 节） | fiber.inertia，飞行中转移的句柄 |
| O-Insert、O-Retire（定义 47） | ctx.use 与其回调的逆（算法 4） |
| O-Remove | 从运行时丢出的纤维，uid 被清除 |
| L-Begin、L-Iter、L-Finish | execute 的迭代循环（算法 1） |
| L-Divert | 迭代边界处失败的守护（算法 1），或 reload 链入 unload |
| L-Leave | refresh 将纤维标记为 UNLOADING（第 10 行） |
| L-Unload | unload 及其惯性链（算法 5） |
| L-Unload 上的守护 | unload 等待已通知的依赖方（第 25 行） |
| L-Raise | 记录在纤维上的错误，其目标被置为 ⊥ |

本节其余部分自底向上构建核心库。第 5.1.1 节实现可逆转效应，它是使上下文被变更的唯一原语；第 5.1.2 节在其上实现响应式余效应；第 5.1.3 节把两者复合进组件生命周期；第 5.1.4 节暴露建立在它们之上的上下文级操作。

#### 5.1.1 效应追踪（Effect Tracking）

本节实现可逆转效应（第 3.1 节）。Cordis 中每个上下文变更都流经单一原语 ctx.effect：余效应提供、组件实例化以及每个其他变更上下文的操作都归结为 ctx.effect 调用，因此经由上下文执行的任何操作都被自动追踪，并在组件卸载时被恢复。操作上，ctx.effect 是 effectΓ^iter 的实现（定义 52）：它接受类型 $\mathfrak{E}_Γ^{\text{iter}}$ 的回调，并把它提升到 $\mathfrak{E}_{\partial Γ}^{\text{iter}}$，产出一个 dispose 闭包，调用它即恢复效应。Cordis 通过这一操作同时接纳 𝔈Γ 与 𝔈Γ^iter（临时多态 ad-hoc polymorphism）；我们以迭代器形式为代表，因为普通效应函数是只产出单个逆的退化迭代器。该操作不检查的是 𝔈Γ* 所携带的见证：回调提供逆，而逆恢复它伴随的效应是组件作者的义务而非运行时验证的性质。演算在定理 61 诉诸它，第 6.1 节界定该义务。

算法 1 展示 ctx.effect 的构造。我们写 𝑓 ∘ 𝑔 表示先运行 𝑔 后运行 𝑓 的处置器（disposer），id 表示无操作；因此把每个新逆前挂即得 LIFO 恢复。

**算法 1 效应追踪**
```
1   async function execute(callback, guard)
2       iter ← callback()
3       inverse ← id
4       while guard()
5            (value, done) ← await iter.next()
6            if value then inverse ← value ∘ inverse
7            if done then break
8       return inverse
9   function effect(ctx, callback)
10      armed ← true
11      task ← execute(callback, () ↦ armed)
12      async function dispose()
13           if not armed then return
14           armed ← false
15           recover ← await task
16           recover()
17      ctx.dispose ← dispose ∘ ctx.dispose
18      return dispose
```

引擎 execute 把回调作为效应迭代器（𝔈Γ^iter，定义 51）驱动，并把每步产出的逆折叠成单一复合。每步之前它咨询调用方提供的守护；守护一旦翻转，迭代停止，只剩迄今累积的逆。这是第 4.3.2 节的步边界中断：$\text{Maybe}(\mathfrak{E}^{\text{iter}})$ 续延由迭代器的 done 标志连同守护实现。
ctx.effect 是 execute 上的薄包装，添加两件事。第一，自我处置：守护报告 armed 标志，返回的 dispose 把 armed 翻为 false，同时中止任何飞行中的迭代并使恢复至多触发一次。触发两次会把逆应用到没有效应应用产生的状态上，彼处没有任何东西约束它恢复什么。第二，父复合：dispose 被前挂到外围上下文的累积逆 ctx.dispose 上，所以子效应的逆是父上的效应自身，这是 $\partial^2 Γ$ 的递归结构。组件级（第 5.1.3 节）复用同一 execute，守护测试 fiber.target 的稳定性而非 armed。

#### 5.1.2 余效应操作（Coeffect Operations）

本节实现响应式余效应（第 3.2 节）。所有余效应操作作用于每个上下文携带的三个符号键槽：
- @@store：值存储 σ : (r : R) ⇀ 𝒱r，从领域符号到类型化值；
- @@isolate：领域表 ρ : Map(K, R)，从余效应键到领域符号；
- @@intercept：拦截表 ι : (k : K) → ℳk，给每个键指派其元数据。

前两个复合为两层解析 k → ρ(k) → σ(ρ(k))：ctx.get(key)（算法 2）从 @@isolate 读领域符号 ρ(k)，再从 @@store 读被绑定值 σ(ρ(k))。ρ 间接层让隔离把键重定向到独立绑定，而 @@intercept 只在绑定被访问时被咨询，调整它的使用方式而非解析结果。我们分两部分实现这些操作：(1) 提供与通知，安装或撤除绑定并把变更传播给依赖方；(2) 隔离与拦截，重塑键的解析方式。
提供与通知。由于 set(k, v) 有类型 𝔈Σ（第 3.1 节），余效应提供是 ctx.effect 调用，继承其自动追踪与恢复。算法 2 实现 ctx.set(key, value)，即具体的 set(k, v)：回调把值绑定到领域符号 ρ(k) 之下的存储中，返回的 dispose 函数移除它。安装与移除都调用 notify 把变更传播给依赖组件。

**算法 2 余效应操作**
```
1   function get(ctx, key)
2       realm ← ctx[@@isolate][key] ▷ ρ(k)
3       return ctx[@@store][realm] ▷ σ(ρ(k))
4   function set(ctx, key, value)
5       function callback()
6           realm ← ctx[@@isolate][key] ▷ ρ(k)
7           ctx[@@store][realm] ← value ▷ σ[ρ(k) ↦ v]
8           notify(ctx, [key])
9           return function()
10               delete ctx[@@store][realm] ▷ σ ∖ ρ(k)
11               notify(ctx, [key])
12      return ctx.effect(callback)
```

算法 3 通过测试每个活跃纤维的 fiber.inject 中是否出现某个被变更的键且解析到同一领域，把每个绑定变更传播给依赖方；若是，调用 refresh（第 5.1.3 节）让该纤维对照新状态重估，并返回它所重估的纤维使调用方能等待它们。这是定义 26 的反应式分类：翻转满足性的变更激活或停用纤维，refresh 的幂等性使中性变更无害。该重估与多样控制流的相互作用在第 5.1.3 节展开。

**算法 3 响应式通知**
```
1   function notify(ctx, keys)
2       affected ← ⌀
3       for fiber in all_fibers do
4            for key in keys do
5                 if key ∈ fiber.inject and fiber.ctx[@@isolate][key] = ctx[@@isolate][key] then
6                       refresh(fiber)
7                       affected ← affected ∪ {fiber}
8                       break
9       return affected
```

绑定对依赖方只有在安装它的纤维处于 ACTIVE 时才计为可用，所以 refresh 让每个声明的键对照活跃提供方而非仅对照存储解析。这是定义 46 的 provided by 关系，正是它使撤除在发生前一步就对依赖方可见：已进入 UNLOADING 的提供方已停止提供，所以它的依赖方在它的绑定全部仍在原位时重算出不满足的目标视图并开始自己的拆除。
隔离与拦截。两个操作在结构上做同一件事：每个都派生一个子上下文，只针对 key 调整一个继承的表，让父保持不动，所以恢复是隐式的：丢弃子上下文即足够，没有要运行的显式逆。ctx.isolate(key, realm) 用 realm（默认新生成的符号）覆盖领域映射 ρ（实现 isolate，定义 29），所以给同一键指派不同符号的两个上下文解析到独立绑定。ctx.intercept(key, metadata) 把元数据并入拦截表 ι（实现 intercept，定义 31）：按该定义，新元数据与上下文已为 key 携带的内容组合，并优先于它。

#### 5.1.3 组件生命周期（Component Lifecycle）

组件经 ctx.use 实例化为纤维。本节把纤维（第 5.1 节引入）赋予第 4.3.3 节惯性状态机的操作意义。两个字段驱动下面的算法：fiber.parent，fiber.ctx 的父上下文，形成组件层次（Γ∞ 的递归结构，第 3.3.1 节）；以及 fiber.inertia，飞行中异步转移的句柄（空闲时为 null）。
算法 4 展示组件实例化。组件把余效应规约 component.inject (d) 与效应函数 component.apply 配对；实例化把组件的配置绑定进 fiber.apply（第 9 行），即生命周期随后运行的配置应用效应函数 (e)。回调函数（第 2 行）是父纤维中追踪的效应：执行它时，经调用 refresh（算法 5）启动子的生命周期；恢复它时，把子的目标强置为 ⊥ 并触发 unload。这是定义 47 的注册原语，callback 是其 O-Insert、callback 返回的闭包是其 O-Retire：实例化是父的普通已追踪效应，所以卸载父级联到其子。

**算法 4 组件实例化**
```
1   function use(ctx, component, config)
2       function callback()
3            refresh(fiber)
4            return function()
5                 fiber.target ← ⊥
6                 unload(fiber)
7       fiber ← Fiber(parent: ctx, inject: component.inject)
8       fiber.ctx ← ctx[fiber ↦ fiber]
9       fiber.apply ← () ↦ component.apply(fiber.ctx, config)
10      ctx.effect(callback)
11      return fiber
```

算法 5 实现第 4.3.3 节的惯性状态机，其中 reload 与 unload 是惯性的：一旦进入，转移在系统对目标状态变化作出响应之前运行到完成。它使用余效应存储上的两个辅助查找：resolve(inject) 返回声明键当前解析到的绑定，provided(fiber) 返回该纤维安装其绑定的键。refresh 函数从余效应存储重算 fiber.target，并在纤维尚未处于转移中时启动 reload 或 unload 任务²。reload 函数记录当前目标并执行组件的效应函数 apply。完成后，它检查目标是否仍匹配：若是，纤维进入 ACTIVE；若否（无论新目标是 ⊥ 还是不同的提供方集），它链入 unload。对称地，unload 按 LIFO 顺序恢复所有已追踪效应，然后进入 INACTIVE 或链入 reload。这个相互递归实现惯性性质：一旦转移开始，它在新转移能启动前完成。

**算法 5 组件生命周期**
```
1   function refresh(fiber)
2       target ← target(γ, n)
3       if target = fiber.target then return
4       fiber.target ← target
5       if fiber.inertia then return
6       if target ≠ ⊥ then
7             fiber.state ← LOADING
8             fiber.inertia ← create_task(reload(fiber))
9       else
10            fiber.state ← UNLOADING ▷ 在任何逆被调度前停止服务
11            fiber.inertia ← create_task(unload(fiber))
12  async function reload(fiber)
13      target0 ← fiber.target
14      fiber.committed ← resolve(fiber.inject) ▷ 承诺该视图
15      recover ← await execute(fiber.apply, () ↦ fiber.target = target0)
16      fiber.dispose ← recover ∘ fiber.dispose
17      if fiber.target = target0 then
18            fiber.state ← ACTIVE
19            notify(fiber.ctx, provided(fiber))
20            fiber.inertia ← null
21      else
22            fiber.state ← UNLOADING
23            fiber.inertia ← create_task(unload(fiber))
24  async function unload(fiber)
25      await all(notify(fiber.ctx, provided(fiber)).map(f ↦ f.await())) ▷ 排空依赖方
26      await fiber.dispose()
27      fiber.dispose ← id
28      fiber.committed ← ⊥
29      if fiber.target = ⊥ then
30            fiber.state ← INACTIVE
31            fiber.inertia ← null
32      else
33            fiber.state ← LOADING
34            fiber.inertia ← create_task(reload(fiber))
```

² create_task 调度一个 async 函数并发运行并向它返回句柄（存在 fiber.inertia）。为语言无关我们显式写出它：在急切调度（如 TypeScript 的 promise）下调用是隐式的、返回的 promise 是句柄，而在惰性调度（如 Python 协程、Rust future）下宿主必须派发任务使其推进。

fiber.target 通过对照当前余效应存储解析每个声明键并把提供它的纤维的 uid 组元组来计算，所以它是 target(γ, n)（定义 46）的摘要。按提供方而非按值识别绑定，使对记录目标的单一比较足够：uid 新抽取且从不复用，所以被替换的提供方不能被误认作替换它的那个，即使两者提供相等的值。由于 notify（第 5.1.2 节）在每次余效应变更时重算目标，纤维恰在其声明键之一开始由不同纤维提供时重载。原地覆写自己绑定的提供方因此不被观察到；想让替换传播出去的组件撤回绑定并重新安装它。
算法在两个互补层级运作。在转移层，reload 与 unload 在完成时检查目标，使惯性链能够跨过转移。在每次转移内的迭代层，效应执行（算法 1）在每个迭代边界检查目标，使单次转移内能够部分回滚。这两个机制对应第 4.3.3 节的转移间链与定理 64 所依赖的转移内过期检查。
三行承载定理 63 的余效应排序，每行所处的位置正是排序成立的原因。reload 在第 14 行承诺已解析视图，unload 在全部逆运行后才丢弃它，所以纤维装载多久就读同一绑定多久，包括它自己的拆除。refresh 在转移任务创建前于第 10 行把纤维标记为 UNLOADING，即 L-Leave 步：纤维停止提供，依赖方在它的任何逆被调度前对照这点重算。unload 随后在第 25 行等待每个被通知的依赖方到达 INACTIVE，即 L-Unload 上的守护；notify 只有当依赖方声明键解析到与提供方相同的领域符号时才接纳它，这是守护要求依赖方从该纤维看到键而非仅仅声明它的运行时形式。等待位于整个恢复之前而非某个被等待的逆之内，因为 fiber.dispose 并发启动纤维的效应，把等待放进其一内会使其余失序。终止遵循定理 66：纤维只等待已经停止可满足的依赖方，而自身是提供方的依赖方同样等待它自己的，所以提供图按需遍历而非预先分析。

#### 5.1.4 上下文访问（Context Access）

第 5.1.2 节的余效应操作构成反射式 API（reflective API）：余效应用 ctx.set(key, value) 写、ctx.get(key) 读，两者都以名字为键。Cordis 在这一反射式 API 之上叠加第二种更原生的扩展与消费上下文的方式：属性访问。组件可以把余效应作为属性 ctx[key] 访问，仿佛它是上下文的原生结构，而不经方法调用。在 TypeScript 中，Cordis 用 Proxy 实现它，其 get 陷阱仲裁每个属性访问。算法 6 展示上下文如何把这样的访问解析成余效应（置于第 5.1.2 节原语 get 之上）。

**算法 6 Proxy 中介的上下文访问**
```
1   function resolve(ctx, key)
2       fiber ← ctx.fiber
3       repeat
4            if key ∈ fiber.committed then return fiber.committed[key]
5            if key ∈ fiber.inject then throw INACTIVE_ACCESS
6            if fiber = root then throw UNDECLARED_ACCESS
7            fiber ← fiber.parent.fiber
```

算法 6 自访问上下文沿纤维链上行：在第一个其已承诺视图绑定 key 的纤维处，访问被授权并返回该绑定；若走到声明 key 却未承诺它的纤维，该纤维未装载、访问失败；若到达根仍无任何声明，访问作为未声明被拒绝。这正是 proxy 与裸 ctx.get 的不同之处：ctx.get(key) 是对存储的查找，返回被绑定值或空、从不失败，而 proxy 对照访问纤维自己的视图解析，并在使用点强制余效应规约 d。读视图而非存储也正是定理 63 所依赖的，因为它把依赖保持到其拆除由该依赖消失触发的组件可读。
这个拒绝是在访问点执行的运行时检查。由于组件的余效应规约 d 是静态声明的，同一违规原则上可在编译时检测，通过在执行前把每个 ctx[key] 对照声明的 d 解析；第 6.4 节讨论宿主语言的类型级依赖声明与编译期元编程如何恰好执行这一中介。

### 5.2 组件加载器（Component Loader）

核心库为组件开发者装备动态组合的命令式原语，如 ctx.effect、ctx.use、ctx.set。应用编排器产生一个独立关切：组装既有组件成运行系统并随运行时调整组合。组件加载器通过引入声明式配置层处理这一关切：编排器把期望组合指定为持久数据结构，加载器把该规约的变更翻译成对应的命令式纤维操作。

#### 5.2.1 声明式配置（Declarative Configuration）

第 4 节把运行系统分解为纤维，每个是一次组件实例化。实例化所需的一切都可以声明，所以编排器可以把整个系统描述为声明式配置：加载器实现为纤维并与之保持同步的持久记录。
条目（entries）。配置由条目组成。每个条目指定一个纤维并管理它，绑定双向运行：加载器通过调整纤维响应条目字段的变更，而修订自身配置或禁用自己的组件把变更回写给它的条目。

**定义 74.** 条目声明单个纤维，记录：
- id——稳定标识符，其组的子列表变化时用作对账键；
- url——要实例化的组件模块的 URL；
- isolate——应用于条目上下文的隔离注解；
- intercept——应用于条目上下文的拦截注解；
- config——绑定进组件以形成其效应函数 apply 的配置；
- disabled——条目是否被管理性关闭。

条目能充当忠实的规约，因为支撑纤维的东西恰是条目记录的东西。定义 67 的支撑集只读 τ、π、d、p，条目给出全部四个：disabled 给出 τ，条目在树中的父给出 π，url 选择声明 d 与 p 的组件。支撑集未读的字段是纤维的运行时状态，实例化也不需要它，而引理 70 把支撑集与静息状态（定义 49）的 Active 纤维等同，只要每个组件安装它声明的每个键即可（定义 69）。
这些条目构成配置树，它是系统装载什么的可信记录。条目可以是对应单纤维的叶，也可以是其组件转而装载更多组件的分支节点。Cordis 为这类分组与嵌套装载提供组件：@cordisjs/group 把子条目列表作为其配置并作为子组装载，@cordisjs/include 装载外部配置文件（YAML 或 JSON）并把其条目作嵌套子树嫁接进来。两者都是依托定义 47 注册原语（算法 4）的普通组件，所以嵌套树停留在演算内，下面的结果对它们成立。
对账（reconciliation）。条目记录变化时，加载器增量对账而非拆掉纤维整体重建。这样对账是健全的，理由由元理论提供。
- 定理 73 使静息状态只是最终配置的函数：无论加载器沿途执行什么实例化与退役、以什么顺序，系统静息于从零装载最终配置本会留下的地方。哪些组件最终被装载只按声明读出，只要每个组件安装它声明的每个键（定义 69）；只某些配置下安装所声明键的组件加载器仍能对账，但那时被装载组件集也应答于那些配置。
- 定理 66 证明系统确实静息，所以对账在其实例化与退役发出后即完成。
- 推论 62 把离开纤维对状态的贡献置零，所以重建一个条目撤回其纤维安装的东西、把它周围的纤维留在原样。
- 定理 63 允许条目被一起实例化，编排器无需安排装载顺序：声明键尚未被提供的纤维在其 L-Begin 处等待，提供方离开的纤维在它之前被停用。因此依赖约束的是纤维何时激活，而非它的模块何时抓取求值，所以加载器并发装载模块——大型配置升速的耗时正处于此。
在条目声明的纤维之上，加载器按条目哪些字段变更分派，并对每个应用扰动最小的操作。
- id、url——重建条目，因为其身份或组件已变；
- isolate——重指派条目的领域（算法 7）；
- intercept——原地更新，因为拦截元数据在读时被咨询、无需重载；
- config——交给组件，组件决定如何应用新负载，典型地对照旧配置做 diff、只在实质变更时重载。特别是 @cordisjs/group 条目的 config 是其子条目列表，所以它把更新作为子 id 上的键控 diff 应用，创建、移除或更新每个子；由于更新幸存子会重入这同一个逐字段分派，组对账与条目更新沿树一起递归；
- disabled——置位时卸载纤维、清除时重载它。
托管领域（managed realms）。核心中的隔离经派生子上下文在一个键覆盖领域表 ρ（第 5.1.2 节），在上下文树静止时它足够。条目可能在运行时在组间移动，所以加载器管理自己的领域，isolate 字段按键在两个作用域规则间选择。值 true 请求局部领域（local realm），对条目私有并由其 id 标记，条目移到哪带到哪；字符串请求全局领域（global realm），由命名该字符串的每个条目共享，所以移动这样的条目改变的是它与哪些条目共享绑定，而非它属于哪个领域。一旦没有条目命名某领域，它被丢弃。
重指派条目领域要确认哪些键换了领域、条目自身是否是变更键处的提供方、以及通知哪些依赖方。中间那个问题是难的，因为领域符号可能由多个纤维共享而其中只有一个是提供方。加载器用定界符（delimiters）回答它：每键一个符号 δk，每个上下文在其下存储自己的标签。定界符写在上下文中并被其后代继承，所以条目的标签与提供方的标签恰在两者于 k 的同一 isolate 作用域内派生时一致，这正是 k 处的绑定是条目自己的、必须随它移动的情形。

**算法 7 隔离领域重指派**
```
1   function patch_isolation(entry, ρ′)
2       ρ ← entry.ctx[@@isolate]
3       store ← entry.ctx[@@store]
4       Δ ← {k | ρ(k) ≠ ρ′(k)} ▷ 领域变更的键
5       for k in Δ do
6            entry.ctx[δk] ← fresh tag
7            diff[k] ← (ρ(k), ρ′(k), entry.ctx[δk], store[ρ(k)].fiber.ctx[δk])
8       entry.ctx[@@isolate] ← ρ′
9       reload(entry.fiber)
10      for k in Δ do
11           (s1, s2, d1, d2) ← diff[k]
12           if d1 = d2 and store[s1] and not store[s2] then ▷ 绑定是条目自己的
13                 store[s2] ← store[s1]
14                 delete store[s1]
15      function affected(fiber, k)
16           (s1, s2, d1, d2) ← diff[k]
17           return fiber.ctx[@@isolate][k] ∈ {s1, s2} and (fiber.ctx[δk] = d1) ≠ (d2 = d1)
18      notify(entry.ctx, Δ, affected) ▷ 代替算法 3 的领域测试
```

该测试归结于定界符的一个性质。δk 下的标签写在条目的上下文上并被从它派生的每个上下文继承，且每次重指派时新抽取，所以对上下文 γ′
$$\gamma'[\delta_k] = d_1 \iff \gamma' \text{ 由条目的上下文派生} \qquad (65)$$

记 own(γ′) 为该条件，d2 = d1 是它在提供方的实例。重指派把满足 own 的上下文从 s1 移到 s2，让其他留在原地，按上面的循环恰在提供方满足 own 时把绑定移到 s2。依赖方在其 k 处自己的领域是绑定所处的领域时看到绑定。在 own 于依赖方与提供方一致处，两者都移或都不移，所以依赖方之后看到绑定恰在其之前看到时。在 own 分离它们处，一边移一边留，所以依赖方获得或失去绑定。不等式正是那个分离，成员测试丢弃 k 在两个领域都未解析的依赖方，移动的任何部分都够不到它们。

#### 5.2.2 热模块替换（Hot Module Replacement）

热模块替换（HMR）把可逆转效应模式应用到模块级：源文件变化时（典型地开发期间），系统原地替换受影响的模块而不重启进程。由于纤维已经框住其组件的全部效应与余效应，本身是组件的模块可以只用纤维操作替换：处置旧纤维恢复组件安装的一切，从重载模块实例化的新纤维重新安装它。因此 HMR 不需要开发者标注的验收边界，有别于 Webpack [46] 或 Vite [47] 的 HMR。
@cordisjs/hmr 组件提供 HMR 引擎，分三个阶段运行。
阶段 1：模块分类。引擎取两个输入：stashed 集（自上次重载以来内容变化的文件 URL）与 externals 集（不能热替换、触发完整重启的模块）。写 get_imports(url) 为 url 直接导入的模块，它分类变更的依赖子图，把每个模块标记为 accepted 或 declined：

**算法 8 模块分类**
```
1   function classify(stashed, externals)
2       accepted ← stashed
3       declined ← externals
4       pending ← ⌀
5       for url in stashed do
6            pending ← pending ∪ (get_imports(url) ∖ (accepted ∪ declined))
7       repeat
8            progress ← false
9            for url in pending do
10                if get_imports(url) ∩ accepted ≠ ⌀ then
11                     accepted ← accepted ∪ {url}
12                     pending ← pending ∖ {url}
13                     progress ← true
14                else if get_imports(url) ⊆ declined then
15                     declined ← declined ∪ {url}
16                     pending ← pending ∖ {url}
17                     progress ← true
18                else
19                     pending ← pending ∪ (get_imports(url) ∖ (accepted ∪ declined))
20      until not progress
21      declined ← declined ∪ pending
22      return (accepted, declined)
```

以 stashed 文件的导入为种子，不动点（fixed point）在其导入之一被接受时接受一个模块、在其全部导入被拒绝时拒绝它；任何未决的、陷入导入环的模块默认为拒绝。
阶段 2：过期条目检测。用 accepted 与 declined，引擎随后把组件条目过滤到过期的那些，其依赖树到达被变更模块。它用 get_dependencies 走每个条目的树，后者收集模块的传递导入并把 declined 作为边界尊重：

**算法 9 过期条目检测**
```
1   function get_dependencies(root, declined)
2       deps ← ⌀
3       function traverse(url)
4            if url ∈ deps or url ∈ declined then return
5            deps ← deps ∪ {url}
6            for child in get_imports(url) do traverse(child)
7       traverse(root)
8       return deps
9   function detect(entries, accepted, declined)
10      stale_entries ← ⌀
11      for entry in entries do
12           tree ← get_dependencies(entry.url, declined)
13           if tree ∩ accepted ≠ ⌀ then
14                 accepted ← accepted ∪ tree
15                 stale_entries ← stale_entries ∪ {entry}
16      return stale_entries
```

条目恰在其树与 accepted 相交时过期；该树随后被并入 accepted，所以沿它的每个过期模块在下一阶段被废止。
阶段 3：事务式重载。最后，引擎重载过期条目。它废止 accepted 模块的缓存³，备份每个被移除模块以支持回滚，随后按 url 重新导入每个过期条目的组件模块并换入新纤维：

**算法 10 事务式模块重载**
```
1   function reload(ctx, accepted, stale_entries)
2       backup ← invalidate_caches(accepted)
3       try
4           for entry in stale_entries do
5                entry.fiber.dispose()
6                entry.fiber ← ctx.use(import(entry.url), entry.config)
7       catch error
8           restore_caches(backup)
9           for entry in stale_entries do
10               entry.fiber.dispose()
11               entry.fiber ← ctx.use(backup[entry.url], entry.config)
12          throw error
```

事务保证确保系统从不进入半重载状态：任何模块导入失败（如因语法错误），缓存被恢复、每个过期条目从 backup[entry.url]——缓存刚被恢复的先前组件——重建，撤销已做的换入。

³ 在 Node.js 上，这意味着清除 ES 模块与 CommonJS 模块系统的缓存，因为经 ES 加载器导入的模块可能同时出现在两者中。

### 5.3 案例研究：Koishi

Koishi 是构建在 Cordis 上的开源聊天机器人应用框架⁴。四年多开发中，它积累了超过 4000 个社区贡献的插件⁵，从即时通讯（IM）适配器与数据库驱动到管理控制台与终端用户功能。其规模与多样使其成为对 Cordis 动态组合在生产环境中的代表性验证。
元框架的表达力与通用性。Koishi 运行作服务器端机器人，其每个功能都实现为第 5.1 节上下文原语之上的插件；Koishi 自己只贡献聊天机器人领域的词汇。同一模型在完全不同的运行时重现：Koishi 的 web 控制台是第二个独立的 Cordis 应用，其插件组合的是浏览器与其用户界面的原语而非服务器的。上述不同设定确立了第 3 节模型的两个性质。(1) 它表达力强：其原语足矣承载完整生产系统，宿主框架只提供领域词汇。(2) 它通用：它固定效应与余效应如何组合，把它们的含义留给每个应用，因此既不预设特定领域也不预设特定运行时。
零认知开销的时间可组合性。第 1.2.1 节调查的插件系统不能卸载单个扩展的效应而不重启扩展宿主的进程。Koishi 例行执行这一操作：编排器从控制台停用插件，其效应被原地撤回；开发期间，HMR 引擎在保存时重新应用被编辑的插件，同时保留系统别处的缓存状态与存活连接。Cordis 使这样的移除不仅可能，而且对插件作者毫不费力。由于经上下文执行的效应被追踪、其逆被自动复合（第 3.1 节），即使无经验的作者也为插件的上下文中介效应获得有序清理而不写卸载路径。这实现了第 1.2.1 节识别其缺位的关注局部性：否则依赖每个作者勤勉的正确性，改由抽象一次性交付。
跨开放生态的空间可组合性。与第 1.2.1 节插件系统（插件间依赖大都不存在）相反，Koishi 生态展现真正的依赖拓扑：IM 适配器提供每个消息平台的访问，数据库驱动提供持久存储，功能插件把这些声明为余效应并访问它们。运行时重配提供方——如切换存储后端或重连适配器——只重新激活其已解析依赖变更的依赖方（第 3.2 节）；依赖不可用的插件保持不活跃直到它出现，而不报错。案例研究证实的是该组合跨独立编写代码成立：插件与其依赖通常由不同作者编写，他们除连接两者的余效应外不协调任何东西，所以响应式余效应在独立贡献者的开放生态中保持组装一致。
有效性威胁。这里的证据来自单一宿主语言中的单一生态，所以不能把范式的优点与 TypeScript 实现或 Koishi 特定领域的优点分开，且它是观察性的而非对照替代架构的受控比较。案例研究确立的因此是存在与采用结果而非量化结果；对照基线测量抽象的过载与其对开发者生产力的影响仍是未来工作。

⁴ Koishi 当前使用 Cordis v3。本文呈现 Cordis v4，它精化效应与余效应语义并重新设计加载器；核心组合模型跨两个版本共享。
⁵ Koishi 使用术语 plugin 指称本文形式化为 component 的概念。

## 6. 讨论（Discussion）

前面各节呈现的形式模型与实现引入了一个动态组合的编程范式。本节考察范式如何扩展到更广泛的工程关切，并讨论设计张力与开放问题。

### 6.1 系统边界（System Boundary）

第 3.1 节的每个效应都携带逆，该逆是什么由系统边界裁定。边界把系统所对照运行的环境分成两部分。(1) 系统能够排他地修改且能恢复到该修改之前的状态的位置在边界内，对它的操作被追踪在 Γ 中、之后可被恢复。(2) 两个能力任一失败的位址在边界外，对它的操作作为 idΓ 起作用，因而不被追踪也不被恢复。本节展开该边界的性质及其对恢复的后果。
余效应带来的边界。余效应通过把外部位置实体化移动边界：它把对该位置的每次访问约束到它提供的一组操作，其中每个操作都能提供逆，所以作为 idΓ 的操作转而进入 Γ 被追踪并被恢复。边界因此按位置而非按介质画线，因为上述两个能力都是位置的性质，实体化改变位置的访问方式而把介质留在原样。例如，内存区只在系统独写时在边界内，其他进程也写它时在边界外；文件只在系统独达时在边界内（如私有路径下的暂存文件），它是其他程序读写路径时在边界外。移动边界本身是权衡，在环境为某位置提供可逆转语义与提供这些语义在每次访问的代价之间。我们在第 6.7 节承接这一暗示的协同设计。
获取与发射。到达边界外的操作一般分两阶段进行。(1) 获取阶段，操作获得访问并在边界内安装记录：open 安装 close 移除的描述符，malloc 保留 free 释放的块，fork 启动 kill 终止的子进程。记录本身是实体化该位置的余效应的一部分（如它维护的映射中的条目），安装该条目是可逆转效应。该记录同时是数据可以离开的通道。(2) 发射阶段，操作经通道推数据，如 write 交给文件的字节、send 放到线路上的数据报，推作为 idΓ 起作用，把数据留在他人可读写之处。两阶段因此落在边界两侧：获取留在边界内，而发射越到边界外。
扣留与补偿。仍必须从发射中恢复的系统有两个可用方法。一个是把发射扣留到产生它的状态确定会持久，即回滚-恢复的输出提交问题 [48]。另一个是补偿 [49]：把状态恢复到应用提供的等价——比定义 33 的 ≃ 更粗——的动作，如删除被创建的文件或退还已收取的款项。这样的动作按逆相同的 LIFO 序复合，所以第 3.1 节的复合转移给它们。元理论则不然：定义 60 的交换是针对 ≃ 证明的，必须对照更粗的等价重新建立。

### 6.2 服务多路复用（Service Multiplexing）

OSGi [50] 之类的动态组件平台围绕服务组织组合：提供方按接口发布的、消费者绑定的功能单元。Cordis 余效应模型呼应这一概念，服务对应键之后的接口。提供服务的组件是其提供方，注入服务的组件是其消费者。单服务可由多个提供方实现，这一多重性可以两种形式实现。(1) 排他绑定：若干实现共享一个接口但一次至多绑定一个；编排器选择绑定哪个实现，在它们之间切换需要卸载一个提供方并装载另一个，瞬时扰动每个消费者的依赖。(2) 服务代理（service broker）：一个作为接口入口点的中心服务被后备提供方与消费者都注入，所以多个提供方共存，代理在它们之间分派每个请求。与排他绑定相比，代理吸收这一扰动：更新后备提供方让代理留在原位，所以消费者看不到其依赖的变化、不触发重载。
服务代理垫底三个能力：负载均衡、滚动更新与跨进程调用。
负载均衡。多个提供方共存时，代理按可配置策略（如轮询、最少负载、延迟加权）或消费者命名的显式目标在它们之间分发请求。由于提供方是普通组件，可以增删以扩缩容量；每个提供方经可逆转效应向代理注册，所以卸载它撤回注册、自动把它从代理的路由集剔除。
滚动更新。运行时升级服务实现归结为受控的提供方转移 [51, 52]。执行转移：新提供方作为额外纤维装载并向代理注册；它一旦 ACTIVE，流量逐步从旧提供方移向新提供方（如调整选择权重），旧提供方不再承载飞行中请求后被卸载。该提供方转移把传统上基础设施级的操作（如容器编排、蓝绿部署）变成应用级组合模式。
跨进程调用。服务代理也可以跨进程边界应用 [53]。每个进程托管自己的带本地提供方的 Cordis 上下文；协调组件把它们互连，把每个当作远程提供方。跨进程服务访问经保留接口的 RPC 机制中介，使分布对消费者透明。一个告诫是跨进程调用招致延迟且可能中途失败，所以同步暴露它会阻塞调用方。因此意在跨进程暴露的接口必须按异步契约设计。

### 6.3 访问控制与沙箱（Access Control and Sandboxing）

给定由独立组件组装的应用，保护应用需要两个互补机制：(1) 约束组件可访问的依赖；(2) 把不可信代码与宿主环境沙箱隔离。Cordis 通过依赖声明与拦截支持第一个；第二个需要外部沙箱。
基于能力（capability）的访问控制。依赖访问机制（第 5.1.4 节）已经构成对 proxy 中介属性的访问控制形式：组件只能访问它声明的依赖；未声明访问抛错。这在结构上类似基于能力的安全 [54–56]——权威由持有一引用而非环境权威赋予。inject 声明充当能力请求，上下文 proxy 充当能力中介。由于这些请求是静态声明的，组件所需的 proxy 中介能力全集在运行前已知，让编排器在装载时审查批准它们，而非在访问发生时才发现。
该中介经拦截机制推广到细粒度策略。访问控制元数据可以由上下文携带或由组件声明（定义 30），提供方在依赖被调用时咨询它以决定请求是否被许可。例如，文件系统依赖可携带声明组件可读写哪些路径的元数据，提供方对照元数据检查每次调用。由于这种拦截活在上下文上而非任何一方的代码中，编排器可以调整它以约束任何组件对依赖的访问而不修改提供方，如给社区组件只读数据库访问、核心组件保留完整访问。且由于拦截只影响依赖如何被调用、不影响它是否被满足，它可以在运行时安装、重配或移除而不触发任何重载、不扰动依赖图。
沙箱不可信组件。组件代码不可信时，语言级访问控制不足：能触及宿主运行时的恶意组件可直接够到底层对象，使这类检查形同虚设。沙箱需要语言级手段够不到的执行边界，如软件故障隔离 [57]、独立语言运行时、沙箱化进程或虚拟化容器 [58]。无论机制如何，不可信组件在其自己的沙箱化上下文中运行，经桥（bridge）够到宿主提供的依赖，推广第 6.2 节的跨进程调用：同一透明论证使该桥接访问与本地注入不可区分。在宿主侧，桥是普通纤维，其能力可由上述访问控制削减。

### 6.4 语言无关性与选择（Language Independence and Selection）

虽然 Cordis 用 TypeScript 实现，上下文范式与语言无关：时空可组合性只由其两个可组合性维度定义，因此可以在任何沿两者满足一定要求的语言中实现。我们沿每个维度依次分析这些要求。
时间可组合性。最基本地，时间可组合性需要闭包：可逆转效应把动作与逆配对，逆必须作为值捕获，连同它恢复的状态，以便拆除时重放。此外，组件的代码与装载它的副作用必须在运行时可引入可撤除。
语言如何满足第二个要求取决于其执行模型。托管运行时中，它采取可编程模块注册表的形式，装载的模块可从注册表逐出、无引用后被垃圾回收；Node.js 例如暴露这样的注册表⁶。原生代码不暴露模块注册表，所以引入与撤除采取显式动态链接与解除链接的形式（如 Unix 的 dlopen/dlclose、Windows 的 LoadLibrary/FreeLibrary）[59]，即把目标代码装载进运行中的进程、稍后分离它。WebAssembly 视其嵌入器择路：托管嵌入器（如 JavaScript 宿主）下模块实例由宿主的收集器回收，原生嵌入器（如 Wasmtime）丢弃它时释放。跨这些机制，可逆转效应模型把装载当作上下文上的效应，逆撤销模块引入的符号、类型或处理器的注册。
空间可组合性。空间可组合性需要组件声明依赖、运行时提供并注入这些依赖的机制。这归结为依赖注入（DI）问题 [38]，它在跨语言有别的两个层面显现：依赖如何被类型化、访问如何被中介。
类型层，语言应提供开发者表达良类型依赖访问的方式。消费者从上下文读其键获得余效应，所以上下文类型（第 3.2.1 节）必须记录每个键的余效应。类型类（Haskell）[60] 与 trait（Rust）[61] 通过 instance 或 impl 让提供方从自己的模块扩展上下文类型达成 [62]。TypeScript 的模块增广 [63] 同样让提供方模块把声明并入上下文类型。
运行时层，依赖访问必须被动态中介：键后的余效应可能随提供方装载与卸载而变，且可能跨上下文不同解析。语言因此需要一种透明干预访问的方式，让消费者代码不变，如 JavaScript 的 Proxy 对象 [64] 或 Python 的描述符协议（__get__）[65]。缺乏这样的原语时，运行时反射 [66, 67] 可以动态中介访问，代价是类型安全与开发者体验。
跨两个层面，元编程设施一并供给类型化与中介。注解 [68] 与装饰器把元数据附着到声明上，处理器把它展开成中介访问的访问器；编译期元编程（如 Rust 过程宏、Scala 宏 [69]、Zig comptime）为每个依赖发出一个类型化声明连同这样的访问器，免除通用拦截原语。

⁶ CommonJS 经 require.cache 暴露模块缓存；ES 模块不提供公共逐出 API，虽然模块仍可经引擎内部接口管理。

### 6.5 相互依赖与组件粒度（Mutual Dependencies and Component Granularity）

响应式余效应模型中，依赖环只是让涉及的组件永久不活跃：给定两个组件 A 与 B，若 A 需要 B 提供的键、B 需要 A 提供的键，两者的满足谓词永不能为真。与并发系统中的死锁（依赖调度、必须发生时检测）不同，该条件只按依赖声明即可预测，所以运行时可以在组件被装载时报告它。
实践中，大多数表面上的相互依赖可以分解成消除环的更细粒度组件。考虑两个组件：服务器（提供网络接口）与访问控制器（强制执行授权策略）。两个组件双向交互：访问控制器中介到达服务器的请求，服务器暴露修改访问控制策略的端点。单体设计会让每个组件依赖另一个。然而两个交互方向是逻辑独立的关切。分解它们产生四个组件：server-core、access-control-core、request-mediation（依赖两个核心以对入站请求应用访问控制）与 policy-management（依赖两个核心以经服务器暴露策略修改）。经此方法环被消除，因为核心互不依赖；只有集成组件依赖两者。
这种分解原则上总是可能，因为每个双向交互都可以分解成独立的单向绑定，但它增加组件数：一般情况下，给定 n 个相互作用的组件，集成组件数可随 n 二次增长，因为相互作用组件的每对在每个交互方向可能需要独立组件。这既不影响正确性也不影响运行时性能（组件是轻量的），而更细粒度可能有益：用户获得只装载所需具体集成绑定的能力，实效上提高系统可组合性。然而它可能影响开发者体验：更多组件需要更多配置、更多命名、理解依赖图更多认知开销。
缓解这一粒度代价是工程关切而非理论问题。实用策略包括包捆绑（把相关细粒度组件分组为单一可安装单元）、约定接线（自动连接名字或类型匹配模式的组件）与脚手架工具（从声明式规约生成样板集成组件）。这些策略保留无环模型的形式保证，同时把创作负担降低到接近单体情形。

### 6.6 依赖类型化与版本化（Dependency Typing and Versioning）

形式模型中，依赖链接纯粹由键身份建立：提供键 k 的组件满足在其依赖集声明 k 的任何组件。类型族 𝒱k 保证单个编译单元内的类型级一致，但组件独立开发构建时该保证破裂，这是组件生态的常见场景。该破绽导致两个不同的问题。
接口漂移（interface drift）。提供方可能在各版本间修改与 k 关联的接口（加字段、改方法签名、改行为契约），对照更早接口编译的消费者继续声明同一键 k。依赖在余效应层被满足（k ∈ dom(σ)），但运行时值不再符合消费者预期，导致类型错误、方法未找到失败或静默行为分歧 [70]。
键碰撞（key collision）。两个独立开发的提供方可能用同一键名 k 指称完全不相关的接口。由于键身份单独建立链接，期待一个提供方接口的消费者会不作兼容检查就接纳另一个的值。与接口漂移（至少提供方与消费者共享共同谱系）不同，键碰撞在期待与实际类型间毫无关系，使产生的失败不可预测、难以诊断。
两个问题指向同一缺口：余效应模型只提供名义链接（按键名）而无版本化或结构链接（按接口兼容性）[71]。我们讨论弥补该缺口的三个方法，从最基础设施耦合到最语言无关。
键命名空间化（key namespacing）。把键空间从 K 扩到 K × P，其中 P 标识接口定义包，按构造消除键碰撞：同名局部名的独立开发接口占据不同键。这是最直接的方案但也最耦合：它把包命名空间嵌入形式模型本身，使系统依赖外部包注册表给出键身份。
对等依赖（peer dependencies）。更轻的耦合是通过宿主语言的包管理器声明版本约束 [72]。这是 Cordis 当前采用的方式。组件依赖语义上是对等依赖：组件不在内部捆绑其依赖，而期待运行时上下文提供它们。支持对等依赖的包管理器（如 npm）可以强制版本兼容：提供键的包版本落在消费者声明的对等范围内之外时，不兼容在安装时被抓住，而非作为运行时失败浮出。然而此方式有两个局限：(1) 它依赖提供方忠实地遵守语义化版本，这是不可执行的惯例；(2) 包管理器典型地每个依赖解析到单版本，妨碍在一个应用内从同一包的多个版本装载组件。
结构兼容性（structural compatibility）。完全语言无关的方式会用兼容性谓词替换成员检查 k ∈ dom(σ)，验证提供方实际接口结构上包含消费者的期待。这类似结构子类型 [73]：提供的接口是所需接口的子类型时提供方满足消费者。挑战在于语言无关地定义该谓词：结构兼容对记录类型（宽度子类型）直截了当，对行为契约（如前置/后置条件 [74]、效应规约 [22]）变得复杂，而参数多态引入有界量化的 [75] 处不可判定。
这三个方法处理问题的不同方面。设计一个在保留余效应模型动态组合保证的同时组合这些方法的统一依赖模型，仍是开放问题。

### 6.7 与语言和操作系统的协同设计（Co-Design with Languages and Operating Systems）

第 6.4 节识别宿主语言为上下文范式必须供给的最小限度。本节承接反问题：与该范式协同设计的语言或操作系统可以超出该最小限度提供什么。
与语言的协同设计。围绕上下文范式设计的语言能在两个方面优于库：它给上下文的语义，以及它给效应与余效应的原语。
这样的语言可以在保留第 3.3 节上下文语义的同时让上下文再次隐式。命令式语言已经让每条语句对照隐式上下文运行，而那单一上下文既不追踪效应也不解析余效应。上下文范式反而区分多个上下文：操作要么修改它所对照运行的上下文、要么从它派生另一个（定义 27）。就地实现修改环境上下文，正如命令式语言所做。派生实现反而引入独立上下文，语言必须为它提供构造。让上下文隐式带来人机工效与安全两个好处。(1) 库实现中，每个涉及效应或余效应的函数把上下文作为普通参数或接收者，如第 5.1 节。语言隐式供给上下文处，函数不再需要拿它。(2) 每个上下文携带自己的生命周期状态与已承诺视图（第 4.1 节）。库实现把上下文作为普通变量传递，所以组件可能经闭包或全局变量误达另一组件的上下文。它在彼处安装的效应于是泄漏出自己的生命周期，在彼处读的余效应逃出它的依赖规约。让上下文隐式封堵两者。
这样的语言还可以让效应与余效应为编译器所知。(1) 对效应，效应迭代器（定义 51）在每步分配闭包以连同逆与它恢复的状态持有它们。有了执行效应的语法，编译器可以为整个迭代发出单一状态机并把那些逆hold在其栈帧中。(2) 对余效应，余效应规约可被接纳进类型系统，带来两个好处。第一，依赖环在编译时被报告而非留给运行时（第 6.5 节）。第二，依赖可以按类型结构而非键身份单一比较，如行类型所做 [28]，这是第 6.6 节结构兼容性的类型级支持。
与操作系统的协同设计。第 1.2.3 节观察到粗粒度动态组合替代品，操作系统在进程粒度供给时间可组合性，其上容器编排器在服务粒度供给空间可组合性。与范式协同设计的操作系统会支持细粒度组合，办法是让组件声明的余效应规约成为它所能及的全体，并把操作系统自己的资源作为余效应提供。
这样的操作系统可以供给第 6.3 节推迟到语言外机制的那个沙箱。它通过把组件框定到它声明的依赖做到：装载组件时提供它们，从组件内不留下任何其他可及，正如 WebAssembly 模块在实例化时从嵌入器收取其导入 [76]。它也可以把第 3.2.3 节的余效应隔离与拦截作为自己的能力提供，为每个组件不同绑定键、中介它所供给的访问。
这样的操作系统还可以把自己资源作为余效应提供。边界外的资源在运行时把每次获取记录在作出它的组件名下即变可逆转（第 6.1 节），而每个运行时都保留自己的记录。把资源作为余效应提供的操作系统只保留一次该记录，因为它是把资源交出的当事方、可以把资源归因于索取它的组件。内存与文件描述符是直接候选，而为恢复追踪它们已在内核接口层做过 [77, 78]。此外，操作系统可以使第 6.1 节只能扣留或补偿的某些操作可逆转。事务性地写入持久存储的系统可以回滚它 [79]，构建在写时复制或不可变存储上的系统移动指针即达更早状态 [80, 81]。

## 7. 相关工作（Related Work）

动态组合与若干既有研究领域相交。我们综述最相关的脉络，并区分我们的贡献与每一支。

### 7.1 效应与余效应系统（Effect and Coeffect Systems）

第 2 节把效应与余效应作为我们工作的理论支柱综述。我们先把现在工业实践中常见的单子效应系统定位，再综述把效应与余效应朝与 Cordis 相关方向扩展的三条研究脉络：把代数效应重铸为能力、给效应可逆转语义、以及把效应与余效应统一在单一分级纪律下。
单子效应系统。一族库把效应编码进既有通用语言的类型系统，把它们表示成运行时执行的单子值。Scala 的 ZIO [82] 把计算建模为 ZIO[R,E,A]，TypeScript 的 Effect-TS [83] 建模为 Effect<A,E,R>，一个其参数描述结果、类型化错误与上下文必须供给的服务的泛型类型；fp-ts 库 [84] 经基于 Reader 的单子变换器编码同一错误与需求通道。两个特质把这些系统与 Cordis 分开。第一，追踪是用单子嵌入买的：程序只有写在效应类型之内才获得它，而 Cordis 把效应作为普通宿主代码上的覆盖层追踪。第二，需求经解释（interpretation）交付——一个供给其操作的已安装服务——而该服务被撤除时它的操作已执行的仍留在原位；Cordis 反而把每个效应与逆配对、随提供方来去重新解析需求（第 3.1 节、第 3.2 节）。
代数效应作为能力。代数效应（第 2.1 节）使效应操作对类型系统可见。与我们工作最近的扩展是 Brachthäuser 等人的 Effekt 语言，它把效应类型重释为能力 [85, 86]：效应类型表达计算从其上下文要求什么，而非它可能产生什么副作用。这一视角与我们一样把上下文当作能力的中介。Cordis 与 Effekt 有两个不同。(1) 目的上，代数效应使效应可见以启用模块化解译，给一个操作许多处理程序语义，而 Cordis 使它们可见以启用追踪与逆转，把每个上下文变换与逆配对。(2) 背景上，Effekt 在类型层静态约束效应，默认基于作用域的推理——能力是次等的、被约束在其词法作用域——并经装箱恢复一等使用，后者通过在类型中追踪捕获的能力解除该限制；Cordis 反而在运行时约束效应，目标是在组件移除时完整恢复资源；第 6.7 节承接让上下文在此意义上成为次等的语言会提供什么。
可逆转效应语义。并行脉络给效应可逆转语义而非解释语义。Heunen 等人 [87] 通过把 Hughes 的箭头改编为 dagger 箭头与逆箭头在可逆转背景中建模副作用，捕捉如序列化与可变存储这类其操作允许逆的效应。这是与我们可逆转效应最近的正式表述：两者都把每个效应与撤销它的手段配对，而非经处理程序交付它。两者不同在可逆转性居于何处、以及要求多少。Heunen 等人在指称的、范畴的背景中工作，可逆转性是全局性质、按构造保证（每个计算都可逆），逆是双侧的、从范畴结构恢复。Cordis 在运行时追踪逆、要求更少：不是整个计算可逆，而是每个原子效应允许一侧逆、由调用方在应用点提供而非推导，任何复合的逆随之由复合得出（第 3.1 节）。
分级类型作为统一效应与余效应。Orchard 等人 [88] 提出分级模态类型作为涵盖效应推理（经分级单子）与余效应推理（经分级共单子）的伞形概念，在 Granule 语言中实现，证明单一类型系统可以同时追踪计算做了什么与需要什么；更近的工作把余效应扩展到命令式类 Java 语言 [89, 90] 与 call-by-push-value [91]。所有这些都在类型层运作：效应与余效应是编译时在词法固定作用域上检查的静态注解。我们的贡献与该分析正交：我们把同一两个概念提升为运行时机制，这让 Cordis 能处理动态组合。时间撤回与空间依赖随被装载组件集演化而重新解析，而非在固定程序文本上一次性敲定。

### 7.2 编程范式（Programming Paradigms）

第 3.3.3 节把上下文范式确立为经显式上下文中介效应与余效应的纪律。两个既有范式值得显式比较：一个共享我们的术语，另一个共享我们对横切关切的处理。
面向上下文的编程（COP）。COP [92, 93] 给语言装备层（layers）——按执行上下文在运行时激活与停用的部分方法与类定义，使行为无需基础代码命名其上下文依赖即适应 [94]。COP 与 Cordis 在把上下文当作一等、运行时可变实体并动态激活停用行为上一致，但相似是名义上的。COP 中"上下文"指环境执行情境（如位置、用户、模式），激活在动态作用的范围内改变方法分派；层不追踪它诱导的副作用也不逆转它们，激活不受依赖满足管辖。Cordis 中上下文是中介效应与余效应的 Γ∞ 实体：激活运行组件的可逆转效应、由响应式余效应满足驱动（第 3.2 节），停用完整逆转它们。COP 变化什么行为运行；Cordis 组合并逆转组件安装什么效应与依赖。它们之差是权衡。COP 把激活折进宿主语言的方法分派，以语言特异性为代价获得动态作用域层范围，而 Cordis 作为语言无关覆盖层在一共享上下文上响应式解析激活。Cordis 因此只能把 COP 的全局、值驱动片段表达为余效应：实现间依赖于上下文的选择，而非动态作用域激活。
面向切面编程（AOP）。AOP [95, 96] 把横切关切模块化为切面：在基础程序中量化选择连接点的切点，以及在每个连接点织入的建议。Cordis 处理同一本会散落各组件的上下文行为问题，但它对切面的对物是余效应：许多组件声明依赖其上的共享中介点，因此横切行为可以在彼处重塑而不编辑任何组件。两个范式随后在两个轴不同。(1) 声明对无感（obliviousness）：AOP 切点是无感的量化的，匹配其代码不知被建议的任意连接点，而 Cordis 把横切约束到每个组件声明的余效应，所以它的范围恰好是该声明的表面。这产生确定性与可追踪性：应用编排器可以在配置层检查并治理什么横切一个组件，而不读也不分析其源码，而 AOP 关切只能经量化它的切面可读。(2) 生命周期集成：Cordis 中横切变更由组件的效应承载，组件卸载时被逆转、响应式传播给其依赖方，所以它是动态组合模型内的一步；动态 AOP 系统 [97, 98] 也能在运行时织入与解织，但作为独立操作，既不绑定组件生命周期也不在被建议代码中触发重新解析。

### 7.3 时间可组合性（Temporal Composability）

时间可组合性涉及在运行程序中替换或移除组件并恢复它安装的效应。先前方法按它们如何处理离开组件的状态与效应划分：把状态前移到后继版本、经开发者编写清理恢复效应、在预先固定的作用域内自动逆转效应、或从运行时经接口干预累积的记录回收资源。
有状态前向迁移。一大批系统不宕机地在运行程序中替换组件，把其状态跨版本前移。它们遵守同一时序纪律：组件只有到达安全、无交互的点才可被交换。Kramer 与 Magee 把该判据确立为静息（quiescence）[51]，Vandewoude 等人后来放宽为扰动更小的安定（tranquility）[52]；我们的滚动更新模式（第 6.2 节）通过卸载提供方前排空飞行中请求强制它。动态软件更新（DSU）随后经手写变换函数前移状态：Hicks 等人的通用 C 语言 DSU [99]、Stoyle 等人经 con-freeness 分析的类型安全更新点 [100]、以及 Hayden 等人的 Kitsune [101] 都把旧版数据映射到新版表示，原地继承堆对象、打开文件与连接，同时重新初始化任何未迁移的剩余。同一纪律扩展到持久状态：Overeem 等人 [102] 经手写升级操作在保持系统可用时转换运行中事件存储数据到新模式版本。Erlang/OTP [15] 在进程层持同一立场，经 code_change/3 迁移状态、经重启受监督进程而非逆转其效应从故障恢复；JavaScript 的热模块替换（如 webpack [46]、Vite [47]）在模块层做同样的事，经 module.hot 或 import.meta.hot API 跨重载把状态交出去。与 Cordis 的模块替换（第 5.2 节）相比，这些方法更优雅地迁移内存状态：Cordis 逆转旧组件的已追踪效应、从干净白板重新应用新组件的，所以组件自己的内存状态不存活过一次重载，除非放在更长命的依赖中；在可逆转效应之上叠加 DSU 式前向迁移是未来工作。Cordis 的方法仍在两个方面更通用：它不需要 DSU 与 HMR 要求的手写迁移函数，且它支持完全卸载组件并恢复其资源，而非只原地升级。
开发者编写的恢复。第二族经开发者手写的清理或补偿逻辑恢复组件效应。插件生命周期惯例（如 OSGi [50]、Eclipse 扩展点、IntelliJ 与 VSCode）把清理委托给开发者写的卸载回调；命令模式 [103] 把操作连同 undo 方法封装以待撤销/重做栈；saga 模型 [49] 把长事务结构化为一串都与补偿动作配对的步骤；代数效应处理程序可以附着在拆除时运行的终结器 [104]；事件溯源 [105] 经追加补偿事件撤回状态而根本不执行逆。在所有这些中逆是不可强制的义务、与操作解耦，所以忘掉一个就静默泄漏资源（第 1.2.1 节有经验记录）。React 的 useEffect 钩子 [106] 最接近把效应与逆结构配对：返回运行时在每次重新执行前与卸载时调用的清理。它的短板是可组合性：钩子只能在组件或另一钩子的顶层调用，绝不能在条件、循环或嵌套函数内，且其效应体既不接受异步函数也不接受迭代器。效应因此不能从其他效应组装、不能与控制流交错，留下无从推导复合逆的东西。Cordis 效应不携带这样的限制：它们是自由复合、可以异步运行的普通操作，且只要求每个原子效应手写逆，任何复合的逆随之由复合推导，所以组装既有效应根本不需要写逆。每个效应与其逆的这种结构配对使完整恢复成为系统的不变式，而非开发者纪律之事。
静态作用域逆转。第三族按构造自动逆转效应，但把逆转约束到预先固定的作用域。软件事务内存 [107, 108]（源自硬件事务内存 [109]）记录读/写日志，使一组内存操作要么提交要么中止，把内存回滚到事务前状态。可逆计算从 Landauer 与 Bennett 的热力学分析 [110, 111] 到 Janus 之类的可逆语言 [112] 走得更远，使整个计算的每步全局可逆。可逆进程演算把回溯构建进语义本身：RCCS [113] 沿每个进程携带记忆、允许在其所导向的过去因果等价时收回一步，Phillips 与 Ulidowski [114] 在保留前向操作语义时一致地为 CCS、ACP、CSP 推导可逆算子。它们的因果一致判据是 Cordis 恢复所遵循的序的并发对应物——累加器按后进先出应用组件自己的逆，第 4.3.1 节的守护把提供方的撤除推迟到其消费者停用（定理 63）。然而范围由语义固定，执行的每个动作保持可逆，而 Cordis 组件为每个原子效应提供逆、其累加器把上下文带回组合开始之处。线性类型 [115]、RAII [4] 与 Rust 所有权系统 [61] 把资源释放绑定到词法区域。每个都在静态固定逆转的作用域与范围；Cordis 相反，不预先固定任何这样的作用域：它在组件生命周期上逆转任意上下文操作，并把词法资源管理当作互补手段，适合单个组件内的局部资源。
插入式回收（interposed reclamation）。第四族不靠组件自己提供逆、径在运行时控制的接口记录其获取来回收组件获取之物。Nooks [77] 包装跨越 Linux 内核与其可装载扩展边界的每次调用，使扩展触及的内核对象经对象追踪器通过，其记录告诉恢复管理器扩展失败时释放什么；影子驱动 [78] 从另一侧捕捉同一调用，记录决定驱动状态的请求与配置，使重启的实例可以恢复到它。Akeso [116] 改由编译器插桩获得记录，把内核执行分成可嵌套的恢复域、记录其状态变更与跨线程依赖，并把故障请求连同每个依赖它的域一起回滚。回收因此来自运行时维护的记录，而非来自开发者记得写的清理，这使这一族成为可逆转效应最接近的系统级先例。它与 Cordis 的差别在词汇与范围。平台固定什么可记录，无论作为每个内核对象类型的释放代码、每驱动类一个影子、还是每个被插桩分配器一个逆，所以组件只能持有平台已懂得如何释放的资源；Cordis 组件反而引入自己的效应并为每个原子效应提供逆（第 3.1 节）。回收也被提交或重启同一扩展的请求界住，而 Cordis 在组件整个生命期逆转并把移除传播给其依赖方，后者再释放自己的效应（第 3.2 节）。

### 7.4 空间可组合性（Spatial Composability）

空间可组合性关注组件对他者的依赖如何声明与绑定。先前机制按绑定如何响应变更划分：初始化时接线依赖、响应整个组件的可用性、或在单个值的粒度传播变更。
初始化时依赖接线。两个既有机制在初始化时把组件互连。依赖注入框架 [38]（如 Spring [117]、Guice、Angular、Inversify）在初始化时把依赖注入组件，UI 框架上下文（如 Vue.js 的 provide/inject 与 React 的 Context API）沿组件树传递它们。有些支持动态作用域（如 Spring 的 prototype/request 作用域、Angular 的分层注入器），但都不响应式地重新解析：提供方在运行时被替换或移除时，既有依赖方既不停用也不重新初始化，且没有提供我们组件状态机那种生命周期管理。Cordis 的响应式余效应（第 3.2 节）供给这：通知机制在满足谓词变化时触发生命周期转移。
可用性响应式组件模型。与我们响应式余效应最近的先例响应服务可用性。OSGi 的 Declarative Services 与 iPOJO [118, 119] 让组件声明提供的与要求的服务，运行时随服务出现与消失自动激活停用它们；iPOJO 的 Gravity 项目 [119] 显式瞄准对变化服务可用性的自主运行时适应，其 provide/require 模型直接预示 Cordis 的 ctx.provide/ctx.get 模式。R-OSGi [53] 经 RPC 把同一抽象透明扩展到分布式设定，把网络故障映射为服务撤除事件，第 6.2 节作为 Cordis 模型的扩展讨论这一模式。所有这些系统经停用回调恢复，它有两个局限。第一，回调是手写的，所以资源安全依赖开发者纪律、忘掉的静默泄漏。第二，回调是同步的：拆除需要与离开依赖异步交换时，框架不提供等待它的协议，迫使人对照可能已过期的引用阻塞等待。Cordis 的响应式余效应封堵两个缺口：停用逆转依赖方累积的效应，其惯性 Unloading 状态（第 4.3.3 节）在响应进一步变更前把异步拆除运行到完成。
值级响应性。函数式响应式编程（FRP）[120] 及其现代化身（如 SolidJS 的信号 [121, 122]、Vue 的响应系统、Angular Signals）在值级粒度传播变更：信号变化时，派生计算在调度器下同步或按计划重新求值 [123]。Cordis 的响应式余效应在组件级粒度作用，添加值级传播不建模的异步生命周期语义。同一粒度差异对一致性反向运行：在一轮（turn）内、按依赖图固定的顺序传播，让 FRP 能要求没有派生计算读到更新的与过期的输入混合物，即无毛刺（glitch freedom）[124]；而 Cordis 没有轮的对应物，编排动作一个个到达，只保证没有单一转移跨过其余效应的两个解析（定理 64）。两者互补而非竞争：Cordis 余效应自身可以携带响应式值，组件只在它实际消费的部分上更新，把组件级响应精化为跨两个层次的更细粒度响应式余效应。

## 8. 结论（Conclusion）

我们通过把经典效应与余效应概念提升为运行时机制，呈现了动态组合的形式基础。可逆转效应处理局部时间可组合性：每个上下文变换携带运行时追踪的逆，追踪与恢复都保持组合，所以上下文在组件移除时被恢复。响应式余效应处理局部空间可组合性：上下文每次变更都对照其余效应规约通知组件，每个变更分类为激活、停用或中性，余效应隔离变化声明键解析到什么、余效应拦截变化绑定如何使用。我们把效应上下文与余效应上下文统一成单一上下文类型，其中余效应上的观察等价为效应供给独立性，构成时空可组合性的编程范式。把这些机制组合成组件概念，给出动态组合演算，其元理论把时空可组合性从单一组件带到交错组件的整个系统。我们把这个范式实现为 Cordis 元框架，核心库提供效应追踪与余效应解析，还有带配置对账与热模块替换的声明式组件加载器。Koishi 案例研究在超过 4000 个社区插件的生产系统中验证了 Cordis 的设计。
在人类策展插件生态之外，一个引人注目的未来验证方向是自进化智能体骨架（第 1.2.2 节），其中 AI 智能体在人类监督很少的情况下持续生成并替换自己的骨架组件。在这样的设定中应用 Cordis 会验证快速组件替换下完整恢复的时间保证，以及频繁拓扑变化下依赖协调的空间保证。这样的验证会证明该范式作为智能体骨架及其他自治系统中可恢复、可协调、持续自进化的基础的适用性。