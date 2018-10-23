[[English|Overcome Universal Compaction Space Amplification]]
## Compaction 策略
RocksDB 默认的 Compact 策略是 Level Compaction，Level Compaction 在大多数情况下工作得很好，但是面对高压力的随机写，就有点扛不住了。此时如果使用 Universal Compaction，可以大大改善性能。

原版 RocksDB Compaction 策略比较:

| Compaction<br/>Style | 写放大(体现写性能) | 空间放大(峰值空间) | SST 尺寸 |
|----------:|--------------------|---------------------|-----------------|
|     Level | **大**，写性能**低** | **小**，一般小于 5% | 尺寸越小，优势越大 |
| Universal | **小**，写性能**高** | **大**，略高于 100% | 与尺寸无关|

从上面的表格可以看出：

* Universal Compaction 主要的问题是峰值空间占用
  * 理论上全量 Compact 时，需要 100% 的额外空间
  * 实际的空间占用还会稍微高一点，比如 110%
* Level Compaction SST 尺寸越小，优势越大
  * Level Compaction 每次选取一个上层文件，找到下层文件中 Key Range 与该上层文件有交集的文件，一起进行 Compact，按照默认配置，每层的总尺寸是上一层的 10 倍(max_bytes_for_level_multiplier)，这样，每次上层文件的 Key Range 会覆盖的 12(期望值) 个下层文件。从而写放大就是 13。每一层 13，如果有 7 层，大约就是 13*(7-1) = 78，在很多实测报告中，写放大是 50 左右。
  * 参与每次 Compact 的文件数是基本不变的，从而单个 SST 尺寸越小，参与每次 Compact 的数据总尺寸就越小，系统表现就越平滑

更多内容，请参考 [Universal Compaction 的官方文档](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)

## TerarkDB 集成
TerarkDB 的核心是 TerarkZipTable，这是使用 Terark 技术对 SST(Static Sorted Table) 的实现。除去内在算法上的不同，外在表现上的主要不同，是 SST 的尺寸，RocksDB 默认的 BlockBasedTable 尺寸是 64M（默认值），改成更小的值（例如 2M）一般也没问题。

但是 TerarkZipTable 只有在尺寸更大时才显示出更大的优势，尺寸小的时候优势不明显。这样的情况，对 Level Compaction 更不友好，所以，再综合写放大因素，TerarkDB 使用了 Universal Compaction。

## 解决 Universal Compaction 的空间放大问题
Universal Compaction 100% 的空间放大来自于：
* 每个 Compact 是个原子操作，Compact 结束时，加载所有新文件，删除所有旧文件
* 全量 Compact 时，参与 Compact 的是所有文件

根据 Universal Compaction 的特点，上面**第二点**是必须的，没法改变，那么，**第一点**，Compact 必须是原子操作吗？

首先澄清概念：

* SST, Static Sorted Table
  * 每个 SST 文件中的 (Key,Value) 集合按照 Key 从小到大排列
* Sorted Run
  * 每个 Sorted Run 由多个（或一个）SST 文件组成
  * 文件按 Key 大小排列，不同文件的 Key 无交集：prev.MaxKey < next.MinKey

现在我们仔细分析一下：
* Compact 的输入是多个 Sorted Run，每个输入 Sorted Run 有至少一个文件
* Compact 的输出是一个 Sorted Run，这个输出 Sorted Run 一般是多个文件
* Compact 过程中，必然存在这样一些时刻：
  * Tx: Input Iterator 刚好划过了某个 Sorted Run 中的某个文件
  * Ty: 完成了一个 Output SST

所有的 Tx 构成一个序列 Tx1, Tx2, Tx3 ....<br/>
所有的 Ty 构成一个序列 Ty1, Ty2, Ty3 ....

在每个 Ty 之前所有的 Tx 对应的那些文件，其中的数据必然完全包含在已完成的（一个或多个） Output SST 中，这样，在这些 Ty 时刻，理论上，我们就可以用新生成的 Output SST 取代相应的 Input SST，从而，整个 Compact 就不必是原子的。

顺着这个思路，我们解决了 Universal Compaction 的空间放大问题，最终，空间放大不取决于整个 Compact 的输入有多大，而是取决于：单个 SST 的尺寸（和相应的临时文件尺寸）；另一方面，因为可以有多个并发运行的 Comapct，所以，最坏情况下额外的空间占用就是 `Compact并发数 * SST.size * 2` （*2 是因为 Temp.size ≈ SST.size）。

从而，我们就可以通过限制 SST 尺寸和并发 Compact 数量来限制空间放大，下表列出各种参数的影响。

|    增大以下配置值          |产生以下后果|
|-------------------------:|:---------|
|max_background_compactions|总体 Compact 的速度更快，写放大更坏|
|max_subcompactions|大的 Compact 的速度更快，写放大更坏|
|target_file_size_base|压缩率更好，搜索性能更好，写放大更更坏|

## 配置建议
根据 `额外空间占用 = Compact并发数 * SST.size * 2`，我们可以计算：

假定我们希望额外空间占用不超过数据库压缩后的尺寸的 20%，那么我们需要保证以下值：
```
max_background_compactions * max_subcompactions * target_file_size_base * 2
```
小于 `磁盘SSD.size * 20%`

TerarkDB 的默认默认设置为：
```
 max_background_compactions = 3
         max_subcompactions = 1
      target_file_size_base = 物理内存总数 / 8
```
该默认设置基于以下假定：<br/>
　　当 SSD 尺寸等于内存总量的 5 倍时(精确量是4.5倍，再留出一点安全范围)，该配置可以确保能把 SSD 填满的数据库，其写放大不超过 20%。

如果有特殊需要，请根据你的实际情况进行调整。