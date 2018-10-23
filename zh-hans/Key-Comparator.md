## 背景
我们知道，不管是标准库的 `std::map`, `std::sort`，还是 C lib 的 `qsort`, `bsearch` ...，都有一个必不可少的 Comparator，这个 Comparator 定义了 Key 的顺序。一般情况下，默认的 Comparator 是“**按字节的字典序**”，比如 `std::string` 的默认比较操作，还有 C lib 的 `memcmp`。

RocksDB(来源于LevelDB) 的 Comparator，和其它的 Comparator 并没有什么本质的不同，只不过它给 Comparator 额外定义了一个 `Name`，这个 `Name` 最大的价值是：它**约定了 Comparator 的行为**。如果仅有一个 Comparator 函数指针，我们就无法知道这个函数的实现对应的是什么行为。`Name`的另一个价值是可以**持久化**。

RocksDB 自身默认的 Comparator 就是“**按字节的字典序**”，名字是 `leveldb.BytewiseComparator`，只要我们看到这个名字，我们就知道，不管它是怎么实现的，它的行为一定是“**按字节的字典序**”。

## Terark
不管是 [Terark MemTable](重新实现-RocksDB-MemTable.html)，还是 [Terark SST](TerarkDB-SST-的创建过程.html)，Key 的排列顺序都是且必须是“**按字节的字典序**”。我们只需要判断 Comparator 的 `Name`，就知道接下来应该使用 Terark 的 MemTable/SST，还是应该 fallback 到 RocksDB 自身的 MemTable/SST。

表面上看，这限制了**灵活性**，但实际上，这种依赖于 Comparator 的灵活性，在绝大多数时候都是不必要的，因为我们总是可以使用其它手段来实现这种灵活性，比如通过对 Key 进行编码，让编码后的顺序是字节序，且这个顺序等价于编码前按 Comparator 的顺序：

- MongoDB 有个编码器，实现了这种编码，从而在 MongoRocks 中，只需要 RocksDB 默认的 `leveldb.BytewiseComparator`
- MyRocks 中(MySQL + RocksDB)，也实现了[这样的编码器](https://github.com/facebook/mysql-5.6/wiki/MyRocks-record-format)，把各种 Index Key，编码成“**按字节的字典序**”
  - 对于忽略大小写的 Collation/Comparator，把 Key 统一编码(归一化)到全大写，并且，把解码每条 Key (恢复到原始 Key) 需要的信息，保存到 Value (KeyValue 的 Value) 中。
  - 虽然 MyRocks (默认)的 Comparator 是“**按字节的字典序**”，但是，却使用了另一个 `Name`，而不是 `leveldb.BytewiseComparator`

### 反向字节字典序
最最令人烦恼的是，MyRocks 为了优化反向扫描的性能(比如 `ORDER BY ... DESC`)，提供一个 “**反向字节字典序**” 的 Comparator，其原因有点可笑却又无可奈何：
- RocksDB 默认的 BlockBasedTable 的反向扫描 (Iterator.Prev) 性能太差
- 通过使用 “**反向字节字典序**” 的 Comparator，就把反向扫描变成了正向扫描(Iterator.Next)……

我们(Terark)的 MemTable 和 SST 的自然顺序都是“**正向字节字典序**”，并且都没有反向扫描 (Iterator.Prev) 性能低下的问题，但是，
如果要 100% 兼容 MyRocks，我们仍必须支持“**反向字节字典序**”，虽然和“**正向字节字典序**”的道理是类似的，但是“**反向字节字典序**”的实现很繁琐，代码的实现更是难以优雅。

实际上，就算是要把顺序反过来，使用“**正向字节字典序**”也是很容易做到的：
- 对于定长字段（各种整数、浮点数……），编码时只需要额外再对每个字节取反即可
- 对于变长字段（主要是变长 String），`ORDER BY ... DESC` 是一种很奇怪的需求，无需优化

## Internal Comparator
RocksDB 是基于 MVCC 的，它在用户提供的 Key 上，追加了一个 (SeqNum, OpType)，SeqNum 总是增加的，OpType 是枚举 {PUT,DELETE,MERGE,...}。

从而，它的 Comparator，还分为 User Comparator 和 Internal Comparator，User Comparator 就是用户提供的 Comparator，Internal Comparator 只是在 User Comparator 的基础上，增加了对 SeqNum 和 OpType 的比较。

实际上，对于“**正向字节字典序**”，只需要把 SeqNum+OpType 编码为 BigEndian，Internal Comparator 和 User Comparator 的实现就是完全一样的！