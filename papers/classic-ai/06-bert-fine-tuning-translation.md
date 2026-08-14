> ⚠️ 注意：该 PDF 文件名标注为 GPT-2（Language Models are Unsupervised Multitask Learners），但实际内容为复旦大学团队发表的《How to Fine-Tune BERT for Text Classification?》（arXiv:1905.05583）。以下翻译严格忠实于 PDF 的实际内容。

# 如何为文本分类任务微调 BERT？（How to Fine-Tune BERT for Text Classification?）
**作者**: Chi Sun, Xipeng Qiu, Yige Xu, Xuanjing Huang
**翻译日期**: 2026-08-14

---

## 摘要（Abstract）

语言模型预训练已被证明有助于学习通用的语言表示。作为当前最先进的预训练语言模型，BERT（Bidirectional Encoder Representations from Transformers，基于 Transformer 的双向编码器表示）在许多语言理解任务上取得了惊人的成绩。本文通过大量实验，系统研究了 BERT 在文本分类任务上的不同微调（fine-tuning）方法，并为 BERT 微调提供了一种通用解决方案。最终，该方案在八个广泛研究的文本分类数据集上取得了新的最佳结果。

## 1. 引言（Introduction）

文本分类是自然语言处理（Natural Language Processing, NLP）中的一个经典问题。该任务要求为给定的文本序列分配预定义的类别。其中重要的中间步骤是文本表示（text representation）。以往的工作使用各种神经模型来学习文本表示，包括卷积模型（Kalchbrenner et al., 2014; Zhang et al., 2015; Conneau et al., 2016; Johnson and Zhang, 2017; Zhang et al., 2017; Shen et al., 2018）、循环模型（Liu et al., 2016; Yogatama et al., 2017; Seo et al., 2017）以及注意力机制（Yang et al., 2016; Lin et al., 2017）。

另一方面，大量研究表明，在大规模语料上预训练（pre-training）的模型对文本分类及其他 NLP 任务都有帮助，可以避免从头训练新模型。一类预训练模型是词嵌入（word embedding），如 word2vec（Mikolov et al., 2013）和 GloVe（Pennington et al., 2014），或者上下文相关词嵌入（contextualized word embedding），如 CoVe（McCann et al., 2017）和 ELMo（Peters et al., 2018）。这些词嵌入常被用作主任务的附加特征。另一类预训练模型是句子级别的。Howard 和 Ruder（2018）提出了 ULMFiT，这是一种针对预训练语言模型的微调方法，在六个广泛研究的文本分类数据集上取得了当时最佳的结果。更近期的工作表明，预训练语言模型通过利用大量无标注数据，有助于学习通用的语言表示：例如 OpenAI GPT（Radford et al., 2018）和 BERT（Devlin et al., 2018）。BERT 基于多层双向 Transformer（Vaswani et al., 2017），在纯文本上通过掩码词预测（masked word prediction）和下一句预测（next sentence prediction）任务进行训练。

尽管 BERT 在许多自然语言理解（Natural Language Understanding, NLU）任务上取得了惊人的成绩，但其潜力尚未被充分挖掘。目前很少有研究进一步增强 BERT，以提升其在目标任务上的表现。

本文研究如何最大化利用 BERT 完成文本分类任务。我们探索了多种微调 BERT 以提升文本分类性能的方法，并设计了大量实验对 BERT 进行详细分析。

本文的贡献如下：

* 我们提出了一种微调预训练 BERT 模型的通用解决方案，包含三个步骤：（1）在任务内训练数据或领域内数据上进一步预训练 BERT；（2）如果存在多个相关任务，可选择用多任务学习（multi-task learning）方式微调 BERT；（3）针对目标任务微调 BERT。
* 我们还研究了针对目标任务的 BERT 微调方法，包括长文本预处理、层选择（layer selection）、分层学习率（layer-wise learning rate）、灾难性遗忘（catastrophic forgetting）以及少样本学习（low-shot learning）等问题。
* 我们在七个广泛研究的英文文本分类数据集和一个中文新闻分类数据集上取得了新的最佳结果。

## 2. 相关工作（Related Work）

在 NLP 领域，借鉴其他任务学习到的知识日益引起人们的兴趣。我们简要回顾两条相关的研究路线：语言模型预训练和多任务学习。

