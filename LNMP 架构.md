
# LNMP 架构

![LNMP 架构图](https://images.gitee.com/uploads/images/2019/0417/173705_cb79e054_922657.jpeg)

### 架构原理
- 提供 web 服务的是 Nginx,php 是作为独立服务存在的，名字叫做 php-fpm，Nginx 直接处理静态请求，动态请求会转发给 php-fpm。Nginx 静态处理能力比 Apache 强得多。

## 1、mysql 安装

##### 1、下载、解压
```bash
$ wget https://cdn.mysql.com//Downloads/MySQL-5.6/mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
$ tar zxvf mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
```

##### 2、创建 mysql 账号
```bash
$ useradd -s /sbin/nologin mysql
```

##### 3、将解压完的目录移动到 /usr/local/mysql
```bash
$ mv mysql-5.6.41-linux-glibc2.12-x86_64 /usr/local/mysql
```

##### 4、初始化数据库
```bash
$ cd /usr/local/mysql/
$ mkdir -p /data/mysql
$ chown -R mysql /data/mysql
$ ./scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql/ --datadir=/data/mysql
```

##### 5、拷贝配置文件
```bash
$ cd support-files/
$ cp my-default.cnf /etc/my.cnf
$ vim /etc/my.cnf（定义 basedir datadir socket）
```

##### 6、拷贝启动脚本
```bash
$ cp mysql.server /etc/init.d/mysqld
```

##### 7、修改启动脚本
```bash
$ vim /etc/init.d/mysql //定义 basedir datadir conf 以及启动参数
 basedir=/usr/local/mysql
 datadir=/data/mysql
# mariadb
 conf=$basedir/my.cnf
 $bindir/mysqld_safe --defaults-file="$conf" --datadir="$datadir" --pid-file="$mysqld_pid_file_path" "$@" &
$ chmod 755 /etc/init.d/mysql
```

##### 8、加入系统服务项，并设为开机启动，启动
```bash
$ chkconfig --add mysql
$ chkconfig mysql on
$ service mysql start 或 /etc/init.d/mysql start

# 检查mysql是否启动
$ ps aux|grep mysqld
$ netstat -lntp

# 手动命令行启动
$ /usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf --user=mysql --datadir=/data/mysql &
$ killall mysqld（比直接 kill pid 安全些，会停止当前的读写再杀死进程）
```

##### 报错
>```bash
> FATAL ERROR: please install the following Perl modules before executing ./scripts/mysql_install_db:
> Data::Dumper
>
> $ yum list | grep perl | grep -i dumper
> $ yum install perl-Data-Dumper
> 
> ./bin/mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
> $ yum install libaio-devel
>```


## 2、php 安装
> 和 LAMP 中安装 PHP 有差别，需要开启 php-fpm 服务

#### 1、下载、解压、创建用户
```bash
$ wget http://cn2.php.net/distributions/php-5.6.37.tar.bz2
$ tar jxvf php-5.6.37.tar.bz2 
$ cd php-5.6.37
$ useradd -s /sbin/nologin php-fpm
```

#### 2、编译、安装
```bash
$ make clean //可将之前编译过的文件全部删除
$ ./configure \
--prefix=/usr/local/php-fpm \
--with-config-file-path=/usr/local/php-fpm/etc \
--enable-fpm \
--with-fpm-user=php-fpm \
--with-fpm-group=php-fpm \
--with-mysql=/usr/local/mariadb \
--with-mysqli=/usr/local/mariadb/bin/mysql_config \
--with-pdo-mysql=/usr/local/mariadb \
--with-mysql-sock=/tmp/mysql.sock \
--with-libxml-dir \
--with-gd \
--with-jpeg-dir \
--with-png-dir \
--with-freetype-dir \
--with-iconv-dir \
--with-zlib-dir \
--with-mcrypt \
--enable-soap \
--enable-gd-native-ttf \
--enable-ftp \
--enable-mbstring \
--enable-exif \
--with-pear \
--with-curl \
--with-openssl \
# --disable-ipv6
$ make && make install
```

#### 报错
> ```bash
> configure: error: wrong mysql library version or lib not found. Check config.log for more information.
>> 将 --with-mysqli 后面的路径删掉重新编译，因为php本身有这个模块，不用再添加 mysql 的配置文件路径。
>
> configure: error: PDO_MYSQL configure failed, MySQL 4.1 needed. Please check config.log for more information.
>> 将 --with-pdo-mysql 后面的路径删掉重新 configure
>
> configure: error: Please reinstall the libcurl distribution -
>    easy.h should be in <curl-dir>/include/curl/
> $ yum install libcurl-devel
> ```

#### 3、拷贝配置和启动脚本
```bash
$ cp php.ini-production /usr/local/php-fpm/etc/php.ini //php 全局配置
$ vim php.ini
466 display_errors = Off
575 error_log = /var/log/php_errors.log
449 error_reporting = E_ALL & ~E_NOTICE

$ cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
$ chmod 755 /etc/init.d/php-fpm 
```

#### 4、加入系统服务项并设为开机启动
```bash
$ chkconfig --add php-fpm
$ chkconfig php-fpm on
```

#### 5、修改配置
```bash
$ cd /usr/local/php-fpm/etc
$ mv php-fpm.conf.default php-fpm.conf //php-fpm 服务相关配置
 [global]
pid = /usr/local/php-fpm/var/run/php-fpm.pid
error_log = /usr/local/php-fpm/var/log/php-fpm.log
[www] //模块名称
listen = /tmp/php-fcgi.sock //监听地址
#listen = 127.0.0.1:9000 //也可以
listen.mode = 0666 //监听上面 sock 本条语句才生效，如果不配置此项，则 sock 文件权限为 440,访问时会 502 Bad Gateway
user = php-fpm
group = php-fpm
pm = dynamic
pm.max_children = 50
pm.start_servers = 20
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500
rlimit_files = 1024

$ useradd -s /sbin/nologin php-fpm
$ /usr/local/php-fpm/sbin/php-fpm -t
```

## 3、nginx 安装
- Nignx 应用场景：web 服务器、反向代理、负载均衡
- nginx使用基于事件的模型和依赖于操作系统的机制来高效地在工作进程间分配请求。工作进程的数量在配置文件中定义，并且可以针对给定配置进行修复或自动调整为可用CPU核心的数量。

```bash
$ cd /usr/local/src
$ wget http://nginx.org/download/nginx-1.14.0.tar.gz
$ tar zxvf nginx-1.14.0.tar.gz
$ cd nginx-1.14.0
$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module(# --with-pcre //做正则的)
# $ yum install -y pcre-devel
$ make && make install
$ /usr/local/nginx/sbin/nginx -t
# 如果有 apache 会占用 80 端口，先停掉才能启动
# 在配置 php nginx 之前，两者无法联系在一起，需要手动配置，才能正常成功解析 php 网站
```

## 4、nginx 启动脚本
- http://wiki.nginx.org/RedHatNginxInitScript
- https://www.nginx.com/resources/wiki/start/topics/examples/initscripts/

```bash
$ vim /etc/init.d/nginx
#!/bin/bash
# chkconfig: - 30 21
# description: startup script for nginx http service.
# Source Function Library
. /etc/init.d/functions
# Nginx Settings

NGINX_SBIN="/usr/local/nginx/sbin/nginx"
NGINX_CONF="/usr/local/nginx/conf/nginx.conf"
NGINX_PID="/usr/local/nginx/logs/nginx.pid"
RETVAL=0
prog="Nginx"
start() {
        echo -n $"Starting $prog: "
        mkdir -p /dev/shm/nginx_temp
        daemon $NGINX_SBIN -c $NGINX_CONF
        RETVAL=$?
        echo
        return $RETVAL
}
stop() {
        echo -n $"Stopping $prog: "
        killproc -p $NGINX_PID $NGINX_SBIN -TERM
        rm -rf /dev/shm/nginx_temp
        RETVAL=$?
        echo
        return $RETVAL
}
reload(){
        echo -n $"Reloading $prog: "
        killproc -p $NGINX_PID $NGINX_SBIN -HUP
        RETVAL=$?
        echo
        return $RETVAL
}
restart(){
        stop
        start
}

configtest(){
    $NGINX_SBIN -c $NGINX_CONF -t
    return 0
}
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  reload)
        reload
        ;;
  restart)
        restart
        ;;
  configtest)
        configtest
        ;;
  *)
        echo $"Usage: $0 {start|stop|reload|restart|configtest}"
        RETVAL=1
esac
exit $RETVAL

$ chmod 755 /etc/init.d/nginx
$ chkconfig --add nginx
$ chkconfig nginx on
$ firewall-cmd --add-port=80/tcp //不建议关闭 firewalld ，临时增加80端口，重启会消失
$ firewall-cmd --add-port=80/tcp --permanent //永久增加80端口
$ firewall-cmd --reload
```

## 5、nginx 配置文件
```bash
$ cd /usr/local/nginx/conf
$ mv nginx.conf{,.bak}
# 配置文件很乱，需要重写
$ vim /usr/local/nginx/conf/nginx.conf
user nobody nobody;//定义启动 nginx 服务的用户
worker_processes 2;//子进程个数
error_log /usr/local/nginx/logs/nginx_error.log crit;
pid /usr/local/nginx/logs/nginx.pid;
worker_rlimit_nofile 51200;//最多可以打开的文件数
events
{
    use epoll;
    worker_connections 6000;//进程最大连接数
}
http
{
    include mime.types;
    default_type application/octet-stream;
    server_names_hash_bucket_size 3526;
    server_names_hash_max_size 4096;
    log_format combined_realip '$remote_addr $http_x_forwarded_for [$time_local]'
    '$host "$request_uri" $status'
    '"$http_referer" "$http_user_agent"';
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 30;
    client_header_timeout 3m;
    client_body_timeout 3m;
    send_timeout 3m;
    connection_pool_size 256;
    client_header_buffer_size 1k;
    large_client_header_buffers 8 4k;
    request_pool_size 4k;
    output_buffers 4 32k;
    postpone_output 1460;
    client_max_body_size 10m;
    client_body_buffer_size 256k;
    client_body_temp_path /usr/local/nginx/client_body_temp;
    proxy_temp_path /usr/local/nginx/proxy_temp;
    fastcgi_temp_path /usr/local/nginx/fastcgi_temp;
    fastcgi_intercept_errors on;
    tcp_nodelay on;
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 8k;
    gzip_comp_level 5;
    gzip_http_version 1.1;
    gzip_types text/plain application/x-javascript text/css text/htm application/xml;
    server
    {
         listen 80 default_server; 
         server_name localhost;
         index index.html index.htm index.php;     
         root /usr/local/nginx/html;
         location ~ \.php$
         {             
           include fastcgi_params;
           fastcgi_pass unix:/tmp/php-fcgi.sock;
           #fastcgi_pass 127.0.0.1:9000;
           fastcgi_index index.php;
           fastcgi_param SCRIPT_FILENAME /usr/local/nginx/html$fastcgi_script_name;
         } 
    }
}

$ curl localhost 
$ curl 127.0.0.1
```

## 6、Nginx 默认虚拟主机
```bash
$ vim /usr/local/nginx/conf/nginx.conf
http
{
    ...
    gzip_types text/plain application/x-javascript text/css text/htm application/xml;
    include vhosts/*.conf;
}
$ mkdir /usr/local/nginx/conf/vhosts
$ vim /usr/local/nginx/conf/vhosts/blog.wd.cc.conf
server
{
    listen 80 default_server; //也可以 default ,这个就是默认虚拟主机
    server_name blog.wd.cc;
    index index.html index.htm index.php;
    root /data/wwwroot/blog.wd.cc;
}
$ mkdir /data/wwwroot/blog.wd.cc -p
$ vim /data/wwwroot/blog.wd.cc/index.php
$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
$ curl localhost
$ curl -x127.0.0.1:80 blog.wd.cc
$ curl -x127.0.0.1:80 111.com/forum.php -I
200 ok

# 如果用第 10 行 sock 监听：
$ curl -x127.0.0.1:80 111.com/forum.php -I
502 Bad Gateway
```

## 安装 WordPress
```
$ wget https://cn.wordpress.org/wordpress-5.0.3-zh_CN.tar.gz
$ tar zxvf wordpress-5.0.3-zh_CN.tar.gz
$ mv wordpress/*  /data/wwwroot/blog.wd.cc/
# 浏览器访问 blog.wd.cc 进行安装

$ yum install expect -y
$ mkpasswd -s 0 -l 12
4iWyrAe0qoim
# 创建数据库
$ mysql -uroot -p
$ mysql> create database blog;
$ mysql> grant all on blog.* to 'root'@'localhost' identified by '4iWyrAe0qoim';

# 安装 wordpress 过程中，需要设定网站程序目录的权限，属主设定为 php-fpm 服务的那个用户
$ chown -R php-fpm  /data/wwwroot/blog.wd.cc
```

## 7、nginx 用户认证
```bash
$ vim /usr/local/nginx/conf/vhosts/aaa.com.conf
 location ~ .*admin\.php$ { //~ 是匹配的意思
      auth_basic "aaa.com auth";
      auth_basic_user_file /usr/local/nginx/conf/.htpasswd;
      #include fastcgi_params;
      #fastcgi_pass unix:/tmp/www.sock;
      #fastcgi_index index.php;
      #fastcgi_param SCRIPT_FILENAME /data/www$fastcgi_script_name;
 }

$ vim /usr/local/nginx/conf/vhost/bbb.com.conf
server
{
    listen 80;
    server_name bbb.com;
    index index.html index.htm index.php;
    root /data/wwwroot/bbb.com;
    #location /admin/;
    location ~ admin.php
    {
        auth_basic 			"Auth";
        auth_basic_user_file /usr/local/nginx/conf/htpasswd;
    }
}
# 如果没有 htpasswd 工具，需要安装：
$ yum install httpd
$ htpasswd -c /usr/local/nginx/conf/htpasswd bbb
$ cat /usr/local/nginx/conf/htpasswd 
$ htpasswd /usr/local/nginx/conf/htpasswd ccc //再次创建不要 -c ，否则会删除之前的
$ /usr/local/nginx/sbin/nginx -t && /usr/local/nginx/sbin/nginx -s reload
$ service nginx configtest
$ service nginx reload

$ curl -x127.0.0.1:80 bbb.com/admin.php //显示 401 说明需要用户名和密码
$ curl -x127.0.0.1:80 -ubbb:bbb bbb.com/admin.php -I
$ curl -x 127.0.0.1:80 bbb.com
: 401 Authorization Required
$ curl -x 127.0.0.1:80 test.com/admin/ -I
 : 401 Unauthorized
 $ curl -x 127.0.0.1:80 test.com/admin/ -u test:123456 -I
 : 200 OK
 $ curl -x 127.0.0.1:80 test.com/admin.php -I
 : 401
```
> nginx location优先级:  
> location /  优先级比 location ~ 要低，也就是说，如果一个请求（如，admin.php）同时满足两个location  
> location /admin.php  
> location ~ *.php$  
> 第一条无效  
> nginx location 文档： https://github.com/ziluoxingjun/nginx/tree/master/location

## 8、nginx 域名跳转
> 目的是为了对搜索引擎友好，加重网站权重,在搜索引擎：site:www.apelearn.com site:ithome.com 检查权重

```bash
$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf
server|
{
    listen 80;
    server_name bbb.com bbb1.com bbb2.com;//支持写多个域名
    index index.html index.htm index.php;
    root /data/www/bbb.com;
 	if ($host != 'bbb.com')
     {
          #rewrite ^/(.*)$ http://bbb.com/$1 permanent;
          #rewrite /(.*)  http://www.blog.com/$1 permanent;
          rewrite http://$host/(.*)$ http://bbb.com/$1 permanent; //permanent 301 redirect 302
     }
}
$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
$ curl -x 127.0.0.1:80 bbb1.com/index.html -I
301 Moved Permanently
```
> 如果是域名跳转，用301； 如果不涉及域名跳转用302 例：rewrite /1.txt  /2.txt  redirect;

## 9、nginx 访问日志
```bash
$ vim /usr/local/nginx/conf/nginx.conf
 19     log_format log_name '$remote_addr $http_x_forwarded_for [$time_local]'
 20     ' $host "$request_uri" $status'
 21     ' "$http_referer" "$http_user_agent"';

$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf
access_log /usr/local/nginx/logs/bbb.com.log log_name;
$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
```

| 变量名                | 说明               |
| --------------------- | ------------------ |
| $remote_addr          | 客户端IP（公网IP） |
| $http_x_forwarded_for | 代理服务器IP       |
| $time_local           | 服务器本地时间     |
| $host                 | 访问主机名（域名） |
| $request_uri          | 访问的 URI 地址    |
| $status               | 状态码             |
| $http_referer         | referer            |
| $http_user_agent      | user_agent         |
| $remote_user          | 记录远程客户端用户名称  |
| $request              | 记录用户的 http 请求起始行信息  |
| $body_bytes_sent      | 记录服务器发送给客户端的响应 body 字节数  |

> Nginx 内置变量：https://github.com/ziluoxingjun/nginx/blob/master/rewrite/variable.md


## 10、nginx 日志切割
### One
```bash
$ vim /usr/local/sbin/nginx_logrotate1.sh
 #!/bin/bash
 date=`date -d "-1 day" +%F`（今天切割昨天的）
 [ -d /tmp/nginx_log ] || mkdir /tmp/nginx_log（判断目录是否存在）
 mv /tmp/nginx_access.log /tmp/nginx_log/$date.log（将日志移动）
 /etc/init.d/nginx reload > /dev/null（重新生成 nginx_access.log）
 cd /tmp/nginx_log/
 gzip -f $date.log（如果日志太大进行压缩 -f 强制覆盖）

vim /usr/local/sbin/nginx_logrotate2.sh
#！/bin/bash
# nginx 日志存放路径为 /usr/local/nginx/logs/
olddate=$(date -d "-1 day" +%Y%m%d)
log_path="/usr/local/nginx/logs"
nginx_pid="/usr/local/nginx/logs/nginx.pid"
cd $log_path
for log i$(ls *.log)
do
    mv $log $log-$olddate
done
/bin/kill -HUP $(cat $nginx_pid） //-HUP -USR1 让进程挂起，睡眠；动态更新配置，重新加载配置而不用重启服务：更改日志名称后，重新生成新的日志文件 相当于 -s reload

$ sh -x /usr/local/sbin/nginx_logrotate.sh //-x 能看到执行过程; 应加入 cron 里，每天 0 点执行切割脚本
清理：
$ find /tmp/ -name *.log-* -type f -mtime +30 |xargs rm
$ crontab -e
 0 0 * * * /bin/bash /usr/local/sbin/nginx_logrotate.sh
```

### Two
```bash
logrotate 工具 每天会定时检查配置文件并执行，不用人工干预
配置文件：/etc/logrotate.conf
子配置文件：/etc/logrotate.d/*

Nginx的日志切割配置文件：/etc/logrotate.d/nginx

$ vim /etc/logrotate.d/nginx
/usr/local/nginx/logs/*.log {
    daily
    dateext
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 640 nginx adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 $(cat /var/run/nginx.pid)
        fi
    endscript
}

$ logrotate -vf /etc/logrotate.d/nginx
# -f 强制执行 
```

## 11、nginx 日志不记录静态文件和过期时间
```bash
$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf
server
{
    listen 80;
    server_name bbb.com bcbc.com;
    index index.html index.htm index.php;
    root /data/wwwroot/bbb.com;
    access_log /usr/local/nginx/logs/bbb.com.log log_name;
    location ~* .*\.(gif|jpg|jpeg|png|ico|bmp|swf)$ // \脱意
    {
        expires 7d; //配置静态文件缓存
        access_log off;
    }
    location ~ .*\.(js|css)$
    {
        expires 12h;
        access_log off;
    }
    location ~ (static|cache)
    {
        expires 2h;
        access_log off;
    }
}
$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
```

## 12、nginx 配置防盗链
```bash
$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf 
 # location ~ .*\.(gif|jpg|jpeg|png|bmp|ico|swf|flv|rar|zip|gz|bz2|xls|docx|ppt)$
 location ~* ^.+\.(flv|rar|zip|doc|ppt|xls)$ // ~* 表示后缀名不区分大小写，正则和上面的一样
        {
                expires 7d;
                valid_referers none blocked server_names *.bbb.com;
                if ($invalid_referer)
                {
                        return 403;
                }
                access_log off;
        }

$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
$ curl -e "http://www.baidu.com/test" -I -x127.0.0.1:80 'http://www.xing.com/static/image/common/logo.png' //测试；-e referer
$ curl -x 127.0.0.1:80 -I bbb.com/1.jpg -e "http://www.baidu.com"
HTTP/1.1 403 Forbidden
$ curl -x 127.0.0.1:80 -I bbb.com/1.jpg -e "http://www.bbb.com" 
HTTP/1.1 200 OK
```
- none 空 referer
- blocked 非法域名：不以 http https 开头的

## 13、nginx 访问控制
- 禁止非法 ip 访问，限制 ip 访问，比如后台只需要管理员登录即可。
- 需求：访问 /admin/ 目录的请求，只允许某几个 IP 访问，其它 deny
```bash
$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf
 # 黑名单
 deny 127.0.0.1 //加入黑名单,如果匹配到此条规则，以下再有 127.0.0.1 的忽略，从上向下匹配
 deny 192.168.1.0/24 //禁掉此网段
 # 白名单
 allow 127.0.0.1;
 deny all;

 location ~ .*admin\.php$ {
           #auth_basic "star auth";
           auth_basic_user_file /usr/local/nginx/conf/.htpasswd;
           include fastcgi_params;
           fastcgi_pass unix:/tmp/www.sock;
           fastcgi_index index.php;
           fastcgi_param SCRIPT_FILENAME /data/www$fastcgi_script_name;
}

 # 限制某个目录
 location /admin/
 {
     allow 192.168.95.1;
     allow 127.0.0.1;
     deny all;
 }

$ /usr/local/nginx/sbin/nginx -t
$ /usr/local/nginx/sbin/nginx -s reload
$ curl -x127.0.0.1:80 bbb.com/admin.php -I （200）
$ curl -x192.168.95.145:80 bbb.com/admin.php -I（403）
$ curl -x192.168.95.145:80 bbb.com/forum.php -I（200）
$ curl -x 127.0.0.1:80 bbb.com/admin/ -I
HTTP/1.1 200 OK
$ curl -x 192.168.95.145:80 test.com/admin/ -I
HTTP/1.1 403 Forbidden

# 限制某个目录下的某类文件
location ~ .*(upload|image)/.*\.php$ //禁止解析 upload|image 目录里的 php
{
    deny all;
}

$ curl -x 127.0.0.1:80 test.com/upload/1.php
HTTP/1.1 403 Forbidden

# 限制 user_agent,nginx 禁止指定 user_agent 访问
if ($http_user_agent ~* 'curl|baidu|qq|360|sogou') //~* 不区分大小写的正则匹配
if ($http_user_agent ~ 'Spider/3.0|YoudaoBot|Tomato')
if ($http_user_agent ~* "python|curl|wget|httpclient|okhttp")
// python 就可以过滤掉 80% 的 Python 爬虫
    {
        return 403;
        return 503;
    }
$  /usr/local/nginx/sbin/nginx -t
$  /usr/local/nginx/sbin/nginx -s reload
$ curl -x127.0.0.1:80 -A "Spider/3.0" bbb.com
HTTP/1.1 403 Forbidden

# 限制uri
if ($request_uri ~ (force|sex))
{
    return 404;
}
```
> https://github.com/ziluoxingjun/nginx/blob/master/rewrite/variable.md

## 14、nginx 解析 php 的配置
```bash
$ vim /usr/local/nginx/conf/nginx.conf
location ~ \.php$
{ 
	include fastcgi_params;
	fastcgi_pass unix:/tmp/php-fcgi.sock;
        #fastcgi_pass 127.0.0.1:9000; // fastcgi_pass 用来指定 php-fpm 监听的地址或 socket
	fastcgi_index index.php;
	fastcgi_param SCRIPT_FILENAME /data/wwwroot/bbb.com$fastcgi_script_name;
        //脚本文件请求的路径,也就是说当访问 127.0.0.1/index.php 的时候，需要读取网站根目录下面的 index.php 文件，如果没有配置这一配置项时，nginx 不会去网站根目录下访问 .php 文件，所以返回空白
}

$ /usr/local/nginx/sbin/nginx -s reload
```
> 如果 fastcgi_pass unix:/tmp/php-fcgi.sock; 配置错误就会 502 Bad Gateway,因为找不到 socket 文件

> /usr/local/php-fpm/etc/php-fpm.conf listen 写的什么就要在 nginx conf 里 fastcgi_pass 写什么

> SCRIPT_FILENAME 后面的路径和上面的 root 要一致

> php 配置文件中 listen.mode 如果没写默认是 440 ，属主和属组为 root ，报 502 错误，需要 socket 文件属主和属组为 nobody

> 还有一种情况，就是 php-fpm 进程资源耗尽了，也是报502错误

## 如何正确配置 Nginx+PHP
> 用PHP实现了一个前端控制器，就是统一入口：把 PHP 请求都发送到同一个文件上，然后在此文件里通过解析「REQUEST_URI」实现路由。
```bash
server {
    listen       80;
    server_name  abc.com;

    root /data/wwwroot/sex.com;
    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ /index.php?$args; //try_files 重定向功能，假如访问abc.com/info,会匹配到此三行，匹配到此项就开始执行try_files,nginx 会去找有没有info这个文件（$uri）,如果没有就继续找info这个目录（$uri/），如果也没有的就直接重定向到 /index.php?$args,$args 就是url问号后边的参数
    }

    location ~ \.php$ {
        try_files $uri = 404; //要考虑一个安全问题：在PHP开启「cgi.fix_pathinfo」的情况下，PHP可能会把错误的文件类型当作PHP文件来解析。如果Nginx和PHP安装在同一台服务器上的话，那么最简单的解决方法是用「try_files」指令做一次过滤

        include fastcgi.conf
        fastcgi_pass unix:/tmp/php-fcgi.sock;
        #fastcgi_pass 127.0.0.1:9000;
    }
}
```
> Nginx有两份fastcgi配置文件，分别是「fastcgi_params」和「fastcgi.conf」，它们没有太大的差异，唯一的区别是后者比前者多了一行「SCRIPT_FILENAME」的定义：
>
> `fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;`
>
> 因为「location」已经做了限定，所以「fastcgi_index」其实也没有必要

## nginx & php-fpm
```bash
Nginx 和 PHP-FPM 的进程间通信有两种方式，一种是 TCP,一种是 UNIX Domain Socket.
其中 TCP 是 IP 加端口，可以跨服务器.而 UNIX Domain Socket 不经过网络，只能用于 Nginx 跟 PHP-FPM 都在同一服务器的场景。用哪种取决于你的 PHP-FPM 配置：
方式1：
php-fpm.conf: listen = 127.0.0.1:9000
nginx.conf: fastcgi_pass 127.0.0.1:9000;
方式2：
php-fpm.conf: listen = /tmp/php-fpm.sock
nginx.conf: fastcgi_pass unix:/tmp/php-fpm.sock;
其中 php-fpm.sock 是一个文件，由 php-fpm 生成，类型是srw-rw----.

UNIX Domain Socket 可用于两个没有亲缘关系的进程，是目前广泛使用的 IPC 机制，比如 X Window 服务器和 GUI 程序之间就是通过 UNIX Domain Socket 通讯的。这种通信方式是发生在系统内核里而不会在网络里传播。UNIX Domain Socket 和长连接都能避免频繁创建 TCP 短连接而导致TIME_WAIT 连接过多的问题。对于进程间通讯的两个程序，UNIX Domain Socket 的流程不会走到 TCP 那层，直接以文件形式，以 stream socket 通讯。如果是 TCP Socket,则需要走到 IP 层，对于非同一台服务器上，TCP Socket 走的就更多了。

UNIX Domain Socket:
Nginx <=> socket <=> PHP-FPM
TCP Socket(本地回环)：
Nginx <=> socket <=> TCP/IP <=> socket <=> PHP-FPM
TCP Socket(Nginx和PHP-FPM位于不同服务器)：
Nginx <=> socket <=> TCP/IP <=> 物理层 <=> 路由器 <=> 物理层 <=> TCP/IP <=> socket <=> PHP-FPM

像 mysql 命令行客户端连接 mysqld 服务也类似有这两种方式：
使用 Unix Socket 连接(默认)：
mysql -uroot -p --protocol=socket --socket=/tmp/mysql.sock
使用TCP连接:
mysql -uroot -p --protocol=tcp --host=127.0.0.1 --port=3306
```

## 15、nginx 代理
> 用户 <------> 代理服务器 <------> web 服务器

```bash
# 本测试中 代理服务器就是 虚拟机，web 服务器是 论坛
# 一个 ip

$ cd /usr/local/nginx/conf/vhosts/
$ vim proxy.conf
server {
       listen 80;
       #server_name www.baidu.com;
       server_name ask.apelearn.com;
       
       location /
       {
            #proxy_pass http://119.75.217.109/;（百度 ip）
             proxy_pass http://47.104.7.242/; //真正要访问的 web 服务器
             proxy_set_header Host $host; // 域名是上面的 server_name
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
}

$ curl ask.apelearn.com/robots.txt 
$ curl -x 127.0.0.1:80 ask.apelearn.com/robots.txt
$ yum install bind-utils
$ dig www.baidu.com //之前的 IP 地址可能失效，可以挖出更多最新的 ip
$ dig ask.apelearn.com
```

## 反向代理
场景设置：
1. A B 两台机器，A 只有内网，B 有内网和外网
2. A 的内网 ip 是 192.168.6.165
3. B 的内网 ip 是 192.168.6.128 B 的外网IP是 192.168.50.129
4. C 为客户端，C 只能访问B的外网 IP，不能访问 A 或者 B 的内网 IP

需求目的：C 要访问到 A 的内网上的网站
```
# 在客户端 hosts 文件里写入 A j里域名
192.168.50.129 blog.zi.cc

# 在 B 中 写入配置
$ vim conf/vhosts/proxy.conf
server {
    listen 80;
    server_name blog.zi.cc;
    location / {
        proxy_pass http://192.168.6.165;
        proxy_set_header Host $host;
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
$ nginx -t && nginx -s reload
$ irewall-cmd --add-port=80/tcp --permanent
$ firewall-cmd --reload
$ iptables -nvL |grep 80
```

## 16、nginx 负载均衡
> 多个 ip 相当于负载均衡

```bash
$ vim /usr/local/nginx/conf/vhosts/load.conf
upstream qqcom //upstream 用来指定多个 web server
{
        ip_hash; //ip_hash 同一用户保证请求在同一机器上
        server 111.161.64.40:80;
        server 111.161.64.48:80;
}
server
{
        listen 80;
        server_name www.qq.com;
        location /
        {
                proxy_pass http://qqcom; //upstream 模块名和 proxy_pass 名称一致
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}

$ curl -x 127.0.0.1:80 www.qq.com
# 在 windows hosts 里写入: 192.168.6.165 www.qq.com 然后在浏览器访问 www.qq.com
```
```
$ vim /usr/local/nginx/conf/vhosts/load2.conf
upstream apelearn
    {
        ip_hash;
        server 115.159.51.96:80 weight=100;
        server 47.104.7.242:80;

    }
    server
    {
        listen 80;
        server_name www.apelearn.com;
        location /
        {
            proxy_pass http://apelearn;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
```

> nginx 不支持 https 443

## 17、生成 SSL 密钥对
```bash
$ rpm -qf $(which openssl)
$ cd /usr/local/nginx/conf
$ openssl genrsa -des3 -out tmp.key 2048 //key文件为私钥 -des3 为加密算法
$ openssl rsa -in tmp.key -out zilo.key //转换 key ，取消密码
$ rm -f tmp.key
$ openssl req -new -key zilo.key -out zilo.csr
//生成证书请求文件，需要这个文件和私钥一直产生公钥文件
//Certificate Signing Request（CSR）
//req 统一生成密钥对和证书请求，也可以指定是否对私钥文件进行加密
$ openssl x509 -req -days 365 -in zilo.csr -signkey zilo.key -out zilo.crt //zilo.crt 为公钥
//x509 命令 进行格式转换及显示证书文件中的 text,module 等信息
```

## 18、nginx 配置 SSL
```bash
$ vim /usr/local/nginx/conf/vhosts/ssl.conf
server
{
    listen 443 ssl;
    server_name bbb.com;
    index index.html index.php index.htm;
    root /data/wwwroot/bbb.com;
    # ssl on;
    ssl_certificate zilo.crt;
    ssl_certificate_key zilo.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
}
报错：nginx: [emerg] unknown directive “ssl” 需要重新编译

$ /usr/local/nginx/sbin/nginx -V
$ cd /usr/local/src/nginx-1.14.0/
$ .configure --help | grep -i ssl
$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module
$ make && make install
$ -t && -s reload
$ /etc/init.d/nginx restart
$ netstat -lntp
:443
$ firewall-cmd --add-port=443/tcp --permanent
$ firewall-cmd --reload
$ vim /etc/hosts
:127.0.0.1 bbb.com
$ curl https://bbb.com/
$ curl -k -H "host:blog.zi.cc" https://192.168.6.165/index.php
```
> 在 windows hosts 中添加： 192.168.95.145 bbb.com 用浏览器访问 https://bbb.com

> 申请证书：  
> 网站：www.wosign.com （沃通）  
> 免费：freessl.org   
>  注册账号，输入域名，开始申请，在这个过程中需要去加一条TXT的记录

> https://github.com/ziluoxingjun/nginx/tree/master/ssl

## 19、php-fpm 的 pool
```bash
$ vim /usr/local/php-fpm/etc/php-fpm.conf
// 在 [global] 部分增加
 include = etc/php-fpm.d/*.conf
$ mkdir /usr/local/php/etc/php-fpm.d/
$ cd /usr/local/php-fpm/etc/php-fpm.d/
$ vim www.conf
 [www]
 listen = /tmp/www.sock
 listen.mode = 666
 user = php-fpm
 group = php-fpm
 pm = dynamic
 pm.max_children = 50
 pm.start_servers = 20
 pm.min_spare_servers = 5
 pm.max_spare_servers = 35
 pm.max_requests = 500
 rlimit_files = 1024
# 每个站点隔离开，单独写一个 pool 在 nignx 配置文件中可以根据不同站点配置不同的 socket
```
```bash
[global]
  pid = /usr/local/php/var/run/php-fpm.pid
  error_log = /usr/local/php/var/log/php-fpm.log
  [www]
  listen = /tmp/php-fcgi.sock
  user = php-fpm
  group = php-fpm
  pm = dynamic
  pm.max_children = 50 //最大子进程数
  pm.start_servers = 20 //启动几个子进程
  pm.min_spare_servers = 5 //空闲时，最少几个子进程
  pm.max_spare_servers = 35
  pm.max_requests = 500 //一个子进程在一个生命周期之内处理多少个请求
  rlimit_files = 1024
  [www1]
  listen = /tmp/php-fcgi1.sock
  user = php-fpm
  group = php-fpm
  pm = dynamic
  pm.max_children = 50
  pm.start_servers = 20
  pm.min_spare_servers = 5
  pm.max_spare_servers = 35
  pm.max_requests = 500
  rlimit_files = 1024
  php_flag[display_errors] = off //出现错误时，错误是否显示在网页中
  php_admin_value[error_log] = /var/log/fpm-php.www.log
  php_admin_flag[log_errors] = on
  php_admin_value[error_reporting] = E_ALL
```

> 如果 pm=static,只有这一条配置生效 pm.max_children = 50

> 可以指定 aaa.com.conf 用 www 这个 pool，bbb.com.conf 用 www1 这个 pool

> 可以在 Nginx 配置文件里此行 -> fastcgi_pass unix:/tmp/php-fcgi.sock,指定用哪个 pool，www.sock www1.sock，也可以都用一个sock（pool），好处是可以分别指定用户名，设置权限，如果一个访问量过大，down 掉了，另一个不受影响。


## 20、php-fpm 慢执行日志
```bash
# 性能追踪
$ vim /usr/local/php-fpm/etc/php-fpm.d/www.conf
 [www]
 request_slowlog_timeout = 2 //超过 2 秒就要记录日志，为什么慢
 slowlog = /usr/local/php-fpm/var/log/www-slow.log //排查慢的原因

$ vim /usr/local/nginx/conf/vhosts/bbb.com.conf
 unix:/tmp/php-fcgi.sock --> unix:/tmp/www.sock
$ /etc/init.d/nginx -s reload
$ /etc/init.d/php-fpm restart

$ vim /data/www/bbb.com/sleep.php
 <?php echo "test slow log";sleep(3); echo "done"; ?>
$ curl -x 127.0.0.1:80 bbb.com/sleep.php
$ cat /usr/local/php-fpm/var/log/www-slow.log
```

## 21、php-fpm 定义 open_basedir
- 有多个网站不适合在 php.ini 中定义
- 在 Apache 虚拟主机配置文件中定义 open_basedir
- 在 php-fpm 中根据不同的网站不同的 pool 定义不同的 open_basedir

```bash
$ vim /usr/local/php-fpm/etc/php-fpm.d/www.conf
 php_admin_value[open_basedir]=/data/wwwroot/bbb.com:/tmp/
$ vim /usr/local/php-fpm/etc/php.ini
//定义 php-fpm 错误日志和日志级别
 error_log = /usr/local/php-fpm/var/log/php-fpm_errors.log
 error_reporting = E_ALL
$ touch /usr/local/php-fpm/var/log/php-fpm_errors.log
$ chmod 777 /usr/local/php-fpm/var/log/php-fpm_errors.log
//以上两步操作防止不能正常写入日志
```


## 22、php-fpm 进程管理
```bash
pm = dynamic
//process manage dynamic 动态进程管理 一开始先启动20个服务，当访问量增加，动态增加子进程，如服务器比较空闲可自动销毁子进程
//static 静态 只有 pm.max_children = 50 生效，启动就50个子进程，不变
pm.max_children = 50
//最大子进程数
pm.start_servers = 20
//启动服务时启动的子进程
pm.min_spare_servers = 5
//在空闲时段，子进程最小数，如果达到数值，php-fpm 会自动派生新的子进程
pm.max_spare_servers = 35
//空闲时段，子进程最大数，如果达到数值，就开始清理空闲的子进程
pm.max_requests = 500
//一个子进程最多处理的请求数，一个 php-fpm 子进程最多可以处理的请求数，当达到这个数值会自动退出
```