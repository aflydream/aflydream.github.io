# 如何提升MySql的插入性能（一）innodb_flush_log_at_trx_commit参数和Mysql连接数

## 前言
最近我们小组接到一个需求，客户端每秒会发送一条数据到服务端，服务端需要用保存数据，而客户端的同时并发大约为1000，持续时间大约为1小时，哇，心里有点爽，终于有点小boss可以练手了。但是经过测试，现有的程序只能撑到50并发，而罪魁祸首主要有Mysql,程序代码，Nginx等。
因此我们需要将它一一击破才行，此篇为通关的第一步

## 测试运行环境
可以说没有给出运行环境的测试都是耍流氓，不同的运行环境跑出的测试结果将大相径庭，如果没有给出运行环境，基准测试或压力测试结果将没有参考意义，请注意，下面开始正经了
```	
head -n 1 /etc/issue # 查看操作系统版本    Ubuntu 16.04.4 LTS
cat /proc/cpuinfo # 查看CPU信息           四核四线程 Intel(R) Core(TM) i5-7500 CPU @ 3.40GHz
free -h # 查看内存使用量和交换区使用量      16G
smartctl --all /dev/sda2 #查看硬盘信息    渣渣机械硬盘 硬盘型号：18F0D56E-325A-4A84-947A-7EDCB0CC1AB2
```

## 安装基准测试工具Sysbench
### 安装
1.下载解压
```
wget https://github.com/akopytov/sysbench/archive/1.0.zip -O "sysbench-1.0.zip"
unzip sysbench-1.0.zip
cd sysbench-1.0
```
2.安装依赖
```
yum install automake libtool –y
```
3.安装
安装之前，确保位于之前解压的sysbench目录中。
```
./autogen.sh
./configure
export LD_LIBRARY_PATH=/usr/local/mysql/include #这里换成机器中mysql路径下的include
make
make install
```
4.安装成功
```
[root@test sysbench-1.0]# sysbench --version
sysbench 1.0.9
```

## 准备阶段
调整同时打开文件数量，不然随着线程数增多将会报FATAL: error 2001: Can't create UNIX socket (24)错误
```
ulimit -n 20480
```

## 开始测试

