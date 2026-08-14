# 通过联合学习对齐和翻译实现神经机器翻译（Neural Machine Translation by Jointly Learning to Align and Translate）

**作者**: Dzmitry Bahdanau（Jacobs University Bremen）, KyungHyun Cho, Yoshua Bengio（Université de Montréal）

**翻译日期**: 2026-08-14

---

## 摘要（Abstract）

神经机器翻译（neural machine translation）是最近提出的一种机器翻译方法。与传统的统计机器翻译不同，神经机器翻译旨在构建一个单一的神经网络，可以联合调优以最大化翻译性能。最近为神经机器翻译提出的模型通常属于编码器-解码器（encoder–decoder）家族，它们把源句子编码成固定长度的向量，解码器再从该向量生成译文。在本文中，我们猜想使用固定长度向量是改进这种基本编码器-解码器架构性能的瓶颈，并提议对此进行扩展：允许模型自动（软性）搜索源句子中与预测目标词相关的部分，而无需显式地将这些部分构造成硬分段。用这种新方法，我们在英法翻译任务上取得了与现有最先进的基于短语（phrase-based）的系统相当的翻译性能。此外，定性分析表明，模型发现的（软）对齐与我们的直觉高度吻合。

## 1. 引言（Introduction）

神经机器翻译是一种新兴的机器翻译方法，最近由 Kalchbrenner 和 Blunsom (2013)、Sutskever 等人 (2014) 以及 Cho 等人 (2014b) 提出。与由许多单独调优的小子组件组成的传统基于短语的翻译系统（例如 Koehn 等人, 2003）不同，神经机器翻译试图构建并训练一个单一的、大型的神经网络，它读取句子并输出正确的译文。

大多数已提出的神经机器翻译模型属于编码器-解码器家族（Sutskever 等人, 2014; Cho 等人, 2014a），每种语言对应一个编码器和一个解码器，或者涉及对每个句子应用一个特定语言的编码器，然后比较其输出（Hermann 和 Blunsom, 2014）。编码器神经网络读取源句子并将其编码成固定长度的向量。解码器随后从编码后的向量输出译文。整个编码器-解码器系统（由一对语言的编码器和解码器组成）被联合训练，以最大化给定源句子时正确译文的概率。

这种编码器-解码器方法的一个潜在问题是，神经网络需要把源句子的所有必要信息压缩进一个固定长度的向量。这可能使神经网络难以处理长句子，尤其是比训练语料中的句子更长的句子。Cho 等人 (2014b) 表明，随着输入句子长度的增加，基本编码器-解码器的性能确实迅速恶化。

为了解决这个问题，我们引入了编码器-解码器模型的一种扩展，它学习联合进行对齐和翻译。每次提出的模型生成译文中的一个词时，它都会（软性）搜索源句子中信息最集中的一组位置。然后，模型基于与这些源位置相关联的上下文向量以及所有先前生成的目标词来预测目标词。

这种方法与基本编码器-解码器最重要的区别在于，它不试图把整个输入句子编码成单个固定长度的向量。相反，它把输入句子编码成一个向量序列，并在解码译文时自适应地选择这些向量的一个子集。这使神经翻译模型不必把源句子的所有信息（无论其长度如何）压缩进固定长度的向量。我们表明这使模型能够更好地处理长句。

在本文中，我们表明所提出的联合学习对齐和翻译的方法比基本编码器-解码器方法取得显著更好的翻译性能。改进在长句上更明显，但在任何长度的句子上都能观察到。在英法翻译任务上，所提出的方法仅用单个模型就取得了与常规基于短语的系统相当或接近的翻译性能。此外，定性分析表明，所提出的模型在源句子和对应目标句子之间找到了语言上合理的（软）对齐。

## 2. 背景：神经机器翻译（Background: Neural Machine Translation）

从概率的角度看，翻译等同于找到使条件概率最大的目标句子 y，即 arg max_y p(y|x)。在神经机器翻译中，我们拟合一个参数化模型，用平行训练语料最大化句子对的条件概率。一旦翻译模型学会了条件分布，给定源句子，就可以通过搜索使条件概率最大的句子来生成对应的译文。

最近，许多论文提出用神经网络直接学习这个条件分布（例如 Kalchbrenner 和 Blunsom, 2013; Cho 等人, 2014a; Sutskever 等人, 2014; Cho 等人, 2014b; Forcada 和 Ñeco, 1997）。这种神经机器翻译方法通常由两个组件组成，第一个编码源句子 x，第二个解码出目标句子 y。例如，Cho 等人 (2014a) 和 Sutskever 等人 (2014) 使用两个循环神经网络（Recurrent Neural Network, RNN），一个把变长源句子编码成固定长度向量，另一个把该向量解码成变长目标句子。

尽管是一个相当新的方法，神经机器翻译已经显示出有前景的结果。Sutskever 等人 (2014) 报告说，基于带长短期记忆（Long Short-Term Memory, LSTM）单元的 RNN 的神经机器翻译，在英法翻译任务上达到了与常规基于短语的机器翻译系统接近的最先进性能¹。给现有翻译系统添加神经组件，例如对短语表中的短语对打分（Cho 等人, 2014a）或对候选译文重新排序（Sutskever 等人, 2014），已经使人们超越了之前的最先进性能水平。

