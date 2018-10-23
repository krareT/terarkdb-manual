# **This solution was discarded because we have a more elegant solution.**

[English](Partial-Read-on-value-data)

## 前言
我们计划通过 RocksDB 做一个 Fuse 的实现，该实现将针对大量的小文件进行优化。

一个文件系统中，一般存有两类数据：元数据(Meta Data) 和 文件数据(File Data).

- 元数据包含文件类型，文件尺寸，更改时间，创建时间等信息
- 文件数据指文件的内容，即便是对小文件，文件数据也比源数据要大很多

在 DB 中，`key` 应该是文件的全路径，`value` 则是文件元数据(meta data)和文件数据(file data)。元数据(meta data)比文件数据(file data)的访问更加频繁，所以我们需要将元数据和文件数据分开来存储。

比较直接的方法是将元数据和文件数据存储与不同的 ColumnFamily 中，或者通过给文件名增加一个额外的前缀作为新的 `key`来单独标示元数据，但这两种方法都需要额外的存储空间（主要是针对 key）。

我们考虑通过以下的方式来实现这个功能：
* 元数据和文件数据连接在一起存储在 DB 的 `value` 中
* 支持对 `value` 的部分读取， 同时保持 API 和 ABI 的兼容

## 让 API 支持对 `value` 的部分读取

基于性能方面的考虑，`struct/class` 的数据成员都是对齐的，也就是在相邻的数据成员中间可能存在内存空隙。经过精心的设计，这些空隙能够被最小化，同时也会失去一些可读性。`RocksDB` 倾向于代码的可读性，所以为我们留下了足够多的空隙。

如果我们在内存空隙中添加新的数据成员，我们不会损失 ABI 的兼容性，我们针对 `rocksdb::ReadOptions` 进行了这项设计，具备新的功能之后，用户使用新的 `librocksdb.so` 并不需要重新编译，当然 API 也保持不变。

If applications changed new data member of `rocksdb::ReadOptions`,  our new feature will take effect. We add two data members on `rocksdb::ReadOptions`:

```c++
struct ReadOptions {
  bool verify_checksums;
  bool fill_cache;
  uint32_t value_data_offset; // read value data from this offset
  const Snapshot* snapshot;
  const Slice* iterate_upper_bound;
  ReadTier read_tier;
  bool tailing;
  bool managed;
  bool total_order_seek;
  bool prefix_same_as_start;
  bool pin_data;
  bool background_purge_on_iterator_cleanup;
  size_t readahead_size;
  bool ignore_range_deletions;
  uint32_t value_data_length; // read at most such length of value data
  ReadOptions();
  ReadOptions(bool cksum, bool cache);
};
```
We stripped comments in the code snippet, our added new fields are:
```c++
  uint32_t value_data_offset; // read value data from this offset
  uint32_t value_data_length; // read at most such length of value data
// These two fields are initialized in constructor:
  value_data_offset = 0;
  value_data_length = UINT32_MAX;
```

## 功能支持
* `iterator` 和 `DB::Get` 均支持该功能.
* `memtable` 支持该功能
* `TerarkZipTable` 支持该功能
* 其他 `SSTable` 不支持