### 2.1 语言模型预训练（Language Model Pre-training）

预训练词嵌入（Mikolov et al., 2013; Pennington et al., 2014）作为现代 NLP 系统的重要组成部分，相比从头学习的嵌入能带来显著的性能提升。词嵌入的推广形式，如句子嵌入（Kiros et al., 2015; Logeswaran and Lee, 2018）或段落嵌入（Le and Mikolov, 2014），也常作为特征用于下游模型。

Peters 等人（2018）将语言模型派生的嵌入拼接起来作为主任务的附加特征，并在多个主要 NLP 基准上推进了当时的最佳结果。除了无监督数据的预训练之外，利用大量监督数据的迁移学习（transfer learning）也能取得良好效果，例如自然语言推断（Conneau et al., 2017）和机器翻译（McCann et al., 2017）。

更近期，在大规模网络上用大量无标注数据预训练语言模型、再在下游任务上进行微调的方法，已经在多个自然语言理解任务上取得突破，例如 OpenAI GPT（Radford et al., 2018）和 BERT（Devlin et al., 2018）。Dai 和 Le（2015）使用了语言模型微调，但在 1 万条标注样本上出现过拟合；而 Howard 和 Ruder（2018）提出 ULMFiT，在文本分类任务上取得了当时最佳的结果。BERT 通过大规模跨领域语料在掩码语言模型任务（Masked Language Model Task）和下一句预测任务（Next Sentence Prediction Task）上预训练。与以往受限于两个单向语言模型（即从左到右和从右到左）组合的双向语言模型（biLM）不同，BERT 使用掩码语言模型（Masked Language Model）来预测被随机掩码或替换的词。BERT 是第一个基于微调、在众多 NLP 任务上取得最佳结果的表示模型，展示了微调方法的巨大潜力。本文在此基础上进一步探索了 BERT 在文本分类任务上的微调方法。

### 2.2 多任务学习（Multi-task learning）

多任务学习（Caruana, 1993; Collobert and Weston, 2008）是另一条相关的研究方向。Rei（2017）和 Liu 等人（2018）用该方法联合训练语言模型与主任务模型。Liu 等人（2019）将 BERT 作为共享的文本编码层，扩展了最初由 Liu 等人（2015）提出的 MT-DNN 模型。MTL 每次都需要从头训练任务，效率较低，而且通常需要仔细权衡各任务特定的目标函数（Chen et al., 2017）。然而，我们可以通过多任务 BERT 微调来规避这一问题，充分利用共享的预训练模型。

## 3. 面向文本分类的 BERT（BERT for Text Classification）

BERT-base 模型包含一个由 12 个 Transformer 块、12 个自注意力头组成的编码器，隐层大小为 768。BERT 接受长度不超过 512 个 token 的序列作为输入，并输出该序列的表示。序列由一个或两个片段（segment）组成，序列的第一个 token 始终是 [CLS]，它包含特殊的分类嵌入；另一个特殊 token [SEP] 用于分隔片段。

对于文本分类任务，BERT 取第一个 token [CLS] 的最终隐状态 h 作为整个序列的表示。在 BERT 顶部添加一个简单的 softmax 分类器来预测标签 c 的概率：

```
p(c|h) = softmax(W h),             (1)
```

其中 W 是任务特定的参数矩阵。我们联合微调 BERT 的所有参数以及 W，通过最大化正确标签的对数概率来实现。

## 4. 方法（Methodology）

当我们将 BERT 适配到目标领域的 NLP 任务时，需要选择合适的微调策略。本文从以下三个方面寻找合适的微调方法：

1）微调策略（Fine-Tuning Strategies）：当针对目标任务微调 BERT 时，利用 BERT 的方式有很多。例如，BERT 的不同层捕捉不同层次的语义和句法信息——那么哪一层对目标任务更好？如何选择更好的优化算法和学习率？
2）进一步预训练（Further Pre-training）：BERT 在通用领域训练，其数据分布与目标领域不同。一个自然的想法是用目标领域数据进一步预训练 BERT。
3）多任务微调（Multi-Task Fine-Tuning）：在没有预训练语言模型的情况下，多任务学习已证明能有效利用多个任务之间的共享知识。当目标领域中有多个可用任务时，一个有趣的问题是：同时在所有任务上微调 BERT 是否仍能带来收益。

