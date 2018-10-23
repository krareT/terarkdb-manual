## 当 SSD 的随机 IO 性能很高时

　　我们在测试中发现，即使对 `mmap` 的内存标记了 `MADV_RANDOM`，当随机访问量很高时（PCIe SSD 的随机 IO 性能很高），仍会导致操作系统失去响应。

　　经过分析，我们认为，`mmap` 时，操作系统无法第一时间得知 page 的实际访问情况，只能通过启发性的算法，将一些 page 从 `mmap` 中暂时移除，如果接下来该 page 被访问到，它就可以通过一个 `minor page fault` 重新变成 active，如果长时间未被访问到，当系统缺乏内存时，它就可以被回收。这样的方式在绝大多数情况下工作良好，但是在这种频繁随机访问的极端情况下，就会对系统造成很大的压力：kswap 进程满载，所有进程的内存申请都被阻塞，表现出整个系统都失去响应的状态。

　　根据一个直观的推测，如果我们使用 `pread` 去进行随机读，那么操作系统就有足够的信息去维护 LRU 链表，从而可以避免 `mmap` 导致的问题……

　　代码很快改完并进入测试，然而情况依旧，`pread` 并没有达到我们预期的效果……

## 神挡杀神，佛挡杀佛

　　什么也不能阻止 TerarkDB 的性能，我们决定自己做一个 user space page cache……

　　为了性能，我们实现了一个 `LruReadonlyPageCache`，顾名思义：LRU，Readonly，PageCache，并且，为了最大化性能，我们只针对高性能 SSD 做优化：**以单个 page 为 cache 的单位，这样就可以使用 hash table 去进行搜索**，当然，多线程的支持是必须的。

　　对于 LruCache，有一点很关键：每次访问时都需要对 lru list 进行修改，将访问到的那个节点移动到 lru head，这个操作需要修改 5 个结点(使用数组下标做链接):
1. 从 lru list 中删除命中结点 x：需要修改命中结点的前后两个结点
   ```c++
   void lru_remove(Node* base, size_t x) {
       auto next = base[x].lru_next;
       auto prev = base[x].lru_prev;
       base[next].lru_prev = prev;
       base[prev].lru_next = next;
   }
   ```
2. 将命中结点 x 插到 lru head 之后（0 是 head 伪结点）：
   ```c++
   void lru_insert_after_head(Node* base, size_t x) {
       auto n = base[0].lru_next;
       base[x].lru_next = n;
       base[x].lru_prev = 0;
       base[n].lru_prev = x;
       base[0].lru_next = x;
   }
   ```
　　其中 head 伪结点总在 CPU cache 中，其它 4 个结点都是随机访问，当然，还有至少一次对 hash bucket 的随机访问（每多一个 hash 冲突，就多一个随机访问），所以，在 cache 命中的情况下，至少需要对内存进行 5 次随机访问，在 cache 很大时，全随机访问几乎总会导致 CPU 的 tlb miss，这样，就变成了 10 次对物理内存的随机访问，并且这些访问都要加锁。这还不包括对 buffer 的 memcpy，好在对 buffer 的 memcpy 不需要加锁，并且我们尽最大可能省略 memcpy。

　　经过我们的仔细实现，最终达到这样的效果：
* 假定每次 `read` 操作读入 `O(1)` 个 page: `O(1)` 的意思是说，不会一次性读入很多个 page
* 那么每个 `read` 操作的时间复杂度都是 `O(1)`，特别是在 mutex 的临界区内，是严格的 `O(1)`
* 即使是关闭一个文件，一次产生大量可回收 page 时，时间复杂度依然是 `O(1)`

　　一切看上去都很美好，然而测试发现，当线程数量较多时，性能离预期相差太远！

　　接下来的解决办法，只有做 `sharding`：创建多个 Lru cache，然后根据 key(file, page_id) 做 hash 进行 sharding, sharding 的目的只有一个，就是减少修改 lru list 时的 mutex 锁冲突。

　　sharding 的效果非常好：在 benchmark 中，32线程，31 shards，500 字节的随机读达到 2300万 op/sec。唯一的“缺点”就是，和单 Lru Cache 相比，sharding lru 并不是严格精确的 Lru，好在 page 数量会非常大，会有几百万，甚至几千万，几亿个 page，在统计意义上，也算是“精确”的 Lru 了。

