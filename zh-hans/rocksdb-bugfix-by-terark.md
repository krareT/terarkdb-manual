# 我们修改的 RocksDB bug

对外，TerarkDB 使用 rocksdb 作为 API，去兼容各种基于 rocksdb 的系统和应用。

对内，TerarkDB 作为 rocksdb 的一个 SSTable 实现，plugin 到 rocksdb。

不同的是，TerarkDB 默认使用 universal compaction，而 rocksdb 原版默认使用 level compaction。虽然总体上，rocksdb 的代码质量比较一般，但是 level compaction 经过了严酷的测试和广泛的实际应用，从而 bug 更少，特殊优化更多。

rocksdb 发展至今，绝大部分使用场景都是以下组合:

| 组件 | 实现 |
|--------|------------------|
|  MemTable | skip list |
|   SSTable | block based table |
|compaction style|level compaction|

## universal compaction 的 allow\_trivial\_move
`CompactionOptionsUniversal::allow_trivial_move` 表示，在 compact 时，如果碰到不重叠的文件，直接把它 move 到相应的位置（目标
 Level 的某个位置），这个值默认是 false，最终会传递给 `Compaction::is_trivial_move_`，我们把它设成了 true，这就是一个噩梦的开始……

在某些条件下，我们发现数据库总是出错，最后定位到是某个 Level 的 SST 顺序乱了，经过若干个不眠夜，我们最终发现，是因为
 `Compaction::is_trivial_move_` 没有初始化……

很低级的 Bug，但是一直在 RocksDB 中存在了好几年，可能是因为使用 Universal Compaction 的人确实太少了！

## 已关闭的文件句柄
在函数 `PosixEnv::NewRandomAccessFile` 中，把打开的 fd 传播到了 `PosixMmapReadableFile::fd_`，但是，该 fd 在函数返回之前就被关闭了，也就是说，通过 `PosixMmapReadableFile::fd_` 长期持有的 fd 是个已关闭的 fd，这个 bug 在原版 rocksdb 中甚至算不上是一个 bug，因为 `PosixMmapReadableFile::fd_` 实际上没有被使用过……

但是，我们为了在 mmap 时仍然可以使用 pread 去读数据，就真的使用了这个 fd，然后悲剧就发生了……

## MergingIterator 的 AddIterator
Code : [MergingIterator::AddIterator](https://github.com/facebook/rocksdb/blob/15f55e5e06feb931fe951a4a57092859d2a6d72e/table/merging_iterator.cc#L67-L71)  
栈对象取指针存入容器，children添加元素导致所有元素指针失效。  
两个C++低级错误！

## 在 Compact 过程中推迟 SST 的加载
在 Compact 过程中，每生成一个 SST 文件，RocksDB 会加载该 SST 并放入 SST Cache，但是，直到整个 Compact 结束，该 SST 才会真正被访问，在 Compact 期间，该 SST 除了浪费内存以外，没有任何用处。

我们把 SST 的加载推迟到整个 Compact 结束时：[PR 3979](https://github.com/facebook/rocksdb/pull/3979)