我们微调 BERT 的整体方法如图 1 所示。

![Figure 1: 微调 BERT 的三种通用方法，以不同颜色标示。]()

### 4.1 微调策略（Fine-Tuning Strategies）

神经网络的不同层可以捕捉不同层次的句法和语义信息（Yosinski et al., 2014; Howard and Ruder, 2018）。

为了将 BERT 适配到目标任务，我们需要考虑几个因素：1）第一个因素是长文本的预处理，因为 BERT 的最大序列长度为 512。2）第二个因素是层选择。官方 BERT-base 模型由嵌入层、12 层编码器和池化层组成。我们需要为文本分类任务选择最有效的层。3）第三个因素是过拟合问题。需要一个更好的优化器配合合适的学习率。

直觉上，BERT 模型的较低层可能包含更通用的信息。我们可以用不同的学习率微调它们。

遵循 Howard 和 Ruder（2018）的做法，我们将参数 θ 划分为 {θ1 , · · · , θL }，其中 θl 表示 BERT 第 l 层的参数。参数按如下方式更新：

```
θtl = θt−1
      l   − η l · ∇θl J(θ),         (2)
```

其中 η l 表示第 l 层的学习率。

我们将基础学习率设为 η L ，并使用 η k−1 = ξ · η k ，其中 ξ 是衰减因子，小于或等于 1。当 ξ < 1 时，较低层的学习率低于较高层。当 ξ = 1 时，所有层的学习率相同，相当于常规随机梯度下降（stochastic gradient descent, SGD）。我们将在第 5.3 节研究这些因素。

### 4.2 进一步预训练（Further Pre-training）

BERT 模型在通用领域语料上预训练。对于特定领域的文本分类任务（如电影评论），其数据分布可能与 BERT 的训练数据不同。因此，我们可以在领域特定数据上，用掩码语言模型和下一句预测任务进一步预训练 BERT。我们进行了三种进一步预训练方法：

1）任务内预训练（Within-task pre-training）：BERT 在目标任务的训练数据上进一步预训练。
2）领域内预训练（In-domain pre-training）：预训练数据取自目标任务所在的领域。例如，有多个数据分布相似的不同情感分类任务，我们可以使用这些任务合并后的训练数据进一步预训练 BERT。
3）跨领域预训练（Cross-domain pre-training）：预训练数据同时取自目标任务的同领域数据和其他不同领域的数据。

我们将在第 5.4 节研究这些不同的进一步预训练方法。

### 4.3 多任务微调（Multi-Task Fine-Tuning）

多任务学习也是共享多个相关监督任务知识的有效方法。与 Liu 等人（2019）类似，我们也在多任务学习框架下微调 BERT 进行文本分类。

所有任务共享 BERT 层和嵌入层。唯一不共享的层是最终分类层，这意味着每个任务都有一个私有的分类器层。实验分析见第 5.5 节。

## 5. 实验（Experiments）

我们研究了七个英文和一个中文文本分类任务的不同微调方法。我们使用基础 BERT 模型：分别采用 uncased BERT-base 模型和中文 BERT-base 模型。

### 5.1 数据集（Datasets）

我们在八个广泛研究的数据集上评估我们的方法。这些数据集的文档数量和文档长度各不相同，涵盖三类常见的文本分类任务：情感分析（sentiment analysis）、问题分类（question classification）和主题分类（topic classification）。每个数据集的统计信息见表 1。

| 数据集 | 类别数 | 类型 | 平均长度 | 最大长度 | 超长比例 | 训练样本数 | 测试样本数 |
|---|---|---|---|---|---|---|---|
| IMDb | 2 | 情感 | 292 | 3,045 | 12.69% | 25,000 | 25,000 |
| Yelp P. | 2 | 情感 | 177 | 2,066 | 4.60% | 560,000 | 38,000 |
| Yelp F. | 5 | 情感 | 179 | 2,342 | 4.60% | 650,000 | 50,000 |
| TREC | 6 | 问题 | 11 | 39 | 0.00% | 5,452 | 500 |
| Yahoo! Answers | 10 | 问题 | 131 | 4,018 | 2.65% | 1,400,000 | 60,000 |
| AG's News | 4 | 主题 | 44 | 221 | 0.00% | 120,000 | 7,600 |
| DBPedia | 14 | 主题 | 67 | 3,841 | 0.00% | 560,000 | 70,000 |
| Sogou News | 6 | 主题 | 737 | 47,988 | 46.23% | 54,000 | 6,000 |