¹ 这里所说的最先进性能，指不使用任何基于神经网络的组件的常规基于短语的系统的性能。

### 2.1 RNN 编码器-解码器（RNN Encoder–Decoder）

在这里，我们简要描述底层框架，即 Cho 等人 (2014a) 和 Sutskever 等人 (2014) 提出的 RNN 编码器-解码器，我们在其之上构建了一种学习同时对齐和翻译的新架构。

在编码器-解码器框架中，编码器把输入句子（向量序列 x = (x₁, · · · , x_{Tx})）读入向量 c²。最常用的方法是使用 RNN：

$$h_t = f(x_t, h_{t-1}) \tag{1}$$

且

$$c = q(\{h_1, \cdots, h_{T_x}\})$$

其中 h_t ∈ Rⁿ 是 t 时刻的隐藏状态，c 是由隐藏状态序列生成的向量。f 和 q 是某个非线性函数。例如，Sutskever 等人 (2014) 用 LSTM 作为 f，并且 q({h₁, · · · , h_T}) = h_T。

² 虽然大多数先前工作（例如 Cho 等人, 2014a; Sutskever 等人, 2014; Kalchbrenner 和 Blunsom, 2013）都把变长输入句子编码成固定长度向量，但这不是必需的，甚至使用变长向量可能是有益的，我们稍后会展示这一点。

解码器通常被训练为在给定上下文向量 c 和所有先前预测的词 {y₁, · · · , y_{t′−1}} 的情况下预测下一个词 y_{t′}。换句话说，解码器通过把联合概率分解为有序的条件概率来定义译文 y 上的概率：

$$p(y) = \prod_{t=1}^{T} p(y_t | \{y_1, \cdots, y_{t-1}\}, c) \tag{2}$$

其中 y = (y₁, · · · , y_{Ty})。用 RNN 时，每个条件概率建模为：

$$p(y_t | \{y_1, \cdots, y_{t-1}\}, c) = g(y_{t-1}, s_t, c) \tag{3}$$

其中 g 是一个非线性、可能是多层的函数，输出 y_t 的概率，s_t 是 RNN 的隐藏状态。应当注意，也可以使用其他架构，例如 RNN 与反卷积神经网络的混合（Kalchbrenner 和 Blunsom, 2013）。

## 3. 学习对齐和翻译（Learning to Align and Translate）

在本节中，我们提出一种新的神经机器翻译架构。新架构由一个双向 RNN 编码器（第 3.2 节）和一个在解码译文时模拟搜索源句子的解码器（第 3.1 节）组成。

### 3.1 解码器：总体描述（Decoder: General Description）

在新模型架构中，我们把式 (2) 中的每个条件概率定义为：

$$p(y_i | y_1, \ldots, y_{i-1}, x) = g(y_{i-1}, s_i, c_i) \tag{4}$$

其中 s_i 是时间 i 的 RNN 隐藏状态，计算方式为：

$$s_i = f(s_{i-1}, y_{i-1}, c_i)$$

应当注意，与现有编码器-解码器方法（见式 (2)）不同，这里的概率以每个目标词 y_i 各自不同的上下文向量 c_i 为条件。

上下文向量 c_i 依赖于注释（annotation）序列 (h₁, · · · , h_{Tx})，编码器把输入句子映射到该序列。每个注释 h_i 包含整个输入序列的信息，并重点关注输入序列中第 i 个词周围的部分。我们将在下一节详细解释注释是如何计算的。

然后，上下文向量 c_i 计算为这些注释 h_i 的加权和：

$$c_i = \sum_{j=1}^{T_x} \alpha_{ij} h_j \tag{5}$$

每个注释 h_j 的权重 α_{ij} 计算为：

$$\alpha_{ij} = \frac{\exp(e_{ij})}{\sum_{k=1}^{T_x} \exp(e_{ik})} \tag{6}$$

其中

$$e_{ij} = a(s_{i-1}, h_j)$$

是一个对齐模型，它对位置 j 周围的输入与位置 i 的输出匹配程度打分。该分数基于 RNN 隐藏状态 s_{i−1}（在生成 y_i 之前，见式 (4)）和输入句子的第 j 个注释 h_j。

我们把对齐模型 a 参数化为一个前馈神经网络，它与所提出系统的所有其他组件联合训练。注意，与传统机器翻译不同，对齐不被视为潜在变量。相反，对齐模型直接计算软对齐（soft alignment），这使得代价函数的梯度可以反向传播。这个梯度可以用来联合训练对齐模型和整个翻译模型。

我们可以把对所有注释取加权和的方法理解为计算期望注释，其中期望是对可能的对齐取的。令 α_{ij} 为目标词 y_i 与源词 x_j 对齐、或者说从 x_j 翻译而来的概率。那么第 i 个上下文向量 c_i 就是以 α_{ij} 为概率的所有注释的期望注释。

