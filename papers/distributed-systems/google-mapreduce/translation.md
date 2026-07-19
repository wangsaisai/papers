# Google MapReduce（中文版）

**原文标题**: MapReduce: Simplified Data Processing on Large Clusters

**作者**: Jeffrey Dean and Sanjay Ghemawat

**说明**: 本论文原文为英文，存在一个流传较广的中文翻译版本（译者 alex）。本文档收录的是该中文译本，保留了原始论文的完整结构，并附有译者的一些注释。

---

## 摘要

MapReduce 是一个编程模型，也是一个处理和生成超大数据集的算法模型的相关实现。用户首先创建一个 Map 函数处理一个基于 key/value pair 的数据集合，输出中间的基于 key/value pair 的数据集合；然后再创建一个 Reduce 函数用来合并所有的具有相同中间 key 值的中间 value 值。现实世界中有很多满足上述处理模型的例子，本论文将详细描述这个模型。

MapReduce 架构的程序能够在大量的普通配置的计算机上实现并行化处理。这个系统在运行时只关心：如何分割输入数据，在大量计算机组成的集群上的调度，集群中计算机的错误处理，管理集群中计算机之间必要的通信。采用 MapReduce 架构可以使那些没有并行计算和分布式处理系统开发经验的程序员有效利用分布式系统的丰富资源。

我们的 MapReduce 实现运行在规模可以灵活调整的由普通机器组成的集群上：一个典型的 MapReduce 计算往往由几千台机器组成、处理以 TB 计算的数据。程序员发现这个系统非常好用：已经实现了数以百计的 MapReduce 程序，在 Google 的集群上，每天都有 1000 多个 MapReduce 程序在执行。

## 1. 介绍

在过去的 5 年里，包括本文作者在内的 Google 的很多程序员，为了处理海量的原始数据，已经实现了数以百计的、专用的计算方法。这些计算方法用来处理大量的原始数据，比如，文档抓取（类似网络爬虫的程序）、Web 请求日志等等；也为了计算处理各种类型的衍生数据，比如倒排索引、Web 文档的图结构的各种表示形势、每台主机上网络爬虫抓取的页面数量的汇总、每天被请求的最多的查询的集合等等。大多数这样的数据处理运算在概念上很容易理解。然而由于输入的数据量巨大，因此要想在可接受的时间内完成运算，只有将这些计算分布在成百上千的主机上。如何处理并行计算、如何分发数据、如何处理错误？所有这些问题综合在一起，需要大量的代码处理，因此也使得原本简单的运算变得难以处理。

为了解决上述复杂的问题，我们设计一个新的抽象模型，使用这个抽象模型，我们只要表述我们想要执行的简单运算即可，而不必关心并行计算、容错、数据分布、负载均衡等复杂的细节，这些问题都被封装在了一个库里面。设计这个抽象模型的灵感来自 Lisp 和许多其他函数式语言的 Map 和 Reduce 的原语。我们意识到我们大多数的运算都包含这样的操作：在输入数据的"逻辑"记录上应用 Map 操作得出一个中间 key/value pair 集合，然后在所有具有相同 key 值的 value 值上应用 Reduce 操作，从而达到合并中间的数据，得到一个想要的结果的目的。使用 MapReduce 模型，再结合用户实现的 Map 和 Reduce 函数，我们就可以非常容易的实现大规模并行化计算；通过 MapReduce 模型自带的"再次执行"（re-execution）功能，也提供了初级的容灾实现方案。

这个工作（实现一个 MapReduce 框架模型）的主要贡献是通过简单的接口来实现自动的并行化和大规模的分布式计算，通过使用 MapReduce 模型接口实现在大量普通的 PC 机上高性能计算。

第二部分描述基本的编程模型和一些使用案例。第三部分描述了一个经过裁剪的、适合我们的基于集群的计算环境的 MapReduce 实现。第四部分描述我们认为在 MapReduce 编程模型中一些实用的技巧。第五部分对于各种不同的任务，测量我们 MapReduce 实现的性能。第六部分揭示了在 Google 内部如何使用 MapReduce 作为基础重写我们的索引系统产品，包括其它一些使用 MapReduce 的经验。第七部分讨论相关的和未来的工作。

