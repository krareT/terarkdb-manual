## 背景
我们知道，KeyValue 数据库，就是通过搜索 Key 得到 Value 的数据库，所以 Key 需要索引，Value 不需要索引，并且，在绝大多数情况下，Key 的长度比 Value 的长度小得多。

TerarkDB 有两个核心概念：CO-Index 和 PA-Zip
* **CO-Index**: 即 **C**ompressed **O**rdered **Index**，把一个类型为 `ByteArray` 的 Key，映射到一个整数 ID，这个 ID 用来访问 PA-Zip 中相应的那条 Value。
* **PA-Zip**: **P**oint **A**ccessible **Zip**，可以看做是一个抽象的 `array<ByteArray>`，核心功能是把 ID 作为抽象数组的下标，去访问该抽象数组的元素，当然，这些元素都是压缩存储的。
* CO-Index 和 PA-Zip 一起，构成一个逻辑上的 `map<Key, Value>`

Terark SST 由 CO-Index 和 PA-Zip 组成，创建 SST 的时候，实际上是创建 CO-Index 和 PA-Zip，既然 Key 很短，Value 很长，为了兼顾性能和内存用量：创建 CO-Index 时，我们会把相应的 Key 都放在内存中，创建 PA-Zip 时，只把关键数据(例如字典)放入内存，这算是最简单的内存控制（暂且叫做**内存控制-1**）。

## 创建 Terark SST 需要对输入扫描两遍
创建 PA-Zip 时，我们需要对输入扫描两遍，第一遍计算 Value 的各种统计特征，并且收集全局字典，第二遍执行压缩。

RocksDB 使用 SSTBuilder 来构建 SST，构建过程中，对每条数据调用 `SSTBuilder.Add(Key,Value)`，构建完成时，调用 `SSTBuilder.Finish()`，以这样的方式，SSTBuilder 就只能对输入序列扫描一次。

如果要对输入做第二次扫描，就需要付出一些努力。最初，我们在第一次扫描的过程中，把拿到的数据保存到一个临时文件中，第二遍扫描直接读那个临时文件。实际上，第二遍扫描只是按第一遍扫描的顺序，把 value 读一遍，所以这个临时文件只需要保存 value。

## 内存控制-2
创建 CO-Index 和 PA-Zip 都需要大量内存，所以进一步的内存限制是必须的。

第一次扫描中，我们总是把 Key 写到临时文件中，到执行 Index 创建时，我们再把这个临时文件读进内存：
* 只有内存不会超限时，我们才会把 Key 文件读进内存，去创建 CO-Index
* 如果内存超限，CO-Index 的创建就必须排队等待

PA-Zip 也一样，如果内存超限(字典需要大量内存)，也必须排队等待。

排队策略也需要仔细设计：
* SST Flush 是把 MemTable 压缩成 SST，这些 SST 的尺寸很小，创建时需要的内存也小，但优先级最高
* Compaction 产生的 SST 大小各异，一般都很大，创建时需要的内存也很大，优先级较低
* 按照小作业优先的原则，大 SST 应该降低优先级，但不能无限等待（大作业被饿死）

## 不需要临时文件的两遍扫描
通过临时文件的方式实现两遍扫描，需要不少磁盘空间，特别是对于我们，Terark SST 的尺寸都很大（尺寸越大，压缩率越高，大多在 GB 级别），磁盘空间的需求就太多了。

在官方版 RocksDB 的框架内，实现两遍扫描没有别的办法，所以我们只有自己去魔改 RocksDB，魔改的过程充满了各种曲折……

最终，我们还是找到了办法：创建一个专用的 Iterator 进行第二遍扫描，这个 Iterator 实际上是 RocksDB 自身第一遍扫描用的那个 Iterator 的深拷贝，所有相关的辅助对象，例如 `Merger`, `RangeDelAggregator` ... 都需要为第二遍扫描的 Iterator 创建一个副本，但`CompactionFilter` 不能深拷贝，两个 Iterator 必须用同一个 `CompactionFilter`。

## Iterator 太慢
貌似所有的问题都已经解决，我们只需要一点磁盘临时空间，和很小的内存，就可以创建出 Terark SST，各项指标非常优异：
* SST 文件尺寸非常小，比 RocksDB 原生的 SST 小得多，因为我们的压缩率很高
* SST 内存占用非常小，因为使用全局压缩算法，内存用量的上限就是 SST 文件尺寸
* SST 随机访问非常快，直接在压缩的数据上进行定点访问，省去了双缓存等不必要的开销

直到有一天，我们在 MyRocks 中碰到一个**数据插入非常慢**的问题，好日子忽然到头了……

在 MyRocks 中，如果某个 table 有 *n* 个索引(包括主键索引)，对每一条 SQL 数据，在 RocksDB 存储层就有 *n* 条 KeyValue 数据，每个索引占一条，其中，辅助索引(Secondary Index)的 value 是空的。如果某个 table 有很多个辅助索引，就会产生很多条 value 为空的数据，这就是 Compact 性能杀手，下面我们解释原因：

Compact 相当于对多路(按Key)有序的输入序列进行归并，产生一个有序的输出序列，其中，可以把多路输入和归并算法封装起来，对外抽象成 `MergeIterator`，归并的过程相当慢，RocksDB 的归并算法用 Heap，但即使用 LoserTree，也快不到哪里去。这里的主要问题在于：