表 1：八个文本分类数据集的统计信息。超长比例是指长度超过 512 的样本所占百分比。

**情感分析** 对于情感分析，我们使用二分类的电影评论 IMDb 数据集（Maas et al., 2011），以及 Zhang 等人（2015）构建的 Yelp 评论数据集的二分类和五分类版本。

**问题分类** 对于问题分类，我们在 TREC 数据集的六分类版本（Voorhees and Tice, 1999）和 Zhang 等人（2015）创建的 Yahoo! Answers 数据集上评估我们的方法。TREC 数据集是一个问题分类数据集，由开放领域的基于事实的问题组成，分为宽泛的语义类别。与其他文档级数据集相比，TREC 数据集是句子级的，且训练样本较少。Yahoo! Answers 数据集是一个大型数据集，有 140 万条训练样本。

**主题分类** 对于主题分类，我们使用 Zhang 等人（2015）创建的大规模 AG's News 和 DBPedia 数据集。为了测试 BERT 对中文文本的有效性，我们为搜狗新闻语料创建了中文训练和测试数据集。与 Zhang 等人（2015）不同，我们直接使用汉字而不是拼音。该数据集是 SogouCA 和 SogouCS 新闻语料的组合（Wang et al., 2008）。我们根据 URL 确定新闻的类别，例如 "sports" 对应 "http://sports.sohu.com"。我们选择了 6 个类别——"sports"（体育）、"house"（房产）、"business"（财经）、"entertainment"（娱乐）、"women"（女人）和 "technology"（科技）。每个类别的训练样本数为 9,000，测试样本数为 1,000。

**数据预处理** 遵循 Devlin 等人（2018）的做法，我们使用包含 30,000 个 token 词表的 WordPiece 嵌入（Wu et al., 2016），并用 ## 表示拆分出的词片段。因此，数据集中文档长度的统计基于词片段。对于 BERT 的进一步预训练，我们使用 spaCy 对英文数据集进行句子切分，在处理中文搜狗新闻数据集时使用 "。"、"？" 和 "！" 作为分隔符。

### 5.2 超参数（Hyperparameters）

我们使用 BERT-base 模型（Devlin et al., 2018），隐层大小为 768，12 个 Transformer 块（Vaswani et al., 2017）和 12 个自注意力头。进一步预训练在 1 块 TITAN Xp GPU 上进行，批大小为 32，最大序列长度为 128，学习率为 5e-5，训练步数为 100,000，预热步数为 10,000。

我们在 4 块 TITAN Xp GPU 上微调 BERT 模型，批大小设为 24 以充分利用 GPU 内存。dropout 概率始终保持在 0.1。我们使用 Adam 优化器，β1 = 0.9，β2 = 0.999。我们使用斜三角学习率（slanted triangular learning rates）（Howard and Ruder, 2018），基础学习率为 2e-5，预热比例为 0.1。我们根据经验将最大 epoch 数设为 4，并在验证集上保存最佳模型用于测试。

### 5.3 实验一：研究不同的微调策略（Exp-I: Investigating Different Fine-Tuning Strategies）

在本小节中，我们使用 IMDb 数据集研究不同的微调策略。官方预训练模型被设为初始编码器。

#### 5.3.1 处理长文本（Dealing with long texts）

BERT 的最大序列长度为 512。将 BERT 应用于文本分类的第一个问题是如何处理长度大于 512 的文本。我们尝试了以下处理长文章的方法。

**截断方法** 通常，文章的关键信息位于开头和结尾。我们使用三种不同的截断方法进行 BERT 微调：

1. head-only：保留前 510 个 token；
2. tail-only：保留最后 510 个 token；
3. head+tail：根据经验选择前 128 个和后 382 个 token。

**分层方法** 输入文本首先被分成 k = L/510 个片段，依次送入 BERT 以获得 k 个文本片段的表示。每个片段的表示取最后一层 [CLS] token 的隐状态。然后我们使用均值池化（mean pooling）、最大池化（max pooling）和自注意力（self-attention）来组合所有片段的表示。

