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


## 配置主从 1

#### 1、在本机安装 2 个 mysql，第一个安装配置与 lamp 中一样，安装第二个：
```bash
cd /usr/local/
cp -r mysql mysql_slave
cd mysql_slave
cp /etc/my.cnf .
vim my.cnf
port            = 3307
socket          = /tmp/mysql_slave.sock
datadir         = /data/mysql_slave
./scripts/mysql_install_db --user=mysql --datadir=/data/mysql_slave
```

#### 2、启动
```bash
cd /etc/init.d/
cp mysqld mysqld_slave
vim !$
basedir=/usr/local/mysql_slave
datadir=/data/mysql_slave
conf=$basedir/my.cnf

/etc/init.d/mysql_slave start    
```

#### 3、登录 mysql
```bash
/usr/local/mysql/bin/mysql 
mysql -S /tmp/mysql.sock
mysql -h127.0.0.1 -P3307
create database db1;
/usr/local/mysql/bin/mysqldump -S /tmp/mysql.sock mysql > 123.sql（备份）
/usr/local/mysql/bin/mysql -S /tmp/mysql.sock db1 < 123.sql（恢复）
use db1;
show tables;
vim /etc/my.cnf
#skip-networking
server-id       = 1
# Uncomment the following if you want to log updates
log-bin=mysql-bin
binlog-do-db=db1，db2（只针对此数据库做主从）
binlog-ignore-db=mysql（不做主从的库）

    mysql> grant replication slave on *.* to 'repl'@'127.0.0.1' identified by '123456';
    mysql> flush privileges;（刷新权限）
    mysql> flush tables with read lock;（将表的读锁死）
    mysql> show master status;
    +------------------+----------+--------------+------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
    +------------------+----------+--------------+------------------+
    | mysql-bin.000001 |      331 |              |                  |
    +------------------+----------+--------------+------------------+
```
#### 4、配置
```bash
# 在从上
vim /usr/local/mysql_slave/my.cnf
#skip-networking
server-id       = 111
replicate-do-db=db1,db2
replicate-ignore-db=mysql

/usr/local/mysql_slave/bin/mysql -S /tmp/mysql_slave.sock -e "create database db1"
/usr/local/mysql_slave/bin/mysql -S /tmp/mysql_slave.sock db1<123.sql
    
/usr/local/mysql_slave/bin/mysql -S /tmp/mysql_slave.sock
mysql> stop slave;
mysql> change master to master_host='127.0.0.1',master_port=3306,master_user='repl',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=331;
mysql> start slave;
mysql> show slave status\G;
Slave_IO_Running: Yes
Slave_SQL_Running: Yes（两个都为 yes 配置成功）

#在主上
mysql> unlock tables;（解锁）
```    

## 配置主从 2

#### 配置主
```bash
    # 安装 mysql，修改 my.cnf 增加 server_id=10, log_bin=zilo1
    $ /etc/init.d/mysqld restart
    # 备份 mysql 库并恢复成 zilo 库，作为测试
    $ mysqldump -uroot mysql > /tmp/mysql.sql
    $ msyql -uroot -e "create database zilo"
    $ msyql -uroot zilo < /tmp/mysql.sql
    # 创建同步数据的用户
    $ grant replication slave on *.* to 'repl'@slave_ip identified by 'passwd';
    $ flush privileges;
    $ flush tables with read lock;
    $ show master status;
    +----------------+----------+--------------+------------------+-------------------+
    | File           | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +----------------+----------+--------------+------------------+-------------------+
    | zilo-01.000001 |   659371 |              |                  |                   |
    +----------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)
    $ mysql> show processlist; //state 状态应该为 Has sent all binlog to slave; waiting for binlog to be updated
```

#### 配置从
```bash
    # 修改 my.cnf server_id=11（和主不能一样）
    $ /etc/init.d/mysqld restart
    # 首先将主上的要测试的数据库拷贝到从上，并恢复到 mysql 数据库中
    $ scp 192.168.95.10:/tmp/mysql_backups/*.sql /tmp/
    $ mysql -uroot -p -e "create database xxx"
    $ mysql -uroot -p xxx < /tmp/xxx.sql
    
    $ mysql -uroot -p
    mysql> stop slave;
    # mysql> reset slave;
    mysql> change master to master_host=master_ip,master_user='repl',master_password='',master_log_file='zilo-01.000001',master_log_pos=659660;
    # master_log_file 和 master_log_pos 为主上执行 show master status 所得
    mysql> start slave;
    mysql> show slave status\G;
    mysql> show processlist; //应该有两行state值为：Waiting for master to send event.Has read all relay log; waiting for the slave I/O thread to update it

    # 到主上执行 unlock tables

    # 重要：下面两项都为 yes 才成功，如果第一项为 connecting 检查防火墙
    Slave_IO_Running: Connecting //连接到主库，并读取主库的日志到本地，生成本地日志文件
    Slave_SQL_Running: Yes //读取本地日志文件，并执行日志里的SQL命令
    Seconds_Behind_Master: NULL //主从延迟的时间
    Last_IO_Errno: 1130
    Last_IO_Error: error connecting to master 'repl@192.168.95.11:3306' - retry-time: 60  retries: 1
    Last_SQL_Errno: 0
    Last_SQL_Error: 
```

#### 额外配置选项
```bash
    # 主 
    binlog_do_db= //仅同步指定的库
    binlog_ignore_db= //忽略指定同步的库
    # 从
    replicate_do_db=
    replicate_ignore_db=
    replicate_do_table= //避免使用
    replicate_ignore_table= //避免使用
    replicate_wild_do_table= //支持通配符% 如，zilo.%
    replicate_wild_ignore_talbe=
```

## 测试主从
```bash
# 主
mysql> select count(*) from db;
mysql> truncate table db;
mysql> drop table db;
# 从
mysql> select count(*) from db;
```

