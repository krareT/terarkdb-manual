## TerarkDB 说明文档

### 1.产品简介
TerarkDB 是由 Terark.com 研发的一款数据库存储引擎，该存储引擎目前可以适配于 MySQL、MongoDB、Redis、SSDB 等产品。

TerarkDB 使用了 RocksDB 的上层框架，我们实际上实现了一个 RocksDB 的 `SSTable` 并且命名为 `TerarkZipTable`. 基于上面的环境变量，我们可以让 RocksDB 使用我们的 SSTable. 我们所有的算法均封装在 `TerarkZipTable ` 并且不影响现有的 `SSTable`, 也就是说您可以不启动我们版本的 SSTable，继续使用默认版本。


TerarkDB 的底层算法完全不同以往的存储引擎（Key 和 Value 的存储均使用自己全新发明的数据结构），相对于传统技术，TerarkDB 可以实现全剧压缩，同时抽取单条数据的时候，不需要全剧解压。

### 2.产品优势
- 更好的压缩率，通常可以比使用了 InnoDB/RocksDB 存储引擎的 MySQL 节省至少一倍的存储空间
- 更强的随机读性能，通常会比其他存储引擎快 3～5 倍

### 3.TerarkDB 的缺点
- TerarkDB 的压缩速度低于官方 RocksDB（因为压缩过程中计算量更大）
- 当 Key 为字符串，或者很长的联合键的情况下：
  - TerarkDB 的**顺序读**略低于官方 RocksDB
  - TerarkDB 的**顺序读**比 TerarkDB 自身的**随机读**无明显优势

### 4.联系我们
- contact@terark.com
- [www.terark.com](http://www.terark.com)
