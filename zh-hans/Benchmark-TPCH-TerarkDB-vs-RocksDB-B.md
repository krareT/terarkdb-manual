# TerarkDB vs. RocksDB 服务器二

## 1. 测试数据集

我们在 TPC-H 数据集中使用 lineitem 表，并将 dbgen lineitem 中的 comment 文本字段的长度设置为 512（默认是 27 字节）。 因此，lineitem 表的平均行尺寸为 615 字节，其中，key 是前三个整数字段直接拼接而成的字符串。

数据集的总大小为 554,539,419,806 bytes，897,617,396 rows。 key 的总尺寸是 22,726,405,004 bytes, 剩余的是 value 总尺寸。

TPC-H dbgen 生成最原始的字符串数据，我们直接在测试中使用这些数据而不对数据进行任何转换。


## 2. 硬件
|                                                                    | 服务器配置                                         |
|--------------------------------------------------------------------|--------------------------------------------------|
| CPU Number                                                         | 2                                                |
| CPU Type                                                           | Xeon E5-2630 v3                                  |
| CPU Freq.                                                          | 2.4 GHz                                          |
| CPU Actual Freq.                                                   | 2.6 GHz                                          |
| Cores per CPU                                                      | 8 cores with16 threads                           |
| Cores in total                                                     | 16 cores with 32 threads                         |
| CPU Cache                                                          | 20M                                              |
| [CPU bogomips](http://www.cnblogs.com/youngerchina/p/5624439.html) | 4793                                             |
| Memory                                                             | 64GB                                             |
| Memory Freq.                                                       | DDR4 1866Hz                                      |
| SSD Capacity                                                       | 480GB x 4                                        |
| SSD IOPS                                                           | 两个 Intel 730 IOPS 89000 <br/>两个 Intel 530 IOPS 41000 |

## 3. DB 参数

|                                      | RocksDB                                                                                                                                                                                                                  | TerarkDB                              |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Level Layers                         | 4                                                                                                                                                                                                                        | 4                                     |
| Compression                          | Level 0 doesn’t compress<br/>Level 1~3 use Snappy                                                                                                                                                                        | All level uses<br/>Terark Compression |
| Compact Type                         | Level based compaction                                                                                                                                                                                                   | Universal compaction                  |
| MemTable Size                        | 1G                                                                                                                                                                                                                       | 1G                                    |
| Cache Size                           | 16G                                                                                                                                                                                                                      | Do not need                           |
| MemTable Number                      | 3                                                                                                                                                                                                                        | 3                                     |
| Write Ahead Log                      | Disabled                                                                                                                                                                                                                 | Disabled                              |
| Compact Threads                      | 2~4                                                                                                                                                                                                                      | 2~4                                   |
| Flush Threads <br/>(MemTable Compression) | 2                                                                                                                                                                                                                        | 2                                     |
| Terark Compress threads              | NA                                                                                                                                                                                                                       | 12                                    |
| Working Threads                      | 25 in total <br/>24 Random Read, 1 Write                                                                                                                                                                                     | 25 in total <br/>24 Random Read, 1 Write                       |
| Target_file_size_base                | Default (64M)                                                                                                                                                                                                            | 1G                                    |
| Target_file_size_multiplier          | 1 (SST size in RocksDB<br/> do not influence performance)                                                                                                                                                                     | 5                                     |

对于两个引擎，均禁用写入限速:

- options.level0_slowdown_writes_trigger = 1000;
- options.level0_stop_writes_trigger = 1000;
- options.soft_pending_compaction_bytes_limit = 2ull<<40
- options.hard_pending_compaction_bytes_limit = 4ull<<40


## 4. 测试结果
### 4.1. OPS Comparison
#### 4.1.1. TerarkDB OPS
![](images/terarkdb_vs_rocksdb_server_b/terarkdb_ops.png)
#### 3.1.2. RocksDB OPS
![](images/terarkdb_vs_rocksdb_server_b/rocksdb_ops.png)

### 4.2. DB Size Comparison
#### 4.2.1. TerarkDB DB Size
![](images/terarkdb_vs_rocksdb_server_b/terarkdb_dbsize.png)
#### 4.2.2. RocksDB DB Size
![](images/terarkdb_vs_rocksdb_server_b/rocksdb_dbsize.png)

### 4.3. Memory Usage Comparison
#### 4.3.1. TerarkDB Memory Usage
![](images/terarkdb_vs_rocksdb_server_b/terarkdb_memory.png)
#### 4.3.2. RocksDB Memory Usage
![](images/terarkdb_vs_rocksdb_server_b/rocksdb_memory.png)

### 4.4. CPU Usage Comparison
#### 4.4.1. TerarkDB CPU Usage
![](images/terarkdb_vs_rocksdb_server_b/terarkdb_cpu.png)
#### 4.4.2. RocksDB CPU Usage
![](images/terarkdb_vs_rocksdb_server_b/rocksdb_cpu.png)



## 5. 测试结果说明

<table>
<tr>
<td width="20%">&nbsp;</td>
<td width="40%">RocksDB (DB Cache 16GB)</td>
<td width="40%">TerarkDB</td>
</tr>

<tr>
<td>0~2 minutes</td>
<td>Read OPS 970K <br/> Write OPS 100K (Apply Level Compaction)</td>
<td>Read OPS 1.92M <br/> Write OPS 120K (Sufficient memory)</td>
</tr>

<tr>
<td>2~15 minutes</td>
<td>Read OPS 620K <br/> Write OPS 96K</td>
<td>Read OPS 850K <br/> Write OPS 100K (Sufficient memory)</td>
</tr>

<tr>
<td>15~30 minutes</td>
<td>Read OPS 480K <br/>Write OPS 83K <br/>(Read and Write decline gradually)</td>
<td>Read begins to fluctuate sharply, with average around 680K. <br/>Write keeps at around 88K, CPU usage close to 100%, IOWait close to 0 (Sufficient memory but compression thread in the background starts to hit the read threads)</td>
</tr>

<tr>
<td>30~60 minutes</td>
<td>Read OPS 220K <br/>Write OPS 61K</td>
<td>Read OPS fluctuates with average 330K, Write OPS keeps at 82K, CPU usage starts to drop (Compaction thread hits the read performance)</td>
</tr>

<tr>
<td>60~120 minutes</td>
<td>Read 17K <br/> Write 39K <br/> (Memory runs out, read and write meet bottleneck)</td>
<td>Read OPS fluctuates with average around 310K. <br/>Write OPS keeps at around 90K.</td>
</tr>

<tr>
<td>3 hours 20 minutes</td>
<td>Read and Write keep dropping</td>
<td>All 550G data completes writing, Read OPS fluctuates between 60 ~ 120K (Compaction thread hits the read performance)</td>
</tr>

<tr>
<td>3~11 hours</td>
<td>Read and Write keep dropping</td>
<td>Read OPS keeps at around 170K <br/>(Data being compressed gradually, more data can be loaded into memory)</td>
</tr>

<tr>
<td>12 hours 40 minutes</td>
<td>Data completes writing, current database size is 234GB <br/>(Continues to compress in the background)</td>
<td>Read OPS keeps growing</td>
</tr>

<tr>
<td>18 hours</td>
<td>Compaction completes with final data size 209GB after compression, Read OPS keeps at around 5K</td>
<td>&nbsp;</td>
</tr>

<tr>
<td>30 hours</td>
<td>&nbsp;</td>
<td>Data compression completes, Read OPS keeps at 2.2M <br/>(47G after compression, all data can be loaded into memory.)</td>
</tr>

</table>