## 2. 编程模型

MapReduce 编程模型的原理是：利用一个输入 key/value pair 集合来产生一个输出的 key/value pair 集合。MapReduce 库的用户用两个函数表达这个计算：Map 和 Reduce。

用户自定义的 Map 函数接受一个输入的 key/value pair 值，然后产生一个中间 key/value pair 值的集合。MapReduce 库把所有具有相同中间 key 值 I 的中间 value 值集合在一起后传递给 reduce 函数。

用户自定义的 Reduce 函数接受一个中间 key 的值 I 和相关的一个 value 值的集合。Reduce 函数合并这些 value 值，形成一个较小的 value 值的集合。一般的，每次 Reduce 函数调用只产生 0 或 1 个输出 value 值。通常我们通过一个迭代器把中间 value 值提供给 Reduce 函数，这样我们就可以处理无法全部放入内存中的大量的 value 值的集合。

### 2.1 例子

例如，计算一个大的文档集合中每个单词出现的次数，下面是伪代码段：

```
map(String key, String value):
  // key: document name
  // value: document contents
  for each word w in value:
    EmitIntermediate(w, "1");

reduce(String key, Iterator values):
  // key: a word
  // values: a list of counts
  int result = 0;
  for each v in values:
    result += ParseInt(v);
  Emit(AsString(result));
```

Map 函数输出文档中的每个词、以及这个词的出现次数（在这个简单的例子里就是 1）。Reduce 函数把 Map 函数产生的每一个特定的词的计数累加起来。

另外，用户编写代码，使用输入和输出文件的名字、可选的调节参数来完成一个符合 MapReduce 模型规范的对象，然后调用 MapReduce 函数，并把这个规范对象传递给它。用户的代码和 MapReduce 库链接在一起（用 C++ 实现）。附录 A 包含了这个实例的全部程序代码。

### 2.2 类型

尽管在前面例子的伪代码中使用了以字符串表示的输入输出值，但是在概念上，用户定义的 Map 和 Reduce 函数都有相关联的类型：

```
map(k1, v1) -> list(k2, v2)
reduce(k2, list(v2)) -> list(v2)
```

比如，输入的 key 和 value 值与输出的 key 和 value 值在类型上推导的域不同。此外，中间 key 和 value 值与输出 key 和 value 值在类型上推导的域相同。

我们的 C++ 中使用字符串类型作为用户自定义函数的输入输出，用户在自己的代码中对字符串进行适当的类型转换。

### 2.3 更多的例子

这里还有一些有趣的简单例子，可以很容易的使用 MapReduce 模型来表示：

- **分布式的 Grep**：Map 函数输出匹配某个模式的一行，Reduce 函数是一个恒等函数，即把中间数据复制到输出。
- **计算 URL 访问频率**：Map 函数处理日志中 web 页面请求的记录，然后输出 (URL, 1)。Reduce 函数把相同 URL 的 value 值都累加起来，产生 (URL, 记录总数) 结果。
- **倒转网络链接图**：Map 函数在源页面（source）中搜索所有的链接目标（target）并输出为 (target, source)。Reduce 函数把给定链接目标（target）的链接组合成一个列表，输出 (target, list(source))。
- **每个主机的检索词向量**：Map 函数为每一个输入文档输出 (主机名, 检索词向量)，其中主机名来自文档的 URL。Reduce 函数接收给定主机的所有文档的检索词向量，并把这些检索词向量加在一起，丢弃掉低频的检索词，输出一个最终的 (主机名, 检索词向量)。
- **倒排索引**：Map 函数分析每个文档输出一个 (词, 文档号) 的列表，Reduce 函数的输入是一个给定词的所有 (词, 文档号)，排序所有的文档号，输出 (词, list(文档号))。
- **分布式排序**：Map 函数从每个记录提取 key，输出 (key, record)。Reduce 函数不改变任何的值。

## 3. 实现

MapReduce 模型可以有多种不同的实现方式。如何正确选择取决于具体的环境。本章节描述一个适用于 Google 内部广泛使用的运算环境的实现：用以太网交换机连接、由普通 PC 机组成的大型集群。

