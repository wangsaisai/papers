# 论文库索引

## 主题分类

| 主题 | 中文名 | 论文数 |
|---|---|---|
| storage-systems | 存储系统 | 4 |
| database-systems | 数据库系统 | 2 |
| distributed-systems | 分布式系统 | 3 |
| ml-systems | 机器学习系统 | 1 |
| llm | 大语言模型 | 3 |
| ai-applications | AI 应用 | 2 |
| math-foundations | 数学与算法基础 | 2 |

## 论文索引

| 主题 | 文件夹 | 标题 | 发布时间 | 翻译日期 | 一句话总结 |
|---|---|---|---|---|---|
| storage-systems | span-db | SpanDB: A Fast, Cost-Effective LSM-tree Based KV Store on Hybrid Storage | 2021-02 | 2026-07-18 | 用一块小而贵的高速SSD专写日志和存热数据，大幅提升KV存储性能且成本可控 |
| storage-systems | diffkv-balanced-io | Differentiated Key-Value Storage Management for Balanced I/O Performance | 2021-07 | 2026-07-18 | 让键值存储的"钥匙"和"抽屉里的东西"分别按不同规则整理，读写扫描三不误 |
| storage-systems | frozenhot-cache-rethinking-cache-management | FrozenHot Cache: Rethinking Cache Management for Modern Hardware | 2023-05 | 2026-07-18 | 把热门数据"冻"起来不加锁直接读，大幅提升高并发下缓存吞吐量 |
| storage-systems | cfs-scaling-metadata-service | CFS: Scaling Metadata Service for Distributed File System via Pruned Scope of Critical Sections | 2023-05 | 2026-07-18 | 把文件元数据拆成"目录结构"和"文件属性"两层分别管，砍掉锁的范围让并发飞起来 |
| database-systems | tao-facebook-distributed-data-store | TAO: Facebook's Distributed Data Store for the Social Graph | 2013 | 2026-07-18 | Facebook 自研的读优化图数据库，每秒支撑十亿次读取 |
| database-systems | what-goes-around-2024 | What Goes Around Comes Around... And Around... | 2024 | 2026-07-18 | SQL和关系模型不断吸收挑战者优点，每次被宣判死刑都活了下来，未来也一样 |
| distributed-systems | heracles-resource-efficiency | Heracles: Improving Resource Efficiency at Scale | 2015 | 2026-07-18 | 让延迟敏感服务和批处理任务安全共享服务器，将平均利用率提升到 90% |
| distributed-systems | declarative-imperative-distributed-logic | The Declarative Imperative: Experiences and Conjectures in Distributed Logic | 2013 | 2026-07-18 | 用数据库查询语言 Datalog 的思路来简化并行和分布式编程 |
| distributed-systems | minflow | MinFlow: High-performance and Cost-efficient Data Passing for I/O-intensive Stateful Serverless Analytics | 2024-02 | 2026-07-18 | 用多级中转+交错调度让一半shuffle流量走本地内存，速度提10倍、成本砍98% |
| ml-systems | tensorflow-large-scale-machine-learning | TensorFlow: Large-Scale Machine Learning on Heterogeneous Distributed Systems | 2015 | 2026-07-18 | 用数据流图统一从手机到超大规模集群的深度学习模型训练和部署 |
| llm | exploring-llm-intelligent-agents | Exploring Large Language Model Based Intelligent Agents: Definitions, Methods, and Prospects | 2024-01 | 2026-07-18 | 综述了用大语言模型构建智能体的方法与前景 |
| llm | llm-augmented-llms-composition | LLM Augmented LLMs: Expanding Capabilities Through Composition | 2024-01 | 2026-07-18 | 将多个专用小模型组合到大模型上实现新能力，无需重新训练 |
| llm | understanding-llms-training-inference | Understanding LLMs: A Comprehensive Overview from Training to Inference | 2024-01 | 2026-07-18 | 系统梳理大语言模型从训练到推理的关键技术和未来方向 |
| ai-applications | healthcare-voice-ai-assistants-trust | Healthcare Voice AI Assistants: Factors Influencing Trust and Intention to Use | 2024-01 | 2026-07-18 | 什么东西决定我们信不信任"AI医生"？ |
| ai-applications | spoken-sign-language-translation-3d-avatars | A Simple Baseline for Spoken Language to Sign Language Translation with 3D Avatars | 2024-01 | 2026-07-18 | 首个用3D虚拟角色将口语实时翻译成手语的系统 |
| math-foundations | art-of-linear-algebra | The Art of Linear Algebra – Graphic Notes on "Linear Algebra for Everyone" | 2021-09 | 2026-07-18 | 用图画把矩阵乘法和五种分解画出来，看一眼就记住 |
| math-foundations | skiplists-probabilistic-alternative-balanced-trees | Skip Lists: A Probabilistic Alternative to Balanced Trees | 1990 | 2026-07-18 | 用抛硬币代替复杂旋转操作的有序链表，实现简单但性能媲美平衡树 |
