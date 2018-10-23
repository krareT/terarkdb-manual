[中文](快速开始.html)

# Try TerarkDB With Minimal Effort
Now you have compiled `terark rocksdb` and downloaded `terark-zip-rocksdb`, if not, see [here...](Home)

## 1. Introduction
Now you can start trying `TerarkDB` by just setting a few environment variables, this is the simplest way is to use `TerarkDB`.

Since `Terark RocksDB` is binary compatible with `Official RocksDB`, you don't even need to recompile existing rocksdb applications. -- **Note:** `Official RocksDB` is not binary compatible among different revisions, now we are binary compatible to [rocksdb 5.9.2](https://github.com/facebook/rocksdb/releases/tag/v5.9.2)

## 2. Who Need This
- If you want to experience TerarkDB in the quick way
- If you have an existing RocksDB application and don't want to change any existing code
- If you have an existing RocksDB application and don't want to recompile
- If you don't need more granular control of TerarkDB

## 3. Try TerarkDB

To enable Terark algorithms, you need to set these environment variables:

```bash
# If you have installed files in `terark-zip-rocksdb-XXX/lib` into
# system lib path(such as /usr/lib64), `LD_LIBRARY_PATH` is not needed.
export LD_LIBRARY_PATH=terark-zip-rocksdb-XXX/lib:$LD_LIBRARY_PATH

env LD_PRELOAD=libterark-zip-rocksdb-r.so \
    TerarkZipTable_localTempDir=/path/to/some/temp/dir \
    TerarkZipTable_indexNestLevel=2 \
    TerarkZipTable_indexCacheRatio=0.005 \
    TerarkZipTable_smallTaskMemory=1G \
    TerarkZipTable_softZipWorkingMemLimit=16G \
    TerarkZipTable_hardZipWorkingMemLimit=32G \
    app_exe_file app_args...
```

For these variables only `LD_PRELOAD` and `TerarkZipTable_localTempDir` are required, others could be left as default, it shall work well in most cases.

For more details, see [TerarkZipTableOptions](https://github.com/terark/terark-zip-rocksdb/blob/master/src/table/terark_zip_table.h#L17).

#### 3.3.1. Full List of Options
All options defined by env var have the prefix `TerarkZipTable_`. For example, for option `localTempDir`, the env variable name is `TerarkZipTable_localTempDir`.

**1) Options defined in class `TerarkZipTableOptions`**

|type|env var `suffix` or<br/>TerarkZipTableOptions::<br/>`member`|Default|Description|
|----|-------|-----------------------|----|
|string|localTempDir|*|For temp files|
|enum<br/>string|entropyAlgo|`NoEntropy`|Space gain is little,<br/>Speed lose is great|
|string|indexType|`IL_256`|For rank-select implementaion|
|int|checksumLevel|3|`3` for checksum on all data|
|int|indexNestLevel|3|Large value reduce index size,<br/>but slowdown index speed|
|int|indexNestScale|8|Large value reduce index size,<br/>but slowdown index speed|
|int|indexTempLevel|0|Large value reduce memory usage for index building, but slowdown index building speed<br/>`-1`: disable indexing temp file<br/>` 0`: be smart, use temp file for large index building|
|int|terarkZipMinLevel|0|Default `0` is good enough|
|int|keyPrefixLen|0(disabled)|For mongorocks/myrocks...|
|int|offsetArrayBlockUnits|0(disabled)|For offset compression|
|bool|disableSecondPassIter|false|`false` reduce disk usage,<br/>but slowdown compact speed|
|bool|useSuffixArrayLocalMatch|false|`true` slowdown compress|
|bool|warmUpIndexOnOpen|true|Avoid pagefault on index search|
|bool|warmUpValueOnOpen|false|Avoid pagefault on value read|
|bool|enableCompressionProbe|true|Probe data compressibility, if not compressible, compression will be disabled|
|float|estimateCompressionRatio|0.2|For reporting estimated output SST file size|
|float|sampleRatio|0.03|sampleRatio for global value compression|
|float|indexCacheRatio|0|typically `0`, or `0.001`|
|int64|softZipWorkingMemLimit|SysTotal/8|Limit zip working memory|
|int64|hardZipWorkingMemLimit|SysTotal/4|...|
|int64|smallTaskMemory|SysTotal/4|...|

**2) Options defined in class `ColumnFamilyOptions`**

|type|env var `suffix` or<br/>ColumnFamilyOptions::`member`|Default|
|----|-------|-----------------------|
|enum<br/>string|compaction_style|universal|
|int|num_levels|5|
|int64|write_buffer_size|SysTotalMem/32|
|int|max_write_buffer_number|3|
|int64|target_file_size_base|SysTotalMem/2|
|int|target_file_size_multiplier|1|
|int|level0_file_num_compaction_trigger|rocksdb<br/>default|

**3) Options defined in class `DBOptions`**

|type|env var `suffix` or <br/>DBOptions::`member`|Default|
|----|-------|-----------------------|
|int|base_background_compactions|3|
|int|max_background_compactions|5|
|int|max_background_flushes|3|
|int|max_subcompactions|1|

**4) Options defined in `ColumnFamilyOptions::compaction_options_universal`**

|type|env var `suffix` or <br/>columnFamilyOptions.compaction_options_universal.`member`|Default|
|---|---------------|-|
|int|min_merge_width|4|
|int|max_merge_width|50|

**5) Other options**

|type|env var full name|Default|
|----|-------|-----------------------|
|string|`TerarkZipTable_blackListColumnFamily`|`empty`|
|int|`DictZipBlobStore_zipThreads`|min(8, physical\_cpu\_num)|

- `TerarkZipTable_blackListColumnFamily`
  - Since once env `TerarkZipTable_localTempDir` is defined, `TerarkZipTable` will take preference
  - Users can put some column families into black list, these column families will not using `TerarkZipTable`
  - Multiple column families should be separated by ',' such as:<br/>
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`TerarkZipTable_blackListColumnFamily=cf1,cf2,cf3`
  - Users may want to put some `log data` or data with short `TTL` into the black list

- `DictZipBlobStore_zipThreads`
  - If this var is not `0`, TerarkDB's SST builder using a pipeline `read -> compress -> write` to do the compress, this pipeline is shared by all SSTs builder, this var is the thread num of `compress` stage in the `read -> compress -> write` pipeline.
  - If this var is larger than physical CPU number, it is reset to the physical CPU number.
  - If this var is `0`, TerarkDB will not use `pipeline`, and each SST's compression is running in the calling thread. 

### 3.4. Launch Your Application
That's it, now you can Launch your application and try everything you want.


## 4. Background Knowledge
We implemented a `SSTable` of RocksDB, and named it as `TerarkZipTable`. With the environment variables in previous section, we can actually make RocksDB use our version of `SSTable`. All of our algorithms are packed into `TerarkZipTable` and don't impact existing RocksDB `SSTable`. You can use both side by side.


## 5. More
`TerarkZipTable` actually provides more granular control on TerarkDB. If you want to know more about it, you need code changes for configuring TerarkDB and re-compile your application, refer to [Try TerarkDB With Full Features](Try-TerarkDB-With-Full-Features.html).