概率 α_{ij} 及其关联的能量 e_{ij} 反映了注释 h_j 相对于先前隐藏状态 s_{i−1} 在决定下一个状态 s_i 和生成 y_i 时的重要性。直观上，这在解码器中实现了一种注意力（attention）机制。解码器决定注意源句子的哪些部分。通过让解码器拥有注意力机制，我们减轻了编码器的负担，使它不必把源句子的所有信息编码进固定长度的向量。有了这种新方法，信息可以分布在整个注释序列中，解码器可以根据需要选择性地检索这些信息。

**图 1：** 所提出模型的图形化说明。模型试图在给定源句子 (x₁, x₂, . . . , x_T) 的情况下生成第 t 个目标词 y_t。

### 3.2 编码器：用于注释序列的双向 RNN（Encoder: Bidirectional RNN for Annotating Sequences）

式 (1) 中描述的常规 RNN 从第一个符号 x₁ 开始按顺序读取输入序列，直到最后一个 x_{Tx}。然而，在我们提出的方案中，我们希望每个词的注释不仅概括前面的词，也概括后面的词。因此，我们提议使用双向 RNN（Bidirectional RNN, BiRNN, Schuster 和 Paliwal, 1997），它最近在语音识别中得到了成功应用（例如 Graves 等人, 2013）。

BiRNN 由前向和后向 RNN 组成。前向 RNN f 按输入序列原有顺序（从 x₁ 到 x_{Tx}）读取序列，并计算前向隐藏状态序列（h₁→, · · · , h_{Tx}→）。后向 RNN f 按相反顺序（从 x_{Tx} 到 x₁）读取序列，产生后向隐藏状态序列（h₁←, · · · , h_{Tx}←）。

我们通过拼接每个词 x_j 的前向隐藏状态 h_j→ 和后向隐藏状态 h_j← 来获得注释，即 h_j = [h_j→; h_j←]。这样，注释 h_j 包含前面词和后面词的概括。由于 RNN 倾向于更好地表示最近的输入，注释 h_j 将聚焦于 x_j 周围的词。这个注释序列随后被解码器和对齐模型用来计算上下文向量（式 (5)–(6)）。

所提出模型的图形化说明见图 1。

## 4. 实验设置（Experiment Settings）

我们在英法翻译任务上评估所提出的方法。我们使用 ACL WMT'14 提供的双语平行语料³。作为比较，我们还报告了 Cho 等人 (2014a) 最近提出的 RNN 编码器-解码器的性能。我们对两个模型使用相同的训练过程和相同的数据集⁴。

³ http://www.statmt.org/wmt14/translation-task.html

⁴ 实现可在 https://github.com/lisa-groundhog/GroundHog 获取。

### 4.1 数据集（Dataset）

WMT'14 包含以下英法平行语料：Europarl（6100 万词）、news commentary（550 万词）、UN（4.21 亿词）以及两个爬取语料（分别为 9000 万和 2.725 亿词），总计 8.5 亿词。按照 Cho 等人 (2014a) 描述的过程，我们用 Axelrod 等人 (2011) 的数据选择方法⁵把合并后的语料缩减到 3.48 亿词。除了上述平行语料外，我们不使用任何单语数据，尽管使用更大的单语语料预训练编码器可能是可行的。我们拼接 news-test-2012 和 news-test-2013 作为开发（验证）集，并在 WMT'14 的测试集（news-test-2014）上评估模型，该测试集包含训练数据中未出现的 3003 个句子。

⁵ 可在 http://www-lium.univ-lemans.fr/~schwenk/cslm_joint_paper/ 在线获取。

在常规分词⁶后，我们使用每种语言中 3 万个最高频词的短表（shortlist）训练模型。任何不在短表中的词都映射到特殊标记（[UNK]）。除了分词外，我们不对数据应用任何其他特殊预处理，例如小写化或词干还原。

⁶ 我们使用了开源机器翻译包 Moses 的分词脚本。

**图 2：** 生成的译文在测试集上的 BLEU 分数随句子长度的变化。结果是基于完整测试集，其中包含模型未知词的句子。

### 4.2 模型（Models）

我们训练两种类型的模型。第一种是 RNN 编码器-解码器（RNNencdec, Cho 等人, 2014a），另一种是所提出的模型，我们称之为 RNNsearch。我们训练每个模型两次：先在有 30 个词以内的句子上训练（RNNencdec-30, RNNsearch-30），再在有 50 个词以内的句子上训练（RNNencdec-50, RNNsearch-50）。

RNNencdec 的编码器和解码器各有 1000 个隐藏单元⁷。RNNsearch 的编码器由前向和后向循环神经网络（RNN）组成，各 1000 个隐藏单元。它的解码器有 1000 个隐藏单元。两种情况下，我们都使用带单个 maxout（Goodfellow 等人, 2013）隐藏层的多层网络来计算每个目标词的条件概率（Pascanu 等人, 2014）。

⁷ 在本文中，"隐藏单元"总是指门控隐藏单元（见附录 A.1.1）。

我们使用 mini-batch 随机梯度下降（SGD）算法和 Adadelta（Zeiler, 2012）训练每个模型。每个 SGD 更新方向用包含 80 个句子的 mini-batch 计算。每个模型训练约 5 天。

