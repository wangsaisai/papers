# 论文库索引

| 文件夹 | 标题 | 发布时间 | 翻译日期 | 一句话总结 |
|---|---|---|---|---|
| frozenhot-cache-rethinking-cache-management | FrozenHot Cache: Rethinking Cache Management for Modern Hardware | 2023-05 | 2026-07-18 | 把热门数据"冻"起来不加锁直接读，大幅提升高并发下缓存吞吐量 |
| minflow | MinFlow: High-performance and Cost-efficient Data Passing for I/O-intensive Stateful Serverless Analytics | 2024-02 | 2026-07-18 | 用多级中转+交错调度让一半shuffle流量走本地内存，速度提10倍、成本砍98% |
| span-db | SpanDB: A Fast, Cost-Effective LSM-tree Based KV Store on Hybrid Storage | 2021-02 | 2026-07-18 | 用一块小而贵的高速SSD专写日志和存热数据，大幅提升KV存储性能且成本可控 |
| diffkv-balanced-io | Differentiated Key-Value Storage Management for Balanced I/O Performance | 2021-07 | 2026-07-18 | 让键值存储的"钥匙"和"抽屉里的东西"分别按不同规则整理，读写扫描三不误 |
| cfs-scaling-metadata-service | CFS: Scaling Metadata Service for Distributed File System via Pruned Scope of Critical Sections | 2023-05 | 2026-07-18 | 把文件元数据拆成"目录结构"和"文件属性"两层分别管，砍掉锁的范围让并发飞起来 |
| art-of-linear-algebra | The Art of Linear Algebra – Graphic Notes on "Linear Algebra for Everyone" | 2021-09 | 2026-07-18 | 用图画把矩阵乘法和五种分解画出来，看一眼就记住 |
| what-goes-around-2024 | What Goes Around Comes Around... And Around... | 2024 | 2026-07-18 | SQL和关系模型不断吸收挑战者优点，每次被宣判死刑都活了下来，未来也一样 |
