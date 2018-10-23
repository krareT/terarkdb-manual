# **This solution was discarded because we have a more elegant solution.**

[中文版](读取-Value-数据的一部分.html)

## Preface
We are planed to implement a fuse filesystem by rocksdb, this filesystem is optimized for large number of small files.

A filesystem has two kinds of data: `meta data` and `file data`.
* `meta data` includes file mode, file size, file update time, creation time, even last access time, ...
* `file data` is the file content, even for small files, `file data` is much larger than `meta data`.

The `key` of the DB should be full path name of a file, and the `value` is `meta data` and `file data`.
`meta data` is accessed more often than `file data`, so we need to store `meta data` and `file data` separately.

A straight forward way is to store `meta data` and `file data` in different column families, or add different prefix for file name as `key`. But these approaches both incurred space overhead, mainly for keys.

We proposed a new solution for this problem:
* `meta data` is stored with `file data` together, they are concatenated as DB's `value`
* Add support for read partial data of `value`, with `API` and `ABI` compatible

## API changes for supporting read partial data of `value`
For performance reasons, struct/class data members are aligned, thus introduce paddings between ajacent data members. With careful design, such paddings can be minimized, but may lose code readability. `rocksdb` prefered code readabilty, so there are enough padding spaces in classes/structures.

If we add new data members at the padding space, we will not break the ABI(Application Binary Interface). We do so for  `rocksdb::ReadOptions`, with this new feature, users can use new `librocksdb.so`, and all existing application does not need to recompile, the behavior of API will keep unchanged.

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
* Unfortunately, new rocksdb version rearranged `ReadOption` fields to eliminate paddings, so new **terark rocksdb(v5.8+)** is not binary compatible with corresponding official rocksdb...

## Supporting Status
* Supported by both iterator and DB::Get.
* Supported by memtable (memtable common code)
* Supported by `TerarkZipTable`
* Not Supported by other SSTable

## Cooperate with Merge Operator
In the fuse filesystem example, we use merge operator to update `meta data`:
```c++
  rocksdb::WriteOptions wopt;
  db->Merge(wopt, pathname, meta_data)； // blind write, no need to touch `file data`.
```
In the merge operator, we just discard old `meta data`, and keep the newest `meta data`, when reading `meta data`, such as in directory iteration, it just need to read `meta data`:
```c++
  // ...
  rocksdb::ReadOptions ropt;
  ropt.value_data_offset = 0;
  ropt.value_data_length = META_LEN; // META_LEN is a constant
                                     // because meta data is fixed length
  db->Get(ropt, pathname, &meta_data);
  // ...
``` 

By this way, we can implement a fuse filesystem with mininal overheads.