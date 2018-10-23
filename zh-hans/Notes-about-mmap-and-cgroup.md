[中文版](mmap,-page-cache-与-cgroup)
## mmap and OS cache
We will not explain the detailed principle of mmap and OS cache, for reference, you can read [memory-faq](https://landley.net/writing/memory-faq.txt) for background knowledge.

Here we describe some notes:

1. When we calling `mmap` on a file, if (some of) the file contents(pages) is already in OS cache, these pages will bring into the mmap'ed area -- user app can access such area without page fault.
1. When we read a (file backed) `mmap` and triggered a `page fault`:
    * This page may be already in the OS cache, but has not registered to this `mmap`, this is called `soft page fault`, or `minor page fault`.
    * If this page is not in OS cache, it needs to be loaded from the backing file, this is called `hard page fault`, or `major page fault`.
1. When a `major page fault` happening, OS needs to load the requested page.
    * In most cases, when a page is requested, pages following this page is likely to be requested soon after.
    * So the OS will perform read ahead: read some following pages together(linux will read ahead 31 pages, so a page fault will read total 32 pages by default).
1. Read ahead pages will only be loaded into OS cache, but **will not** be registered into the `requesting mmap`(just the requested page will be registered to `mmap`). Read ahead pages will be registered into `mmap` in a later `minor page fault`.
1. When `mmap` data is accessed randomly, read ahead is useless, and will greatly hurt the performance. So we need to do something for this situation, posix provided a solution for this issue: by calling `posix_madvise` with `POSIX_MADV_RANDOM` on the target memory area.
    * We had encountered this issue: when random read terarkdb on fast SSD with insufficient memory without `POSIX_MADV_RANDOM`, `mmap` generated lots of major page faults, SSD bandwith(5GB+/sec) was exhausted, and memory usage of terarkdb process was stay very low! The (useless) read ahead pages were not registered to terarkdb's `mmap`, even the low frequent accessed pages were evicted out, the process memory usage dropped down and the performance dropped down further more.

## cgroup memory limit
In [cgroup doc](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt), it says:
```
2.3 Shared Page Accounting

Shared pages are accounted on the basis of the first touch approach. The
cgroup that first touches a page is accounted for the page. The principle
behind this approach is that a cgroup that aggressively uses a shared
page will eventually get charged for it (once it is uncharged from
the cgroup that brought it in -- this will happen on memory pressure).
```

This means that, if we access a file by `read`(read existing file) or `write`(write new file) system call, and `mmap` this file later, we can bypass the cgroup memory limit! This is because pages of the file content was brought into OS cache by `read` or `write` system call, and later `mmap` for these pages will not charged by cgroup. Thus the process memory will be much larger than cgroup memory limit(we can see process memory by `top` or any other commands).
