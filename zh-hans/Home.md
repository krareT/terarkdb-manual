[[中文 Chinese|首页]]

# TerarkDB Documentation

## [TerarkDB](https://github.com/Terark/terarkdb) = [terark rocksdb](https://github.com/Terark/rocksdb) + [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb)
- [terark rocksdb](https://github.com/Terark/rocksdb) is open source and can be compiled by yourself
- [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb) is open source but can **NOT** be compiled by yourself

`terark-zip-rocksdb` is for `RocksDB`'s `SSTable` implementation: `TerarkZipTable`.

`TerarkZipTable` use terark's searchable compression technology and achieved all of these:<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(1) much higher random read performance<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(2) much higher compression ratio (disk file is much smaller)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(3) much lower memory usage ([mmap](http://man7.org/linux/man-pages/man2/mmap.2.html) on compressed file, no double caching)

We forked [official rocksdb](https://github.com/facebook/rocksdb) as [terark rocksdb](https://github.com/Terark/rocksdb) and changed for better support `TerarkZipTable`(in [terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb)).

Now by using `TerarkDB`, you are able to store more data on disk (3x+ more than snappy, 2x+ more than zstd), and load more data into memory(10x+ more), and improve read performance greatly! All data are accessed at memory speed!

## Notes
- Without `terark-zip-rocksdb`, [terark rocksdb](https://github.com/Terark/rocksdb) works same as [official rocksdb](https://github.com/facebook/rocksdb)
- With `terark-zip-rocksdb`, you need [[a little config | Try TerarkDB With Minimal Effort]] to enable terark technology

## 1. Compile and Download
### 1.1 Compile <a href="https://github.com/terark/rocksdb">terark rocksdb</a> by yourself
```bash
  git clone https://github.com/terark/terarkdb.git
  cd  terarkdb
  git submodule init
  git submodule update
  make -C rocksdb shared_lib DEBUG_LEVEL=0
```
### 1.2 [Download precompiled terark-zip-rocksdb](http://www.terark.com/download/terarkdb/latest)
**Remember:**
- You can not compile [terark-zip-rocksdb](https://github.com/terark/terark-zip-rocksdb) by yourself
- Without `terark-zip-rocksdb`, `terark rocksdb` works exactly the same as `official rocksdb`

The download file names like:
- terark-zip-rocksdb-Linux-x86_64-g++-5.4-`bmi2-1`.tgz<br/>
  &nbsp;&nbsp;&nbsp;-- `bmi2-1` means this software can only run on intel-haswell or newer CPU
- terark-zip-rocksdb-Linux-x86_64-g++-5.4-`bmi2-0`.tgz<br/>
  &nbsp;&nbsp;&nbsp;-- `bmi2-0` means this software can run on older CPU (but the CPU must support popcnt & SSE4.2)

The downloaded package contains all dependencies you need as below:
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

## 2. Now, you can try TerarkDB
- [[Try TerarkDB With Minimal Effort]]
  - If you want to try TerarkDB the quickest way, do not need to re-compile your existing application.
- [[Try TerarkDB With Full Features]]
  - If you want to try TerarkDB's full features (e.g. more granular control on TerarkDB), with a little change of your existing application code.

## 3. Restrictions
- User [[Key Comparator]] is not supported, you should encode your keys in byte lexical order
- `EnvOptions::use_mmap_reads` must be `true`, can be set by `DBOptions::allow_mmap_reads`

## 4. [[Tuning Guide]]

## 5. License
This software is open source.

### 5.1 submodule rocksdb
[submodule rocksdb](https://github.com/Terark/rocksdb) is our fork of rocksdb and can be compiled by yourself.

[License of submodule rocksdb](https://github.com/Terark/rocksdb/blob/master/LICENSE) is same as offical rocksdb(BSD clause 3).

### 5.2 submodule terark-zip-rocksdb
[submodule terark-zip-rocksdb](https://github.com/Terark/terark-zip-rocksdb) implements an [[MemTable | 重新实现 RocksDB MemTable]] and an [[SSTable | TerarkDB SST building]] for [submodule rocksdb](https://github.com/Terark/rocksdb), [terark-zip-rocksdb license](https://github.com/Terark/terark-zip-rocksdb/blob/master/LICENSE) is Apache 2.0, with NOTES:
  * You can read or redistribute or use the source code under Apache 2.0 license
  * You can not compile this software by yourself, since this software depends on our proprietary core algorithms, which requires a commercial license
  * You can [download](http://www.terark.com/download/terarkdb/latest) the precompiled binary library of this software

## 6. Contacts
- [Terark.com](http://www.terark.com)
- contact@terark.com
