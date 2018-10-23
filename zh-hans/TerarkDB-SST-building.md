[[中文 Chinese|TerarkDB SST 的创建过程]]
## Background
We know, for a (Key, Value) store, in most cases, key length is much smaller than value length, and key index is required to search values by keys.

A core concept of TerarkDB is [CO-Index] and [PA-Zip]:
* **CO-Index**: **C**ompressed **O**rdered **Index**, mapping a `ByteArray` key to an integer *id*, the *id* is used to access PA-Zip.
* **PA-Zip**: **P**oint **A**ccessible **Zip**, an abstract `array<ByteArray>` which support random access an array item by its integer *id*, used for value compression. 

For speed and memory usage, in SST building, we put all key data of CO-Index in memory, and just put critical data(dictionary) of PA-Zip in memory, this is the basic memory control(Memory control 1).

## TerarkDB needs two-pass scan on input data
During SST building, RocksDB iterate the input and send each item by `SSTBuilder.Add(Key,Value)`, when finishing a SST, `SSTBuilder.Finish()` will be called. That is to say: the input sequence can only be read once(one pass)! 

To read the input sequence again(second-pass scan), we need some extra work to do. At the beginning, we write input sequence to a temporary file in the first-pass scan, to read the input sequence second-pass(in `SSTBuilder.Finish()`), we just read the temporary file.

Accurately, the second pass scan is just need to scan the `value` part, so the temporary file for second pass scan just contains `value`.

## Memory control 2
We always write key data to a file during first pass scan, this file will be read into memory when the index building is scheduled:
* If configured memory is sufficient, the index building will be executed at once.
* Else the index building will be `delayed(by queuing)` until memory is sufficient.

## Trivial two-pass scan without temporary file
Writing value sequence to temporary file works, but requires disk/SSD space, when output SST is large, the temporary space usage is larger. This issue is much more serious to us, because our SST file is much larger(in GB) than RocksDB's native SST(in MB), this is unacceptable!

In RocksDB world, there is no idea, so we need to hack RocksDB, reading and hacking RocksDB is full of pain...

Then we got the solution: create a dedicated iterator to perform the second-pass scan, this iterator is a deep clone of native RocksDB's first-pass scan iterator, all helper objects `Merger`, `RangeDelAggregator`... must be cloned altogether， but `CompactionFilter` must not be cloned!

## Iterator is too slow...
It seems we got a perfect solution: Now we can build our SST with limited memory and disk/SSD space, and...
* Terark SST is much smaller than RocksDB native SST in SST file size
* Terark SST is much smaller than RocksDB native SST in query time memory usage
* Terark SST is much faster than RocksDB native SST on random point search

The good days ends when we encountered a table with many indices in MyRocks...

MyRocks produce *n* (Key, Value) pairs per row for a table with *n* indices(including primary index), and (Key, Value) pairs for all secondary indices entry has empty `Value` part.

In RocksDB compaction, multiple sorted runs(1 sorted run can have *many* files) are **merged** into one sorted run.

The input iterator is a multi-run merging(e.g. by a heap of keys, per-key-per-run), and the merging operation is quite slow:
* When the value length is much larger than key length, the overall throughput is dominated by the value, thus the slowness of merging is not significant.
* When the value length is small(e.g. empty in MyRocks index), the overall throughput is dominated by the merging, the slowness is very extraordinary.

The second-pass scan/merge performs same as first-pass scan/merge, the slowness becomes 2x slower!

## Omit two-pass scan for small value
On these situations, number of (Key,Value) pair is very large, and the length of value is very small.

So, first we need to define "`small values`", this is simple: if the average value length is less than the average key length, we say these values are "small values".

The next job is to recognize small value!

### Recognize small values
To recognize small values, the straight forward way is to scan all (key,value) pairs and compute the average length. But our goal is to omit the second-pass scan:
* If only compute lengths in first-pass scan, the second-pass scan is still required.
* If we write all values to file in first-pass scan, the space usage is too large.

How should we do? This problem had not taken our much time -- As long as we realized this problem, the solution will be near us...

**Speculatively writing value data**:
At the beginning of first-pass scan, we write the values along with the keys, until **the accumulate values length is up to a configured size(such as 5MB)** and the scanned average value length is larger than key length. The **bold** text is what the **Speculation** means.

A very simple strategy, but it is very efficient!

### Add `Seek()` to `CompactionIterator`

In most cases, each output SST only has one physical (Key,Value) store, the above solution is sufficed. But there are more complicated scenarios:

In MyRocks and MongoRocks(and other RocksDB based database):
* Each key is prefixed by an uint32 id which identifier a subset
* Each subset is for an index/table
* An SST can have many subset: data of same subset is adjacent in key space.

When subset1 is finished in first-pass scan(next larger prefix id is met, which is for subset2), and subset1 satisfing `small value` condition, when subset2 is finished later and subset2 does not satisfing `small value` condition, then subset2 needs second pass scan.

Now we know, in the input sequence, subset1 is before subset2, and second pass scan needs to be omited for subset1 and kept for subset2. So `skip` is required for `CompactionIterator` which is used for scanning the input sequence.

We realize `skip` feature by `Seek`, although functionality of `Seek` is stronger than `skip`, the implementation effort and performance is the same.

The algorithm for `Seek` is simple, but the implementation detail is nasity: `UserKey`, MVCC `SeqNum` ... must be correctly handled.

## Concurrent index building
*subset1 and subset2 in this section is the same as the above*.

When subset1 is finished in first-pass scan(a different prefix id is met), if memory is sufficient at this time, the index builder for subset1 can be concurrently executed in another thread while the first-pass scan (for subset2...) is continuing in current thread.

## Ideal concurrent execute first & second pass scan
If memory is sufficient, first & second pass scan can be executed in a pipeline manner:
* We know iterator1 for first pass scan, iterator2 for second pass scan, and iterator2 is after iterator1
* If subset1 is finished in first-pass scan, second-pass scan on subset1 can be started in another thread while first-pass scan of subset2 is running in current thread

But this solution can only be realized in an ideal world, RocksDB has a customizable/plugin-able `CompactionFilter`, which may destroy the ideal world(This is shown in MyRocks and MongoRocks).

