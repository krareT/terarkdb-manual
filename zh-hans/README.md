## TerarkDB 说明文档

## [TerarkDB](https://github.com/Terark/terarkdb) = [terark rocksdb](https://github.com/Terark/rocksdb) + [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb)
- [terark rocksdb](https://github.com/Terark/rocksdb) 是我们修改版的 RocksDB，完全开源并且您可以自行编译
- [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb) 开源但您无法自行编译(依赖于个别私有库)

`terark-zip-rocksdb` 使用 Terark 特有技术实现了 `RocksDB` 的  [MemTable](重新实现-RocksDB-MemTable.html) 和 [SSTable](TerarkDB-SST-的创建过程.html)。

`TerarkZipTable` 使用 Terark 的可检索压缩技术达到了以下效果：

- 更高的随机读性能
- 更高的压缩率 (磁盘文件更小)<br/>
- 更低的内存使用 (对压缩文件使用 [mmap](http://man7.org/linux/man-pages/man2/mmap.2.html), 没有双缓存问题)

我们修改了 [原版的 RocksDB](https://github.com/facebook/rocksdb)，称为 [Terark RocksDB](https://github.com/Terark/rocksdb), 以方便更好的支持我们特有的 [MemTable](重新实现-RocksDB-MemTable.html) 和 [SSTable](TerarkDB-SST-的创建过程.html)(由 [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb) 实现).

使用 `TerarkDB` 您现在可以在 磁盘/SSD 上存储更多的数据（比 snappy 算法多三倍以上），同时可以加载更多数据进入内存（10倍以上），显著地提升随机读性能。

## 注意
- 如果不加载 `terark-zip-rocksdb`, [terark rocksdb](https://github.com/Terark/rocksdb) 和官方 [official rocksdb](https://github.com/facebook/rocksdb) 的使用和表现并无区别
- 如果没有获得商业授权, `terark-zip-rocksdb` 将会在一个月后过期并停止运行, **请不要在生产环境使用** 您可以通过官网联系我们获得商业授权。

## 1. 编译或下载
### 1.1 自行编译 <a href="https://github.com/terark/rocksdb">Terark RocksDB</a>
```bash
  git clone https://github.com/terark/terarkdb.git
  cd  terarkdb
  git submodule init
  git submodule update
  make -C rocksdb shared_lib DEBUG_LEVEL=0
```
### 1.2 下载预编译版本(推荐)
- [terark-zip-rocksdb](http://www.terark.com/download/terarkdb/latest)

**提示:**
- 你无法自行编译 [terark-zip-rocksdb](https://github.com/terark/terark-zip-rocksdb) 
- 不加载 `terark-zip-rocksdb` 的情况下 `Terark RocksDB` 和原版 RocksDB 表现完全一样

下载后的文件如下所示:
- terark-zip-rocksdb-Linux-x86_64-g++-5.4-`bmi2-1`.tgz<br/>
  &nbsp;&nbsp;&nbsp;-- `bmi2-1` 表示需要 bmi2 支持，只能运行在 intel-haswell 或更新的 CPU
- terark-zip-rocksdb-Linux-x86_64-g++-5.4-`bmi2-0`.tgz<br/>
  &nbsp;&nbsp;&nbsp;-- `bmi2-0` 表示可以运行在老一些的 CPT (但至少需要支持 popcnt 和 SSE4.2)

下载后的文件，包含了所有的依赖库:
```bash
  $ cd terark-zip-rocksdb-Linux-x86_64-g++-4.8-bmi2-1/lib
  # just show filename and symbol links

  $ ls -l libterark*-r.so # "-r" for "release version"
  libterark-core-g++-4.8-r.so
  libterark-core-r.so -> libterark-core-g++-4.8-r.so
  libterark-fsa-g++-4.8-r.so
  libterark-fsa-r.so -> libterark-fsa-g++-4.8-r.so
  libterark-zbs-g++-4.8-r.so
  libterark-zbs-r.so -> libterark-zbs-g++-4.8-r.so
  libterark-zip-rocksdb-g++-4.8-r.so
  libterark-zip-rocksdb-r.so -> libterark-zip-rocksdb-g++-4.8-r.so

  $ ls -l libterark*-d.so # "-d" for "debug version"
  libterark-core-d.so -> libterark-core-g++-4.8-d.so
  libterark-core-g++-4.8-d.so
  libterark-fsa-d.so -> libterark-fsa-g++-4.8-d.so
  libterark-fsa-g++-4.8-d.so
  libterark-zbs-d.so -> libterark-zbs-g++-4.8-d.so
  libterark-zbs-g++-4.8-d.so
  libterark-zip-rocksdb-d.so -> libterark-zip-rocksdb-g++-4.8-d.so
  libterark-zip-rocksdb-g++-4.8-d.so
```

## 2. 使用 TerarkDB
- [快速开始](快速开始.html)
  - 在您现有的 RocksDB 应用下直接使用 TerarkDB，无需重新编译应用层代码
- [使用 TerarkDB 的完整功能](完整功能.html)
  - 包含 TerarkDB 的全部功能（更细粒度的调用接口），可能需要稍微修改您应用层的代码.

## 3. 限制
- 用户自定义的 `comparator` 暂不支持，您需要按照字节序来处理您的所有 key. 
- `EnvOptions::use_mmap_reads` 必须设置为 `true`, 可以通过 `DBOptions::allow_mmap_reads` 设置

## 4. [优化指南](Tuning-Guide.html)

## 5. 授权协议
本软件自身是一款开源软件

### 5.1 `RocksDB` 子模块(git submodule)
[Submodule RocksDB](https://github.com/Terark/rocksdb) is our fork of RocksDB and can be compiled by yourself.

[Submodule RocksDB 的授权协议](https://github.com/Terark/rocksdb/blob/master/LICENSE) 和官方 RocksDB 相同(BSD clause 3).

### 5.2 `terark-zip-rocksdb` 子模块
[Submodule terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb) 使用 Terark 独创的技术实现了 RocksDB 的  [MemTable](重新实现-RocksDB-MemTable.html) 和 [SSTable](TerarkDB-SST-的创建过程.html), [terark-zip-rocksdb license](https://github.com/Terark/terark-zip-rocksdb/blob/master/LICENSE) 遵循 Apache 2.0，并增加了额外内容:
  * 您可以在 Apache 2.0 协议下使用和分发本产品
  * 由于此软件包含我们部分的专利算法，您不能自行编译本软件，除非获得我们的商业授权
  * 您可以下载预编译的 [TerarkDB](http://www.terark.com/download/terarkdb/latest)

## 6. Contacts
- [Terark.com](http://www.terark.com)
- contact@terark.com

