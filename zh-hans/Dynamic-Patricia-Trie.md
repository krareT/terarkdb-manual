# Dynamic Patricia Trie
一提到 Trie，很多人可能会想到 Double Array Trie，但是，在这里，必须断了 Double Array Trie 这个念头。

这个 Dynamic Patricia Trie 是要为 MemTable 服务的，所以性能需求十分苛刻，按照预期：

1. 插入性能至少要达到 rbtree 的 150%
2. 精确搜索的性能至少要达到 rbtree 的 300%
3. 范围搜索 (lower_bound 语义) 至少要与 rbtree 持平
4. 最坏情况下的内存占用不能超过 rbtree

## 设计思想
1. 使用 MemPool，用 uint32 偏移代替 64 bit 指针
   * 偏移对齐到 4，寻址空间扩大 4 倍，从 4G 扩大到 16G
3. 使用 Copy On Write 解决并发问题
4. 多个并发级别: 

|并发级别|限制条件|
|-------|--------|
|NoWriteReadOnly|只读，不可修改|
|SingleThreadStrict|插入**导致** Iterator 失效|
|SingleThreadShared|插入**保持** Iterator 有效|
|OneWriteMultiRead|插入**保持** Iterator 有效|
|OneWritePerSubTrie|同 OneWriteMultiRead，但是多个 SubTrie 可以同时写|
|*MultiWriteMultiRead*|*插入**保持** Iterator 有效*|

## 内存分配
*在自动机的语境中，**结点**与**状态**是等价的，文中有些地方会两者混用，比如：**孩子结点**，**终止状态**。*

- 首先，内存池是必需的：内存分配释放必须要快
- 其次，内存池仅包含单块内存：内存地址 = 内存池基地址 + 池内偏移

从内存池中分配出来的内存块用一个整数偏移来表示，偏移和尺寸都对齐到 4，这样做有两个好处：
1. 性能更高：CPU 访问对齐的数据更快
1. 寻址能力更大：32 位的寻址能力是 4GB，对齐到 4，寻址能力就扩大到 16GB

为了最小化内存用量，光是把 64 位的指针换成 32 位的偏移是远远不够的：
1. 一个 `Patricia Trie` 结点最多有 `256` 个孩子指针
1. 最大需要 254 字节的压缩路径（保存字符串）
1. 如果该结点是终止状态，还需要 `ValueSize` 大小的空间保存该节点对应的 value
1. 结点的最大尺寸超过 1KB
1. 一般情况下，结点的平均尺寸也就 10 字节左右

所以，结点的大小不能是固定的，而是要**按需分配**！

## Copy On Write & Access Token
写线程插入一条数据，会修改一些结点，在多线程场景下……

**Copy On Write** 不用解释。

**Access Token** 包含 **Reader Token** 和 **Writer Token**，它表示，每个 Token 可以通过某种方式持有对一些结点的引用，在该 Token Invalidate 这些引用之前，这些引用必须保持有效。

严格讲，只有 Access Token 是必须的：如果结点大小是固定的，并且不是带有路径压缩的 Patricia Trie，那么：
1. 每个结点都有 256 个孩子结点的指针，NULL 指针表示该孩子不存在
1. 修改指针是原子操作，所以我们可以原子性地在父节点中修改一个指向孩子结点的指针

从而，我们只需要确保，即便孩子指针已被修改为新值，旧的仍然被 Token 引用的孩子继续保持有效，那么，其它并发的操作就是有效的：要么访问新值，要么访问旧值，绝对不会出现未定义行为。在这里，只有 Access Token，没有 Copy On Write。

现在，结点的尺寸和布局不是固定的，修改无法通过原子操作完成，只能对整个读操作和整个写操作加锁，然而，即便使用读写锁，性能的损失也非常巨大。

所以，我们采用了 Copy On Write 机制：一个写操作（例如整个插入操作）所有的修改，都是在源结点的拷贝上进行修改，所有的新结点（和新节点上面的链接关系）完全构造好之后，再修改父结点上的指针。此时，需要结合 Copy On Write 与 Access Token。

