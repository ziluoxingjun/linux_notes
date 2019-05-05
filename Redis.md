
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
$ 127.0.0.1:6379> LRAGE list1 0 -1
$ 127.0.0.1:6379> LPOP list1
$ 127.0.0.1:6379> RPOP list1
```
添加左边元素：LPUSH               语法：LPUSH key value [value ...]  ，返回添加后的列表元素的总个数

添加右边元素：RPUSH              语法：RPUSH key value [value ...]  ，返回添加后的列表元素的总个数

移除左边第一个元素：LPOP        语法：LPOP key  ，返回被移除的元素值

移除右边第一个元素：RPOP        语法：RPOP key ，返回被移除的元素值 

列表元素个数：LLEN                语法：LLEN key， 不存在时返回0，redis是直接读取现成的值，并不是统计个数

获取列表片段：LRANGE           语法：LRANGE key start stop，如果start比stop靠后时返回空列表，0 -1 返回整个列表

                                                    正数时：start 开始索引值，stop结束索引值（索引从0开始）

                                                    负数时：例如 lrange num -2 -1，-2表示最右边第二个，-1表示最右边第一个，

删除指定值：LREM                  语法：LREM key count value，返回被删除的个数

                                                   count>0，从左边开始删除前count个值为value的元素

                                                   count<0，从右边开始删除前|count|个值为value的元素

                                                   count=0，删除所有值为value的元素

索引元素值：LINDEX               语法：LINDEX key index ，返回索引的元素值，-1表示从最右边的第一位

设置元素值：LSET                  语法：LSET key index value

保留列表片段：LTRIM              语法：LTRIM key start stop，start、top 参考lrange命令

一个列表转移另一个列表：RPOPLPUSH      语法：RPOPLPUSH source desctination ，从source列表转移到desctination列表，

                                                                 该命令分两步看，首先source列表RPOP右移除，再desctination列表LPUSH