| 方法 | IMDb | Sogou |
|---|---|---|
| head-only | 5.63 | 2.58 |
| tail-only | 5.44 | 3.17 |
| head+tail | 5.42 | 2.43 |
| hier. mean | 5.89 | 2.83 |
| hier. max | 5.71 | 2.47 |
| hier. self-attention | 5.49 | 2.65 |

表 2：IMDb 和中文搜狗新闻数据集的测试错误率（%）。

表 2 展示了上述方法的有效性。head+tail 截断方法在 IMDb 和搜狗数据集上取得了最好的性能。因此，我们在后续实验中使用该方法处理长文本。

#### 5.3.2 来自不同层的特征（Features from Different layers）

BERT 的每一层捕捉输入文本的不同特征。我们研究了来自不同层特征的有效性，然后微调模型并记录测试错误率上的表现。

| 层 | 测试错误率(%) |
|---|---|
| Layer-0 | 11.07 |
| Layer-1 | 9.81 |
| Layer-2 | 9.29 |
| Layer-3 | 8.66 |
| Layer-4 | 7.83 |
| Layer-5 | 6.83 |
| Layer-6 | 6.83 |
| Layer-7 | 6.41 |
| Layer-8 | 6.04 |
| Layer-9 | 5.70 |
| Layer-10 | 5.46 |
| Layer-11 | 5.42 |
| First 4 Layers + concat | 8.69 |
| First 4 Layers + mean | 9.09 |
| First 4 Layers + max | 8.76 |
| Last 4 Layers + concat | 5.43 |
| Last 4 Layers + mean | 5.44 |
| Last 4 Layers + max | 5.42 |
| All 12 Layers + concat | 5.44 |

表 3：在 IMDb 数据集上使用不同层微调 BERT。

表 3 展示了用不同层微调 BERT 的性能。来自 BERT 最后一层的特征表现最好。因此，我们在后续实验中使用该设置。

#### 5.3.3 灾难性遗忘（Catastrophic Forgetting）

灾难性遗忘（McCloskey and Cohen, 1989）通常是迁移学习中的常见问题，指在学习新知识的过程中，预训练得到的知识被抹去。因此，我们也研究了 BERT 是否受灾难性遗忘问题的影响。

我们用不同的学习率微调 BERT，IMDb 上错误率的学习曲线如图 2 所示。

![图 2：灾难性遗忘]()

我们发现，较低的学习率（如 2e-5）是让 BERT 克服灾难性遗忘问题的必要条件。当使用激进的学习率 4e-4 时，训练集无法收敛。

#### 5.3.4 分层递减学习率（Layer-wise Decreasing Layer Rate）

表 4 展示了不同基础学习率和衰减因子（见公式 (2)）在 IMDb 数据集上的性能。我们发现，给较低层分配较低的学习率对微调 BERT 是有效的，合适的设置是 ξ=0.95，lr=2.0e-5。

| 学习率 | 衰减因子 ξ | 测试错误率(%) |
|---|---|---|
| 2.5e-5 | 1.00 | 5.52 |
| 2.5e-5 | 0.95 | 5.46 |
| 2.5e-5 | 0.90 | 5.44 |
| 2.5e-5 | 0.85 | 5.58 |
| 2.0e-5 | 1.00 | 5.42 |
| 2.0e-5 | 0.95 | 5.40 |
| 2.0e-5 | 0.90 | 5.52 |
| 2.0e-5 | 0.85 | 5.65 |

表 4：分层递减学习率。

### 5.4 实验二：研究进一步预训练（Exp-II: Investigating the Further Pretraining）

除了用监督学习微调 BERT 之外，我们还可以用无监督的掩码语言模型和下一句预测任务在训练数据上进一步预训练 BERT。本节研究进一步预训练的有效性。在以下实验中，微调阶段我们使用实验一中得到的最佳策略。

#### 5.4.1 任务内进一步预训练（Within-Task Further Pre-Training）

因此，我们首先研究任务内进一步预训练的有效性。我们使用不同步数进一步预训练的模型，然后用文本分类任务微调它们。

如图 3 所示，进一步预训练有助于提升 BERT 在目标任务上的性能，在 100K 训练步数后达到最佳性能。

![图 3：在 IMDb 数据集上不同进一步预训练步数的收益。BERT-ITPT-FiT 表示 "BERT + 任务内预训练 + 微调"。]()

