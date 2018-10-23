[English](Engine-test-using-Dockerfile.html)

**1. 获取 `Dockerfile` 用于构建基础环境, 放在任意目录中，如 `terarkdb-tests/Dockerfile`:**

```
# use ubuntu 16.04
FROM hub.c.163.com/library/ubuntu:16.04

# tsing hua deb source
RUN echo '# Tsinghua deb source \n\
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse  \n\
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse \n\
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse \n\
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse'  >> /etc/apt/sources.list

RUN apt-get update
RUN apt-get install vim -y

# install gcc
RUN apt-get install build-essential -y; exit 0
RUN apt-get install gcc-5 -y
RUN apt-get install gcc-4.8 -y
RUN apt-get install libnuma-dev -y

RUN apt-get update
RUN apt-get install wget -y

RUN apt-get install perl -y
RUN apt-get install libaio1 libaio-dev -y

# extra tools
RUN apt-get install libreadline6 libreadline6-dev -y
# RUN apt-get install openjdk-8-jre -y; exit 0
RUN apt-get install git -y; exit 0
RUN apt-get install python -y; exit 0

# get terarkdb-tests
RUN cd /home && wget http://terark-downloads.oss-cn-qingdao.aliyuncs.com/docker_assets/terarkdb-tests-pkg.tar.gz
RUN cd /home && tar -zxvf terarkdb-tests-pkg.tar.gz
```

**2. 构建镜像**

```
cd terarkdb-tests
docker build -t terarkdb/tests .  # 镜像名称命名为 `terarkdb/tests`
```

**3. 通过镜像创建容器并进入容器 shell**

```
docker run -it --name test_name \
-v /data/publicdata/wikipedia:/data \
-v /path/to/terarkdb-tests-docker-data:/database \
terarkdb/tests
```

其中:
- `/data/publicdata/wikipedia` 下存储 `wikipidea.txt` 和 `wikipidea_key.txt` 数据文件
- `/path/to/terarkdb-tests-docker-data` 为数据库文件存储路径，一般使用 SSD 盘，需要提前创建一个 `tempdir` 目录
- 也可以另挂载一个 SSD 盘目录来单独作为 `tempdir` 目录

**4. 进入镜像后:**

引擎的测试分为两个阶段， 分别是数据加载阶段，和数据读写混合阶段：

```
cd /home/terarkdb-tests-pkg

# 执行数据加载测试，改测试会读取 `wikipidea.txt` 数据文件并插入数据库
bash terarkdb_load_wikipedia.sh

# 数据加载完成后可以执行运行操作
bash terarkdb_run_wikipedia.sh
```

以上两个脚本中的参数，分别如下所示：

**4.1. `terarkdb_load_wikipedia.sh` 表示数据加载阶段（只写）的启动脚本：**

```
./bench_terarkdb
--action=load
--db=/database/terarkdb
--fields_num=15
--key_fields=0,1,2
--load_data_path=/data/wikipedia.txt
--terocksdb_tmpdir=/tempdir
--logdir=/database/log
--write_rate_limit=0
--auto_slowdown_write=0
--enable_auto_compact=0
--rocksdb_memtable=vector
--flush_threads=2
--disable_wal
--num_levels=9
--index_nest_level=2
--compact_threads=2
--use_universal_compaction=1
--target_file_size_multiplier=2
--mysql_passwd=******
--alt_engine_name=terarkdb_load_wikipedia
```

重点参数说明：

- `write_rate_limit`: 设定写速度，使写操作尽量按该速度进行。默认为 30MB/s。当设置为零时 `auto_slowdown_write` 才会生效。
- `auto_slowdown_write`: 仅在 `--write_rate_limit` 设置为零时生效。为 1 时，可能会因为 compact 太慢，导致写降速，为 0 时，对写速度不做限制，总是尽最大速度写入。在进行纯写测试时应设为0。
- `enable_auto_compact`: 是否开启自动 compact，纯写操作时应设为 0
- `rocksdb_memtable`: 设值该值时，必为 vector。设定时使用使用 VectorRepFactory，不设置时 RocksDB 默认使用 memtable。仅在 `--action=load` 时有用，使写性能更佳。
- `flush_threads`: 将 MemTable 刷新到 SST 文件的线程数，在我们的测试中设为2时性能最好，设为3、4时性能都不如设为2时。


**4.2. `terarkdb_run_wikipedia.sh` 表示数据压测阶段(读写混合)启动脚本：**

```
./bench_terarkdb
--action=run
--db=/database/terarkdb
--fields_num=15
--key_fields=0,1,2
--insert_data_path=/data/wikipedia.txt
--keys_data_path=/data/wikipedia_key.txt
--terocksdb_tmpdir=/tempdir
--logdir=/database/log
--write_rate_limit=10M
--flush_threads=2
--disable_wal
--num_levels=9
--index_nest_level=2
--compact_threads=2
--use_universal_compaction=1
--target_file_size_multiplier=2
--plan_config=0:90:10:0
--plan_config=1:100:0:0
--thread_plan_map=0-19:1
--thread_plan_map=20-20:0
--thread_num=21
--mysql_passwd=******
--alt_engine_name=terarkdb_run_wikipedia
```

重点参数说明：

- `thread_num`：设定run线程数量。
- `thread_plan_map`：将左闭右闭区间的线程标记为指定标记，如--thread_plan_map=0-19:1将0到19线程标记为1，--thread_plan_map=20-20:0将20线程标记为0。
- `plan_config`：设定2）中标记的线程的读写更新操作比例（读：写：更新），如 `--plan_config=1:100:0:0` 将标记为1的线程的读写更新比例设为 100：0：0，`--plan_config=0:90:10:0` 将标记为0的线程的读写更新比例设为90：10：0。
- 读写混合时，应将写速度适当降低，因为过快的写操作会影响到读操作。这时 `--write_rate_limit=10M` 将写速度设置为10MB/s，并仅使用一个线程来写，且该线程读写更新比列为 `90：10：0`。这时 `enable_auto_compact` 需要设置为1（默认值）以开启自动compact，否则会影响读操作。

完整参数说明：[选项参数options](https://github.com/Terark/terarkdb-tests/blob/master/docs/%E9%A6%96%E9%A1%B5.md#选项参数options)


**5. 使用用户自定义的测试数据集:**

数据加载阶段，修改以下参数即可：

```
...
--fields_num=N
--key_fields=n0,n1,n2
--load_data_path=/path/to/wikipedia.txt
...
```

说明：
- load_data_path 为需要加载的数据文件
- fields_num 为一行中共有多少（N）项，每项之间使用 \t 分割
- key_fields 使用逗号（,）分割的数字列表，表示使用每行中的第 n0、n1、n2 项的组合作为 key，索引从 0 开始


数据压测阶段：

```
...
--fields_num=15
--key_fields=0,1,2
--insert_data_path=/path/to/wikipedia.txt
--keys_data_path=/path/to/wikipedia_key.txt
...
```

说明：
- fields_num 和 key_fields 同上加载数据阶段
- insert_data_path 为插入操作中的插入数据的数据源文件
- keys_data_path 为读操作中使用的所有 key 的文件（不需要 shuffle）