模型训练完成后，我们用束搜索寻找近似最大化条件概率的译文（例如 Graves, 2012; Boulanger-Lewandowski 等人, 2013）。Sutskever 等人 (2014) 用这种方法从他们的神经机器翻译模型生成译文。关于实验所用模型架构和训练过程的更多细节，见附录 A 和 B。

## 5. 结果（Results）

### 5.1 定量结果（Quantitative Results）

在表 1 中，我们列出了用 BLEU 分数衡量的翻译性能。从表中可以清楚地看到，在所有情况下，所提出的 RNNsearch 都优于常规 RNNencdec。更重要的是，当只考虑由已知词组成的句子时，RNNsearch 的性能与常规基于短语的翻译系统（Moses）一样高。考虑到 Moses 除了我们用来训练 RNNsearch 和 RNNencdec 的平行语料之外，还使用了单独的单语语料（4.18 亿词），这是一个显著的成就。

**图 3：** RNNsearch-50 找到的四个示例对齐。每幅图的 x 轴和 y 轴分别对应源句子（英语）和生成的译文（法语）中的词。每个像素以灰度（0: 黑色, 1: 白色）显示第 j 个源词注释对第 i 个目标词的权重 α_{ij}（见式 (6)）。(a) 一个任意句子。(b–d) 从测试集中没有未知词、长度为 10 到 20 个词的句子中随机选取的三个样本。

提出该方法的一个动机是基本编码器-解码器方法使用固定长度的上下文向量。我们猜想这一限制可能使基本编码器-解码器方法在长句上表现不佳。在图 2 中，我们看到 RNNencdec 的性能随句子长度的增加急剧下降。另一方面，RNNsearch-30 和 RNNsearch-50 都对句子长度更稳健。特别是 RNNsearch-50，即使句子长度达到 50 或更长也没有性能恶化。所提出模型优于基本编码器-解码器的这一优势，还由 RNNsearch-30 甚至胜过 RNNencdec-50 的事实进一步证实（见表 1）。

**表 1：** 训练好的模型在测试集上计算的 BLEU 分数。第二列和第三列分别显示所有句子上的分数，以及本身和参考译文中都没有任何未知词的句子上的分数。注意，RNNsearch-50? 训练的时间长得多，直到开发集上的性能停止改善。(◦) 当只评估没有未知词的句子时，我们禁止模型生成 [UNK] 标记。

### 5.2 定性分析（Qualitative Analysis）

#### 5.2.1 对齐（Alignment）

所提出的方法提供了一种直观的方式来检查生成译文中的词与源句子中的词之间的（软）对齐。这可以通过可视化式 (6) 中的注释权重 α_{ij} 来实现，如图 3 所示。每幅图中矩阵的每一行表示与注释相关的权重。从中我们可以看出，在生成目标词时，源句子中的哪些位置被认为更重要。

从图 3 的对齐可以看出，英语和法语之间的词对齐在很大程度上是单调的。我们看到每个矩阵的对角线上有很强的权重。然而，我们也观察到一些非平凡、非单调的对齐。形容词和名词在法语和英语中的顺序通常不同，我们在图 3 (a) 中看到一个例子。从这幅图中，我们看到模型正确地把短语 [European Economic Area] 翻译成 [zone économique européen]。RNNsearch 能够正确地把 [zone] 与 [Area] 对齐，跳过两个词（[European] 和 [Economic]），然后一次回看一个词，完成整个短语 [zone économique européenne]。

软对齐相对于硬对齐（hard-alignment）的优势，例如可以从图 3 (d) 中看出。考虑被翻译成 [l'homme] 的源短语 [the man]。任何硬对齐都会把 [the] 映射到 [l']、[man] 映射到 [homme]。这对翻译没有帮助，因为必须考虑 [the] 后面的词才能决定它应该翻译成 [le]、[la]、[les] 还是 [l']。我们的软对齐通过让模型同时观察 [the] 和 [man] 自然地解决了这个问题，在这个例子中，我们看到模型能够正确地把 [the] 翻译成 [l']。我们在图 3 中展示的所有情况下都观察到类似的行为。软对齐的另一个好处是它自然地处理长度不同的源短语和目标短语，而无需一种反直觉地把某些词映射到无处（[NULL]）的方式（例如 Koehn, 2010 第 4 章和第 5 章）。

#### 5.2.2 长句（Long Sentences）

从图 2 可以清楚地看到，所提出的模型（RNNsearch）在翻译长句方面比常规模型（RNNencdec）好得多。这很可能是因为 RNNsearch 不需要把长句完美地编码成固定长度的向量，而只需要准确编码输入句子中围绕特定词的部分。

作为例子，考虑测试集中的这个源句子：

> An admitting privilege is the right of a doctor to admit a patient to a hospital or a medical centre to carry out a diagnosis or a procedure, based on his status as a health care worker at a hospital.

RNNencdec-50 把这个句子翻译为：

> Un privilège d'admission est le droit d'un médecin de reconnaître un patient à l'hôpital ou un centre médical d'un diagnostic ou de prendre un diagnostic en fonction de son état de santé.

RNNencdec-50 正确翻译了源句子直到 [a medical center]。然而，从那里开始（下划线部分），它偏离了源句子的原意。例如，它把源句子中的 [based on his status as a health care worker at a hospital] 替换为 [en fonction de son état de santé]（意为"based on his state of health"，即"基于他的健康状况"）。