在我们的环境里包括：
1. x86 架构、运行 Linux 操作系统、双处理器、2-4GB 内存的机器。
2. 普通的网络硬件设备，每个机器的带宽为百兆或者千兆。
3. 集群中包含成百上千的机器，机器故障是常态。
4. 存储为廉价的内置 IDE 硬盘。一个内部分布式文件系统（GFS）用来管理存储在这些磁盘上的数据。
5. 用户提交工作（job）给调度系统。每个工作都包含一系列的任务（task），调度系统将这些任务调度到集群中多台可用的机器上。

### 3.1 执行概括

通过将 Map 调用的输入数据自动分割为 M 个数据片段的集合，Map 调用被分布到多台机器上执行。输入的数据片段能够在不同的机器上并行处理。使用分区函数将 Map 调用产生的中间 key 值分成 R 个不同分区（例如，hash(key) mod R），Reduce 调用也被分布到多台机器上执行。分区数量（R）和分区函数由用户来指定。

当用户调用 MapReduce 函数时，将发生下面的一系列动作：

1. 用户程序首先调用的 MapReduce 库将输入文件分成 M 个数据片段，每个数据片段的大小一般从 16MB 到 64MB。然后用户程序在机群中创建大量的程序副本。
2. 这些程序副本中有一个特殊的程序——master。副本中其它的程序都是 worker 程序，由 master 分配任务。有 M 个 Map 任务和 R 个 Reduce 任务将被分配，master 将一个 Map 任务或 Reduce 任务分配给一个空闲的 worker。
3. 被分配了 map 任务的 worker 程序读取相关的输入数据片段，从输入的数据片段中解析出 key/value pair，然后把 key/value pair 传递给用户自定义的 Map 函数，由 Map 函数生成并输出的中间 key/value pair，并缓存在内存中。
4. 缓存中的 key/value pair 通过分区函数分成 R 个区域，之后周期性的写入到本地磁盘上。缓存的 key/value pair 在本地磁盘上的存储位置将被回传给 master，由 master 负责把这些存储位置再传送给 Reduce worker。
5. 当 Reduce worker 程序接收到 master 程序发来的数据存储位置信息后，使用 RPC 从 Map worker 所在主机的磁盘上读取这些缓存数据。当 Reduce worker 读取了所有的中间数据后，通过对 key 进行排序后使得具有相同 key 值的数据聚合在一起。
6. Reduce worker 程序遍历排序后的中间数据，对于每一个唯一的中间 key 值，Reduce worker 程序将这个 key 值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追加到所属分区的输出文件。
7. 当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对 MapReduce 调用才返回。

### 3.2 Master 数据结构

Master 持有一些数据结构，它存储每一个 Map 和 Reduce 任务的状态（空闲、工作中或完成），以及 Worker 机器（非空闲任务的机器）的标识。

Master 就像一个数据管道，中间文件存储区域的位置信息通过这个管道从 Map 传递到 Reduce。因此，对于每个已经完成的 Map 任务，master 存储了 Map 任务产生的 R 个中间文件存储区域的大小和位置。

### 3.3 容错

**Worker 故障**：master 周期性的 ping 每个 worker。如果在一个约定的时间范围内没有收到 worker 返回的信息，master 将把这个 worker 标记为失效。所有由这个失效的 worker 完成的 Map 任务被重设为初始的空闲状态，之后这些任务就可以被安排给其他的 worker。已经完成的 Reduce 任务的输出存储在全局文件系统上，因此不需要再次执行。

**Master 失败**：如果这个 master 任务失效了，可以从最后一个检查点（checkpoint）开始启动另一个 master 进程。然而由于只有一个 master 进程，我们现在的实现是如果 master 失效，就中止 MapReduce 运算。客户可以检查到这个状态，并且可以根据需要重新执行 MapReduce 操作。

**在失效方面的处理机制**：当用户提供的 Map 和 Reduce 操作是输入确定性函数时，我们的分布式实现在任何情况下的输出都和所有程序没有出现任何错误、顺序的执行产生的输出是一样的。这个特性通过 Map 和 Reduce 任务的输出是原子提交来完成。

