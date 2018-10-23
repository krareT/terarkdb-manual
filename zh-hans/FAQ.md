[中文](常见问题.html)

# FAQ

## 1. How is TerarkDB related with RocksDB?
TerarkDB implemented a SSTable with Terark’s algorithms and data structures, and gains much better random read performance and much better compression ratio.

## 2. Does TerarkDB change RocksDB code?
Yes, TerarkDB changed a little bit of RocksDB code:
- Add two-pass scan capability to reduce disk/ssd usage during Terark SSTable building(This does not impact existing SSTables).
- Add TerarkDB config by environment var(users can replace official librocksdb.so by TerarkDB’s librocksdb.so and set env var to enable TerarkDB’s SSTable).

## 3. How is TerarkDB compatible with RocksDB?
TerarkDB doesn’t change any user API of RocksDB, from the user’s perspective TerarkDB is 100% compatible with RocksDB, this compatibility is on ABI level, user applications even need not to recompile.

## 4. How is TerarkDB compatible with MongoDB and MySQL?
TerarkDB is compatible with MongoDB and MySQL through MyRocks and MongoRocks.

## 5. Since the core algorithm of TerarkDB is proprietary, how to be compliant with MongoDB’s AGPL license?
- All Terark code related to MongoDB is open source (MongoDB itself and MongoRocks), these do not use any TerarkDB code or API.
- The core of TerarkDB is a plug-in for RocksDB, so it is not directly applied into MongoDB -- it is loaded as a dynamic library for librocksdb.so, and TerarkDB is compliant with RocksDB’s license.

## 6. How much memory is needed during compression?
- For key, the peak memory is about total_key_length * 1.3 during compression.
- For value, the memory usage is about input_size*18%, and never exceeds 12G during compression.

To give you an example, to compress 2TB of data, in which "keys" are 30GB:
- TerarkDB would need ~39GB of RAM to compress the keys (into index).
- TerarkDB would need ~12GB of RAM to compress the values.
- Key compressing and Value compressing can be in parallel if the memory is sufficient, and in serial if the memory is not sufficient.

## 7. Why use universal compaction, instead of level compaction?
Level compaction has larger write amplification, resulting slower compression speed and shorter SSD lifetime.

We also improved 'universal compaction' than RocksDB's native version.

If TerarkDB use " level compaction", all things will work well.

## 8. TerarkDB is faster in random read and smaller in compressed data size, but what is the trade-off?
The compression speed is slower, and use more memory during compression. This does not impact instant writes. SINGLE thread compression speed is about 15~50 MB/s, so multiple threads are used for compression.

## 9. What is the lock mechanism? Will TerarkDB lock the whole SSTable during the compression?
TerarkDB has nothing to do with lock, SSTable in RocksDB itself is read-only after compaction, and write-only during compaction. So we did not make any change on locking.

## 10. What happens if a huge SSTable needs to be compressed? (e.g. more than 5TB)
RocksDB’s SStable will not be that large. 

TerarkDB supports data of huge size like that, but each SSTable is not necessarily that large, a single SSTable is hundreds of GB at most.

## 11. What is the max size of a single SSTable?
It depends, usually the max number of keys in an SSTable can not exceed 1.5 billions, and the total length of all keys can be much larger(practically 30GB should be safe). 

For values, the total length of compressed values can not exceeds 128PB(practically this limit will never be hit), the number of values is the same as the number of keys.

People will unlikely to generate such large single SSTable and hit the limit, and TerarkDB will finish such large SSTable and create a new SSTable.

## 12. What is the random and sequential read speed?
It depends on the data set and memory limit.

On TPC-H lineitem data with unlimited memory (row len 615 bytes, key len 23 bytes, text field length 512 bytes), for a SINGLE thread:

- For key, ~10MB/s on Xeon 2630 v3, up to ~80MB/s on a different data set.
- For value, ~500 MB/s on Xeon 2630 v3, up to ~7GB/s on a different data set.

As a general rule, below showing the relative comparable magnitudes of read speed:
- When Memory is <strong>slightly</strong> limited(all data fit in memory by TerarkDB, not by Other DBs)
<table>
<tr>
  <th>DB</th>
  <th>Read mode</th>
  <th>Speed</th>
  <th colspan="2">Description</th>
</tr>
<tr>
  <th rowspan="2">TerarkDB</th>
  <td>sequential</td>
  <td align="right">8000</td>
  <td rowspan="2">all data exactly in memory<br/>no page cache miss</td>
  <td>CPU Cache & TLB miss is <strong>medium</strong></td>
</tr>
<tr>
  <td>random</td>
  <td align="right">4000</td>
  <td>CPU Cache & TLB miss is <strong>heavy</strong></td>
</tr>
<tr>
  <th rowspan="2">Other DB</th>
  <td>sequential</td>
  <td align="right">10000</td>
  <td colspan="2">hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">10</td>
  <td colspan="2">no hot data, heavy cache miss</td>
</tr>
</table>

- When Memory is <strong>extremely</strong> limited

<table>
<tr>
  <th>DB</th>
  <th>Read mode</th>
  <th>Speed</th>
  <th>Description</th>