另一方面，RNNsearch-50 生成了以下正确的译文，保留了输入句子的完整含义，没有遗漏任何细节：

> Un privilège d'admission est le droit d'un médecin d'admettre un patient à un hôpital ou un centre médical pour effectuer un diagnostic ou une procédure, selon son statut de travailleur des soins de santé à l'hôpital.

让我们考虑测试集中的另一个句子：

> This kind of experience is part of Disney's efforts to "extend the lifetime of its series and build new relationships with audiences via digital platforms that are becoming ever more important," he added.

RNNencdec-50 的翻译是：

> Ce type d'expérience fait partie des initiatives du Disney pour "prolonger la durée de vie de ses nouvelles et de développer des liens avec les lecteurs numériques qui deviennent plus complexes.

与前面的例子一样，RNNencdec 在生成大约 30 个词之后开始偏离源句子的实际含义（见下划线短语）。在那之后，翻译质量恶化，出现基本错误，例如缺少结束引号。

同样，RNNsearch-50 能够正确翻译这个长句：

> Ce genre d'expérience fait partie des efforts de Disney pour "prolonger la durée de vie de ses séries et créer de nouvelles relations avec des publics via des plateformes numériques de plus en plus importantes", a-t-il ajouté.

结合已经展示的定量结果，这些定性观察证实了我们的猜想：RNNsearch 架构比标准 RNNencdec 模型能够可靠得多地翻译长句。在附录 C 中，我们提供了一些由 RNNencdec-50、RNNsearch-50 和 Google 翻译生成的长源句子示例译文，以及参考译文。

## 6. 相关工作（Related Work）

### 6.1 学习对齐（Learning to Align）

最近，Graves (2013) 在手写合成背景下提出了一种类似的方法，把输出符号与输入符号对齐。手写合成是这样一项任务：模型被要求生成给定字符序列的手写体。在他的工作中，他使用高斯核的混合来计算注释的权重，其中每个核的位置、宽度和混合系数由对齐模型预测。更具体地说，他的对齐被限制为预测单调递增的位置。与我们的方法的主要区别是，在 (Graves, 2013) 中，注释权重的众数只朝一个方向移动。在机器翻译的背景下，这是一个严重的限制，因为（长距离）重排序往往是生成语法正确的译文所必需的（例如英语到德语）。

另一方面，我们的方法需要对译文中的每个词计算源句子中每个词的注释权重。对于输入和输出句子大多只有 15–40 个词的翻译任务来说，这个缺点并不严重。然而，这可能会限制所提出方案在其他任务上的适用性。

### 6.2 用于机器翻译的神经网络（Neural Networks for Machine Translation）

自 Bengio 等人 (2003) 引入神经概率语言模型（用神经网络对给定固定数量前词时该词的条件概率建模）以来，神经网络已被广泛用于机器翻译。然而，神经网络的作用在很大程度上仅限于向现有统计机器翻译系统提供单个特征，或对现有系统提供的候选译文列表重新排序。

例如，Schwenk (2012) 提出用前馈神经网络计算一对源短语和目标短语的分数，并把该分数作为基于短语的统计机器翻译系统的附加特征。更近期的，Kalchbrenner 和 Blunsom (2013) 以及 Devlin 等人 (2014) 报告了神经网络作为现有翻译系统子组件的成功使用。传统上，训练为译文侧语言模型的神经网络被用于对候选译文列表进行重打分或重新排序（例如 Schwenk 等人, 2006）。

虽然上述方法已被证明能改进最先进机器翻译系统的翻译性能，但我们更感兴趣的是一个更宏大的目标：设计一个完全基于神经网络的全新翻译系统。因此，我们在本文中考虑的神经机器翻译方法与这些早期工作截然不同。我们的模型不是作为现有系统的一部分使用神经网络，而是独立工作，直接从源句子生成译文。

## 7. 结论（Conclusion）

神经机器翻译的常规方法——编码器-解码器方法——把整个输入句子编码成固定长度的向量，译文从该向量解码。基于 Cho 等人 (2014b) 和 Pouget-Abadie 等人 (2014) 报告的近期实证研究，我们猜想固定长度上下文向量的使用对翻译长句是有问题的。

在本文中，我们提出了一种解决这个问题的全新架构。我们扩展了基本编码器-解码器，让模型在生成每个目标词时（软性）搜索一组输入词，或者搜索编码器计算的它们的注释。这使模型不必把整个源句子编码成固定长度的向量，也让模型只关注与生成下一个目标词相关的信息。这对神经机器翻译系统在较长句子上取得良好结果的能力产生了重大的积极影响。与传统机器翻译系统不同，翻译系统的所有部分（包括对齐机制）都被联合训练，以追求产生正确译文的更好的对数概率。

我们在英法翻译任务上测试了所提出的模型 RNNsearch。实验表明，无论句子长度如何，所提出的 RNNsearch 都显著优于常规编码器-解码器模型（RNNencdec），并且对源句子的长度稳健得多。通过检查 RNNsearch 生成的（软）对齐的定性分析，我们能够得出结论：模型在生成正确译文时，能把每个目标词与源句子中的相关词或它们的注释正确对齐。

