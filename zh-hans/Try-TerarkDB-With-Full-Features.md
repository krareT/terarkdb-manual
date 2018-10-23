[[中文 | 完整功能]]

# Try TerarkDB With Full Features
Now you have compiled terark rocksdb and downloaded terark-zip-rocksdb, if not, see [here...](Home)

## 1. Introduction

## 2. Who Need This
- People who need more granular control on `SSTable` (Requires code changes in your application).
- People who want to know more details about `TerarkZipTable` (Our implementation of `SSTable`).

**NOTE:** If you just want to experience `TerarkDB` without code changes of your application, please refer to [[Quick Start | Try TerarkDB With Minimal Effort]]

## 3. RocksDB utility tools for TerarkDB

In this RocksDB fork, we add some extra options to use TerarkDB, e.g.:

|Utility|TerarkDB option|
|-------|--------------|
|ldb|`--use_terarkdb=1`|

### 4. TerarkDB Restrictions
- User comparator is not supported, you should encode your keys byte lexical order
- `EnvOptions::use_mmap_reads` must be `true`, can be set by `DBOptions::allow_mmap_reads`

## 5. RocksdDB Cautions & Notes
- **`table_factory`** is a member of **`ColumnFamilyOptions`**, not **`DBOptions`**!
- If calling **`rocksdb::DB::Open()`** with column families, you must set **`table_factory`** for each `ColumnFamilyDescriptor`
```
// Caution: When calling this `Open` overload, you must 
// set column_families[i].options.table_factory
//
// You may pass an rocksdb::Option object as db_options, but this
// db_options.table_factory will NOT be used!!
//
static Status Open(const DBOptions& db_options, const std::string& name,
                   const std::vector<ColumnFamilyDescriptor>& column_families,
                   std::vector<ColumnFamilyHandle*>* handles, DB** dbptr);
```

## 6. Using TerarkZipTable

### 6.1. Now just for C++

- Compile flags
```makefile
CXXFLAGS += -I/path/to/terark-zip-rocksdb/src
```
- Linker flags
```makefile
LDFLAGS += -L/path/to/terark-zip-rocksdb-lib
LDFLAGS += -lterark-zip-rocksdb-r
LDFLAGS += -lterark-zbs-r -lterark-fsa-r -lterark-core-r
```

- C++ code

```c++
#include <table/terark_zip_table.h>
/// other includes...

  ///....
  TerarkZipTableOptions opt;

  /// TerarkZipTable needs to create temp files during compression
  opt.localTempDir = "/path/to/some/temp/dir"; // default is "/tmp"

  /// 0 : check sum nothing
  /// 1 : check sum meta data and index, check on file load
  /// 2 : check sum all data, not check on file load, checksum is for
  ///     each record, this incurs 4 bytes overhead for each record
  /// 3 : check sum all data with one checksum value, not checksum each record,
  ///     if checksum doesn't match, load will fail
  opt.checksumLevel = 3; // default 1

  ///    < 0 : only last level using terarkZip
  ///          this is equivalent to terarkZipMinLevel == num_levels-1
  /// others : use terarkZip when curlevel >= terarkZipMinLevel
  ///          this includes the two special cases:
  ///                   == 0 : all levels using terarkZip
  ///          >= num_levels : all levels using fallback TableFactory
  /// it shown that set terarkZipMinLevel = 0 is the best choice
  /// if mixed with rocksdb's native SST, those SSTs may using too much
  /// memory & SSD, which degrades the performance
  opt.terarkZipMinLevel = 0; // default

  /// optional
  opt.softZipWorkingMemLimit = 16ull << 30; // default
  opt.hardZipWorkingMemLimit = 32ull << 30; // default

  /// to let rocksdb compaction algo know the estimate SST file size
  opt.estimateCompressionRatio = 0.2;

  /// the global dictionary size over all value size
  opt.sampleRatio = 0.03;
 
  /// other opt are tricky, just use default

  /// rocksdb options when using terark-zip-rocksdb:

  /// fallback can be NULL
  auto fallback = NewBlockBasedTableFactory(); // or NewAdaptiveTableFactory();
  auto factory = NewTerarkZipTableFactory(opt, fallback);
  options.table_factory.reset(factory);

  /// terark-zip use mmap
  options.allow_mmap_reads = true;

  /// universal compaction reduce write amplification and is more friendly for
  /// large SST file, terark SST is better on larger SST file.
  /// although universal compaction needs 2x SSD space on worst case, but
  /// with terark-zip's high compression, the used SSD space is much smaller
  /// than rocksdb's block compression schema
  options.compaction_style = rocksdb::kCompactionStyleUniversal;

  /// larger MemTable yield larger level0 SST file
  /// larger SST file make terark-zip better
  options.write_buffer_size     =  1ull << 30; // 1G
  options.target_file_size_base =  1ull << 30; // 1G

  /// single sst file size on greater levels should be larger
  /// filesize(level[n+1]) = filesize(level[n]) * target_file_size_multiplier
  options.target_file_size_multiplier = 2; // can be larger, such as 3,5,10

  /// turn off rocksdb write slowdown, optional. If write slowdown is enabled
  /// and write was really slow down, you may doubt that terark-zip caused it
  options.level0_slowdown_writes_trigger = INT_MAX;
  options.level0_stop_writes_trigger = INT_MAX;
  options.soft_pending_compaction_bytes_limit = 0;
  options.hard_pending_compaction_bytes_limit = 0;
```
