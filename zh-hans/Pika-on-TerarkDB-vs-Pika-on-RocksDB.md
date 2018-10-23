## Pika on RocksDb verses Pika on TerarkDB

### 关于Pika

&emsp;&emsp;Pika 是一种支持持久化大容量存储的类 Redis 储存系统。除去解决了 Redis 无法应对大容量数据的问题，Pika 还可以提供支持全同步和部分同步的主从机制以提供更好的性能。

&emsp;&emsp;Pika 的架构图如下：

![](images/pika_vs_terark/pika_constructrue.png)

&emsp;&emsp;正如图中所示，Pika 使用 Blackwidow 桥接了一个轻薄的壳层Pika Process Layer 和 RocksDB 这一核心数据库，通过 Blackwidow 将储存于 RocksDB 中的 key-value 对形式的数据封装成 Redis 格式的数据结构（Strings, Hash, List, Set, ZSet），而壳层则是通过 Redis 协议与外界进行交互。

&emsp;&emsp;显然，作为数据库核心的 RocksDB 是影响整个系统性能的最大因子。

### 关于RocksDB

&emsp;&emsp;RocksDB 作为一种继承了 LevelDB 衣钵的 key-value 数据库，使用了和 LevelDB 一样的 Log-Structured 结构用以储存数据，同时允许高度灵活的配置使得其可以应对各种不同的应用场景与需求。我们不妨也看看 RocksDB 的架构设计：

![](images/pika_vs_terark/RocksDB_Structure.png)

&emsp;&emsp;简要的解释一下图中表示的架构：对于 RocksDB 的每个 DBImpl 实例中都存在唯一的 Wal Log 和 Manifest , 供所有 ColumnFamily 共享。从内存的组织形式上来看，ColumnFamily 及其御下的所有 MemTable 和 Immutable Memtables 组成了数据的实质储存形式，在调控逻辑的管控下与外界物理储存进行IO交互；从逻辑的组织形式上来看，VersionStorage 将物理储存中的 SST File 组织成 LSM Tree（Log Structured Merge Trees）的结构，用以提升系统的读写性能。这两种组织形式通过 RocksDB 的内部组件进行整合，使得其逻辑得以自洽，从而将其组成一个逻辑完整可用的储存系统。

&emsp;&emsp;本质上，所有的 ColumnFamily 是 VersionStorage 的中间件，是 RocksDB 为了实现 LSM Tree 结构本身的一种具体实现方法, 虽然 LSM Tree 配合 block compression 较传统的 B-Tree 结构提高了不少的性能，但它也有瑕疵。

### 关于TerarkDB

&emsp;&emsp;TerarkDB 与 RocksDB 一样，同样是使用了 Log-Strucured 结构作为数据组织形式的基本逻辑，而不同的则是 Terark 使用了自研的Searchable Compression, 借由 Succinct Data Structure 强大的信息压缩能力，TerarkDB 可以在接近信息论下界的空间内来表达具体的 Objects, 用以承载复杂的 indexing 需求。与此同时，再对 key-value 进行特殊的压缩，使得每条数据在可索引的情况下能够在不损失性能的同时到极高的压缩率。

&emsp;&emsp;这种可以单点索引的方式，使得随机索引的能力大大增强，且不会出现 block compression 在解压某块中的单条或少数数据时被迫要解压整个块，从而浪费了计算力和内存的情况。与之相反，这种方式反而提高了内存的利用率，使得实际运行时需求的内存大大减少。

&emsp;&emsp;TerarkDB 兼容了 RocksDB 的上层 API，让我们看看 Pika 在 RocksDB 和 TerarkDB 上的具体情况。

### 性能测试数据

#### 测试平台

CPU : Intel® Xeon® Processor E5-2630 v3 * 2

SSD : Intel® SSDSC2BP48 0420 IOPS 89000

#### 测试结果

当内存小于数据量，即普通工况下：

![](images/pika_vs_terark/fig1.png)

当内存大于数据量的情况下：

<img src=images/pika_vs_terark/fig2.png width="65%" height="65%" />

### 结果解读

&emsp;&emsp;在内存有限，数据量大于内存的情况下，即平常的实际工况中，Pika on TerarkDB 的性能远高于原版；而在内存远大于数据量的情况，Pika on TerarkDB 的性能则稍低于原版。

&emsp;&emsp;出现这样情况的原因很简单，在平常的实际工况中，TerarkDB的独有技术可以有效提高内存与IO效能，自然好于原版；而在内存可以肆意使用时，原版可以将数据全部缓冲至内存，自然会快一些。