也许更重要的是，所提出的方法取得了与现有基于短语的统计机器翻译相当的翻译性能。考虑到所提出的架构、乃至整个神经机器翻译家族只是今年才被提出，这是一个惊人的结果。我们相信这里提出的架构是通向更好的机器翻译、乃至更好地理解自然语言的有希望的一步。

留给未来的挑战之一是更好地处理未知词或罕见词。要让模型得到更广泛的使用，并在所有场景下匹配当前最先进的机器翻译系统的性能，这一点是必需的。

## 致谢（Acknowledgments）

作者感谢 Theano 的开发者们（Bergstra 等人, 2010; Bastien 等人, 2012）。我们感谢以下机构提供的研究经费和计算支持：NSERC、Calcul Québec、Compute Canada、加拿大研究讲席计划（Canada Research Chairs）和 CIFAR。Bahdanau 感谢 Planet Intelligent Systems GmbH 的支持。我们还感谢 Felix Hill、Bart van Merriënboer、Jean Pouget-Abadie、Coline Devin 和 Tae-Ho Kim。

## 参考文献（References）

参考文献略

## 附录 A 模型架构（Model Architecture）

### A.1 架构选择（Architectural Choices）

第 3 节中提出的方案是一个通用框架，其中可以自由定义，例如循环神经网络（RNN）的激活函数 f 和对齐模型 a。在这里，我们描述本文实验中所做的选择。

#### A.1.1 循环神经网络（Recurrent Neural Network）

对于 RNN 的激活函数 f，我们使用 Cho 等人 (2014a) 最近提出的门控隐藏单元（gated hidden unit）。门控隐藏单元是逐元素 tanh 等常规简单单元的替代方案。这种门控单元与 Hochreiter 和 Schmidhuber (1997) 早先提出的长短期记忆（LSTM）单元类似，同样具有更好地建模和学习长期依赖的能力。这之所以可能，是因为展开的 RNN 中存在一些计算路径，其导数乘积接近 1。这些路径使梯度能够轻松地向后流动，而不会遭受太严重的消失效应（Hochreiter, 1991; Bengio 等人, 1994; Pascanu 等人, 2013a）。因此，使用 LSTM 单元来代替这里描述的门控隐藏单元也是可行的，Sutskever 等人 (2014) 就在类似的背景下这样做了。

采用 n 个门控隐藏单元⁸的 RNN 的新状态 s_i 计算如下：

$$s_i = f(s_{i-1}, y_{i-1}, c_i) = (1 - z_i) \circ s_{i-1} + z_i \circ \tilde{s}_i$$

其中 ∘ 是逐元素乘法，z_i 是更新门（update gate）的输出（见下文）。提出的更新状态 s̃_i 计算如下：

$$\tilde{s}_i = \tanh(W e(y_{i-1}) + U[r_i \circ s_{i-1}] + C c_i)$$

其中 e(y_{i−1}) ∈ Rᵐ 是词 y_{i−1} 的 m 维嵌入，r_i 是重置门（reset gate）的输出（见下文）。当 y_i 表示为 1-of-K 向量时，e(y_i) 就是嵌入矩阵 E ∈ R^{m×K} 的一列。只要可能，我们就省略偏置项，使等式更简洁。

更新门 z_i 允许每个隐藏单元保持其先前的激活，重置门 r_i 控制先前状态中应重置多少、以及哪些信息。我们按如下方式计算它们：

$$z_i = \sigma(W_z e(y_{i-1}) + U_z s_{i-1} + C_z c_i)$$

$$r_i = \sigma(W_r e(y_{i-1}) + U_r s_{i-1} + C_r c_i)$$

其中 σ(·) 是逻辑 sigmoid 函数。

在解码器的每一步，我们计算输出概率（式 (4)）为一个多层函数（Pascanu 等人, 2014）。我们使用单层 maxout 单元（Goodfellow 等人, 2013）作为隐藏层，并用 softmax 函数归一化输出概率（每个词一个）（见式 (6)）。

⁸ 这里我们展示解码器的公式。同样的公式可以用在编码器中，只需忽略上下文向量 c_i 及其相关项。

#### A.1.2 对齐模型（Alignment Model）

设计对齐模型时应考虑到：对长度为 T_x 和 T_y 的每一对句子，模型需要被评估 T_x × T_y 次。为了减少计算量，我们使用单层多层感知机：

$$a(s_{i-1}, h_j) = v_a^{\top} \tanh(W_a s_{i-1} + U_a h_j)$$

其中 W_a ∈ R^{n×n}、U_a ∈ R^{n×2n} 和 v_a ∈ Rⁿ 是权重矩阵。由于 U_a h_j 不依赖于 i，我们可以提前预计算它，以最小化计算代价。

### A.2 模型的详细描述（Detailed Description of the Model）

#### A.2.1 编码器（Encoder）

在本节中，我们详细描述实验中使用的所提出模型（RNNsearch）的架构（见第 4–5 节）。从现在起，为了增加可读性，我们省略所有偏置项。模型以 1-of-K 编码的词向量的源句子作为输入：

$$x = (x_1, \ldots, x_{T_x}), \quad x_i \in \mathbb{R}^{K_x}$$

