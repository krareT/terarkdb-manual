## RocksDB 默认的 MemTable 是 SkipList

## Terark 实现了一个 RBTree 的 MemTable

## 多线程并发：双层 RBTree

### 基于 Array Index 的 RBTree

## 基于 PatriciaTrie 的 MemTable
### 使用 MemPool，用 uint32 偏移代替 64 bit 指针
#### 偏移对齐到 4，寻址空间扩大 4 倍，从 4G 扩大到 16G
### 使用 Copy On Write 解决并发问题
#### 多个并发级别: 

|并发级别|限制条件|
|-------|--------|
|NoWriteReadOnly|只读，不可修改|
|SingleThreadStrict|插入导致 Iterator 失效|
|SingleThreadShared|插入保持 Iterator 有效|
|OneWriteMultiRead|插入保持 Iterator 有效|
