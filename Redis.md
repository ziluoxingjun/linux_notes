
## Redis 介绍
Redis是一个键值对存储数据库，属于一种NoSQL，其数据存储在内存里，读写速度非常快，据说是可以达到10w并发。支持数据持久化。它属于单线程服务，但这不影响它的高并发特性。

类似键值对数据库还有Memcached，但Redis比Memcached支持更多类型的数据。Mecached只支持string类型的数据，但Redis除了支持string外，还支持hash，set，list，zset(有序集合)

## Redis 安装启动服务
```bash
$ wget http://download.redis.io/releases/redis-5.0.4.tar.gz
$ tar zxf redis-5.0.4.tar.gz
$ cd redis-5.0.4
$ make && make install

$ cp redis.conf /etc/
$ vim /etc/redis.conf
136 daemonize yes //后台启动
171 logfile "/var/log/redis.log"

$ redis-server /etc/redis.conf
$ pkill redis-server
```

## systemd 管理 Redis 服务
CentOS7下编写服务管理脚本
```bash
$ vim /usr/lib/systemd/system/redis.service ##内容如下
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/bin/redis-server /etc/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
##到此结束

$ ln -s /usr/lib/systemd/system/redis.service /etc/systemd/system/multi-user.target.wants/redis.service
$ systemctl daemon-reload
$ systemctl start redis

$ redis-cli  
$ redis-cli -h ip -p port
$ redis-cli -a 'password'
```

## Redis 数据类型
#### string
```bash
$ 127.0.0.1:6379> set key1 "violet"
$ 127.0.0.1:6379> get key1
$ 127.0.0.1:6379> mset key1 1  key2 'a'  key3 'lambo'
$ 127.0.0.1:6379> mget key1 key2
$ 127.0.0.1:6379> keys *
```

#### list
list 是一个列表结构，主要功能是 push、pop、获取一个范围的所有值等等。操作中 key 理解为链表的名字。list 用于消息队列。
内部使用双向链表实现，所以获取越接近两端的元素速度越快，但通过索引访问时会比较慢
```bash
$ 127.0.0.1:6379> LPUSH list1 "violet"
$ 127.0.0.1:6379> LPUSH list1  "1 2 3"
$ 127.0.0.1:6379> LPUSH list1 "aaa" "bbb" "ccc"
$ 127.0.0.1:6379> RPUSH list1 "ddd"
$ 127.0.0.1:6379> LRAGE list1 0 -1
$ 127.0.0.1:6379> LPOP list1
$ 127.0.0.1:6379> RPOP list1
$ 127.0.0.1:6379> LLEN list1
$ 127.0.0.1:6379> LREM list1 1 'nogood'
$ 127.0.0.1:6379> LINDEX list1 0
$ 127.0.0.1:6379> LSET list1 0 'nogood'
```

#### set
set 是无序集合，和我们数学中的集合概念相似，集合类型值具有唯一性，常用操作是向集合添加、删除、判断某个值是否存在，有对多个集合求交并差等操作。操作中key理解为集合的名字,集合内部是使用值为空的散列表实现的。
```bash
$ 127.0.0.1:6379> SADD set1 a b c
$ 127.0.0.1:6379> SADD set1 d
$ 127.0.0.1:6379> SMEMBERS set1  //读取所有元素
$ 127.0.0.1:6379> SISMEMBER s2 b
$ 127.0.0.1:6379> SREM set1 c  //删除元素
$ 127.0.0.1:6379> SADD set2 1 a b 
$ 127.0.0.1:6379> SINTER set1 set2 //交集
$ 127.0.0.1:6379> SUNION set1 set2 //并集
$ 127.0.0.1:6379> SDIFF set1 set2 //差集
$ 127.0.0.1:6379> SSCARD s2
$ 127.0.0.1:6379> SPOP s1
```

#### zset (sorted set)
sorted set是有序集合，它比set多了一个权重参数score，使得集合中的元素能够按 score 进行有序排列。
```bash
$ 127.0.0.1:6379> ZADD set3 12 abc
$ 127.0.0.1:6379> ZADD set3 2 "cde 123"
$ 127.0.0.1:6379> ZADD set3 24 "123-aaa"
$ 127.0.0.1:6379> ZADD set3 4 "a123a"
$ 127.0.0.1:6379> ZRANGE set3 0 -1
$ 127.0.0.1:6379> ZREVRANGE set3 0 -1  //倒序
$ 127.0.0.1:6379> ZSCORE z1 abc
```

