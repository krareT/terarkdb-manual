## This benchmark is based on TPCH data
We've changed `TPC-H`'s `dbgen` a little to generate the data we need, mostly changed its field length.

In the tables of `TPC-H`, `lineitem` has the largest table size, so we use `lineitem` for testing. **Notice**: TPC-H dbgen uses `|` as record separator.

TPC-H lineitem has a `comment` field, which is raw text format. This field conttribute the most to the compression. The field in `dbgen` is hard coded as 27 bytes. To make it flexiable and fit our needs, we added a feature that let us change the field length via environemnt variable. And besides that we added a new `dbgen.sh` to generate compressed db tables directly. [dbgen.sh intro](https://github.com/rockeet/tpch-dbgen)


## Platform
- CPU : Intel(R) Xeon(R) CPU E5-2682 v4 @ 2.50GHz 40 core
- Memory : 224GB
- DISK : 2TB SSD

## Compression
| Original Data Size | Record Size | TerarkDB Compressed Size | RocksDB Compressed Size |
|--------------------|-------------|--------------------------|-------------------------|
| 40 GB              | 128 bytes   | 8.1 GB                   | 19 GB                   |
| 104 GB             | 512 bytes   | 12 GB                    | 40 GB                   |
| 415 GB             | 512 bytes   | 34 GB                    | 170 GB                  |
| 128 GB             | 2048 bytes  | 4.4 GB                   | 35 GB                   |



## OPS
- Read Only
  - About 2,000,000/seconds
- Read Write Mixed
  - Reading keeps a pretty high performance, from 600,000/s to 1,200,000/s.
  - Writing keeps around 100,000/seconds

![](images/random_read.png)


## Random Read Latency

| Read Latency | Read Write Mixed | Read<br/>(No Background Compression) | Read<br/>(With Background Compression) |
|---------|------------------|---------------------------------|-----------------------------------|
| < 100ms | 100%             | 99.99%                          | 100%                              |
| < 15ms  | 99.99%           | 99.99%                          | 100%                              |
| < 5ms   | 99.98%           | 99.99%                          | 100%                              |
| < 2ms   | 99.94%           | 99.99%                          | 100%                              |
| < 0.5ms | 94.70%           | 99.99%                          | 99.99%                            |