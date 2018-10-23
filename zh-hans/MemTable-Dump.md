## 背景
MemTable **写满**之后会标记为 `Immutable` 并进行 `Flush`，这个机制看上去工作良好……

使用 Terark SST，一个 1G 的 MemTable，Flush 大约需要 10~20 秒。

使用 BlockBased Table，一个 64M 的 MemTable，Flush 大约需要 100 毫秒。

## PCIe SSD
企业级 PCIe SSD 的写性能可以达到 5GB/sec，读性能可以达到 6GB/sec，MemTable Flush 最快也只能达到这个性能的 10%。

## Terark MemTable
我们(Terark)实现了一个新的 MemTable：基于 Patricia Trie 的 MemTable，这个 MemTable 除了速度快以外，其数据结构的内存形式与持久化形式完全相同，从而，可以直接把内存 Dump 到硬盘，这个 Dump 的性能，是大块数据(比如1GB)的连续写入，可以达到 SSD 硬件的上限(比如5GB/sec)。

## 现实意义
MemTable 直接 Dump 成 SST，因为 Dump 很快，在高写入负载下，就不会出现 MemTable Flush 跟不上写入速度的情况了。从而，在 DB 重启时，(每个 ColumnFamily)只需要重放(replay) 一个 WAL Log，大大加快重启速度。