#### hash
散列类型,可以认为是多维度string
```bash
$ 127.0.0.1:6379> hset hash1 name zizi
$ 127.0.0.1:6379> hget hash1 name
$ 127.0.0.1:6379> hset hash1  age 30
$ 127.0.0.1:6379> hget hash1 age
$ 127.0.0.1:6379> hgetall hash1
$ 127.0.0.1:6379> hmset
$ 127.0.0.1:6379> hmget 
$ 127.0.0.1:6379> HEXISTS 
$ 127.0.0.1:6379> HDEL
$ 127.0.0.1:6379> HKEYS
$ 127.0.0.1:6379> HVALS
$ 127.0.0.1:6379> HLEN
```

## Redis常见操作
```bash
$ keys *    //取出所有key
$ keys my* //模糊匹配
$ exists name  //有name键 返回1 ，否则返回0；
$ del  key1 // 删除一个key    //成功返回1 ，否则返回0；
$ EXPIRE key1 100  //设置key1 100s后过期
$ ttl key // 查看键 还有多长时间过期，单位是s,当 key 不存在时，返回 -2 。 当 key 存在但没有设置剩余生存时间时，返回 -1 。 否则，返回 key 的剩余生存时间。
$ select  0  //代表选择当前数据库，默认进入0 数据库
$ move age 1  // 把age 移动到1 数据库
$ persist key1   //取消key1的过期时间
$ randomkey //随机返回一个key
$ rename oldname newname //重命名key
$ type key1 //返回键的类型
$ dbsize  //返回当前数据库中key的数目
$ info  //返回redis数据库状态信息
$ flushdb //清空当前数据库中所有的键
$ flushall    //清空所有数据库中的所有的key
$ bgsave //保存数据到 rdb文件中，在后台运行
$ save //作用同上，但是在前台运行
$ config get * //获取所有配置参数
$ config get dir  //获取配置参数
$ config set dir  //更改配置参数
# 数据恢复： 首先定义或者确定dir目录和dbfilename，然后把备份的rdb文件放到dir目录下面，重启redis服务即可恢复数据
```

