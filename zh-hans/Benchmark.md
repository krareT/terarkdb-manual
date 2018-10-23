# Benchmark

We are continuously providing benchmark results, here are some of them:

## 1. TerarkDB Benchmark

- [Benchmark TPCH Overview](Benchmark-TPCH-Overview.html)
- [Benchmark Amazon Movies](Benchmark-Amazon-Movies.html)
- [Benchmark by db_bench](Benchmark-by-db_bench.html)

## 2. TerarkDB vs. RocksDB
We have benchmarked TerarkDB and RocksDB with two different servers:

- [Benchmark TPCH TerarkDB vs RocksDB A](Benchmark-TPCH-TerarkDB-vs-RocksDB-A.html)
- [Benchmark TPCH TerarkDB vs RocksDB B](Benchmark-TPCH-TerarkDB-vs-RocksDB-B.html)

|   | Server A               | Server B                   |
|---|------------------------|----------------------------|
| CPU Number                                                         | 2                                         | 2                                                |
| CPU Type                                                           | Xeon E5-2680 v3                           | Xeon E5-2630 v3                                  |
| CPU Freq.                                                          | 2.5 GHz                                   | 2.4 GHz                                          |
| CPU Actual Freq.                                                   | 2.5 GHz                                   | 2.6 GHz                                          |
| Cores per CPU                                                      | 8 cores with 16 threads                   | 8 cores with16 threads                           |
| Cores in total                                                     | 16 cores 32 threads                       | 16 cores with 32 threads                         |
| CPU Cache                                                          | 30M                                       | 20M                                              |
| [CPU bogomips](http://www.cnblogs.com/youngerchina/p/5624439.html) | 4988                                      | 4793                                             |
| Memory                                                             | 64GB                                      | 64GB                                             |
| Memory Freq.                                                       | DDR4 2133Hz                               | DDR4 1866Hz                                      |
| SSD Capacity                                                       | 2TB x 1                                   | 480GB x 4                                        |
| SSD IOPS                                       | 20000<br/>(Network virtual SSD, not local SSD) | Two Intel 730 IOPS 89000<br/>Two Intel 530 IOPS 41000 |

## 3. 其他分布式数据库的性能测试报告
### TiDB
1. [TiDB Sysbench 官方测试报告(中文版)](https://github.com/pingcap/docs-cn/blob/master/benchmark/sysbench.md)
1. [TiDB Sysbench official report English](https://github.com/pingcap/docs/blob/master/benchmark/sysbench.md)
1. [sysbench issue #5328](https://github.com/pingcap/tidb/issues/5328)