#### 5.4.2 领域内与跨领域进一步预训练（In-Domain and Cross-Domain Further Pre-Training）

除了目标任务的训练数据之外，我们还可以在同领域的数据上进一步预训练 BERT。本小节研究用领域内和跨领域数据进一步预训练 BERT 是否能继续提升性能。

我们将七个英文数据集划分为三个领域：主题（topic）、情感（sentiment）和问题（question）。这种划分方式并不严格正确，因此我们还进行了大量的跨任务预训练实验，将每个任务视为一个不同的领域。

结果如表 5 所示。我们发现，几乎所有进一步预训练模型在全部七个数据集上都优于原始 BERT-base 模型（表 5 中 "w/o pretrain" 行）。总体而言，领域内预训练通常比任务内预训练带来更好的性能。在规模较小的句子级 TREC 数据集上，任务内预训练对性能有害，而利用 Yah. A. 语料的领域内预训练可以在 TREC 上取得更好的结果。

| 领域 | 数据集 | IMDb | Yelp P. | Yelp F. | TREC | Yah. A. | AG's News | DBPedia |
|---|---|---|---|---|---|---|---|---|
| sentiment | IMDb | 4.37 | 2.18 | 29.60 | 2.60 | 22.39 | 5.24 | 0.68 |
| sentiment | Yelp P. | 5.24 | 1.92 | 29.37 | 2.00 | 22.38 | 5.14 | 0.65 |
| sentiment | Yelp F. | 5.18 | 1.94 | 29.42 | 2.40 | 22.33 | 5.43 | 0.65 |
| sentiment | all sentiment | 4.88 | 1.87 | 29.25 | 3.00 | 22.35 | 5.34 | 0.67 |
| question | TREC | 5.65 | 2.09 | 29.35 | 3.20 | 22.17 | 5.12 | 0.66 |
| question | Yah. A. | 5.52 | 2.08 | 29.31 | 1.80 | 22.38 | 5.16 | 0.67 |
| question | all question | 5.68 | 2.14 | 29.52 | 2.20 | 21.86 | 5.21 | 0.68 |
| topic | AG's News | 5.97 | 2.15 | 29.38 | 2.00 | 22.32 | 4.80 | 0.68 |
| topic | DBPedia | 5.80 | 2.13 | 29.47 | 2.60 | 22.30 | 5.13 | 0.68 |
| topic | all topic | 5.85 | 2.20 | 29.68 | 2.60 | 22.28 | 4.88 | 0.65 |
| - | all | 5.18 | 1.97 | 29.20 | 2.80 | 21.94 | 5.08 | 0.67 |
| - | w/o pretrain | 5.40 | 2.28 | 30.06 | 2.80 | 22.42 | 5.25 | 0.71 |

表 5：七个数据集上领域内和跨领域进一步预训练的性能。每个模型都进一步预训练了 100k 步。第一列表示不同的进一步预训练数据集。"all sentiment" 表示该数据集由情感领域的所有训练数据集组成。"all" 表示该数据集由全部七个训练数据集组成。注意 Yelp P. 和 Yelp F. 中的部分数据存在重叠，例如 Yelp P. 测试集中的部分数据会出现在 Yelp F. 的训练集中，因此我们在进一步预训练时从训练集中移除了这部分数据。

跨领域预训练（表 5 中的 "all" 行）总体上没有带来明显收益。这是合理的，因为 BERT 已经在通用领域上训练过。

我们还发现，在情感领域，IMDb 和 Yelp 并没有互相帮助。原因可能是 IMDb 和 Yelp 分别是电影和美食两类情感任务，数据分布存在显著差异。

#### 5.4.3 与以往模型的比较（Comparisons to Previous Models）

我们将我们的模型与多种不同方法进行比较：基于 CNN 的方法，如字符级 CNN（Char-level CNN）（Zhang et al., 2015）、VDCNN（Conneau et al., 2016）和 DPCNN（Johnson and Zhang, 2017）；基于 RNN 的模型，如 D-LSTM（Yogatama et al., 2017）、Skim-LSTM（Seo et al., 2017）和分层注意力网络（hierarchical attention networks）（Yang et al., 2016）；基于特征的迁移学习方法，如 region embedding（Qiao et al., 2018）和 CoVe（McCann et al., 2017）；以及语言模型微调方法（ULMFiT）（Howard and Ruder, 2018），它是当前文本分类的最先进方法。

