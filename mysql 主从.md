## mysql 主从介绍

> mysql 主从官方称谓叫做 Replication，AB 复制，简单讲就是 A 和 B 两台机器做主从后，在 A 上写数据，B 上也会跟着写数据，两者是实时同步的。

> mysql 主从是基于 binlog(Binary log) 的，主上需要开启 binlog 才能进行主从。

#### 主从过程大致有 3 个步骤：
1. 主将更改操作记录到 binlog 中。
2. 从将主的 binlog (sql语句) 同步到从本机上并记录在relay log 中，为中继日志。
3. 从根据 relay log里面的 sql 语句按顺序执行。
- 主上有一个 log dump 线程，用来和从的 I/O 线程传递 binlog 。
- 从上有两个线程，其中 I/O 线程用来同步主的 binlog 并生成 relay log，
另外一个 sql 线程用来把 relay log 里面的语句落地。

> A --> change data --> bin_log --transfer--> B --> repl_log --> change data