## 配置文件讲解
#### 网络相关
```bash
bind 127.0.0.1
# 指定绑定IP，如果想绑定多个IP，可以一行写多个IP，空格分开 bind ip1 ip2

protected-mode yes
# 设置为yes，则开启了安全模式，当redis.conf中没有定义bind的ip时，也就是说redis将会绑定全网IP， 并且也没有设置访问密码，这两个条件满足时，当远程的机器访问redis时，就会被限制了。建议开启。

port 6379
# 定义监听的端口

tcp-backlog 511
# 关于backlog的理解，需要先搞清楚TCP三次握手。这个tcp-backlog定义了一个队列的长度。这个队列指的是， TCP三次握手中最后一次握手完成后的那个状态的连接（下图的accept queue）。
# 该参数设定的值不能大于内核的somaxconn的值，要想设置的非常高，那么首先要将内核参数somaxconn的值提升。 somaxconn，定义了系统中每一个端口最大的监听队列的长度，这是个全局的参数,默认值为128. 限制了每个端口接收新tcp连接侦听队列的大小。对于一个经常处理新连接的高负载 web服务环境来说，默认的128太小了。大多数环境这个值建议增加到2048或者更多。
# 调整内核参数： echo "net.core.somaxconn = 2048" >> /etc/sysctl.conf; sysctl -p;sysctl -a |grep somaxconn

timeout 0
# 当客户端处于空闲状态时，redis服务端会主动关闭连接，这个参数用来定义客户端空闲多少秒，服务端关闭连接，如果是0则不限制。

tcp-keepalive 300
# TCP连接保活策略，可以通过tcp-keepalive配置项来进行设置，单位为秒，假如设置为60秒，则server端会每60秒向连接空闲的客户端发起一次ACK请求， 以检查客户端是否已经挂掉，对于无响应的客户端则会关闭其连接。所以关闭一个连接最长需要120秒的时间。如果设置为0，则不会进行保活检测。
```
![TCP 三次握手](https://gitee.com/uploads/images/2019/0506/151002_040f74a0_922657.png "tcp 三次握手.png")

#### 通用配置
```bash
daemonize yes
# 启动的时候后台启动

supervised no
# 是否通过upstart或systemd管理守护进程。默认no没有服务监控，其它选项有upstart, systemd, auto

pidfile /var/run/redis_6379.pid
# pid文件路径

loglevel notice
# 日志级别

logfile ""
# 日志路径和名字，默认为空，当不以后台启动时，日志直接输出到屏幕，当后台启动时，日志输出到/dev/null

databases 16
# 定义redis的数据库个数，默认为16个（0-15），可以使用select id，来选择对应的数据库。

always-show-logo yes
# redis启动时，会打印ASCII艺术logo
```

#### 快照相关配置
```bash
save
# 定义数据持久化的策略，这个持久化指的是RDB格式的数据。save 900 1指的是900秒内至少有一个key发生改变，满足这个条件就会触发持久化。 要想关闭RDB格式的持久化，需要改为这样save ""

stop-writes-on-bgsave-error yes
# 当save过程中出现失败的情况时，或者有某些错误时，总之导致了内存中的数据和磁盘中的数据不一致了。该参数定义此时是否继续进行save的操作。

rdbcompression yes
# 是否在save时将string对象进行压缩，压缩算法为LFZ。开启该功能会额外消耗CPU资源。

rdbchecksum yes
# 当save完成后，是否使用CRC64算法校验rdb文件。

dbfilename dump.rdb
# save的rdb文件名字，路径在dir定义的目录下

dir ./
# 定义rdb文件路径
```

#### 副本相关配置
```bash
replicaof
  +------------------+      +---------------+
  |      Master      | ---> |    Replica    |
  | (receive writes) |      |  (exact copy) |
  +------------------+      +---------------+
# 定义主的ip和端口，5.0版本以前使用slaveof，该参数是在从上定义的。

masterauth
# 定义主的密码，该参数是在从上定义的。该密码其实就是在主上由requirepass参数定义的密码。

replica-serve-stale-data yes
# 当从Redis失去了与主Redis的连接，或者主从同步正在进行中时，Redis该如何处理外部发来的访问请求呢？这里，从Redis可以有两种选择：
# yes,即使主从断了，从依然响应客户端的请求。
# no,主从断开了，则从会提示客户端"SYNC with master in progress"，但有些指令还可以使用 INFO, replicaOF, AUTH, PING, SHUTDOWN, REPLCONF, ROLE, CONFIG,SUBSCRIBE, UNSUBSCRIBE, PSUBSCRIBE, PUNSUBSCRIBE, PUBLISH, PUBSUB, COMMAND, POST, HOST: and LATENCY

replica-read-only yes
# 定义从是否只读

repl-diskless-sync no
# 定义主从同步数据的策略，它有两种策略，一个是磁盘形式，一个是socket（就是这个diskless）形式。 磁盘形式，就是先将数据写到rdb文件里，然后传输rdb文件到从上。 socket形式，就是直接通过网络传输变更的数据到从上的rdb文件里。 repl-diskless-sync no，表示使用磁盘的形式。

repl-diskless-sync-delay 5
# 如果使用了通过socket的形式传输数据，则需要考虑一个主下面有多个从的情况，因为一旦基于Diskless的复制传送开始， 主就无法顾及新的从的到来。所以，就有了这个延迟的设置，比如延迟5秒，这样在主从传输数据之前，所有的从都被识别了， 这样主就可以多开几个线程来同时给所有的从进行数据传输了。

repl-disable-tcp-nodelay no
# 是否关闭tcp_nodelay功能。 关于tcp_nodelay有个nagle算法，假如需要频繁的发送一些小包数据，比如说1个字节，以IPv4为例的话， 则每个包都要附带40字节的头，也就是说，总计41个字节的数据里，其中只有1个字节是我们需要的数据。为了解决这个问题，出现了Nagle算法。 它规定：如果包的大小满足MSS，那么可以立即发送，否则数据会被放到缓冲区，等到已经发送的包被确认了之后才能继续发送。 通过这样的规定，可以降低网络里小包的数量，从而提升网络性能。
# 该参数设置为no，即使用tcp_nodelay，数据传输到slave的延迟将会减少但要使用更多的带宽。 反之，不使用tcp_nodelay，这样Redis主将使用更少的TCP包和带宽来向slaves发送数据。 但是这将使数据传输到slave上有延迟，Linux内核的默认配置会达到40毫秒。

repl-backlog-size 1mb
# 首先解释一下，这里的backlog是主上的一个内存缓冲区，它存储的数据是当主和从断开连接时，主无法将数据传给从了，这时候主先将更新的数据 暂时存放在缓存去里。如果主从再次连接时，就不需要重新传输所有数据，而是只需要传输缓冲区的这一部分即可。
# 这个参数用来定义该缓冲区的大小。

repl-backlog-ttl 3600
# 如果主Redis等了一段时间之后，还是无法连接到从Redis，那么缓冲队列中的数据将被清理掉。我们可以设置主Redis要等待的时间长度。 如果设置为0，则表示永远不清理。默认是1个小时。

replica-priority 100
# 我们可以给众多的从Redis设置优先级，在主Redis持续工作不正常的情况，优先级高的从Redis将会升级为主Redis。而编号越小，优先级越高。 比如一个主Redis有三个从Redis，优先级编号分别为10、100、25，那么编号为10的从Redis将会被首先选中升级为主Redis。 当优先级被设置为0时，这个从Redis将永远也不会被选中。默认的优先级为100。

min-replicas-to-write 3 /min-replicas-max-lag 10
# 假如主Redis发现有超过M个从Redis的连接延时大于N秒，那么主Redis就停止接受外来的写请求。这是因为从Redis一般会每秒钟都向主Redis发出PING， 而主Redis会记录每一个从Redis最近一次发来PING的时间点，所以主Redis能够了解每一个从Redis的运行情况。
# min-replicas-to-write 3 /min-replicas-max-lag 10表示，假如有大于等于3个从Redis的连接延迟大于10秒，那么主Redis就不再接受外部的写请求。 上述两个配置中有一个被置为0，则这个特性将被关闭。默认情况下min-slaves-to-write为0，而min-slaves-max-lag为10。
```

#### 安全配置
```bash
requirepass foobared
# 设置登录redis的密码

rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
# 将一些关键指令修改名字，比如将CONFIG命令改为b840fc02d524045429941cc15f59e41cb7be6c52，也可以直接禁用CONFIG指令 rename-command CONFIG ""
```

#### 客户端
```bash
maxclients 10000
# 允许最大客户端数
```

#### AOF相关
```bash
appendonly no
# 默认情况下，Redis会异步的将数据持久化到磁盘（RDB）。这种模式在大部分应用程序中已被验证是很有效的，但是在一些问题发生时， 比如断电，则这种机制可能会导致数分钟的写请求丢失。如上半部分中介绍的，AOF是一种更好的保持数据一致性的方式。 即使当服务器断电时，也仅会有1秒钟的写请求丢失，当Redis进程出现问题且操作系统运行正常时，甚至只会丢失一条写请求。
# 官方建议，AOF机制和RDB机制可以同时使用，不会有任何冲突。

appendfilename "appendonly.aof"
# 定义aof文件名字

appendfsync everysec
# 使用AOF机制做持久化时，调用fsync()函数有三个模式：
#（1）no：不调用fsync()。而是让操作系统自行决定sync的时间。这种模式下，Redis的性能会最快。
#（2）always：在每次写请求后都调用fsync()。这种模式下，Redis会相对较慢，但数据最安全。
#（3）everysec：每秒钟调用一次fsync()。这是性能和安全的折衷。
#默认情况下为everysec。

no-appendfsync-on-rewrite no
# 当fsync方式设置为always或everysec时，如果后台持久化进程需要执行一个很大的磁盘IO操作，那么Redis可能会在fsync()调用时卡住。 目前尚未修复这个问题，这是因为即使我们在另一个新的线程中去执行fsync()，也会阻塞住同步写调用。
# 为了缓解这个问题，我们可以使用该配置项，这样的话，当BGSAVE或BGWRITEAOF运行时，fsync()在主进程中的调用会被阻止。 这意味着当另一路进程正在对AOF文件进行重构时，Redis的持久化功能就失效了，就好像我们设置了"appendsync no"一样。 如果Redis有时延问题，那么可以将该选项设置为yes。否则请保持no，因为这是保证数据完整性的最安全的选择。

auto-aof-rewrite-percentage 100/auto-aof-rewrite-min-size 64mb
# 我们允许Redis自动重写aof。当aof增长到一定规模时，Redis会隐式调用BGREWRITEAOF来重写log文件，以缩减文件体积。
# Redis是这样工作的：Redis会记录上次重写时的aof大小。假如Redis自启动至今还没有进行过重写，那么启动时aof文件的大小会被作为基准值。 这个基准值会和当前的aof大小进行比较。如果当前aof大小超出所设置的增长比例，则会触发重写。另外还需要设置一个最小大小，是为了防止在aof很小时就触发重写。
# 如果设置auto-aof-rewrite-percentage为0，则会关闭此重写功能。

aof-load-truncated yes
# 由于某种原因（aof文件损坏）有可能导致利用aof文件恢复redis数据时发生异常，该参数决定redis服务接下来的行为。
# 如果设置为yes，则aof文件会被加载（但数据一定不全），并且会记录日志说明情况。
# 如果设置为no，则redis服务根本就启动不起来。

aof-use-rdb-preamble yes
# 为了让用户能够同时拥有RDB和AOF两种持久化的优点， 从Redis 4.0版本开始，就推出了一个能够“鱼和熊掌兼得”的持久化方案 —— RDB-AOF 混合持久化： 这种持久化能够通过 AOF 重写操作创建出一个同时包含RDB数据和AOF数据的AOF文件， 其中RDB数据位于AOF文件的开头， 它们储存了服务器开始执行重写操作时的数据库状态：至于那些在重写操作执行之后执行的Redis命令，则会继续以AOF格式追加到AOF文件的末尾，也即是RDB数据之后。
```

#### slow log
```bash
slowlog-log-slower-than 10000
# Redis也有跟MySQL类似的慢查询日志，该参数定义一个查询执行时间超过10000微秒则会记录日志。其中1秒=1000000微秒。

slowlog-max-len 128
# 该参数定义慢查询日志最大的条数。其实，Redis的slow log也是保存在内存中，也是一种k/v形态的数据。

$ slowlog get //列出所有的慢查询日志
$ slowlog get 2 //只列出2条
$ slowlog len //查看慢查询日志条数
```

## PHP 中使用 Redis
#### php安装redis扩展模块
1. 使用pecl安装
```bash
$ /usr/local/php-fpm/bin/pecl install redis
$ vim  /usr/local/php/etc/php.ini
925 extension = redis.so
```
2. 通过源码安装
```bash
$ wget https://github.com/phpredis/phpredis/archive/4.3.0.tar.gz
$ mv 4.3.0.tar.gz  php-redis.tar.gz
$ tar zxvf php-redis.tar.gz
$ cd phpredis-4.3.0/
$ /usr/local/php-fpm/bin/phpize
$ ./configure --with-php-config=/usr/local/php-fpm/bin/php-config
$ make && make install
$ vim  /usr/local/php/etc/php.ini
925 extension = redis.so
```

#### php 中使用 redis 存储 session
```bash
$ vim /usr/local/php-fpm/etc/php.ini//更改或增加
1417 session.save_handler = "redis" 
1446 session.save_path = "tcp://127.0.0.1:6379" 

# 或者apache虚拟主机配置文件中也可以这样配置：
php_value session.save_handler "redis" 
php_value session.save_path "tcp://127.0.0.1:6379" 
 
# 或者php-fpm配置文件对应的pool中增加：
php_value[session.save_handler] = redis
php_value[session.save_path] = "tcp://127.0.0.1:6379"

$ /usr/loca/php-fpm/bin/php -i |grep session.save
session.save_handler => redis => redis
session.save_path => tcp://127.0.0.1:6379 => tcp://127.0.0.1:6379

# 创建测试文件
$ wget http://study.lishiming.net/.mem_se.txt
$ mv .mem_se.txt session.php
$ /usr/local/php/bin/php session.php
$ redis-cli
$ 127.0.0.1:6379> keys *
```
```php
<?php 
    session_start(); 
    if (!isset($_SESSION['TEST'])) { 
        $_SESSION['TEST'] = time(); 
    } 
    $_SESSION['TEST3'] = time(); 
    print $_SESSION['TEST']; 
    print "<br><br>"; 
    print $_SESSION['TEST3']; 
    print "<br><br>"; 
    print session_id(); 
?> 
```

## Redis 主从配置
```bash
# 为了节省资源，我们可以在一台机器上启动两个redis服务
$ cp /etc/redis.conf  /etc/redis2.conf
$ vim /etc/redis2.conf
port 6380
pidfile /var/run/redis_6380.pid
logfile "/var/log/redis2.log"
dbfilename dump2.rdb
dir ./
replicaof 127.0.0.1 6379 //增加一行

# 如果主上设置了密码，还需要增加
masterauth aminglinux>com //设置主的密码

# 启动之前不要忘记创建新的dir目录
redis-server /etc/redis2.conf

# 测试：在主上创建新的key，在从上查看
# 注意：redis主从和mysql主从不一样，redis主从不用事先同步数据，它会自动同步过去
```

## Redis sentinel
Redis Sentinel 是 Redis 高可用的实现方案。Sentinel 是一个管理多个 Redis 实例的工具，它可以实现对 Redis 的监控、通知、自动故障转移。

#### Redis Sentinel 的主要功能
Sentinel 的主要功能包括主节点存活检测、主从运行情况检测、自动故障转移（failover）、主从切换。Redis 的 Sentinel 最小配置是一主一从。 Redis 的 Sentinel 系统可以用来管理多个 Redis 服务器，该系统可以执行以下四个任务：
1. 监控:
Sentinel 会不断的检查主服务器和从服务器是否正常运行。
2. 通知:
当被监控的某个Redis服务器出现问题，Sentinel通过API脚本向管理员或者其他的应用程序发送通知。
3. 自动故障转移:
当主节点不能正常工作时，Sentinel会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点，并且将其他的从节点指向新的主节点。
4. 配置提供者:
在 Redis Sentinel 模式下，客户端应用在初始化时连接的是 Sentinel 节点集合，从中获取主节点的信息。

#### Redis Sentinel 的工作流程
Sentinel负责监控集群中的所有主、从Redis，当发现主故障时，Sentinel会在所有的从中选一个成为新的主。
并且会把其余的从变为新主的从。同时那台有问题的旧主也会变为新主的从，也就是说当旧的主即使恢复时，
并不会恢复原来的主身份，而是作为新主的一个从。

在Redis高可用架构中，Sentinel往往不是只有一个，而是有3个或者以上。目的是为了让其更加可靠，毕竟主
和从切换角色这个过程还是蛮复杂的。

> http://www.cnblogs.com/jifeng/p/5138961.html

#### 相关概念
- 主观失效:
SDOWN（subjectively down），直接翻译的为”主观”失效，即当前sentinel实例认为某个redis服务为”不可用”状态.

- 客观失效:
ODOWN（objectively down），直接翻译为”客观”失效，即多个sentinel实例都认为master处于”SDOWN”状态,那么此时master将处于ODOWN,ODOWN可以简单理解为master已经被集群确定为”不可用”，将会开启failover

#### 环境准备
准备3台机器，其中每台机器上都有两个角色，分配如下：

| 主机名 | IP:PORT             | 角色           |
| ------ | ------------------- | -------------- |
| test1  | 192.168.6.165:6379  | Redis Master   |
| test1  | 192.168.6.165:6380  | Sentinel1      |
| test2  | 192.168.6.166:6379  | Redis Repli1   |
| test2  | 192.168.6.166:6380  | Sentinel2      |
| test3  | 192.168.6.167:6379  | Redis Repli2   |
| test3  | 192.168.6.167:6380  | Sentinel3      |

#### 部署
1. 安装 Redis
```bash
$ vim /etc/redis.conf
bind 192.168.6.165/166/167
daemonize yes
logfile "/var/log/redis.log"
```
2. 部署 Redis主从
```bash
$ vim /etc/redis.conf
replicaof 192.168.6.165 6379 //在两个从上配置

$ firewalld-cmd --permanent --add-port 6379-6380/tcp
$ firewalld-cmd --reload
```
3. 部署 Sentinel
```bash
# 三台Sentinel配置文件是一样的，编辑配置文件
$ vim /etc/sentinel.conf
# 端口
port 6380

# 是否后台启动
daemonize yes

# pid文件路径
pidfile /var/run/redis-sentinel.pid

# 日志文件路径
logfile "/var/log/sentinel.log"

# 定义工作目录
dir /tmp

# 定义Redis主的别名, IP, 端口，这里的2指的是需要至少2个Sentinel认为主Redis挂了才最终会采取下一步行为
sentinel monitor mymaster 192.168.6.165 6379 2

# 如果mymaster 30秒内没有响应，则认为其主观失效
sentinel down-after-milliseconds mymaster 30000

# 如果master重新选出来后，其它slave节点能同时并行从新master同步数据的台数有多少个，显然该值越大，所有slave节
##点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保守的设置
##为1，同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢。
sentinel parallel-syncs mymaster 1

# 该参数指定一个时间段，在该时间段内没有实现故障转移成功，则会再一次发起故障转移的操作，单位毫秒
sentinel failover-timeout mymaster 180000

# 不允许使用SENTINEL SET设置notification-script和client-reconfig-script。
sentinel deny-scripts-reconfig yes
```

#### 启动服务
> 启动顺序：主Redis -> 从Redis -> Sentinel1/2/3
```bash
# Sentinel 启动命令
$ redis-sentinel /etc/sentinel.conf
$ redis-server --sentinel /etc/sentinel.conf
```

#### Sentinel操作
```bash
$ redis-cli -h 192.168.6.165 -p 6380 //连接 sentinel

$ sentinel master mymaster
# 输出被监控的主节点的状态信息

$ sentinel slaves mymaster
# 查看mymaster的从信息

$ sentinel sentinels mymaster
# 查看其他Sentinel信息
```

#### 测试
- 停止Redis从
- 停止Redis主
```bash
$ redis-cli -h 192.168.6.165
$ 192.168.6.165:6379> shutdown
```
- 停止sentinel1

#### 客户端连接问题
> 使用sentinel后，客户端（如 php）如何连Redis呢？

> https://blog.51cto.com/chenql/1958910

## Redis Cluster
Redis Cluster为Redis官方提供的一种分布式集群解决方案。它支持在线节点增加和减少。 集群中的节点角色可能是主，也可能是从，但需要保证每个主节点都要有对应的从节点， 这样保证了其高可用。

Redis Cluster采用了分布式系统的分片（分区）的思路，每个主节点为一个分片，这样也就意味着 存储的数据是分散在所有分片中的。当增加节点或删除主节点时，原存储在某个主节点中的数据 会自动再次分配到其他主节点。

![redis_cluster](https://gitee.com/uploads/images/2019/0507/141811_c4e0da66_922657.png "redis_cluster.png")

如图，各节点间是相互通信的，通信端口为各节点Redis服务端口+10000，这个端口是固定的，所以注意防火墙设置， 节点之间通过二进制协议通信，这样的目的是减少带宽消耗。

在Redis Cluster中有一个概念slot，我们翻译为槽。Slot数量是固定的，为16384个。这些slot会均匀地分布到各个 节点上。另外Redis的键和值会根据hash算法存储在对应的slot中。简单讲，对于一个键值对，存的时候在哪里是通过 hash算法算出来的，那么取得时候也会算一下，知道值在哪个slot上。根据slot编号再到对应的节点上去取。

Redis Cluster无法保证数据的强一致性，这是因为当数据存储时，只要在主节点上存好了，就会告诉客户端存好了， 如果等所有从节点上都同步完再跟客户端确认，那么会有很大的延迟，这个对于客户端来讲是无法容忍的。所以， 最终Redis Cluster只好放弃了数据强一致性，而把性能放在了首位。

** 企业中用的较多的是 Codis 集群方案 **

### Redis Cluster搭建
#### 角色分配
| 主机名 | IP:PORT             | 角色           |
| ------ | ------------------- | -------------- |
| test1  | 192.168.6.165:6379  | Redis Master   |
| test2  | 192.168.6.166:6379  | Redis Master   |
| test3  | 192.168.6.167:6379  | Redis Master   |
| test1  | 192.168.6.165:6380  | Redis Repli    |
| test2  | 192.168.6.166:6380  | Redis Repli    |
| test3  | 192.168.6.167:6380  | Redis Repli    |

#### 安装配置 Redis
```bash
$ vim /etc/redis_6379.conf
bind 192.168.6.165
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis_6379.log"
dir /var/redis_6379
appendonly yes 
#开启集群
cluster-enabled yes  
#集群的配置文件，首次启动会自动创建
cluster-config-file nodes-6379.conf  
#集群节点连接超时时间，15秒
cluster-node-timeout 15000 

# 如果开启了firewalld，所有机器都需要增加如下规则：
$ 
$ firewall-cmd --permanent --add-port  16379-16380/tcp
$ firewall-cmd --reload

$ redis-server /etc/redis_6379.conf
$ redis-server /etc/redis_6380.conf
```

#### 部署 Cluster
```bash
# 构建集群的命令 --cluster-replicas 1 表示每个主对应一个从
$ redis-cli --cluster create 192.168.6.165:6379 192.168.6.165:6380 192.168.6.166:6379 192.168.6.166:6380 192.168.6.167:6379 192.168.6.167:6380 --cluster-replicas 1
```

#### 连接集群
```bash
# 可以在任何一个节点上去连接集群
$ redis-cli -c -h 192.168.6.165 -p 6380
$ 192.168.6.166:6380> set k1 'hello'
```
> 说明：在创建key的过程中，它会把不同的key分配到不同的slot中，即使我们登录到了165:6380，但在写入数据时，它会选择其他节点。

#### 管理集群
```bash
# 查看集群情况：
$ redis-cli  --cluster check 192.168.6.165:6379

# 删除集群节点
$ redis-cli --cluster del-node 192.168.6.167:6380 nodeid
# 这里必须是没有槽的节点，所以必须先移除槽，否则报错 被删除的node重启后，依然记得集群中的其它节点，这是需要执行cluster forget nodeid来忘记其它节点

# 添加集群节点
$ redis-cli --cluster add-node 192.168.6.166:6380 192.168.6.165:6379  #这样添加的节点为主
$ redis-cli --cluster add-node 192.168.6.166:6380 192.168.6.165:6379 --cluster-slave  # 这样添加的节点为从
$ redis-cli --cluster add-node 192.168.6.166:6380 192.168.6.165:6379 --cluster-slave --cluster-master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e #添加从并指定主

# 平衡各节点槽数量
$ redis-cli --cluster rebalance --cluster-threshold 1 192.168.6.166:6379

# 在线迁移槽
$ redis-cli --cluster reshard 192.168.121.200:6001
# 选择一个目标节点的id 源选择all

# 给添加的节点分配slot
$ redis-cli --cluster reshard 192.168.6.166:6379
How many slots do you want to move (from 1 to 16384)? #定义要分配多少slot
What is the receiving node ID? #定义接收slot的nodeid，即新的master id
Source node #1: #定义第一个源master的id，如果想在所有master上拿slot，直接敲all
Source node #2: #定义第二个源master的id，如果不再继续有新的源，直接敲done

# 将集群外部redis实例中的数据导入到集群中去
$ redis-cli --cluster import 192.168.6.167:6379 --cluster-from 192.168.6.200:6379 --cluster-copy
# Cluster-from后面跟外部redis的ip和port 如果只使用cluster-copy，则要导入集群中的key不能在，否则如下： 如果集群中已有同样的key，如果需要替换，可以cluster-copy和cluster-replace联用，这样集群中的key就会被替换为外部的

$ redis-cli --cluster fix 192.168.6.165:6379
```