如果数据的总量(KeyValue 的总字节数)相同
* 当 value 很**长**的时候，KeyValue 的条目数就很**小**，整体的性能取决于 value 的解压速度，`MergeIterator` 慢一点无所谓
* 当 value 很**短**的时候，KeyValue 的条目数就很**大**，整体的性能取决于 `MergeIterator`
  * MyRocks 辅助索引的 value 甚至短到了 0，`MergeIterator` 就几乎完全决定了总体的性能

第二遍扫描(归并)的行为和第一遍扫描完全一样，这样就又慢了一倍。

## 省略 短 value 的第二遍扫描
我们首先需要精确定义什么是“短 value”，最简单的定义就是：平均长度小于某个值`SVL`(Short Value Length)。`SVL`可以是个定值，但更合理的方案是使用相对值，相对于 Key 的长度。在我们的实现中，`SVL = 2 * avg(KeyLen)`。

接下来，就是如何识别“短 value”

## 识别“短 value”
要识别“短 value”，最简单直接的方法就是计算出它的平均长度：在第一遍扫描中累加长度，在 **Finish** 中计算出平均长度。但是这样做，我们只有在 **Finish** 时才能知道是否需要第二遍扫描，这时已经太晚了……

好在，我们知道，有一种策略叫做“Speculation”，也就是“投机”，或者“冒险”：

在第一遍扫描开始的时候，将 value 写入临时文件，**直到写入的尺寸大于某个预设值(例如 5M)** 并且 当前的平均长度大于 `SLV`。**粗体部分**诠释了“**冒险**”二字，因为 value 的长度分布可能并不均匀，只有一定数量的 value 才有统计意义，5M 是一个拍脑袋值，应该足够。

这样，在绝大多数情况，我们的**冒险**都会成功！非常简单的策略，但是非常有效！

### 给 `CompactionIterator` 增加 `Seek()` 功能

在大多数情况下，单个 SST 只有单个物理的 `map<Key,Value>` 对象，上面的这些策略足以应付。但是，存在一些更复杂的情况：
<table><tr><td>
首先，因为 RocksDB 作为 KeyValue 存储，是按 Key 有序的，从而，前缀相同的 Key 集合，在逻辑上是相邻的，在 RocksDB API 上就体现在：通过 <code>Iterator.Seek</code> 搜索一个前缀 <code>P</code>，会将 Iterator 定位到以 <code>P</code> 为前缀的 Key (有序)集合的第一个(最小的) KeyValue 所在的位置，接着调用 <code>Iterator.Next</code>，遍历就可以得到所有以 <code>P</code> 为前缀的 KeyValue 集合。利用这个特性，可以实现更多的以 <code>P</code> 为前缀的搜索功能：Range 搜索，前缀搜索，等等。
</td></tr></table>

MyRocks 和 MongoRocks (以及其它一些基于 RocksDB 的数据库) 就利用这个特性实现所有需要跨表跨库跨索引的功能：
* 每个 Key 包含一个前缀，该前缀用来区分一个 KeyValue 子集
* 每个 KeyValue 子集就是一个 **表/索引** 的实现
* 每个 SST 可以包含多个不同的前缀 `P`，从而包括多个 **表/索引**
 
在 SSTBuilder 的第一遍扫描中，如果 KeyValue 的 subset1 已扫描结束(碰到了下一个*大一点的*前缀，对应 subset2)，并且 subset1 满足 `短 Value` 条件，在扫描 subset2 的过程中，发现 subset2 不满足 `短 Value` 条件。

到此为止，我们知道 subset1 在前，subset2 在后，其中 subset1 不需要第二遍扫描，subset2 需要第二遍扫描。所以，我们就需要在第二遍扫描的 `CompactionIterator` 中 **跳过** subset1 对应的那个 Key Range。我们通过 `Seek` 来实现这个 **跳过** 的功能（一般而言，功能越强大，需要的代价越大，在这里 `Seek` 的功能比 **跳过** 更强大，但实际上 `Seek` 的实现难度和性能和 **跳过** 都差不多）。

`Seek` 需要的算法并不难，但其中需要处理很多繁琐的细节，比如需要正确处理 UserKey, MVCC SeqNum 等等。

## 并发的 Index 构建
在 SSTBuilder 的第一遍扫描中，在 KeyValue 的 subset1 扫描结束时(碰到了下一个 *大一点的* 前缀，对应 subset2)，如果此时内存用量没有超限，subset1 的 Index 构建就可以立即在另一个线程中开始，而当前线程同时接着扫描 subset2, subset3 ...

更进一步，如果 subset1 不需要第二遍扫描，它的 PA-Zip(Value 存储) 也可以并发构建。

## 不能并发构建 PA-Zip 的情况
理论上讲，subset1 的 PA-Zip 是否可以并发构建与它是否需要第二遍扫描并没有关系，只要内存、CPU 够用，就应该尽量并发执行更多的工作：
* 我们知道第一遍扫描用的是 Iterator1，第二遍扫描用的是 Iterator2，并且，Iterator1 (的位置)领先于 Iterator2
* 如果 subset1 的第一遍已经扫描完成，并且 subset1 也需要第二遍扫描，那么仍然可以立刻并发执行 subset1 的第二遍扫描和 subset2 的第一遍扫描

现实是 RocksDB 支持插拔式的 `CompactionFilter`，我们无法预测这个 `CompactionFilter` 会有什么陷阱，前面讲到，Iterator2 不会对 `CompactionFilter` 进行深拷贝，这就是原因——我们在 MyRocks 和 MongoRocks 中的 `CompactionFilter` 中发现了这一点……