我们通过将 BERT 模型的特征作为带自注意力的 biLSTM（Lin et al., 2017）的输入嵌入来实现 BERT-Feat。BERT-IDPT-FiT 的结果对应于表 5 中的 "all sentiment"、"all question" 和 "all topic" 行，BERT-CDPT-FiT 的结果对应于其中的 "all" 行。

如表 6 所示，BERT-Feat 的性能优于除 ULMFiT 之外的所有其他基线。除了在 DBpedia 数据集上略逊于 BERT-Feat 之外，BERT-FiT 在其他七个数据集上都优于 BERT-Feat。此外，三种进一步预训练模型都优于 BERT-FiT 模型。以 BERT-Feat 为参照，我们计算了其他 BERT-FiT 模型在每个数据集上的平均提升百分比。BERT-IDPT-FiT 表现最好，平均错误率降低了 18.57%。

| 模型 | IMDb | Yelp P. | Yelp F. | TREC | Yah. A. | AG | DBP | Sogou | Avg. ∆ |
|---|---|---|---|---|---|---|---|---|---|
| Char-level CNN(Zhang et al., 2015) | / | 4.88 | 37.95 | / | 28.80 | 9.51 | 1.55 | 3.80 | / |
| VDCNN (Conneau et al., 2016) | / | 4.28 | 35.28 | / | 26.57 | 8.67 | 1.29 | 3.28 | / |
| DPCNN (Johnson and Zhang, 2017) | / | 2.64 | 30.58 | / | 23.90 | 6.87 | 0.88 | 3.48∗ | / |
| D-LSTM (Yogatama et al., 2017) | / | 7.40 | 40.40 | / | 26.30 | 7.90 | 1.30 | 5.10 | / |
| Standard LSTM (Seo et al., 2017) | 8.90 | / | / | / | / | 6.50 | / | / | / |
| Skim-LSTM (Seo et al., 2017) | 8.80 | / | / | / | / | 6.40 | / | / | / |
| HAN (Yang et al., 2016) | / | / | / | / | 24.20 | / | / | / | / |
| Region Emb. (Qiao et al., 2018) | / | 3.60 | 35.10 | / | 26.30 | 7.20 | 1.10 | 2.40 | / |
| CoVe (McCann et al., 2017) | 8.20 | / | / | 4.20 | / | / | / | / | / |
| ULMFiT (Howard and Ruder, 2018) | 4.60 | 2.16 | 29.98 | 3.60 | / | 5.01 | 0.80 | / | / |
| BERT-Feat | 6.79 | 2.39 | 30.47 | 4.20 | 22.72 | 5.92 | 0.70 | 2.50 | - |
| BERT-FiT | 5.40 | 2.28 | 30.06 | 2.80 | 22.42 | 5.25 | 0.71 | 2.43 | 9.22% |
| BERT-ITPT-FiT | 4.37 | 1.92 | 29.42 | 3.20 | 22.38 | 4.80 | 0.68 | 1.93 | 16.07% |
| BERT-IDPT-FiT | 4.88 | 1.87 | 29.25 | 2.20 | 21.86 | 4.88 | 0.65 | / | 18.57% |
| BERT-CDPT-FiT | 5.18 | 1.97 | 29.20 | 2.80 | 21.94 | 5.08 | 0.67 | / | 14.38% |

表 6：八个文本分类数据集上的测试错误率（%）。此前模型中不带 ∗ 的结果是其论文中报告的结果。/ 表示未报告。∗ 表示来自我们实现的结果，因为搜狗数据集与他们不同。BERT-Feat 表示 "BERT 作为特征"。BERT-FiT 表示 "BERT + 微调"。BERT-ITPT-FiT 表示 "BERT + 任务内预训练 + 微调"。BERT-IDPT-FiT 表示 "BERT + 领域内预训练 + 微调"。BERT-CDPT-FiT 表示 "BERT + 跨领域预训练 + 微调"。

### 5.5 实验三：多任务微调（Exp-III: Multi-task Fine-tuning）

当文本分类任务有多个数据集时，为了充分利用这些可用数据，我们进一步考虑了带多任务学习的微调步骤。我们使用四个英文文本分类数据集（IMDb、Yelp P.、AG 和 DBP）。由于 Yelp F. 的测试集与 Yelp P. 的训练集存在重叠，我们排除了 Yelp F. 数据集，同时也排除了问题领域的两个数据集。

