# 论文库索引

## 主题分类

| 主题 | 中文名 | 论文数 |
|---|---|---|
| storage-systems | 存储系统 | 9 |
| database-systems | 数据库系统 | 5 |
| distributed-systems | 分布式系统 | 10 |
| ml-systems | 机器学习系统 | 1 |
| time-scaling-theory | 时间缩放理论 | 1 |
| llm | 大语言模型 | 4 |
| ai-applications | AI 应用 | 2 |
| math-foundations | 数学与算法基础 | 9 |
| └─ dengyu | 邓煜团队系列 | 4 |
| └─ wanghong | 王虹团队系列 | 2 |
| information-retrieval | 信息检索 | 1 |
| research-methodology | 科研方法论 | 1 |
| cognitive-neuroscience | 认知神经科学 | 1 |
| ai-agents | AI 智能体 | 1 |
| classic-ai | AI 经典论文 | 18 |

## 论文索引

| 主题 | 文件夹 | 标题 | 发布时间 | 翻译日期 | 一句话总结 |
|---|---|---|---|---|---|
| storage-systems | span-db | SpanDB: A Fast, Cost-Effective LSM-tree Based KV Store on Hybrid Storage | 2021-02 | 2026-07-18 | 用一块小而贵的高速SSD专写日志和存热数据，大幅提升KV存储性能且成本可控 |
| storage-systems | diffkv-balanced-io | Differentiated Key-Value Storage Management for Balanced I/O Performance | 2021-07 | 2026-07-18 | 让键值存储的"钥匙"和"抽屉里的东西"分别按不同规则整理，读写扫描三不误 |
| storage-systems | frozenhot-cache-rethinking-cache-management | FrozenHot Cache: Rethinking Cache Management for Modern Hardware | 2023-05 | 2026-07-18 | 把热门数据"冻"起来不加锁直接读，大幅提升高并发下缓存吞吐量 |
| storage-systems | cfs-scaling-metadata-service | CFS: Scaling Metadata Service for Distributed File System via Pruned Scope of Critical Sections | 2023-05 | 2026-07-18 | 把文件元数据拆成"目录结构"和"文件属性"两层分别管，砍掉锁的范围让并发飞起来 |
| storage-systems | scaling-memcache-facebook | Scaling Memcache at Facebook | 2013 | 2026-07-19 | Facebook 把简单的缓存工具改造成全球最大分布式缓存，扛住每秒十亿次请求 |
| database-systems | tao-facebook-distributed-data-store | TAO: Facebook's Distributed Data Store for the Social Graph | 2013 | 2026-07-18 | Facebook 自研的读优化图数据库，每秒支撑十亿次读取 |
| database-systems | what-goes-around-2024 | What Goes Around Comes Around... And Around... | 2024 | 2026-07-18 | SQL和关系模型不断吸收挑战者优点，每次被宣判死刑都活了下来，未来也一样 |
| distributed-systems | heracles-resource-efficiency | Heracles: Improving Resource Efficiency at Scale | 2015 | 2026-07-18 | 让延迟敏感服务和批处理任务安全共享服务器，将平均利用率提升到 90% |
| distributed-systems | declarative-imperative-distributed-logic | The Declarative Imperative: Experiences and Conjectures in Distributed Logic | 2013 | 2026-07-18 | 用数据库查询语言 Datalog 的思路来简化并行和分布式编程 |
| distributed-systems | minflow | MinFlow: High-performance and Cost-efficient Data Passing for I/O-intensive Stateful Serverless Analytics | 2024-02 | 2026-07-18 | 用多级中转+交错调度让一半shuffle流量走本地内存，速度提10倍、成本砍98% |
| time-scaling-theory | time-scaling-theory | A Time Scaling Theory for Multi-Layer Electronic Systems | 2026-05 | 2026-07-19 | 不再靠缩小晶体管，而是用压缩时间作为统一标尺来推动芯片进步 |
| ml-systems | tensorflow-large-scale-machine-learning | TensorFlow: Large-Scale Machine Learning on Heterogeneous Distributed Systems | 2015 | 2026-07-18 | 用数据流图统一从手机到超大规模集群的深度学习模型训练和部署 |
| llm | exploring-llm-intelligent-agents | Exploring Large Language Model Based Intelligent Agents: Definitions, Methods, and Prospects | 2024-01 | 2026-07-18 | 综述了用大语言模型构建智能体的方法与前景 |
| llm | llm-augmented-llms-composition | LLM Augmented LLMs: Expanding Capabilities Through Composition | 2024-01 | 2026-07-18 | 将多个专用小模型组合到大模型上实现新能力，无需重新训练 |
| llm | understanding-llms-training-inference | Understanding LLMs: A Comprehensive Overview from Training to Inference | 2024-01 | 2026-07-18 | 系统梳理大语言模型从训练到推理的关键技术和未来方向 |
| llm | longhorizon-harness | LongHorizon-Harness: Advancing Long-Horizon Agents for Real-World Tasks | 2026-08 | 2026-08-08 | 把长任务拆成"规划-执行-验收"小循环、只信外部验证过的事实，长时程任务成功率大幅提升 |
| ai-applications | healthcare-voice-ai-assistants-trust | Healthcare Voice AI Assistants: Factors Influencing Trust and Intention to Use | 2024-01 | 2026-07-18 | 什么东西决定我们信不信任"AI医生"？ |
| ai-applications | spoken-sign-language-translation-3d-avatars | A Simple Baseline for Spoken Language to Sign Language Translation with 3D Avatars | 2024-01 | 2026-07-18 | 首个用3D虚拟角色将口语实时翻译成手语的系统 |
| dengyu | math-foundations/dengyu/boltzmann-equation-hard-sphere-long-time | Long Time Derivation of the Boltzmann Equation from Hard Sphere Dynamics | 2024-08 | 2026-07-19 | 用数学严格证明了从弹来弹去的硬球微粒可在任意长时间导出玻尔兹曼方程 |
| dengyu | math-foundations/dengyu/hilbert-sixth-problem-fluid-equations | Hilbert's Sixth Problem: Derivation of Fluid Equations via Boltzmann's Kinetic Theory | 2025-03 | 2026-07-19 | 从硬球粒子严格推导出流体力学方程，完成了希尔伯特第六问题的完整解答 |
| dengyu | math-foundations/dengyu/wave-kinetic-equation-full-derivation | Full Derivation of the Wave Kinetic Equation | 2021-04 | 2026-07-19 | 从非线性薛定谔方程严格推导出波动力学方程，波湍流理论的核心猜想获证 |
| dengyu | math-foundations/dengyu/propagation-chaos-wave-kinetic-theory | Propagation of Chaos and the Higher Order Statistics in the Wave Kinetic Theory | 2021-10 | 2026-07-19 | 证明波系统中初始独立的波动模式在演化中始终保持统计独立 |
| wanghong | math-foundations/wanghong/kakeya-set-conjecture-three-dimensions | Volume Estimates for Unions of Convex Sets, and the Kakeya Set Conjecture in Three Dimensions | 2025-02 | 2026-07-19 | 证明三维 Kakeya 集必须具有满维数 3，解决了近百年未决的几何猜想 |
| wanghong | math-foundations/wanghong/furstenberg-sets-estimate-plane | Furstenberg Sets Estimate in the Plane | 2023-08 | 2026-07-19 | 完全解决了二维 Furstenberg 集猜想，给出分形维数的最优下界 |
| math-foundations | art-of-linear-algebra | The Art of Linear Algebra – Graphic Notes on "Linear Algebra for Everyone" | 2021-09 | 2026-07-18 | 用图画把矩阵乘法和五种分解画出来，看一眼就记住 |
| math-foundations | skiplists-probabilistic-alternative-balanced-trees | Skip Lists: A Probabilistic Alternative to Balanced Trees | 1990 | 2026-07-18 | 用抛硬币代替复杂旋转操作的有序链表，实现简单但性能媲美平衡树 |
| storage-systems | anna-kvs-any-scale | Anna: A KVS For Any Scale | 2018 | 2026-07-19 | 用格（Lattice）和消息传递实现跨任意规模的高性能键值存储 |
| storage-systems | polarfs-low-latency | PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System | 2018 | 2026-07-19 | 给云数据库造了一个又快又稳的网络硬盘，延迟跟本地硬盘差不多 |
| storage-systems | google-bigtable | Google Bigtable（中文译本） | 2006 | 2026-07-19 | Bigtable 是 Google 设计的分布式结构化数据存储系统 |
| storage-systems | google-file-system | Google File System（中文译本） | 2003 | 2026-07-19 | GFS 是一个为大规模数据密集型应用设计的可伸缩分布式文件系统 |
| database-systems | spanner-globally-distributed-database | Spanner: Google's Globally-Distributed Database | 2012 | 2026-07-19 | 让全球各地的数据库像一台机器一样保持数据一致，不怕地震也不怕断网 |
| database-systems | druid-real-time-analytical-store | Druid: A Real-time Analytical Data Store | 2014 | 2026-07-19 | Druid 是一个分布式、面向列、支持实时数据摄入的 OLAP 数据存储系统 |
| database-systems | coordination-avoidance-database-systems | Coordination Avoidance in Database Systems | 2015 | 2026-07-19 | 提出不变式合流框架，为数据库确定何时可完全避免协调提供了充要条件 |
| distributed-systems | dapper-distributed-tracing | Dapper, a Large-Scale Distributed Systems Tracing Infrastructure | 2010 | 2026-07-19 | 谷歌用极低开销的采样追踪技术，让海量微服务的行为"尽收眼底" |
| distributed-systems | dremel-interactive-analysis | Dremel: Interactive Analysis of Web-Scale Datasets | 2010 | 2026-07-19 | Google 用列式存储+多级服务树实现万亿级数据的秒级交互式查询 |
| distributed-systems | percolator-incremental-processing | Large-scale Incremental Processing Using Distributed Transactions and Notifications | 2010 | 2026-07-19 | 用分布式事务实现PB级数据的增量更新，索引延迟降低100倍 |
| distributed-systems | pregel-graph-processing | Pregel: A System for Large-Scale Graph Processing | 2010 | 2026-07-19 | 像顶点一样思考，用超步同步+消息传递来分布式处理百亿级图 |
| distributed-systems | availability-globally-distributed-storage | Availability in Globally Distributed Storage Systems | 2010 | 2026-07-19 | Google用一年数据证明：节点关联故障才是云存储不可用的元凶 |
| distributed-systems | blazes-coordination-analysis | Blazes: Coordination Analysis for Distributed Programs | 2013 | 2026-07-19 | 自动分析分布式程序中哪里需要协调，并自动生成最省事的协调代码 |
| distributed-systems | google-mapreduce | Google MapReduce（中文译本） | 2004 | 2026-07-19 | Google设计的简单编程模型，让程序员无需关心分布式细节即可自动处理TB级数据 |
| math-foundations | mathematical-theory-communication | A Mathematical Theory of Communication（中文译本） | 1948 | 2026-07-19 | 香农定义熵，推导出无失真压缩和噪声信道下可靠通信的理论极限 |
| information-retrieval | anatomy-web-search-engine | The Anatomy of a Large-Scale Hypertextual Web Search Engine | 1998 | 2026-07-19 | 利用链接分析（PageRank）实现高质量网络搜索 |
| research-methodology | how-to-read-paper | How to Read a Paper | 2016 | 2026-07-19 | 三遍阅读法让你高效吃透一篇论文 |
| cognitive-neuroscience | compressive-learning-higher-order-network | Compressive learning scaffolds higher-order network structure to enhance human knowledge acquisition | 2026-07 | 2026-08-11 | 先学知识网络的"枢纽骨架"再学细节，学习更快，脑机制在背侧前扣带回 |
| ai-agents | self-evolving-coding-agents | Self-Evolving Coding Agents | 2026-08 | 2026-08-12 | 综述梳理"会自我进化的编程智能体"：5类进化对象×3种时机×3类证据的分类框架 |
| classic-ai | 01-word2vec | 01-word2vec：Efficient Estimation of Word Representations in Vector Space | 2013 | 2026-08-14 | 分布式词表示起点，Skip-gram 让词义向量化，NLP 表示学习的开端 |
| classic-ai | 02-seq2seq | 02-Seq2Seq：Sequence to Sequence Learning with Neural Networks | 2014 | 2026-08-14 | 用两个 LSTM 把任意长度序列映射为序列，机器翻译的端到端范式 |
| classic-ai | 03-attention | 03-Attention：Neural Machine Translation by Jointly Learning to Align and Translate | 2014 | 2026-08-14 | 引入注意力机制，解码时动态对齐输入，解决长句翻译瓶颈 |
| classic-ai | 04-transformer | 04-Transformer：Attention Is All You Need | 2017 | 2026-08-14 | 纯注意力架构取代 RNN/CNN，现代 LLM 的地基 |
| classic-ai | 05-bert | 05-BERT：Pre-training of Deep Bidirectional Transformers for Language Understanding | 2018 | 2026-08-14 | 掩码语言建模+双向 Transformer，预训练-微调范式奠基之作 |
| classic-ai | 06-bert-fine-tuning | 06-BERT 微调：How to Fine-Tune BERT for Text Classification？ | 2019 | 2026-08-14 | 系统实验 BERT 文本分类微调策略，提出"继续预训练+多任务微调+目标任务微调"三步法 |
| classic-ai | 07-gpt3 | 07-GPT-3：Language Models are Few-Shot Learners | 2020 | 2026-08-14 | 千亿参数规模定律，上下文学习（In-Context Learning）诞生 |
| classic-ai | 08-rlhf | 08-RLHF：Training Language Models to Follow Instructions with Human Feedback | 2022 | 2026-08-14 | 用人类反馈强化学习让模型对齐指令，ChatGPT 的技术源头 |
| classic-ai | 09-gan | 09-GAN：Generative Adversarial Networks | 2014 | 2026-08-14 | 生成器与判别器对抗博弈，生成模型的里程碑 |
| classic-ai | 10-ddpm | 10-DDPM：Denoising Diffusion Probabilistic Models | 2020 | 2026-08-14 | 扩散模型奠基，逐步去噪生成，Stable Diffusion 等的基础 |
| classic-ai | 11-dqn | 11-DQN：Playing Atari with Deep Reinforcement Learning | 2013 | 2026-08-14 | 深度 Q 网络，从像素直接学习玩游戏，深度强化学习起点 |
| classic-ai | 12-dropout | 12-Dropout：Improving Neural Networks by Preventing Co-Adaptation | 2014 | 2026-08-14 | 随机丢弃神经元抑制过拟合，最常用的正则化技巧 |
| classic-ai | 13-adam | 13-Adam：A Method for Stochastic Optimization | 2015 | 2026-08-14 | 自适应学习率优化器，训练神经网络的默认选择 |
| classic-ai | 14-batchnorm | 14-BatchNorm：Batch Normalization | 2015 | 2026-08-14 | 归一化层内激活稳定训练，让深层网络可以训练 |
| classic-ai | 15-resnet | 15-ResNet：Deep Residual Learning for Image Recognition | 2015 | 2026-08-14 | 残差连接让百层网络可训练，CV 架构设计基石 |
| classic-ai | 16-unet | 16-U-Net：Convolutional Networks for Biomedical Image Segmentation | 2015 | 2026-08-14 | 编码-解码+跳跃连接，图像分割经典架构 |
| classic-ai | 17-yolo | 17-YOLO：You Only Look Once | 2016 | 2026-08-14 | 单阶段实时目标检测，速度与精度的平衡 |
| classic-ai | 18-mask-rcnn | 18-Mask R-CNN | 2017 | 2026-08-14 | 检测+分割统一框架，实例分割的经典方法 |
