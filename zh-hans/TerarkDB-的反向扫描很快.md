按照 TerarkDB 的设计哲学，至少在软件层面，反向扫描没有理由比正向扫描更慢。

然而，这个被我们认为是自然而然的事情，却被很多主流的数据库认为是极难实现，甚至不可能实现的。于是，他们发明了一种叫做 `降序索引` 的东西：

| 数据库 | 降序索引（或其变形） |
|--------|--------------------|
|MySQL 8.0| [Descending Index](https://www.percona.com/blog/2016/10/20/mysql-8-0-descending-indexes-can-speedup-your-queries/)|
| MyRocks (MySQL + RocksDB) | [Reverse Column Family](https://github.com/facebook/mysql-5.6/wiki/Schema-Design) |
| PostgreSQL | [Descending Index](https://www.enterprisedb.com/blog/creating-descending-indexes) |
| MongoDB | [Index Direction](https://stackoverflow.com/questions/10329104/why-does-direction-of-index-matter-in-mongodb) |
| hbase | [Reverse Scan](https://issues.apache.org/jira/browse/HBASE-4811)|

## 为什么主流数据库的反向扫描慢？
### 1. 索引使用了前缀压缩
传统数据库使用的块压缩有很多缺点，但是反向扫描慢这个问题，不是块压缩的锅，而是传统数据库索引中经常使用的另一种技术`前缀压缩`的锅！

在数据库中，Key 会有大量的重复前缀，所以，传统的存储引擎对索引(Key)使用了前缀压缩：如果当前 Key 与前一个 Key 的共同前缀比较长(比如超过了 4 个字节），就不保存这个共同前缀，而是保存一个对共同前缀的引用。这样，前向顺序扫描时，我们只需要保存上一个 Key，当前 Key 就可以很快的解码出来，所以，前缀压缩的前向顺序扫描非常快。

但是，搜索的时候怎么办，如果只有知道上一个 Key，才能知道下一个 Key，那就只能顺序扫描，无法用二分搜索等算法加速，就算是把这样的扫描限制在 Page 内部，性能也是无法忍受的！

所以，存储引擎会每间隔一些 Key（比如间隔 100 个），就保存一个不压缩的完整的 Key，这样，先用二分搜索找到间隔点，然后从间隔点开始顺序扫描，找到目标 Key。

但是，在日常的数据库操作中，相比随机搜索，更多的操作是`扫描`，比如：
```mysql
SELECT * FROM SomeTable WHERE timestamp >= "2018-09-14 12:00"
ORDER BY timestamp DESC LIMIT 10;
## 降序排列 -------^^^^^
```
在这里，会用到索引的 Iterator（也叫做 Cursor），在 Iterator 上需要执行一次 `随机搜索` 和 10 次 `扫描`：
* 对于 `逆向索引`，Iterator 需要`前进` 10 次，速度很**快**
* 对于 `正向索引`，Iterator 需要`后退` 10 次，速度很**慢**，因为需要解压一个间隔区间

这样，在前缀压缩中存储间隔点的优化，对仅执行一次的 `随机搜索`，可以得到巨大的性能提升，然而，`逆向扫描` 则至少需要解压整个间隔区间：
* 对于 `limit 10`，需要取 10 条结果，但需要解压 100 条 Key（每个间隔区间包含 100 个 Key）

### 2. 机械硬盘 & SSD
现存的数据结构和算法仍是主要为机械硬盘优化的，在机械硬盘上的前向扫描就是顺序读取，和块压缩一样，前缀压缩完美匹配了机械硬盘的特性。

然而，时代变了，SSD 基本取代了机械硬盘，现在极少有运行在机械硬盘上的数据库。对 SSD 而言，按 Page 逆向读取并没有太多的性能损失，并且，越新的 SSD，逆向读取的性能损失越小。

## TerarkDB 天生对逆向扫描与正向扫描一视同仁
如果 TerarkDB 的正向扫描比传统 DB 的正向扫描快 1 倍，那么逆向扫描就至少快 10 倍。

从而，对于 TerarkDB，`Reverse Index` 是个负优化，主要是加重了 TerarkDB 实现的复杂性：比如在 RocksDB 中，为了支持 Reverse Column Family，我们做了很多[额外的事情](Key-Comparator.html)。
