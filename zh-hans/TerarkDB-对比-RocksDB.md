[English](TerarkDB-vs-RocksDB.html)
## 简单概括
1. API 接口兼容 RocksDB
1. 随机读更快
1. 压缩率更高（磁盘空间占用）
1. 内存用量更小（对压缩文件使用 mmap, 没有双缓存问题）
1. 快速启动（DB::Open(ExistingLargeDB) 的速度远快于 RocksDB）

## 技术实现
1. 对 RocksDB 本身做了少量修改
1. 替换了 RocksDB 的 SST(Static Sorted Table) 实现，即 TerarkZipTable，或者叫 Terark SST
1. 替换了 RocksDB 的 MemTable 实现，使用 Patricia Trie

## TerarkZipTable
`TerarkZipTable` 是将 Terark 特有的压缩技术 CO-Index 和 PA-Zip，适配到 RocksDB 的 SST 接口。从而将技术落地到工程：兼容整个 RocksDB 生态。

1. CO-Index 全称 Compressed Ordered Index
主要使用 Nest Succinct Trie 实现，用来压缩 Key 集合，可以直接在压缩的数据上搜索，将 Key 映射到一个内部的整数 ID

2. PA-Zip 全称 Point Accessible Zip
是一种全局压缩技术，用来压缩 Value 集合，可以直接在压缩的数据上根据整数 ID 定点提取单条 Value

## Patricia Trie MemTable
基于 Copy On Write 的 [Dynamic Patricia Trie](Dynamic-Patricia-Trie.html)，典型场景下，算法级裸性能提高 5 倍，内存用量降低 3 倍(相同 MemTable 尺寸可装入 3 倍数据)。集成进 RocksDB后（性能倍被 RocksDB 框架拉低），总体性能提高约 40%。

Patricia Trie MemTable 可以直接 Dump 文件，从而 MemTable Flush 的性能获得大幅提升。

[MemTable 的这些改进](重新实现-RocksDB-MemTable.html)，需要同时修改 RocksDB 自身，相关 Pull Request: [PR ####]()。

## 参数调整及优化
1. 使用 universal compaction，减小写放大
1. 扩大 memtable，减小 L0 to L0 compact
1. 如果写入量很高，可以仅对底层 Level(level 数字更大) 的 SST 使用 Terark，高层 Level 使用 RocksDB 原生 SST
   * 默认所有 Level 都使用 Terark ，主要是为了平衡读写性能，因为高层 Level 一般访问更频繁，使用 Terark SST 内存消耗更小，从而改善总体性能

## TerarkDB 不需要 Bloom Filter
Bloom Filter 用来做否定测试，也就是说，如果 Bloom Filter 搜索 Key 失败，则该 Key 一定不存在，如果 Bloom Filter 搜索成功，该 Key 可能存在，也可能不存在。

Bloom Filter 的优点是占用空间很小（一般情况下平均每个 Key 1~2 字节），搜索速度又很快，当然缺点就是它只能确认 Key 不存在。

### 传统 SST 使用 Bloom Filter 来加速高层 Level 中 SST 的搜索
高层（Level 值更小的 Level）的数据尺寸更小，并且都是新数据，访问更频繁，未命中的概率更高，而传统索引搜索速度较慢，内存占用高，所以，用 Bloom Filter 可以实现很好的加速效果，并且相对内存占用较低。

然而，Terark SST 用了 CO-Index，CO-Index 的压缩率和性能都非常高，和 Bloom Filter 相比，甚至在很多情况下空间和搜索性能同时占优，而且还是确定性的搜索（存在就是存在，不存在就是不存在），所以 Terark SST 就完全舍弃了 Bloom Filter。

更多信息，请参考 [TerarkDB 文档](首页)

