# Benchmark by db_bench

`db_bench` is in official rocksdb, many people are using `db_bench`.

To use `db_bench` of [TerarkDB's rocksdb](https://github.com/Terark/rocksdb), you need a little extra work:

```
env LD_LIBRARY_PATH=/path/to/terark-zip-rocksdb-xxx/lib:$LD_LIBRARY_PATH LD_PRELOAD=libterark-zip-rocksdb-r.so ./db_bench --benchmarks=fillrandom,readrandom
```

