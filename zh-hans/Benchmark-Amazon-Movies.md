# 目录
- 1 前言
- 2 测试方式
  - 2.1 硬件配置
- 3 读性能
  - 3.1 数据远小于内存（内存 64 GB）
  - 3.2 数据远大于内存（内存 3 GB）

# 1 前言
基于 [Terark](http://terark.com/zh/index) 的可检索压缩技术（Searchable Compression Technology）研发的 [TerarkDB](https://github.com/Terark/terarkdb) 是一款高压缩，快速随机访问的存储引擎，同时为读多写少的场景进行了单独优化。本测试针对其在两种内存限制情况下性能进行测试，即内存远大于数据及内存远小于数据两种内存限制情形。同时，在相同条件下测试了 Facebook 的 [RocksDB](https://github.com/facebook/rocksdb) 引擎和 MongoDB 
默认引擎 [WiredTiger](https://github.com/wiredtiger/wiredtiger) 的性能作为对比。

由于本测试是针对**存储引擎**本身而不是针对数据库的，因此减少了上层开销的干扰，更加清晰地展现了三者之间在不同内存限制条件下表现出的性能的差距。当然，我们也有针对 [MySQL 数据库](https://github.com/Terark/mysql-on-terarkdb)和 [Mongo 数据库](https://github.com/Terark/mongo-on-terarkdb)的测试。

# 2 测试方式
- 测试工具
  - [Terark DB BenchmarkTest (https://github.com/Terark/terarkdb-tests)](https://github.com/Terark/terarkdb-tests)
- 测试数据
  - 本测试中使用 [Amazon movie data (~ 8 million reviews)](https://snap.stanford.edu/data/web-Movies.html) 数据进行测试
- 测试数据集尺寸
  - 约为 9.1 GB
  - 约 800 万条数据
  - 平均每条数据大约 1 KB
- 测试使用的引擎
  - MongoDB 默认存储引擎 [WiredTiger](https://github.com/wiredtiger/wiredtiger)
  - Facebook 的 [RocksDB](https://github.com/facebook/rocksdb) 引擎
  - Terark 的 [TerarkDB](https://github.com/Terark/terarkdb) 引擎

## 2.1 测试平台
  - 测试平台软硬件配置：
<table border="0">
<tr><th align="right">CPU</th><td align="left">Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz x2 (共 16 核 32 线程)</td></tr>
<tr><th align="right">内存</th><td align="left">Samsumg 16G @ 1866 MHz x4 (共 64 GB)</td></tr>
<tr><th align="right">硬盘</th><td align="left">INTEL SSDSC2BP480G4 (480 GB SSD)</td></tr>
<tr><th align="right">操作系统</th><td align="left">CentOS Linux release 7.3.1611 (Core)</td></tr>
</table>

# 3 随机读取性能
我们在开始随机读取性能测试之前，首先批量的将所有数据写入数据库，然后重启服务器后开始测试。

- RocksDB、TerarkDB 内存受限情形我们使用 cgroup 实现
- WiredTiger 的读文件操作利用 Linux 系统的 read() 函数实现，这将导致文件被读入系统缓存中而对用户不可见，因此这部分内存开销无法使用 cgroup 限制。我们通过修改 Linux 内核参数直接限制系统可用内存约为 3 GB 来达到限制 WiredTigher 总内存开销的目的
- RocksDB 设置 --use_universal_compaction=0 --bloom_bits=8 --block_size=4K 
- RocksDB、Wiredtiger 在限制内存的情况下设置 --cache_size=512M
- WiredTiger 设置 --use_lsm=0
- TerarkDB 使用默认配置选项

## 3.1 数据远小于内存（内存 64GB）
- 以下为数据压缩后大小与内存占用
- 由于压缩后数据库的尺寸（Storage Size）与读测试的内存限制无关，后面不再重复该图表
![Data Storage Size (GB)](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/amazon_movies_parser/2-Data_Storage_Size_GB.svg)
![Memory Usage (GB) - Memory Unlimited](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/amazon_movies_parser/2-Memory_Usage_-_Memory_Unlimited.svg)
![Random Read (QPS) - Memory Unlimited](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/amazon_movies_parser/2-Random_Read_-_Memory_Unlimited.svg)

## 3.2 数据远大于内存（内存 3GB）
- TerarkDB 需要的内存接近 3 GB，性能几乎不受影响
- RocksDB、Wiredtiger 被内存限制影响较大
![Random Read (QPS) - Memory Limited (3GB)](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/amazon_movies_parser/2-Random_Read_-_Memory_Limit_3G.svg)

