[English](Notes-about-mmap-and-cgroup)
## mmap 与 page cache

我这里不会介绍 mmap 和操作系统的的原理等背景知识，这些背景知识你可以参考 [memory-faq](https://landley.net/writing/memory-faq.txt)。

这里只讲几个容易被忽略的要点（以下讲的 page fault 不包含因写保护、权限等问题产生的 page fault）：
1. 当我们在一个文件上调用 `mmap` 时，如果这个文件的（一些）内容已经在操作系统的 page cache 中了，那么这些 page 会直接映射到这个 `mmap` 空间中，也就是说，访问这些 page 时不会产生 page fault。

1. 如果我们访问（基于文件的） mmap 的内存时触发了一个 page fault 。
    * 这个 page 可能已经在操作系统缓存中了，但是还没有注册（关联）到这个 mmap 中，这叫做 `soft page fault` 或者 `minor page fault`。
    * 如果这个 page 不在操作系统缓存中，它就需要从文件中读进来，这叫做 `hard page fault` 或者 `major page fault`。
1. 当一个 `major page fault` 发生时，操作系统需要加载这个 page。
    * 在大多数情况下，如果我们访问一个 page，紧邻这个 page 的后面一些 page 也大概率会很快就被访问。
    * 所以操作系统会执行 read ahead（前向预读，有些操作系统还会执行 read behind，即后向预读），对于 linux，默认会前向预读 31 个 page，再加上那个 page 本身，相当于一次 `major page fault` 会载入 32 个 page。
1. 前向预读的那些 page 只会放入操作系统 cache，不会放入引发这个前向预读的 `mmap`，只有被用户层访问到的那个 page 才会放入 `mmap`。预读的这些 page 只有在后来的 `minor page fault` 中才会放入 `mmap`。
1. **随机访问** `mmap` 的数据时，read ahead 毫无用处，并且还会极大地降低系统性能。当然，操作系统也提供了这种场景下的应对机制，对于 posix 系统，用 `POSIX_MADV_RANDOM` 参数调用 `posix_madvise`，可以告诉操作系统：这块内存我们是随机访问的，请不要进行 read ahead。

## 随机访问中使用 read ahead 有多可怕
曾经：512G内存 + 6T SSD(iops 50万)，对 TerarkDB 进行随机读压测，但是没有开启 `POSIX_MADV_RANDOM`，结果是 QPS 低的一塌糊涂，机器负载高到完全失去响应！

监控显示： `major page fault` 很高，SSD 5GB+ 的带宽全被耗尽，但同时 TerarkDB 进程的内存用量却很低！

**天生异象，必有妖孽**：

每次对 `mmap` 内存的随机访问，都会导致 read ahead，这些 read ahead 进来的 page 既毫无用处，又被操作系统当作“很有可能被访问”的数据，从而导致访问频率稍低，但仍然真正会被访问到的那些 page，都被挤出去了！这一点表现在进程刚启动时内存用量是 140GB （TerarkDB 默认对 index 进行 preload，所以这 140G 主要是 index），几分钟后就降到了 **80GB**，但是，使用 vmtouch 却能看到 terarkdb 的文件占用了接近 **500GB** 的系统 cache！
<table><tr><td>
<p>
mmap 能看到的内存(进程的 SHR 内存)只有 80GB，但这些 mmap 关联的文件却消耗了 500GB 的系统缓存，这样就可以推测出：read ahead 进来的数据并没有马上关联到 mmap！
</p>
<p>
再深入探究一下，我们就知道：read ahead 进来的数据，要等待一个对 mmap 的访问，产生一个 page fault，在该 page fault 的处理中，将来自 read ahead 的 page cache，关联到该 mmap，这就是前面说的 minor page fault。
</p>
</td></tr></table>

## cgroup 内存限制
在 [cgroup 文档](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) 讲到：
```
2.3 Shared Page Accounting

Shared pages are accounted on the basis of the first touch approach. The
cgroup that first touches a page is accounted for the page. The principle
behind this approach is that a cgroup that aggressively uses a shared
page will eventually get charged for it (once it is uncharged from
the cgroup that brought it in -- this will happen on memory pressure).
```
这就意味着：如果我们先通过系统调用 `read` 读文件，或者系统调用 `write` 写文件，然后再 `mmap` 这个文件，我们就可以绕过 cgroup 的内存限制了！因为文件内容已经通过 `read` 或 `write` 进入了操作系统缓存中，再调用 `mmap` 时，已在操作系统缓存中的那些 page 不会被 cgroup 记账。此时，如果我们通过 `top` 等监控命令查看，就会发现进程的内存占用远大于 cgroup 内存限制。

从这一点看，cgroup 的隔离性还是太差，只能在完全受控的环境中使用。

好消息是，Linux 新版中，已在考虑将 File System Cache 也纳入 cgroup 记账……