我们分别使用官方 uncased BERT-base 权重和在全部七个英文分类数据集上进一步预训练的权重进行实验。为了在每个子任务上取得更好的分类结果，在联合微调之后，我们用较低的学习率在各自的数据集上额外微调几步。

表 7 显示，基于 BERT 的多任务微调带来了性能提升。然而，多任务微调对 Yelp P. 和 AG 上的 BERT-CDPT 似乎没有帮助。多任务微调与跨领域预训练可能是可以互相替代的方法，因为 BERT-CDPT 模型已经包含了丰富的领域特定信息，多任务学习对于提升相关文本分类子任务的泛化能力可能并非必要。

| 方法 | IMDb | Yelp P. | AG | DBP |
|---|---|---|---|---|
| BERT-FiT | 5.40 | 2.28 | 5.25 | 0.71 |
| BERT-MFiT-FiT | 5.36 | 2.19 | 5.20 | 0.68 |
| BERT-CDPT-FiT | 5.18 | 1.97 | 5.08 | 0.67 |
| BERT-CDPT-MFiT-FiT | 4.96 | 2.06 | 5.13 | 0.67 |

表 7：带多任务微调的测试错误率（%）。

### 5.6 实验四：少样本学习（Exp-IV: Few-Shot Learning）

预训练模型的好处之一是能够用很少的训练数据为下游任务训练模型。我们在不同数量的训练样本上评估 BERT-FiT 和 BERT-ITPT-FiT。我们从 IMDb 训练数据中选取子集，分别输入 BERT-FiT 和 BERT-ITPT-FiT。结果显示在图 4 中。

![图 4：IMDb 数据集上不同训练样本比例下的测试错误率(%)。]()

该实验结果表明，BERT 给小型数据带来了显著的性能提升。进一步预训练的 BERT 可以进一步提升性能，仅用 0.4% 的训练数据就能将错误率从 17.26% 降到 9.23%。

### 5.7 实验五：在大规模 BERT 上进一步预训练（Exp-V: Further Pre-Training on BERT Large）

本小节研究 BERTLARGE 模型是否与 BERTBASE 有类似的发现。我们在 1 块 Tesla-V100-PCIE 32G GPU 上进一步预训练 Google 的预训练 BERTLARGE 模型，批大小为 24，最大序列长度为 128，训练 120K 步。对于目标任务分类器的 BERT 微调，我们将批大小设为 24，在 4 块 Tesla-V100-PCIE 32G GPU 上微调 BERTLARGE，最大序列长度为 512。

如表 8 所示，ULMFiT 在几乎所有任务上都优于 BERTBASE，但不如 BERTLARGE。然而，在任务特定的进一步预训练之后，情况发生了变化：即使 BERTBASE 也在所有任务上超过了 ULMFiT。带有任务特定进一步预训练的 BERTLARGE 微调取得了最佳结果。

| 模型 | IMDb | Yelp P. | Yelp F. | AG | DBP |
|---|---|---|---|---|---|
| ULMFiT | 4.60 | 2.16 | 29.98 | 5.01 | 0.80 |
| BERTBASE | 5.40 | 2.28 | 30.06 | 5.25 | 0.71 |
| + ITPT | 4.37 | 1.92 | 29.42 | 4.80 | 0.68 |
| BERTLARGE | 4.86 | 2.04 | 29.25 | 4.86 | 0.62 |
| + ITPT | 4.21 | 1.81 | 28.62 | 4.66 | 0.61 |

表 8：五个文本分类数据集上的测试错误率（%）。

## 6. 结论（Conclusion）

本文通过大量实验研究了针对文本分类任务微调 BERT 的不同方法。我们得到以下实验发现：1）BERT 的顶层对文本分类更有用；2）在合适的分层递减学习率下，BERT 能够克服灾难性遗忘问题；3）任务内和领域内进一步预训练能显著提升其性能；4）先进行多任务微调对单任务微调也有帮助，但其收益小于进一步预训练；5）BERT 能够提升小规模数据上的任务表现。

基于以上发现，我们在八个广泛研究的文本分类数据集上取得了最佳性能。未来，我们将进一步探究 BERT 的工作机理。

参考文献略
