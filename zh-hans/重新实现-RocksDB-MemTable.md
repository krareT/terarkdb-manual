TerarkDB 最初的，也是最最重要的模块，是 [Terark SST](TerarkDB-SST-的创建过程.html)，但是，MemTable 是数据进入 DB 的第一个入口，它的性能，绝对不能忽视……

## 背景
在某互联网公司做 TerarkSQL(TerarkDB + MySQL) 的 POC 时，有一个场景，TerarkSQL 的表现远不如预期，写入速度甚至只有官方原版 MySQL 的 70%。这是我们万万不能接受的一个结果，经过仔细排查，我们发现，SkipList 相关的时间开销（主要是 Comparator 耗时）占了 60% 以上！

在这个场景中，数据条目数量非常大（20 多亿条），但表的结构很简单，类似这样：
```sql
CREATE TABLE Counting(
    name     varchar(100) PRIMARY KEY, # 实际平均长度约 30 字节
    count    INT(10)
);
```
每天会进行一次批量数据更新，大约更新 1亿条 数据，这个更新过程，MySQL 原版需要 12分钟，TerarkSQL 需要 17分钟。

在 MyRocks(RocksDB+MySQL) 中，`Counting` 表对应到存储引擎上：name 字段就是 key，count 字段是 value，对于一般的场景，value 尺寸是远大于 key 的，但在这个场景中，key 的尺寸却远大于 value，一下就命中了 MyRocks 的软肋……

虽然锅是 MyRocks 的，但客户只看结果，不管你在其他方面表现有多好，只要有一个地方不好，就会失去客户的认可。

## RocksDB 的 MemTable
架构上，RocksDB MemTable 的设计目标是可插拔的不同实现的，在具体实现上，RocksDB 使用 SkipList 作为默认的 MemTable，其最大的优点是可以并发插入。

然而，RocksDB MemTable 的架构设计实际上有诸多的问题：

1. MemTable 不允许插入失败：每个 MemTable 有内存上限（`write_buffer_size`），是否达到内存上限，通过一种很猥琐的方式来判定：
   * MemTable 需要自己实现一个虚函数，用来报告自己的内存用量
   * `管理层`根据 MemTable 报告的内存用量，决定是否冻结当前 MemTable 并创建新的 MemTable
   * 如此，引发了一个很严重的问题：[#4056](https://github.com/facebook/rocksdb/issues/4056)
   * 这个缺陷非常致命，直接导致内存用量无法精确控制，严重阻碍优化方案的实现；更致命的是相关的代码错综复杂，几乎无法修改，我们进行了大量的尝试，最终还是只能放弃，退回到次优化的实现
2. MemTable 限死了 KeyValue 的编码方式：都由带 var int 前缀的方式编码，这又带来了至少两个严重的问题：
   <br/>&nbsp;&nbsp;&nbsp;(1) var int 解码需要 CPU 时间，并且 key value 都无法自然对齐（对齐到 4 字节或 8 字节）
   <br/>&nbsp;&nbsp;&nbsp;(2) 新的 MemTable 无法使用别的存储方式，例如基于 Trie 树的 MemTable 不需要保存 Key，而是在搜索/扫描的过程中重建 Key
   * 这个缺陷可以通过重构来解决，我们已经提交了相关的 pull request
3. 工厂机制形同摆设，新的 MemTable 不能无缝 plugin，这个缺陷我们也提交了相关的 pull request

架构上的改进，如果没有高效的实现来验证，总是缺乏一些说服力，我们经过不懈的努力，在算法层面获得了 5 倍以上的性能提升，同时 MemTable 的内存用量还大大降低。当然，这个提升，在整个系统层面会被其它部分拖后腿，最终效果是：在 MyRocks 中的一些场景下，我们可以获得 40% 的性能提升。

## 第一次尝试：使用 rbtree

我们启用自己的 MemTable: rbtree，基于红黑树的的 MemTable，红黑树最坏情况下的理论时间复杂度优于 SkipList，但是不支持并发写。

最终的测试结果有一点改善，从 17分钟降到了 15分钟，仍然比 InnoDB 的 12分钟要慢。

## 第二次尝试：使用动态 Patricia Trie

我们知道，Trie 是一种 DFA（确定性的有穷自动机），Patricia Trie 是附加了路径压缩的自动机，而自动机天生就是用来做匹配和搜索的。

并且，TerarkDB 中，TerarkZipTable 的索引就是用了 NLT(Nested Louds Trie)，更全面准确的名称，应该是：Nested Louds Succinct Patricia Trie。但 NLT 是静态的，创建之后不可修改，只能用在创建之后无需修改的 SST 上，不能用在需要动态插入数据的 MemTable 上。

早在多年前，我们就把自动机算法引入了常规的工程实践中，例如 [把自动机用作 Key-Value 存储](http://nark.cc/p/?p=172)，如果使用我们已有的技术，可以实现基于 DFA 的 MemTable，但是预期中的性能肯定不够高，内存占用也会比较大。

## 所以我们从头实现了一个 [Dynamic Patricia Trie](Dynamic-Patricia-Trie.html)
基于这个 Patrcia，我们实现了 PatriciaMemTable，设计上，我们的 Patricia 位于 Terark DFA 体系，和 RocksDB 完全独立，也就是说，PatriciaMemTable 把 [Dynamic Patricia Trie](Dynamic-Patricia-Trie.html) 作为一个基本构造块，PatriciaMemTable 是 [Dynamic Patricia Trie](Dynamic-Patricia-Trie.html) 到 RocksDB MemTable 的适配层。

## [Comparator 问题](Key-Comparator.html)