　　接下来，在接近全命中的场景下，和操作系统的 `pread` 对比了一下，竟然比操作系统的 `pread` 还要慢，操作系统的 `pread` 达到了 2600万 op/sec！我们知道，操作系统的 `pread` 是个系统调用，系统调用本身就比函数调用慢不少呢……但是，再仔细想想，操作系统自身的一些内存可以不用分页，减少了 tlb miss 的开销，并且，正因为操作系统的 `pread` 要达到这么快，它才不能去维护精确的 lru 链表，导致极端情况下的性能崩溃。

　　要比操作系统 `pread` 快，只有增大 shards，进一步减小锁冲突，仍然是 32 线程，shards 增到 61 时，随机读达到 2800万 op/sec

## 其它一些实现细节
### 碰到一个编译器优化 Bug（gcc-4.8）
有个多线程协调的地方，需要 yield 当前线程，然后再检查条件，我们在测试中一直使用 gcc-7.1，所有测试均顺利通过，当我们使用 gcc-4.8 编译并部署进生产系统，出现了非预期的死循环。

我们很快发现是编译器的问题，但到底是哪一块代码编译出错，却费了一番纠结，因为 gcc-4.8 使用 -O0 和 -O1 都没有问题，只有 -O2 以上才会出问题，而 -O2 以上编出来的代码很难调试……

最终，发现了一段可疑代码：
```c++
  // 没错，这就是下面要说的 避免多次 read 同一 page
  while (!nodes[p].is_loaded) {
    // waiting for other threads to load the page
    std::this_thread::yield();
  }
```
在目标代码中，yield 之后，没有重新读 `nodes[p].is_loaded`，该值的加载被提到了循环外！

解决的办法很简单，把 `is_loaded` 的定义为 volatile 即可！

### 避免多次 read 同一 page

在某些情况下，多个线程可能会同时去读某个相同的 page，这个 case 发生的概率很低，但我们仍然在一开始就做了处理。

虽然，对于 page cache 来说，即使没有处理这种情况，对性能的冲击也不会太大，但是，如果缓存的是重量级对象，这可能会造成非常严重的问题。例如（*以下为题外话*）：<hr>
RocksDB 中 SST 是作为 Cachable Object 来管理的，参数 max_open_files 是 SST Cache 的容量。该参数在 [MyRocks](https://github.com/facebook/mysql-5.6) 的 [fb-prod201704 release](https://github.com/facebook/mysql-5.6/blob/fb-prod201704/storage/rocksdb/ha_rocksdb.cc#L4101-L4108) 中被画蛇添足，强制设成非 -1 ，在 RocksDB 中，这就意味着，**打开数据库时，总是不加载任何 SST，只有在第一次用到某个 SST 时，才加载该 SST**，从而导致了一系列严重的问题：
* 导致非预期的，不可控的，难以忍受的延时（加载 SST 很慢）
* 更严重的是，我们的单个 SST(TerarkZipTable) 尺寸经常在 GB 级别，并且加载时默认会进行数据校验，这需要把整个 SST 读一遍，例如加载一个
50GB 的 SST 需要好几分钟
  * 从而，一个正常情况下预期延迟小于 1 毫秒 的 Query，实际耗时若干分钟！
* 这个问题进一步暴露出 RocksDB Cache 严重的设计问题：**多线程同时 Get 某个对象时，该对象会被多次创建……**
  * 80 个 Mysql 连接，就是 80 个服务线程！
  * 相同 SST 被加载 80 次，相同文件就 mmap 80 次，就需要 80 倍的虚存！
  * 效果就是系统卡死，虚存急剧飙升，乍看上去还以为是出了严重的内存泄露……
* 其实，从根上讲，至少在 OLTP 的场景下，**SST 对象应该是永久驻留的**，不应该被当作 Cachable Object 去换入换出。

所以，真问题，一定要解决；伪问题，不要画蛇添足！识别真问题与伪问题，是一个程序员的基本修养。

最终，我们修复了 RocksDB Cache 的多线程 Get 问题，还有 MyRocks 那个画蛇添足的问题([MyRocks 官方也修复了这个画蛇添足的问题](https://github.com/facebook/mysql-5.6/commit/9c1183075fc52c765e323c4a74fe295892c5ba13#diff-5a5525d170a985d103b518fa307b17ccL4105))。

---*题外话结束*---
<hr>