本次测试只测试插入，因而选用insert.lua脚本，其他脚本请进入**/sysbench/sysbench-1.0/tests/include/oltp_legacy/**中查看并使用
```
cd /sysbench/sysbench-1.0
# 准备数据
sysbench --test=./tests/include/oltp_legacy/insert.lua --tables=1 --table-size=200000 --threads=16 --time=120 --report-interval=3 --mysql-user=root --mysql-password=123456 --mysql-host=localhost --max-requests=0 --mysql-db=sbtest prepare
# 执行测试
sysbench --test=./tests/include/oltp_legacy/insert.lua --tables=1 --table-size=200000 --threads=16 --time=120 --report-interval=3 --mysql-user=root --mysql-password=123456 --mysql-host=localhost --max-requests=0 --mysql-db=sbtest run
# 执行测试后，清理数据
sysbench --test=./tests/include/oltp_legacy/insert.lua --tables=1 --table-size=200000 --threads=16 --time=120 --report-interval=3 --mysql-user=root --mysql-password=123456 --mysql-host=localhost --max-requests=0 --mysql-db=sbtest cleanup
```


## 测试样例
如下为一个测试样例，上面的部分为测试过程，下半部分为测试的一些参数
```
Running the test with following options:
Number of threads: 5
Report intermediate results every 3 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 3s ] thds: 5 tps: 791.12 qps: 791.12 (r/w/o: 0.00/791.12/0.00) lat (ms,95%): 35.59 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 5 tps: 10868.36 qps: 10868.36 (r/w/o: 0.00/10868.36/0.00) lat (ms,95%): 0.47 err/s: 0.00 reconn/s: 0.00
[ 9s ] thds: 5 tps: 13490.09 qps: 13490.09 (r/w/o: 0.00/13490.09/0.00) lat (ms,95%): 0.47 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 5 tps: 13360.02 qps: 13360.02 (r/w/o: 0.00/13360.02/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 15s ] thds: 5 tps: 10619.36 qps: 10619.36 (r/w/o: 0.00/10619.36/0.00) lat (ms,95%): 0.45 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 5 tps: 10563.19 qps: 10563.19 (r/w/o: 0.00/10563.19/0.00) lat (ms,95%): 0.49 err/s: 0.00 reconn/s: 0.00
[ 21s ] thds: 5 tps: 13543.55 qps: 13543.55 (r/w/o: 0.00/13543.55/0.00) lat (ms,95%): 0.50 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 5 tps: 9801.62 qps: 9801.62 (r/w/o: 0.00/9801.62/0.00) lat (ms,95%): 0.41 err/s: 0.00 reconn/s: 0.00
[ 27s ] thds: 5 tps: 14806.13 qps: 14806.13 (r/w/o: 0.00/14806.13/0.00) lat (ms,95%): 0.43 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 5 tps: 10318.21 qps: 10318.21 (r/w/o: 0.00/10318.21/0.00) lat (ms,95%): 0.45 err/s: 0.00 reconn/s: 0.00
[ 33s ] thds: 5 tps: 12240.22 qps: 12240.22 (r/w/o: 0.00/12240.22/0.00) lat (ms,95%): 0.47 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 5 tps: 12890.49 qps: 12890.49 (r/w/o: 0.00/12890.49/0.00) lat (ms,95%): 0.43 err/s: 0.00 reconn/s: 0.00
[ 39s ] thds: 5 tps: 9354.06 qps: 9354.06 (r/w/o: 0.00/9354.06/0.00) lat (ms,95%): 0.52 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 5 tps: 10884.25 qps: 10884.25 (r/w/o: 0.00/10884.25/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 45s ] thds: 5 tps: 7628.99 qps: 7628.99 (r/w/o: 0.00/7628.99/0.00) lat (ms,95%): 0.44 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 5 tps: 11770.19 qps: 11770.19 (r/w/o: 0.00/11770.19/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 51s ] thds: 5 tps: 9290.30 qps: 9290.30 (r/w/o: 0.00/9290.30/0.00) lat (ms,95%): 0.48 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 5 tps: 9802.77 qps: 9802.77 (r/w/o: 0.00/9802.77/0.00) lat (ms,95%): 0.53 err/s: 0.00 reconn/s: 0.00
[ 57s ] thds: 5 tps: 9642.22 qps: 9642.56 (r/w/o: 0.00/9642.56/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 5 tps: 13468.39 qps: 13468.05 (r/w/o: 0.00/13468.05/0.00) lat (ms,95%): 0.50 err/s: 0.00 reconn/s: 0.00
[ 63s ] thds: 5 tps: 10889.59 qps: 10889.59 (r/w/o: 0.00/10889.59/0.00) lat (ms,95%): 0.42 err/s: 0.00 reconn/s: 0.00
[ 66s ] thds: 5 tps: 11606.92 qps: 11606.92 (r/w/o: 0.00/11606.92/0.00) lat (ms,95%): 0.41 err/s: 0.00 reconn/s: 0.00
[ 69s ] thds: 5 tps: 12245.12 qps: 12245.12 (r/w/o: 0.00/12245.12/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 72s ] thds: 5 tps: 9698.47 qps: 9698.47 (r/w/o: 0.00/9698.47/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 75s ] thds: 5 tps: 11216.82 qps: 11216.82 (r/w/o: 0.00/11216.82/0.00) lat (ms,95%): 0.47 err/s: 0.00 reconn/s: 0.00
[ 78s ] thds: 5 tps: 6156.04 qps: 6156.04 (r/w/o: 0.00/6156.04/0.00) lat (ms,95%): 0.43 err/s: 0.00 reconn/s: 0.00
[ 81s ] thds: 5 tps: 9751.58 qps: 9751.58 (r/w/o: 0.00/9751.58/0.00) lat (ms,95%): 0.42 err/s: 0.00 reconn/s: 0.00
[ 84s ] thds: 5 tps: 11319.16 qps: 11319.16 (r/w/o: 0.00/11319.16/0.00) lat (ms,95%): 0.42 err/s: 0.00 reconn/s: 0.00
[ 87s ] thds: 5 tps: 8658.20 qps: 8658.20 (r/w/o: 0.00/8658.20/0.00) lat (ms,95%): 0.47 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 5 tps: 4762.77 qps: 4762.77 (r/w/o: 0.00/4762.77/0.00) lat (ms,95%): 0.51 err/s: 0.00 reconn/s: 0.00
[ 93s ] thds: 5 tps: 13365.72 qps: 13365.72 (r/w/o: 0.00/13365.72/0.00) lat (ms,95%): 0.42 err/s: 0.00 reconn/s: 0.00
[ 96s ] thds: 5 tps: 8843.30 qps: 8843.30 (r/w/o: 0.00/8843.30/0.00) lat (ms,95%): 0.37 err/s: 0.00 reconn/s: 0.00
[ 99s ] thds: 5 tps: 7045.58 qps: 7045.58 (r/w/o: 0.00/7045.58/0.00) lat (ms,95%): 0.43 err/s: 0.00 reconn/s: 0.00
[ 102s ] thds: 5 tps: 10553.68 qps: 10553.68 (r/w/o: 0.00/10553.68/0.00) lat (ms,95%): 0.45 err/s: 0.00 reconn/s: 0.00
[ 105s ] thds: 5 tps: 9271.14 qps: 9271.14 (r/w/o: 0.00/9271.14/0.00) lat (ms,95%): 0.44 err/s: 0.00 reconn/s: 0.00
[ 108s ] thds: 5 tps: 9715.30 qps: 9715.30 (r/w/o: 0.00/9715.30/0.00) lat (ms,95%): 0.49 err/s: 0.00 reconn/s: 0.00
[ 111s ] thds: 5 tps: 8085.12 qps: 8085.12 (r/w/o: 0.00/8085.12/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
[ 114s ] thds: 5 tps: 11750.16 qps: 11750.16 (r/w/o: 0.00/11750.16/0.00) lat (ms,95%): 0.42 err/s: 0.00 reconn/s: 0.00
[ 117s ] thds: 5 tps: 10508.96 qps: 10508.96 (r/w/o: 0.00/10508.96/0.00) lat (ms,95%): 0.39 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 5 tps: 8244.93 qps: 8244.93 (r/w/o: 0.00/8244.93/0.00) lat (ms,95%): 0.46 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           1226449
        other:                           0
        total:                           1226449
    transactions:                        1226449 (10220.16 per sec.)
    queries:                             1226449 (10220.16 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0017s
    total number of events:              1226449

Latency (ms):
         min:                                    0.04
         avg:                                    0.49
         max:                                  835.97
         95th percentile:                        0.46
         sum:                               598585.94

Threads fairness:
    events (avg/stddev):           245289.8000/2231.38
    execution time (avg/stddev):   119.7172/0.00
```

其中:  
**transactions**: 事务总数及tps  
**queries**: 查询总数及qps

运行过程中：  
1. 没有异常，并且已入库  
2. cpu最高占用285%(最高400%),内存最高占用22%



## 测试结果
### 设置innodb_flush_log_at_trx_commit=1
```
SET GLOBAL innodb_flush_log_at_trx_commit=1;
SHOW GLOBAL VARIABLES LIKE 'innodb_flush_log%';
```
|线程数/连接数|qps|tps|
|:----- |:-----|----- |
|1|17.61|17.61|
|4|38.59|38.59|
|50|457|457|
|100|802|802|
|200|1418|1418|
|500|1913|1913|
|1000|2159|2159|
|5000|2431|2431|
|5500|2079|2079|


### 设置innodb_flush_log_at_trx_commit=2
```
SET GLOBAL innodb_flush_log_at_trx_commit=2;
SHOW GLOBAL VARIABLES LIKE 'innodb_flush_log%';
```

|线程数/连接数|qps|tps|
|:----- |:-----|----- |
|1|7617|7617|
|4|11108|11108|
|5|10220|10220|
|6|8560|8560|
|10|8851|8851|
|50|7038|7038|
|100|5285|5285|
|200|5601|5601|
|500|4786|4786|
|1000|855|855|

以上结果仅供参考，即使是相同运行环境相同的线程数，执行多次结果也会有很大不同，并不能代表有同样的机器同样的执行方式就能一定跑出多少qps


# 总结
#### 何为innodb_flush_log_at_trx_commit？
从测试结果看设置innodb_flush_log_at_trx_commit=2后吞吐率大大提升，那么何为innodb_flush_log_at_trx_commit？  
**简而言之，innodb_flush_log_at_trx_commit 参数指定了 InnoDB 在事务提交后的日志写入频率**。  
**当 innodb_flush_log_at_trx_commit 取值为 0 的时候**，log buffer 会 每秒写入到日志文件并刷写（flush）到磁盘。但每次事务提交不会有任何影响，也就是 log buffer 的刷写操作和事务提交操作没有关系。在这种情况下，MySQL性能最好，但如果 mysqld 进程崩溃，通常会导致最后 1s 的日志丢失。  
**当取值为 1 时**，每次事务提交时，log buffer 会被写入到日志文件并刷写到磁盘。这也是默认值。这是最安全的配置，但由于每次事务都需要进行磁盘I/O，所以也最慢。  
**当取值为 2 时**，每次事务提交会写入日志文件，但并不会立即刷写到磁盘，日志文件会每秒刷写一次到磁盘。这时如果 mysqld 进程崩溃，由于日志已经写入到系统缓存，所以并不会丢失数据；在操作系统崩溃的情况下，通常会导致最后 1s 的日志丢失。
上面说到的「最后 1s」并不是绝对的，有的时候会丢失更多数据。有时候由于调度的问题，每秒刷写（once-per-second flushing）并不能保证 100% 执行。对于一些数据一致性和完整性要求不高的应用，配置为 2 就足够了；如果为了最高性能，可以设置为 0。有些应用，如支付服务，对一致性和完整性要求很高，所以即使最慢，也最好设置为 1

#### 对innodb_flush_log_at_trx_commit=2时测试结果的理解
可以看到在线程数为4~10的时候吞吐率是比较大的，之后开始下降
，难道线程数不是越多越好么？  
其实不是的，别忘了我们运行环境是在四核四核心的机器上运行的，**即使是单核CPU的计算机也能“同时”运行数百个线程。但我们都[应该]知道这只不过是操作系统用时间分片玩的一个小把戏**。一颗CPU核心同一时刻只能执行一个线程，然后操作系统切换上下文，核心开始执行另一个线程的代码，以此类推。给定一颗CPU核心，其顺序执行A和B永远比通过时间分片“同时”执行A和B要快，这是一条计算机科学的基本法则。一旦线程的数量超过了CPU核心的数量，再增加线程数系统就只会更慢，而不是更快。   
然而我们用的是渣渣机械硬盘，磁盘通常是由一些旋转着的金属碟片和一个装在步进马达上的读写头组成的。读/写头同一时刻只能出现在一个地方，然后它必须“寻址”到另外一个位置来执行另一次读写操作。所以就有了寻址的耗时，此外还有旋回耗时，读写头需要等待碟片上的目标数据“旋转到位”才能进行操作。使用缓存当然是能够提升性能的，但上述原理仍然成立。
在这一时间段（即"I/O等待"）内，线程是在“阻塞”着等待磁盘，此时操作系统可以将那个空闲的CPU核心用于服务其他线程。所以，由于线程总是在I/O上阻塞，我们可以让线程/连接数比CPU核心多一些，这样能够在同样的时间内完成更多的工作。我们需要更多的线程。  
以下的公式是由PostgreSQL提供的：**连接数 = ((核心数 * 2) + 有效磁盘数**)，四核四线程一个渣渣机械硬盘的话大约为9

打个比方，一个程序猿小哥哥有两个项目要做，一定是做完A在做完B快，而不是写A项目写一行代码切换到B写一行代码在切换回来，更多时间浪费在来回切换上了

#### 对innodb_flush_log_at_trx_commit=1时测试结果的理解
可以看到线程数越多，吞吐率越大，当达到某个临界值后开始下降。
实际上innodb_flush_log_at_trx_commit=1时每次事务提交时并刷写到磁盘，别忘记了我们是渣渣机械硬盘，而刷新磁盘磁头寻址这段等待时间是浪费掉的，这种设置方式要比设置为2时影响要大得多，而这段时间完全可以用来执行其他任务，所以线程数多在刷新磁盘这段时间创建其他链接，省下的时间越多。然而核心数有限，线程数越多同样切换时间片也在浪费时间，但这两个值终究会有一个临界值，达到临界值后吞吐率便开始下降  

打个比方，现在，一个程序猿小哥哥有两个项目要做，并且每个项目的一个模块做完都要等待猿老大的审核才可以，那么实际上程序猿小哥哥可以利用审核的这段时间去写其他模块这样效率会更高，而这里程序猿小哥哥是在来回切换做A,B项目，所以也是有切换的浪费时间，
当节约的时间和浪费的时间达到一个平衡点以后，这种优势将不存在  
虽然这种方式可以加大吞吐率，但是我们不建议这种方式，因为线程数越多系统浪费的资源也会越多

# 后言
事实上，设置innodb_flush_log_at_trx_commit时候我们还是需要根据具体的需求，在一般对于数据完整性不是很高的情况下，我们可以根据公式**连接数 = ((核心数 * 2) + 有效磁盘数**进行设置，但是公式不是绝对的，想要最佳性能还需要多次测试才行，不同运行环境都会有很大差异，即使是相同的运行环境也不能测试一次就找到最佳的连接数。  
即使对于数据完整性和可用性高，可以适当加大连接数，也不要将连接数设置过大，这样会造成系统资源的浪费，往往三高（高性能，高并发，高可用）中有一个最重要的，而不能完全兼得。  

我们知道了如何提升mysql的插入性能，那么程序连接数据库后如何提升呢？我们从程序连接mysql通常会有四个参数需要设置，初始化连接数，最小连接数，最大连接数，过期时间，这些参数要设置多少才合适呢？且听下回分解

# 本篇文章离不开以下优秀的作者  
[数据库连接池到底应该设置多大?](https://blog.csdn.net/w05980598/article/details/78797310)  
[CPU 扫盲（核心数/线程数）](https://durant35.github.io/2017/05/16/hsw_CPUWipeoutIlliteracy/)  
[详解MySQL基准测试和sysbench工具](https://www.cnblogs.com/kismetv/p/7615738.html)  
[MySQL参数：innodb_flush_log_at_trx_commit 和 sync_binlog](http://liyangliang.me/posts/2014/03/innodb_flush_log_at_trx_commit-and-sync_binlog/)  