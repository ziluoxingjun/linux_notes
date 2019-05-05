
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
keys *    //取出所有key
keys my* //模糊匹配
exists name  //有name键 返回1 ，否则返回0；
del  key1 // 删除一个key    //成功返回1 ，否则返回0；
EXPIRE key1 100  //设置key1 100s后过期
ttl key // 查看键 还有多长时间过期，单位是s,当 key 不存在时，返回 -2 。 当 key 存在但没有设置剩余生存时间时，返回 -1 。 否则，返回 key 的剩余生存时间。
select  0  //代表选择当前数据库，默认进入0 数据库
move age 1  // 把age 移动到1 数据库
persist key1   //取消key1的过期时间
randomkey //随机返回一个key
rename oldname newname //重命名key
type key1 //返回键的类型
dbsize  //返回当前数据库中key的数目
info  //返回redis数据库状态信息
flushdb //清空当前数据库中所有的键
flushall    //清空所有数据库中的所有的key
bgsave //保存数据到 rdb文件中，在后台运行
save //作用同上，但是在前台运行
config get * //获取所有配置参数
config get dir  //获取配置参数
config set dir  //更改配置参数
数据恢复： 首先定义或者确定dir目录和dbfilename，然后把备份的rdb文件放到dir目录下面，重启redis服务即可恢复数据
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
# 调整内核参数： echo "net.core.somaxconn = 2048" >> /etc/sysctl.conf; sysctl -p

timeout 0
# 当客户端处于空闲状态时，redis服务端会主动关闭连接，这个参数用来定义客户端空闲多少秒，服务端关闭连接，如果是0则不限制。

tcp-keepalive 300
# TCP连接保活策略，可以通过tcp-keepalive配置项来进行设置，单位为秒，假如设置为60秒，则server端会每60秒向连接空闲的客户端发起一次ACK请求， 以检查客户端是否已经挂掉，对于无响应的客户端则会关闭其连接。所以关闭一个连接最长需要120秒的时间。如果设置为0，则不会进行保活检测。
```