### 一些实现细节
1. Patricia Trie 保持一个版本计数器，每次写操作会使版本计数器加 1
1. Token 构造时拷贝 Trie 版本计数器的当前值
1. 所有 Token 通过循环双链表连接，新 Token 放入表尾，表头 Token 版本计数器最小
1. Token 可以 update，update 时把该 Token 移动到 List 尾部
1. 被 Copy On Write 的 Trie 结点放入 LazyFreeQueue，每个 Queue 元素 = (nodePtr, version)
```c++
    // 伪代码，省略加锁、延迟删除等具体细节
    if (LazyFreeQueue.head.version < TokenList.head.version)
        MemPool.Delete(LazyFreeQueue.pop().nodePtr); 
```

## 并发级别
### 0: NoWriteReadOnly
这个场景最简单：
1. Patricia Trie 完全创建好，不再需要修改，只有读操作
1. 持久化到文件的 Patricia Trie 从文件加载(mmap)进来，只读，不写
1. 只有读操作，不需要 Token，并发(读)能力最大

### 1: SingleThreadStrict
单线程，不需要任何并发控制，也不需要 Token，修改会导致之前的 Iterator 失效。牺牲了这么多特性，获得的收益有：
1. 不需要 Copy On Write，可以对结点原地修改
1. 不需要 Access Token
1. 分配结点时，如果超出了内存池的容量，可以 realloc 一块更大内存

### 2: SingleThreadShared
单线程情况下，很多时候，我们需要保证修改操作不会破坏已有的 Iterator，此时：
1. 需要 Copy On Write
1. 需要 Reader Token，但不需要 Writer Token
1. 可以 realloc

### 3: OneWriteMultiRead
一个写线程，多个读线程
1. 写线程分配内存时不需要加锁，甚至连连 Lock Free 都不需要。
1. 需要 Reader Token，但不需要 Writer Token
1. 不能 realloc，从而内存池的容量必须一开始就固定下来
   * 因为整个内存池中只有一大块内存，realloc 会导致整块内存被搬移到其它位置并释放原内存块，而其他线程可能正在读取原内存块。

因为写操作不需要加锁，所以写性能可以达到极致，实测和 SingleThread 几乎相同。

### 4: OneWritePerSubTrie
多个 SubTrie 共用一个内存池，对没有 SubTrie，可以有一个写线程，多个读线程
1. 使用 Lock Free 内存池，减小内存分配释放的开销
1. 需要 Reader Token，但不需要 Writer Token
   * 每个 SubTrie 拥有自己的 Reader Token List，从而减小 List 锁冲突
1. 不能 realloc，这一点和 OneWriteMultiRead 相同

虽然共用一个内存池，但多个 SubTrie 可以并发写，一开始的设计中没有这个并发级别，在 RocksDB `WriteBatchWithIndex` 的实现过程中，我们发现，如果有这样一个并发级别，对多个 ColumnFamily 的支持会更好。

### 5: MultiWriteMultiRead
最彻底的并发：
1. 使用 Lock Free 内存池，减小内存分配释放的开销
1. 需要 Reader Token，也需要 Writer Token
1. 不能 realloc

在实测中，多核 CPU 上使用多线程并发插入，性能反而比单线程插入大幅下降……

## 实测性能
我们达到的性能远超预期，特别是，动态随机插入的性能，比 `strVec` 还快！
这个性能是非常惊人的，Patricia 内存占用更小，并且是 **有序的**，**动态的**，**并发的**，性能还要比静态的 `sort` 更高！
<!--
<table>
<tr><th align="right">RbTree</th>
    <td>
       使用 <br/><code>std::set&lt;string&gt;</code>，
       无特殊优化
    </td>
</tr>
<tr><th align="right">strVec<br/>sort</th>
    <td>
    1. 使用一块连续内存保存所有的 StringData<br/>
    2. 另外一块连续内存是索引，指向 StringData 中的偏移和相应 String 的长度
    </td>
