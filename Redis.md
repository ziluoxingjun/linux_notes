
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