</tr>
<tr>
  <th rowspan="2">TerarkDB</th>
  <td>sequential</td>
  <td align="right">5000</td>
  <td>hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">100</td>
  <td>no hot data, heavy cache miss</td>
</tr>
<tr>
  <th rowspan="2">Other DB</th>
  <td>sequential</td>
  <td align="right">10000</td>
  <td>hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">1</td>
  <td>no hot data, <strong>very</strong> heavy cache miss</td>
</tr>
</table>

## 13. What is the compression (write) speed?
It depends on the data set.

On TPC-H lineitem data (row len 615 bytes, key len 23 bytes, text field length 512 bytes), for a SINGLE thread:

- For key, ~9MB/s on Xeon 2630 v3, up to ~90MB/s on a different data set.
- For value, ~40 MB/s on Xeon 2630 v3, up to ~120MB/s on a different data set.

## 14. What is the compression ratio?
It depends on the data set.

On wikipedia data, all english text of ~109G, is compressed into ~23G.

On TPC-H lineitem of ~550G, keys are ~22G, values are ~528G. Average row len is 615 bytes, in which key is 23 bytes, value is 592 bytes, in which the configurable text field is 512 bytes:

- All keys are compressed into ~5.3G.
- All values are compressed into ~24G. (So the SSTable size is 29.3G)
 
## 15. How does the length of key/value impact the compression ratio?
- For key, < 80 bytes will be best fit, and of course >80 Bytes still works.
- For value, between 50 bytes and 30KB will be best fit, and it works for other length too.

## 16. Are there any more details(e.g. pseudo code) about TerarkDB?
the high level pseudo code:
```c++
int id = indexStore.find(key); // indexStore is mmap'ed
if (id >= 0) {
  // metadata is mmap, content is mmap or pread or by user cache
  value = valueStore.get(id);
}
```
## 17. Have you run any benchmarks comparing just the PA-Zip algorithm with other standard compression algorithms (particularly LZ77) for parameters such as compression ratio, compression/decompression performance.

If we ignore the Point Accessible feature, and only compare compression capabilities.
For different data set, the metrics would be vary, here is a rough metric(on [Amazon movie data](https://snap.stanford.edu/data/web-Movies.html)):

| compression<br/>ratio | compression<br/>speed | decompression<br/>speed | Point<br/>Accessible |
|:-----:|:---:|:-----:|:----:|
|PA-Zip | 5x | 3x | 50x| Yes |
|bzip2  | 5x | 1x | 1x | No  |
|gzip   | 3x | 3x | 3x | No  |
|snappy | 2x |15x |25x | No  |
|zstd   | 3x |10x |15x | No  |

## 18. How does compression ratio and performance compare to (for example) using something like Zstandard and training it on a large portion of a given table and using it to compress each row.

We had benchmarked zstd with training mode, the trained result(dictionary) is too small and can not be large enough. zstd compression speed with such training mode is much slower(compress each record with pre-trained dictionary) than PA-Zip, and the compression ratio is worse.

Almost all traditional compression algorithms support such kind of "training mode", and rocksdb also supports such feature (it just sampling fixed size fragments as dictionary):
Set `rocksdb::CompressionOptions::max_dict_bytes` (in `options.h`) to a nonzero value indicating the maximum per-file dictionary size.

For decompression(Point Access), zstd is also much slower than PA-Zip.

PA-Zip is dedicated for Point Access, we hadn't find any other algorithm has such feature.

## 19. Is PA-Zip only addressable or does it allow more?
Yes, PA-Zip does not support more features, For example:
<table>
<tr>
<th align="left">
Does it supprot regex search?<br/>
(I don't mean search all records one by one!)
</th>
<td>
PA-Zip does not support regex search.<br/>
CO-Index support regex search!
</td>
</tr>
<tr>
<th align="left">
 Is it only addressable<br/>("seekable extract functionality")?
</th>
<td>
Yes! It just does what it can do well:<br/><strong>Point Access</strong> -- extract a record by an integer id
</td>
</tr>
</table>


## 20. In your estimate, how much of the improved performance and compression ratio is due to CO-index and how much is due to PA-Zip?

It depends, if the "value" is relatively smaller, CO-Index contributes more, if the "value" is relatively larger, PA-Zip contributes more. As a general perspective:

Point Search on CO-Index: 20MB/s, if average keylen is 20 bytes, the QPS is 1M.
Point Access on PA-Zip: 500MB/s, if average keylen is 500 bytes, the QPS is 1M.
Comprise the CO-Index and PA-Zip above, the QPS is 500K. (single thread)

## 21. Can TerarkDB efficiently handle many DB instances(such as 1000 instances) on a single machine?

Technically, running many DB instance on a single machine is a very bad design, there is no any DB can efficiently handle such cases, esp. for TerarkDB, TerarkDB is highly optimized for big database instance, and is likely worse than other DB on very small DB instances(e.g. smaller than 1GB).

The real requirement which using many DB instances is to make a `namespace`, a good solution should be **Using PrefixID**:<br/>
&nbsp;&nbsp;&nbsp;Databases based on RocksDB(such as MyRocks and MongoRocks) using `PrefixID` to emulate DB namespace, `PrefixID` schema can handle millions of DB namespaces.