</tr>
<tr><th align="right">NLT</th>
</tr>
<tr><th align="right">Patricia</th>
</tr>
</table>
-->
1. 这里的 `strVec` 是指以 `StringVector` 作为存储，进行排序、查找
2. `StringVector` 进行了深度优化：
   - 使用一块连续内存保存所有的 StringData
   - 另外一块连续内存是索引，指向 StringData 中的偏移和相应 String 的长度
3. **插入**指在 `StringVector` 上逐个 Append String，然后排序
   - Append 开始前预分配大小刚够容纳所有 String 的内存
   - Patricia 的**插入**，使用的并发级别是 `OneWriteMultiRead`
   - `NLT` 的**插入**，指 build 整个 NLT
   - `RbTree` 和 `Patricia` 的**插入**就是通常意义的插入
4. **点查**指使用 std::binary_search 语义的搜索
5. **LB** 指 lower_bound 语义的搜索，使用 Iterator 定位 **LB** 以后可以前进后退
   - strVec.lower_bound 经过了深度优化，比 std::lower_bound 更快
   - `NLT`, `RbTree`, `Patricia` 的 **LB** 采用各自的优化实现

### 点查 & LB(lower_bound)
- `strVec` 和 `RbTree` 的点查基于 **LB** 实现，所以比它们自己的 **LB** 要慢一点
- `Patricia` 和 `NLT` 的点查相当于在 DFA 上执行匹配，原则上比它们自己的 **LB** 要快
- `Patricia` 和 `NLT` 的 **LB** 通过一个重量级的 **Iterator** 对象来实现
  - Iterator 需要一个内部栈，**LB** 过程中还需要重建 Key……
  - 所以，尽管 `Patricia` 的 **LB** 经过了极高度的优化，性能仍然比**点查**慢
- 对于 `NLT`，不管是点查还是 LB，搜索过程中都需要重建压缩路径
  - 重建压缩路径是一个昂贵的操作，占用了大部分 CPU 时间
  - 所以，`NLT` 的点查，并没有比 LB 快多少（两者几乎一样）

按照最开始的预期，不管是 `Patricia` 还是 `NLT`，他们的点查要比 LB 快得多，但实测结果是各自的点查和 LB 性能相差无几！使用 Profiling 工具，我们最终发现，性能瓶颈是[内存墙](https://en.wikipedia.org/wiki/Random-access_memory#Memory_wall)，在 Patricia 的点查里面，对 DFA 状态的访存（随机访问），占了 90% 的时间，所以，可以看到，顺序和随机搜索，性能差了那么多，其实就是随机访存的缘故！

NLT 的点查与 LB 性能比 Patricia 更接近，是因为在 NLT 中，内存的访问更加随机，内存墙造成的影响更大！

下面是性能对比图，测试用的数据是维基百科英文版的标题:

平均长度|20.624	
------:|-------:
数据总条数|29,381,585	
数据总尺寸|635,342,915

我们还加入了和静态 Patricia `NLT`(Nest Louds Trie) 的对比
- NLT 的**插入**指从排序好的 `StringVector` 创建 NLT

图中 **OPS** 表示每秒**插入**或**搜索**多少个 Key（1 M op/sec 表示每秒 100万 次操作），把 OPS 折算成吞吐率，要乘以 Key 的平均长度，对这个维基百科数据，要乘以 20。

### 内存占用
对于 Patricia Trie，在大部分情况下，随机插入比顺序插入要占用更多内存，极端情况下，内存占用会多出 20%。即便如此，内存占用仍比 RbTree 低很多：

![patricia-wikipedia-memory](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/patricia/wikipedia-memory.svg)

### 性能对比：数据已排序
![patricia-wikipedia-sorted](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/patricia/wikipedia-sorted-2.svg)

### 性能对比：数据随机打乱
![patricia-wikipedia-rand](https://cdn.rawgit.com/wiki/Terark/terarkdb/images/patricia/wikipedia-rand-2.svg)

这个性能意味着：把这个 Patricia Trie 用在 RocksDB 的 MemTable 上，性能会比 RocksDB 自身的 `vector MemTable` 更高！要知道，`vector MemTable` 为了写性能而牺牲了读性能，读(搜索)是 O(n) 的，这相当于放弃了读(搜索)功能！
