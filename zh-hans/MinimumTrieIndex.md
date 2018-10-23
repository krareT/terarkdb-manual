Nested Succinct Trie 作为变长 Key 的索引，有优异的压缩率和随机搜索性能，不足之处是顺序遍历的性能较差。

我们发现，很多 Key 集合有一些特征可以利用：抽取每个 Key 的`最短可区分前缀`，对这个前缀集合创建一个 Trie 树，每个 Key 前缀对应的后缀，以另外的方式存储……

## 仍以 MySQL 为例

还是 [UintIndex](UintIndex.html) 中的那个表：
```mysql
CREATE TABLE Student(
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) INDEX,
    dorm_id INT INDEX,
    -- others ...
);
```
这次我们看 `name` 的  Secondary Index，在 MyRocks 中，`Secondary Key` 的逻辑结构如下：
<table><tr><td>PrefixID</td><td>SecondaryKey</td><td>PrimaryKey</td></tr></table>

SST Builder 并不知道索引的 Schema，在把 `PrefixID` 剔除之后，剩余的部分是 `name+id`，SST Builder 并不知道 `name+id` 中，name 和
 id 的分界点。

所以我们需要对 Key 集合扫描一遍，找出每个 Key 的`最短可区分前缀`，这个前缀可能比 name 长，也可能比 name 短：
* 如果有多个 Student 同名，就会导致`最短可区分前缀`比 `name` 长，这个前缀还要加上 `id` 的前几个字节
* 如果多个相邻的 Student 的 `姓` 都不同，例如 {`李明`, `刘军`, `刘明`} 相邻，那么我们可以确定 `刘军` 的`最短可区分前缀`就是 `刘`，比 name 本身要短

无论如何，只需要经过一遍扫描，我们就可以把每个 Key 分为 `最短可区分前缀` 和 `剩余后缀` 两部分，前缀部分使用 Nested Succinct Trie 来表达，后面一部分，使用 `ZipOffsetStrVec` 来表达，这样在空间占用上能达到最优。

与 BlobStore 类似，使用 LOUDS Succinct Trie，需要对后缀进行 Reorder：在 LOUDS Trie 中，Key 的编号不是字典序，从而需要对相应的 
后缀进行 Reorder，才能与与前缀对应起来。

然而，`ZipOffsetStrVec` 仍有缺点，它的随机访问稍微有点慢，所以，我们可以再进行折衷，对`剩余后缀`部分建立统计直方图（Histogram），利用直方图给出的信息，采用不同的存储方式：
* 如果直方图表明`后缀`的最小长度在某一范围（例如 4~8 字节），并且平均长度比最小长度没有大太多
  * 此时直接将后缀存储为元素长度为最小长度的数组，对那些比较长的后缀，也仅截取最小长度，其余的归入前缀
  * 这种情况很可能前缀就是 Secondary Key，后缀就是 Primary Key，并且 Secondary Key 是 Unique 的
* 如果直方图表明大部分`后缀`都是固定长度（例如 4 字节），但仍有少部分（例如 5%）后缀的长度较短（例如少于 4 字节）
  * 此时直接将后缀存储为元素长度为大于最小长度的某个值（例如 4 字节）
  * 此时就存在一个前缀对应多个后缀的情况，可以用 Rank Select 来进行解决，使用类似 [CompositeUintIndex](CompositeUintIndex.html) 的方式
    * 这种情况下，Reorder 会更复杂一些
  * 使用了 Rank Select，前缀唯一（无重复）的情况就可以通过 `RankSelectAllZero` 来实现，作为一种特殊解

## 推广到 MongoDB
MongoDB(MongoRocks) 的 `UniqueIndex`，不太需要这种优化，但是它的 `StandardIndex`，也就是 `NonUniqueIndex`，其存储方式跟 MySQL 是非常类似的：
<table><tr><td>PrefixID</td><td>IndexKey</td><td>RecordID</td></tr></table>

从而也可以使用这种优化。
