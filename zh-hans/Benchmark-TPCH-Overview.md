## 该基准测试基于 TPC-H 数据
我们对 TPC-H 的 dbgen 程序进行了轻微的修改以生成我们需要的数据，主要的修改是改变它的字段长度。

在 TPC-H 的各种表中, lineitem 表的尺寸最大, 因此我们使用 lineitem 来进行测试. 

**Notice**: TPC-H dbgen 使用 `|` 作为分隔符.

TPC-H lineitem 有一个 comment 字段, 这个字段是原始的纯文本数据并且是压缩算法的主要压缩对象。这个字段在原始的 dbgen 中被硬编码为 27 bytes，为了让它能够更加适应我们的实际需要，我们添加了一个用来修改这个字段长度的环境变量。另外我们创建了一个 `dbgen.sh` 来直接生成表数据：[dbgen.sh intro](https://github.com/rockeet/tpch-dbgen)


## 环境
- CPU : Intel(R) Xeon(R) CPU E5-2682 v4 @ 2.50GHz 40 core
- Memory : 224GB
- DISK : 2TB SSD

## 压缩率
| 原始数据 | Record Size | TerarkDB 压缩 | RocksDB 压缩 |
|--------------------|-------------|--------------------------|-------------------------|
| 40 GB              | 128 bytes   | 8.1 GB                   | 19 GB                   |
| 104 GB             | 512 bytes   | 12 GB                    | 40 GB                   |
| 415 GB             | 512 bytes   | 34 GB                    | 170 GB                  |
| 128 GB             | 2048 bytes  | 4.4 GB                   | 35 GB                   |



## OPS
- Read Only
  - 约 2,000,000/seconds
- Read Write Mixed
  - 读测试性能非常高, 从 600,000/s 到 1,200,000/s.
  - 写速度约 100,000/seconds

![](images/random_read.png)


## Random Read Latency

| Read Latency | Read Write Mixed | Read<br/>(无后台压缩) | Read<br/>(有后台压缩) |
|---------|------------------|---------------------------------|-----------------------------------|
| < 100ms | 100%             | 99.99%                          | 100%                              |
| < 15ms  | 99.99%           | 99.99%                          | 100%                              |
| < 5ms   | 99.98%           | 99.99%                          | 100%                              |
| < 2ms   | 99.94%           | 99.99%                          | 100%                              |
| < 0.5ms | 94.70%           | 99.99%                          | 99.99%                            |
