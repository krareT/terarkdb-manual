[[中文 Chinese| 使用-Dockerfile-进行存储引擎测试]]

**1. Using `Dockerfile` to build a test environemnt, place the Dockerfile below into any directory, e.g. `terarkdb-tests/Dockerfile`:**

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

**2. Build docker image**

```
cd terarkdb-tests
docker build -t terarkdb/tests .  # change docker image name to `terarkdb/tests`
```

**3. Create contianer by docker and start it**

```
docker run -it --name test_name \
-v /data/publicdata/wikipedia:/data \
-v /path/to/terarkdb-tests-docker-data:/database \
terarkdb/tests
```

Explain:
- `/data/publicdata/wikipedia` Place downloaded data file wikipedia here, if don't have it, please contact us or use your own datafile, datafile format please refer to `https://github.com/Terark/terarkdb-tests/blob/master/docs/home.md`
- `/path/to/terarkdb-tests-docker-data` database working dir, please use SSD，need to create a `tempdir` dir first.
- You can also use another mounted directory as `tempdir`

**4. After start the container:**

Engine test has two different phases: data loading and test read/write：

```
cd /home/terarkdb-tests-pkg

# execute data loading, this will load data from `wikipidea.txt`
bash terarkdb_load_wikipedia.sh

# After data loading, execute read/write test
bash terarkdb_run_wikipedia.sh
```

Params explanition：

**4.1. `terarkdb_load_wikipedia.sh` means data loading phase script：**

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

Important params：

- `write_rate_limit`: Default is 30MB/s, when set to 0, `auto_slowdown_write` will be used(if set).
- `auto_slowdown_write`: Only enabled when `--write_rate_limit` is set to 0. If set to 1，it may cause slower write speed since compact is slow. If set to 0 it will try its best to write.
- `enable_auto_compact`: When test write performance, should set to 0
- `rocksdb_memtable`: must use `vector` or default RocksDB memtable will be used, only useful on data loading phase.
- `flush_threads`: Threads number that flush MemTable into SST file, default is 2.


**4.2. `terarkdb_run_wikipedia.sh` represent data read/write testing script：**

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

Important params：

- `thread_num`：Threads of the running thread.
- `thread_plan_map`：thread label setting, e.g. `--thread_plan_map=0-19:1` will label `0 to 19` threads as `1`，`--thread_plan_map=20-20:0` will label `thread 20` as `0`
- `plan_config`：Work with the settings above. e.g. `--plan_config=1:100:0:0` will make `thread label 1` has the read/write/update percent `100%, 0%, 0%`. `--plan_config=0:90:10:0` will mark `thread label 0` with `90%, 10%, 0%` read/write/update.
- When test mixed read/write, you should slow down write speed, since faster write speed will influence read performance. use `--write_rate_limit=10M` to set write limit as `10MB/s` and use only one thread to write.


**5. Use your own dataset:**

Read the docs of the test program：[Test Program Docs](https://github.com/Terark/terarkdb-tests/blob/master/docs/home.md)


Data load phase, change these params：

```
...
--fields_num=N
--key_fields=n0,n1,n2
--load_data_path=/path/to/wikipedia.txt
...
```

Explain：
- load_data_path : dataset file
- fields_num : How many items（N）in a line，each item splitted by `\t`
- key_fields : Splitted by `,`, e.g. `1,3,5` means use `1,3,5` columns as key


Compress phase：


```
...
--fields_num=15
--key_fields=0,1,2
--insert_data_path=/path/to/wikipedia.txt
--keys_data_path=/path/to/wikipedia_key.txt
...
```

Explain：
- fields_num and key_fields : same as loading phase
- insert_data_path : Dataset of insert operations
- keys_data_path : key set of read operations