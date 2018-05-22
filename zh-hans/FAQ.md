# 常见问题

## TerarkDB 和 RocksDB 是什么关系?
TerarkDB 使用自己的算法和数据结构实现了一个 SSTable，获得了更好的随机读性能和更好的压缩率.

## TerarkDB 是否有修改 RocksDB 的代码?
是的，TerarkDB 对 RocksDB 做了少量修改:
- 增加了在 TerarkDB SSTable 创建时的两边扫描支持，以减少磁盘使用(不影响现有的其他 SSTable).
- 增加了 TerarkDB 环境变量配置(这样用户可以在使用我们的 `librocksdb.so` 替换官方版本后，直接设置环境变量就可以支持 TerarkDB).

## TerarkDB 是否兼容 RocksDB?
TerarkDB 没有修改任何 RocksDB 用户层的 API，从用户的角度 TerarkDB 和 RocksDB 100% 兼容，用户在替换为 TerarkDB 后甚至不需要重新编译应用。

## TerarkDB 是否兼容 MongoDB 和 MySQL?
TerarkDB 通过 MongoRocks 和 MyRocks 兼容 MongoDB 和 MySQL。

## TerarkDB 作为商业产品，如何和 MongoDB 的 AGPL 协议兼容？
- 我们所有和 MongoDB 相关的代码，目前均已经开源（包括修改版的 MongoDB 和  MongoRocks 的修改）
- TerarkDB 目前以插件的形式嵌入 RocksDB，可以理解为开源产品的商业插件--我们通过动态库的方式使用私有库

## 在压缩过程中，对内存的需求如何？
- 对于 key 来说，内存占用的峰值大约是 `key 总长度 x 1.3`
- 对于 value 来说，内存占用大约是 `输入长度 x 18%`, 并且限定为上限是 12G

举例来说，假设要压缩 2TB 的原始数据，key 的总长为 30GB:
- TerarkDB 大概需要 39GB 的内存来压缩 key（创建 index）
- 大概需要 12GB 的内存来压缩数据本身
- key 和 value 在内存充足的情况下可以并行，在内存紧张的情况下可以顺序进行

## 为什么使用 universal compaction 而不是 level compaction?
Level compaction 写放大更为严重，会导致更低的写入速度以及更短的 SSD 寿命

## TerarkDB 随机读更快，并且压缩率也更高，那么损失的是什么？
我们的压缩速度更慢，并且在压缩过程中需要占用更多的内存. 当然，这并不会阻塞实时写入（异步压缩），单线程的写入，在普通的服务器上一般可以达到 15～50M/s, 多线程压缩的时候，速度可以满足据大多数场景使用

## 锁机制是什么样的？当压缩的时候，TerarkDB 会锁住整个表么？
TerarkDB 在压缩过程中并没有使用锁，如同 RocksDB 一样，SSTable 在压缩后就变成只读的，在压缩过程中是只写的，我们并没有对此部分逻辑进行修改

## 当一个巨大的 SSTable 需要压缩的时候，会出现什么问题(e.g. 5TB 大的一个)
一般不会出现如此大的 SSTable，当然 TerarkDB 支持这么大的 SSTable，但是这并没有必要，一般情况下几百 GB 是足够的


## 单个 SSTable 的最大尺寸能有多大？
视情况而有所不同，通常一个 SSTable 中的 KEY 不能超过 15 亿个，并且 KEY 的总长度可以很大（比如 30GB 是很安全的）

对于 Value 而言, 压缩后的总数据不能超过 128PB（实践中几乎不会出现），VALUE 的数量限制和 KEY 相同。


## 随机和顺序读的速度如何?
取决于数据集的大小和内存限制。

在 TPC-H lineitem 数据集上且不限制内存（每行 615 字节，KEY 23 字节，文本 512 字节），单线程：

- 对于 key, 大约 ~10MB/s, 在其他数据集上最高 ~80MB/s（Xeon 2630 v3）.
- 对于 value, 大约 ~500 MB/s, 在其他数据集上最高 ~7GB/s （Xeon 2630 v3）.

下表是一个相对的读速度对比：
- When Memory is <strong>slightly</strong> limited(all data fit in memory by TerarkDB, not by Other DBs)
<table>
<tr>
  <th>DB</th>
  <th>Read mode</th>
  <th>Speed</th>
  <th colspan="2">Description</th>
</tr>
<tr>
  <th rowspan="2">TerarkDB</th>
  <td>sequential</td>
  <td align="right">3000</td>
  <td rowspan="2">all data exactly in memory<br/>no page cache miss</td>
  <td>CPU Cache & TLB miss is <strong>medium</strong></td>
</tr>
<tr>
  <td>random</td>
  <td align="right">2000</td>
  <td>CPU Cache & TLB miss is <strong>heavy</strong></td>
</tr>
<tr>
  <th rowspan="2">Other DB</th>
  <td>sequential</td>
  <td align="right">10000</td>
  <td colspan="2">hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">10</td>
  <td colspan="2">no hot data, heavy cache miss</td>
</tr>
</table>

- When Memory is <strong>extremely</strong> limited

<table>
<tr>
  <th>DB</th>
  <th>Read mode</th>
  <th>Speed</th>
  <th>Description</th>
</tr>
<tr>
  <th rowspan="2">TerarkDB</th>
  <td>sequential</td>
  <td align="right">2000</td>
  <td>hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">10</td>
  <td>no hot data, heavy cache miss</td>
</tr>
<tr>
  <th rowspan="2">Other DB</th>
  <td>sequential</td>
  <td align="right">10000</td>
  <td>hot data always fit in cache</td>
</tr>
<tr>
  <td>random</td>
  <td align="right">1</td>
  <td>no hot data, <strong>very</strong> heavy cache miss</td>
</tr>
</table>

## 压缩速度怎么样？
取决于数据集

在 TPC-H lineitem 数据集上（行长度 615 bytes, key 23 bytes, text 长 512 ）, 单线程：

- key 约 9MB/s, 在其他数据集上最高 90MB/s（CPU Xeon 263 v3）
- value 约 40MB/s, 在其他数据集上最高 120MB/s(CPU Xeon 2630 v3)

## 压缩率总体来说怎么样?
这取决于测试数据集

在 wikipedia 数据集上，所有的英文文本大约 109GB，我们可以压缩到 23GB

在 TPC-H lineitem 数据集上（约550GB，key 22G，value 525GBB，平均记录长度 615 bytes）：
- key 压缩后是 5.3GB 左右
- value 压缩后是 24GB 左右（SSTable 尺寸是 29.3GB）

## key/value 的长度，对压缩率有何影响
- 对于 key 来说，小于 80 bytes 是最理想的，当然其他尺寸也不会太差
- 对于 value，50 bytes 到 30KB 是最理想的，其他尺寸同样不会差很多

