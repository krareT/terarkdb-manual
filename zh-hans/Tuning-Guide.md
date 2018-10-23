[中文 Chinese](TerarkDB-参数调节.html)
# TerarkDB Tuning Guide
There are some best practices you should know about TerarkDB. We will add those information ASAP, meanwhile if you need help you can post an [issue](https://github.com/Terark/terarkdb/issues) in this repository.

## 1. SSTable Size

SSTable size is controlled by 2 options:
- `ColumnFamilyOptions::target_file_size_base`
- `ColumnFamilyOptions::target_file_size_multiplier`

In most cases, you can keep TerarkDB's configuration as default, it will work just fine. But if you meet cases like these:
- Your record `key` is related to the insert time (mostly in time series DB).
- Your record `value` is ordered by time while inserting.

You may need to use a smaller SSTable size, since client users are likely to delete data that is in the same SSTable, which will frequently causes whole SSTable deletions.

## 2. Write triggers
|config name|RocksDB default|
|----------:|:------:|
|level0_file_num_compaction_trigger|4|
|level0_slowdown_writes_trigger|20|
|level0_stop_writes_trigger|36|

### `level0_file_num_compaction_trigger`
Number of files to trigger level-0 compaction. A value <0 means that level-0 compaction will not be triggered by number of files at all.
### `level0_slowdown_writes_trigger` and `level0_stop_writes_trigger`
TerarkDB is faster on read, and slower on write, we should pay read for write, so these 2 config for TerarkDB should be larger than RocksDB.