并输出 1-of-K 编码的词向量的译文句子：

$$y = (y_1, \ldots, y_{T_y}), \quad y_i \in \mathbb{R}^{K_y}$$

其中 K_x 和 K_y 分别是源语言和目标语言的词表大小。T_x 和 T_y 分别表示源句子和目标句子的长度。

首先，计算双向循环神经网络（BiRNN）的前向状态：

$$\overrightarrow{h}_i = (1 - \overrightarrow{z}_i) \circ \overrightarrow{h}_{i-1} + \overrightarrow{z}_i \circ \overrightarrow{\tilde{h}}_i, \quad \text{if } i > 0; \quad 0, \quad \text{if } i = 0$$

其中

$$\overrightarrow{\tilde{h}}_i = \tanh(W E x_i + U(\overrightarrow{r}_i \circ \overrightarrow{h}_{i-1}))$$

$$\overrightarrow{z}_i = \sigma(W_z E x_i + U_z \overrightarrow{h}_{i-1})$$

$$\overrightarrow{r}_i = \sigma(W_r E x_i + U_r \overrightarrow{h}_{i-1})$$

E ∈ R^{m×Kx} 是词嵌入矩阵。W, W_z, W_r ∈ R^{n×m}，U, U_z, U_r ∈ R^{n×n} 是权重矩阵。m 和 n 分别是词嵌入维度和隐藏单元数量。σ(·) 照例是逻辑 sigmoid 函数。

后向状态（h₁←, · · · , h_{Tx}←）以类似方式计算。与权重矩阵不同，我们在前向和后向 RNN 之间共享词嵌入矩阵 E。

我们拼接前向和后向状态以获得注释 (h₁, h₂, · · · , h_{Tx})，其中：

$$h_i = \begin{bmatrix} \overrightarrow{h}_i \\ \overleftarrow{h}_i \end{bmatrix} \tag{7}$$

#### A.2.2 解码器（Decoder）

在给定编码器注释的情况下，解码器的隐藏状态 s_i 计算如下：

$$s_i = (1 - z_i) \circ s_{i-1} + z_i \circ \tilde{s}_i$$

其中

$$\tilde{s}_i = \tanh(W E y_{i-1} + U[r_i \circ s_{i-1}] + C c_i)$$

$$z_i = \sigma(W_z E y_{i-1} + U_z s_{i-1} + C_z c_i)$$

$$r_i = \sigma(W_r E y_{i-1} + U_r s_{i-1} + C_r c_i)$$

E 是目标语言的词嵌入矩阵。W, W_z, W_r ∈ R^{n×m}，U, U_z, U_r ∈ R^{n×n}，C, C_z, C_r ∈ R^{n×2n} 是权重。同样，m 和 n 分别是词嵌入维度和隐藏单元数量。初始隐藏状态 s₀ 计算为 s₀ = tanh(W_s h₁←)，其中 W_s ∈ R^{n×n}。

上下文向量 c_i 每一步都由对齐模型重新计算：

$$c_i = \sum_{j=1}^{T_x} \alpha_{ij} h_j$$

其中

$$\alpha_{ij} = \frac{\exp(e_{ij})}{\sum_{k=1}^{T_x} \exp(e_{ik})}$$

$$e_{ij} = v_a^{\top} \tanh(W_a s_{i-1} + U_a h_j)$$

而 h_j 是源句子中的第 j 个注释（见式 (7)）。v_a ∈ Rⁿ′，W_a ∈ R^{n′×n} 和 U_a ∈ R^{n′×2n} 是权重矩阵。注意，如果我们把 c_i 固定为 h_{Tx}→，模型就退化为 RNN 编码器-解码器（Cho 等人, 2014a）。

用解码器状态 s_{i−1}、上下文 c_i 和最后生成的词 y_{i−1}，我们定义目标词 y_i 的概率为：

$$p(y_i | s_i, y_{i-1}, c_i) \propto \exp(y_i^{\top} W_o t_i)$$

其中

$$t_i = \left[ \max(\tilde{t}_{i,2j-1}, \tilde{t}_{i,2j}) \right]_{j=1,\ldots,l}$$

而 t̃_{i,k} 是由下式计算的向量 t̃_i 的第 k 个元素：

$$\tilde{t}_i = U_o s_{i-1} + V_o E y_{i-1} + C_o c_i$$

W_o ∈ R^{Ky×l}，U_o ∈ R^{2l×n}，V_o ∈ R^{2l×m} 和 C_o ∈ R^{2l×2n} 是权重矩阵。这可以理解为带单个 maxout 隐藏层（Goodfellow 等人, 2013）的深度输出（deep output）（Pascanu 等人, 2014）。

#### A.2.3 模型规模（Model Size）

对于本文使用的所有模型，隐藏层大小 n 为 1000，词嵌入维度 m 为 620，深度输出中 maxout 隐藏层的大小 l 为 500。对齐模型中的隐藏单元数量 n′ 为 1000。

**表 2：** 训练统计和相关信息。每次更新对应于使用单个 mini-batch 更新一次参数。一个 epoch 是完整遍历训练集一次。NLL 是训练集或开发集句子的平均条件对数概率。注意，句子的长度各不相同。