### 3.4 存储位置

我们通过尽量把输入数据（由 GFS 管理）存储在集群中机器的本地磁盘上来节省网络带宽。GFS 把每个文件按 64MB 一个 Block 分隔，每个 Block 保存在多台机器上。MapReduce 的 master 在调度 Map 任务时会考虑输入文件的位置信息，尽量将一个 Map 任务调度在包含相关输入数据拷贝的机器上执行。

### 3.5 任务粒度

我们把 Map 拆分成了 M 个片段、把 Reduce 拆分成 R 个片段执行。理想情况下，M 和 R 应当比集群中 worker 的机器数量要多得多。实际使用时 M 值使得每一个独立任务都是处理大约 16M 到 64M 的输入数据。我们通常会用这样的比例来执行 MapReduce：M = 200000，R = 5000，使用 2000 台 worker 机器。

### 3.6 备用任务

影响一个 MapReduce 的总执行时间最通常的因素是"落伍者"（straggler）：在运算过程中，如果有一台机器花了很长的时间才完成最后几个 Map 或 Reduce 任务，导致 MapReduce 操作总的执行时间超过预期。我们有一个通用的机制来减少"落伍者"出现的情况：当一个 MapReduce 操作接近完成的时候，master 调度备用（backup）任务进程来执行剩下的、处于处理中状态（in-progress）的任务。无论是最初的执行进程、还是备用任务进程完成了任务，我们都把这个任务标记成为已经完成。

## 4. 技巧

### 4.1 分区函数

一个缺省的分区函数是使用 hash 方法（比如 hash(key) mod R）进行分区。用户也可以提供专门的分区函数，例如使用 "hash(Hostname(urlkey)) mod R" 作为分区函数就可以把所有来自同一个主机的 URLs 保存在同一个输出文件中。

### 4.2 顺序保证

我们确保在给定的分区中，中间 key/value pair 数据的处理顺序是按照 key 值增量顺序处理的。

### 4.3 Combiner 函数

在某些情况下，Map 函数产生的中间 key 值的重复数据会占很大的比重。我们允许用户指定一个可选的 combiner 函数，combiner 函数首先在本地将这些记录进行一次合并，然后将合并的结果再通过网络发送出去。

### 4.4 输入和输出的类型

MapReduce 库支持几种不同的格式的输入数据，比如文本模式的输入数据的每一行被视为是一个 key/value pair。用户可以通过提供一个简单的 Reader 接口实现就能够支持一个新的输入类型。

### 4.5 副作用

在某些情况下，MapReduce 的使用者发现，如果在 Map 和/或 Reduce 操作过程中增加辅助的输出文件会比较省事。我们依靠程序 writer 把这种"副作用"变成原子的和幂等的。

### 4.6 跳过损坏的记录

我们提供了一种执行模式，在这种模式下，为了保证整个处理能继续进行，MapReduce 会检测哪些记录导致确定性的 crash，并且跳过这些记录不处理。

### 4.7 本地执行

为了简化调试、profile 和小规模测试，我们开发了一套 MapReduce 库的本地实现版本，通过使用本地版本的 MapReduce 库，MapReduce 操作在本地计算机上顺序的执行。

### 4.8 状态信息

master 使用嵌入式的 HTTP 服务器显示一组状态信息页面，用户可以监控各种执行状态。

### 4.9 计数器

MapReduce 库使用计数器统计不同事件发生次数，这些计数器的值周期性的从各个单独的 worker 机器上传递给 master。计数器机制对于 MapReduce 操作的完整性检查非常有用。

## 5. 性能

本节用在一个大型集群上运行的两个计算来衡量 MapReduce 的性能：一个计算在大约 1TB 的数据中进行特定的模式匹配（Grep），另一个计算对大约 1TB 的数据进行排序。

### 5.1 集群配置

所有这些程序都运行在一个大约由 1800 台机器构成的集群上。每台机器配置 2 个 2G 主频、支持超线程的 Intel Xeon 处理器，4GB 的物理内存，两个 160GB 的 IDE 硬盘和一个千兆以太网卡。

### 5.2 GREP

分布式的 grep 程序扫描大概 10¹⁰ 个由 100 个字节组成的记录，查找出现概率较小的 3 个字符的模式。当 1764 台 worker 参与计算时，处理速度达到了 30GB/s。整个计算过程从开始到结束一共花了大概 150 秒。

### 5.3 排序

排序程序处理 10¹⁰ 个 100 个字节组成的记录（约 1TB 数据）。排序程序由不到 50 行代码组成。整个运算消耗了 891 秒，和 TeraSort benchmark 的最高纪录 1057 秒相差不多。

### 5.4 高效的 backup 任务

关闭备用任务后，排序程序多花了 44% 的执行时间（1283 秒 vs 891 秒）。

### 5.5 失效的机器

在程序开始后有意的 kill 了 1746 个 worker 中的 200 个，整个运算在 933 秒内完成（只比正常执行多消耗了 5% 的时间）。

## 6. 经验

MapReduce 库能广泛应用于我们日常工作中遇到的各类问题，包括：
- 大规模机器学习问题
- Google News 和 Froogle 产品的集群问题
- 从公众查询产品（比如 Google 的 Zeitgeist）的报告中抽取数据
- 从大量的新应用和新产品的网页中提取有用信息
- 大规模的图形计算

截至 2004 年 9 月，Google 内部已有约 900 个不同的 MapReduce 程序。

### 6.1 大规模索引

MapReduce 最成功的应用就是重写了 Google 网络搜索服务所使用到的 index 系统。索引系统的输入数据超过 20TB。使用 MapReduce 后，计算的代码行数从原来的 3800 行 C++ 代码减少到大概 700 行代码。

## 7. 相关工作

讨论与 Bulk Synchronous Programming、MPI、active disks、Charlotte System、NOW-Sort、River、BAD-FS、TACC 等相关系统的比较。

## 8. 结束语

MapReduce 编程模型在 Google 内部成功应用于多个领域。成功归结为几个方面：易于使用；大量不同类型的问题都可以通过 MapReduce 简单解决；在数千台计算机组成的大型集群上灵活部署运行。

## 9. 感谢

略（原文为英文感谢词）

## 10. 参考资料

[1] Andrea C. Arpaci-Dusseau et al. High-performance sorting on networks of workstations. SIGMOD 1997.
[2] Remzi H. Arpaci-Dusseau et al. Cluster I/O with River. IOPADS 1999.
[3] Arash Baratloo et al. Charlotte: Metacomputing on the web. ICPADS 1996.
[4] Luiz A. Barroso et al. Web search for a planet: The Google cluster architecture. IEEE Micro 2003.
[5] John Bent et al. Explicit control in a batch-aware distributed file system. NSDI 2004.
[6] Guy E. Blelloch. Scans as primitive parallel operations. IEEE Trans. Computers 1989.
[7] Armando Fox et al. Cluster-based scalable network services. SOSP 1997.
[8] Sanjay Ghemawat et al. The Google file system. SOSP 2003.
[9] S. Gorlatch. Systematic efficient parallelization of scan. Euro-Par 1996.
[10] Jim Gray. Sort benchmark home page.
[11] William Gropp et al. Using MPI. MIT Press 1999.
[12] L. Huston et al. Diamond: A storage architecture for early discard. FAST 2004.
[13] Richard E. Ladner, Michael J. Fischer. Parallel prefix computation. JACM 1980.
[14] Michael O. Rabin. Efficient dispersal of information. JACM 1989.
[15] Erik Riedel et al. Active disks for large-scale data processing. IEEE Computer 2001.
[16] Douglas Thain et al. Distributed computing in practice: The Condor experience. 2004.
[17] L. G. Valiant. A bridging model for parallel computation. CACM 1997.
[18] Jim Wyllie. Spsort: How to sort a terabyte quickly.

## 附录 A：单词频率统计

包含一个完整的 C++ 程序，用于统计在一组命令行指定的输入文件中每一个不同的单词出现频率。程序实现了 WordCounter（Mapper）和 Adder（Reducer）类。