## 附录 B 训练过程（Training Procedure）

### B.1 参数初始化（Parameter Initialization）

我们把循环权重矩阵 U, U_z, U_r（前向）和 U←, U_z←, U_r←（后向）初始化为随机正交矩阵。对于 W_a 和 U_a，我们从均值为 0、方差为 0.001²的高斯分布中采样每个元素来初始化。V_a 的所有元素和所有偏置向量都初始化为零。任何其他权重矩阵都从均值为 0、方差为 0.01²的高斯分布中采样初始化。

### B.2 训练（Training）

我们使用随机梯度下降（SGD）算法。使用 Adadelta（Zeiler, 2012）自动调整每个参数的学习率（ε = 10⁻⁶，ρ = 0.95）。每次代价函数梯度的 L2 范数超过预定阈值 1 时，我们将其显式归一化到该阈值（Pascanu 等人, 2013b）。每个 SGD 更新方向用包含 80 个句子的 mini-batch 计算。

在每次更新时，我们的实现所需时间与 mini-batch 中最长句子的长度成正比。因此，为了尽量减少计算浪费，在每第 20 次更新之前，我们取出 1600 个句子对，按长度排序，并分成 20 个 mini-batch。训练数据在训练前打乱一次，并按这种方式顺序遍历。表 2 中我们展示了训练实验中所有模型的相关统计。

## 附录 C 长句子的译文（Translations of Long Sentences）

**表 3：** RNNenc-50 和 RNNsearch-50 从测试集中选出的长源句子（30 个词或以上）生成的译文。对每个源句子，我们还给出了标准参考译文。Google 翻译的译文生成于 2014 年 8 月 27 日。

| 模型 | 译文 |
|---|---|
| 源句 | An admitting privilege is the right of a doctor to admit a patient to a hospital or a medical centre to carry out a diagnosis or a procedure, based on his status as a health care worker at a hospital. |
| 参考译文 | Le privilège d'admission est le droit d'un médecin, en vertu de son statut de membre soignant d'un hôpital, d'admettre un patient dans un hôpital ou un centre médical afin d'y délivrer un diagnostic ou un traitement. |
| RNNenc-50 | Un privilège d'admission est le droit d'un médecin de reconnaître un patient à l'hôpital ou un centre médical d'un diagnostic ou de prendre un diagnostic en fonction de son état de santé. |
| RNNsearch-50 | Un privilège d'admission est le droit d'un médecin d'admettre un patient à un hôpital ou un centre médical pour effectuer un diagnostic ou une procédure, selon son statut de travailleur des soins de santé à l'hôpital. |
| Google 翻译 | Un privilège admettre est le droit d'un médecin d'admettre un patient dans un hôpital ou un centre médical pour effectuer un diagnostic ou une procédure, fondée sur sa situation en tant que travailleur de soins de santé dans un hôpital. |
| 源句 | This kind of experience is part of Disney's efforts to "extend the lifetime of its series and build new relationships with audiences via digital platforms that are becoming ever more important," he added. |
| 参考译文 | Ce type d'expérience entre dans le cadre des efforts de Disney pour "étendre la durée de vie de ses séries et construire de nouvelles relations avec son public grâce à des plateformes numériques qui sont de plus en plus importantes", a-t-il ajouté. |
| RNNenc-50 | Ce type d'expérience fait partie des initiatives du Disney pour "prolonger la durée de vie de ses nouvelles et de développer des liens avec les lecteurs numériques qui deviennent plus complexes. |
| RNNsearch-50 | Ce genre d'expérience fait partie des efforts de Disney pour "prolonger la durée de vie de ses séries et créer de nouvelles relations avec des publics via des plateformes numériques de plus en plus importantes", a-t-il ajouté. |
| Google 翻译 | Ce genre d'expérience fait partie des efforts de Disney à "étendre la durée de vie de sa série et construire de nouvelles relations avec le public par le biais des plates-formes numériques qui deviennent de plus en plus important", at-il ajouté. |
| 源句 | In a press conference on Thursday, Mr Blair stated that there was nothing in this video that might constitute a "reasonable motive" that could lead to criminal charges being brought against the mayor. |
| 参考译文 | En conférence de presse, jeudi, M. Blair a affirmé qu'il n'y avait rien dans cette vidéo qui puisse constituer des "motifs raisonnables" pouvant mener au dépôt d'une accusation criminelle contre le maire. |
| RNNenc-50 | Lors de la conférence de presse de jeudi, M. Blair a dit qu'il n'y avait rien dans cette vidéo qui pourrait constituer une "motivation raisonnable" pouvant entraîner des accusations criminelles portées contre le maire. |
| RNNsearch-50 | Lors d'une conférence de presse jeudi, M. Blair a déclaré qu'il n'y avait rien dans cette vidéo qui pourrait constituer un "motif raisonnable" qui pourrait conduire à des accusations criminelles contre le maire. |
| Google 翻译 | Lors d'une conférence de presse jeudi, M. Blair a déclaré qu'il n'y avait rien dans cette vidéo qui pourrait constituer un "motif raisonnable" qui pourrait mener à des accusations criminelles portes contre le maire. |