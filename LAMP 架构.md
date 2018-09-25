## LAMP 架构
![LAMP 架构图](https://images.gitee.com/uploads/images/2018/0911/143122_f3f75dee_922657.png)

#### httpd php Mysql 三者工作原理

1. Apache 和 php 是一个整体，php 以模块的形式和 Apache 结合在一起，Apache 不能直接和 Mysql 相互通信，必须要通过 php 模块去 mysql 拿数据，php 拿到数据后交给 Apache, Apache再交给用户，以上操作称为动态请求。
2. 访问一个网站论坛，首先登录，在浏览器里输入网址登录时，请求发给 Apache ，Apache 检查请求是静态还是动态，登录论坛需要输入用户名和密码，Apache 拿到用户名和密码需要通过 php 模块去数据库比对，比对成功 Apache 会返回一个登录的状态，这个过程为动态请求。
3. 用户请求一张图片，文件等，Apache 就会从服务器上的某个目录下拿到图片再返回给用户，这个过程没有和 Mysql 打交道，这个过程为静态请求。文件不能存放在数据库中。

## 1、mysql/MariaDB 安装
> MariaDB 5.5 版本对应 Mysql 5.5 版本，MariaDB 10.0 版本对应 Mysql 5.6 版本

##### 1、下载、解压
```bash
$ wget https://cdn.mysql.com//Downloads/MySQL-5.6/mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
$ wget http://mirrors.ustc.edu.cn/mariadb/mariadb-10.2.14/bintar-linux-glibc_214-x86_64/mariadb-10.2.14-linux-glibc_214-x86_64.tar.gz
$ tar zxvf mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
```

##### 2、创建 mysql 账号
```bash
$ useradd -s /sbin/nologin mysql
```

##### 3、将解压完的目录移动到 /usr/local/mysql
```bash
$ mv mysql-5.6.41-linux-glibc2.12-x86_64 /usr/local/mysql
$ mv mariadb-10.2.14-linux-glibc_214-x86_64 /usr/local/mariadb
```

##### 4、初始化数据库
```bash
$ cd /usr/local/mysql/
$ mkdir -p /data/mysql
$ chown -R mysql /data/mysql
$ ./scripts/mysql_install_db --user=mysql --datadir=/data/mysql
$ ./scripts/mysql_install_db --user=mysql --basedir=/usr/local/mariadb/ --datadir=/data/mariadb
```

##### 5、拷贝配置文件
```bash
$ cd support-files/
$ cp my-large.cnf /etc/my.cnf (my-fault.cnf)
$ cp my-small.cnf /usr/local/mariadb/my.cnf
$ vim /usr/local/mariadb/my.cnf（定义 basedir datadir socket）
```

##### 6、拷贝启动脚本
```bash
$ cp mysql.server /etc/init.d/mysqld
$ cp mysql.server /etc/init.d/mariadb
```

##### 7、修改启动脚本
```bash
$ vim /etc/init.d/mysqld
$ vim /etc/init.d/mariadb //定义 basedir datadir conf 以及启动参数
 basedir=/usr/local/mysql
 datadir=/data/mysql
# mariadb
 conf=$basedir/my.cnf
 $bindir/mysqld_safe --defaults-file="$conf" --datadir="$datadir" --pid-file="$mysqld_pid_file_path" "$@" &
$ chmod 755 /etc/init.d/mysqld
```

##### 8、加入系统服务项，并设为开机启动，启动
```bash
$ chkconfig --add mysqld
$ chkconfig mysqld on
$ service mysqld start 或 /etc/init.d/mysqld start
$ /etc/init.d/mariadb start

# 检查mysql是否启动
$ ps aux|grep mysqld
$ netstat -lntp

# 手动命令行启动
$ /usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf --user=mysql --datadir=/data/mysql &
$ killall mysqld（比直接 kill pid 安全些，会停止当前的读写再杀死进程）
```

##### 报错
> ```bash
> FATAL ERROR: please install the following Perl modules before executing ./scripts/mysql_install_db:
> Data::Dumper
>
> $ yum list | grep perl | grep -i dumper
> $ yum install perl-Data-Dumper
> ```

Mysql 两个引擎：innodb myisam（存储量和存储空间都比较小）

## 2、Apache 源码编译安装

##### 1、下载软件包、解压
```bash
$ cd /usr/local/src
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/httpd/httpd-2.4.34.tar.bz2
$ wget http://mirrors.hust.edu.cn/apache/apr/apr-1.5.2.tar.gz
$ wget http://mirrors.hust.edu.cn/apache/apr/apr-util-1.5.4.tar.gz
# apr 和 apr-util 是一个通用函数库，httpd 依赖的一个包,可以让 httpd 不关心底层的操作系统平台，可以很方便的移植,跨平台应用，需要底层的包支持就是 apr
$ tar jxf httpd-2.4.34.tar.bz2
$ tar zxvf apr-1.5.2.tar.gz
$ tar zxvf apr-util-1.5.4.tar.gz
```

##### 2、安装 httpd 2.4 版本前安装依赖
```bash
# httpd 2.4 版本需要先安装 apr apr-util
配置编译参数：
$ cd /usr/local/apr-5.2
$ ./configure --prefix=/usr/local/apr
$ make && make install
$ cd /usr/local/apr-util-1.5.4
$ ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr
$ make && make install
# httpd 2.4 版本需要先安装以上
```

##### 报错
> ```bash
> xml/apr_xml.c:35:19: 致命错误：expat.h：没有那个文件或目录
> #include <expat.h>
>                   ^
> 编译中断。
> make[1]: *** [xml/apr_xml.lo] 错误 1
> make[1]: 离开目录“/usr/local/src/apr-util-1.6.1”
> make: *** [all-recursive] 错误 1
> $ yum install expat-devel
> ```

##### 3、编译安装 httpd 2.4
```bash
$ ./configure \
--prefix=/usr/local/apache2.4 \
--with-apr=/usr/local/apr \
--with-apr-util=/usr/local/apr-util \
--enable-so \
--enable-mods-shared=most //大多数动态扩展模块
$ make && make install
$ ll /usr/local/apache2.4/modules
$ /usr/local/apache2.4/bin/httpd -M //查看加载的模块
```

##### 报错
> ```bash
> configure: error: pcre-config for libpcre not found. PCRE is required and available from http://pcre.org/
> $ yum install pcre-devel
>
> collect2: error: ld returned 1 exit status
> make[2]: *** [htpasswd] 错误 1
> make[2]: 离开目录“/usr/local/src/httpd-2.4.34/support”
> make[1]: *** [all-recursive] 错误 1
> make[1]: 离开目录“/usr/local/src/httpd-2.4.34/support”
> make: *** [all-recursive] 错误 1
> ** apr 用 1.5 版本即可解决 **
> ```

##### 4、编译安装 httpd 2.2 
```bash
# httpd 2.2 安装无需先安装依赖
$ ./configure \
--prefix=/usr/local/apache2.2 \ //指定安装到位置
--with-included-apr \
--enable-so \ //表示启用DSO，支持动态扩展模块
--enable-deflate=shared \ //表示动态共享的方式编译 deflate
--enable-expires=shared \
--enable-rewrite=shared \
--with-pcre //正则相关的库
$ make && make install 
```
> DSO:DSO是Dynamic Shared Objects（动态共享目标）的缩写，它提供了一种在运行时将特殊格式的代码在程序运行需要时，将需要的部分从外存调入内存执行的方法。Apache 支持动态共享模块，也支持静态模块，静态的话，会把需要的目标直接编译进apache的可执行文件中，相比较动态，虽然省去了加载共享模块的步骤，但是也加大了二进制执行文件的空间，变得臃肿。

###### 报错
> ```bash
> error: mod_deflate has been requested but can not be built due to prerequisite failures
> $ yum install -y zlib-devel
> # 为了避免在make的时候出现错误，所以最好是提前先安装好一些库文件:
> $ yum install -y pcre pcre-devel apr apr-devel
> ```

##### 5、启动
```bash
$ /usr/local/apache2/bin/apachectl start
$ ps aux|grep httpd
```

##### 6、相关命令
```bash
$ /usr/local/apache2/bin/apachectl -M //列出模块
$ /usr/local/apache2/bin/apachectl -l //列出静态模块
$ /usr/local/apache2/bin/apachectl -t //检查配置文件是否有语法错误
$ /usr/local/apache2/bin/apachectl restart //先将进程杀死再开启新的进程
$ /usr/local/apache2/bin/apachectl gracefull //进程还在，将配置文件重新加载
Loaded Modules:
 core_module (static) //静态，直接将模块编译进主脚本，二进制文件中，也就是核心文件 httpd 中
 so_module (static)
 http_module (static)
 mpm_event_module (static)
 authn_file_module (shared) //扩展模块
```

##### 报错
> ```bash
> httpd: apr_sockaddr_info_get() failed for xing
> httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1 for ServerName
> 
> AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using fe80::94cd:961b:56ca:8df2. > Set the 'ServerName' directive globally to suppress this message
>
> 如果出现上面的错误，需要在配置文件中打开一行(/usr/local/apache2/conf/httpd.conf)
> ServerName www.example.com 80
> ```

## 3、php 编译安装

##### 1、下载、解压
```bash
$ wget http://cn2.php.net/get/php-5.6.37.tar.gz/from/this/mirror/php-5.6.37.tar.gz
$ wget http://cn2.php.net/distributions/php-5.6.37.tar.bz2 ^C
$ wget http://cn2.php.net/distributions/php-7.1.21.tar.bz2 ^C
$ tar jxvf php-5.6.37.tar.bz2
```

##### 2、php5 编译安装
```bash
$ cd php-5.6.37
$ ./configure \
--prefix=/usr/local/php \
--with-apxs2=/usr/local/apache2.4/bin/apxs \ //自动帮助我们安装拓展模块的
--with-config-file-path=/usr/local/php/etc  \
--with-mysql=/usr/local/mysql \ //依赖于mysql,php7 不需要此参数
--with-pdo-mysql=/usr/local/mysql \
--with-mysqli=/usr/local/mysql/bin/mysql_config \
--with-libxml-dir \
--with-gd \
--with-jpeg-dir \
--with-png-dir \
--with-freetype-dir \
--with-iconv-dir \
--with-zlib-dir \
--with-bz2 \
--with-openssl \
--with-mcrypt \
--enable-soap \
--enable-gd-native-ttf \
--enable-mbstring \
--enable-sockets \
--enable-exif \
$ make && make install
```

##### 3、php7 编译安装
```bash
$ cd php-7.1.21
$ ./configure \
--prefix=/usr/local/php7 \
--with-apxs2=/usr/local/apache2.4/bin/apxs \
--with-config-file-path=/usr/local/php7/etc  \
--with-pdo-mysql=/usr/local/mysql \
--with-mysqli=/usr/local/mysql/bin/mysql_config \
--with-libxml-dir \
--with-gd \
--with-jpeg-dir \
--with-png-dir \
--with-freetype-dir \
--with-iconv-dir \
--with-zlib-dir \
--with-bz2 \
--with-openssl \
--with-mcrypt \
--enable-soap \
--enable-gd-native-ttf \
--enable-mbstring \
--enable-sockets \
--enable-exif \
$ make && make install
```

##### 4、拷贝配置文件
```bash
$ cp php.ini-production /usr/local/php/etc/php.ini //php.ini-production 是线上生产环境用；php.ini-development 是开发环境用
```

##### 5、php 相关命令
```bash
$ ls /usr/local/apache2.4/modules/libphp5.so
$ /usr/local/php/bin/php -m //查看所加载模块，静态
$ /usr/local/php/bin/php -i //查看相关配置
$ cat /usr/local/apache2/build/config.nice apache //配置编译参数
```

##### 报错
> ```bash
> configure: error: xml2-config not found. Please check your libxml2 installation.
> $ yum install libxml2-devel
>
> configure: error: Cannot find OpenSSL's <evp.h>
> $ yum install openssl-devel
>
> configure: error: Please reinstall the BZip2 distribution
> $ yum install bzip2-devel
>
> configure: error: jpeglib.h not found.
> $ yum install libjpeg-turbo-devel
>
> configure: error: png.h not found.
> $ yum install libpng-devel
>
> configure: error: freetype-config not found.
> $ yum install freetype-devel
>
> configure: error: mcrypt.h not found. Please reinstall libmcrypt.
> $ yum install epel-release
> $ yum install libmcrypt-devel
> ```

## 4、Apache 和 php 结合

##### 1、配置 httpd 支持 php
```bash
# httpd 主配置文件 /usr/local/apache2.4/conf/httpd.conf
$ vim /usr/local/apache2.4/conf/httpd.conf //修改4个地方
 193 ServerName
 202 Require all granted
 251 DirectoryIndex index.html index.php
 389 AddType application/x-httpd-php .php
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ netstat -lntp
$ curl localhost
$ vim /usr/local/apache2.4/htdocs/test.php
 <?php
 	phpinfo();
 	echo 123;
 ?>
$ /usr/local/php/bin/php -i
# 有可能 80 端口没有打开，手动增加
$ iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

## 5、Apache 默认虚拟主机
> 一台服务器可以访问多个网站，每个网站都是一个虚拟主机
> 任何一个域名解析到这台机器，都可以访问的虚拟主机就是默认虚拟主机

```bash
$ vim /usr/local/apache2.4/conf/httpd.conf
 # Virtual hosts
 477 Include conf/extra/httpd-vhosts.conf //去掉前面的 # 符号

$ vim /usr/local/apache2/conf/extra/httpd-vhosts.conf 
 <VirtualHost *:80>
     DocumentRoot "/data/wwwroot/xing.com"
     ServerName xing.com
     ServerAlias www.xing.com
     ErrorLog logs/xing.com-error.log
     CustomLog logs/xing.com-access.log
 </VirtualHost>
 <VirtualHost *:80>
     DocumentRoot "/data/wwwroot/abc.com"
     ServerName abc.com
 </VirtualHost>
$ mkdir -p /data/wwwroot/xing.com
$ mkdir -p /data/wwwroot/abc.com
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x192.168.95.10:80 xing.com
```

## 6、Apache 用户认证

#### 1、针对目录认证
```bash
$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.abc.com
    <Directory /data/wwwroot/abc.com> //指定认证的目录
        AllowOverride AuthConfig //相当于打开认证的开关
        AuthName "abc.com user auth" //自定义认证的名字，作用不大
        AuthType Basic //认证的类型，一般为 Basic
        AuthUserFile /data/.htpasswd //指定密码文件所在位置
        require valid-user //指定需要认证的用户为全部可用用户，也就是 .htpasswd 里面定义的用户
    </Directory>
    ErrorLog "logs/abc.com-error.log"
    CustomLog "logs/abc.com-access.log" common
</VirtualHost>

# 生成 .htpasswd 文件
$ /usr/local/apache2.4/bin/htpasswd -h
$ /usr/local/apache2.4/bin/htpasswd -cm /data/.htpasswd abc //创建第二个用户不用加 -c 选项
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x192.168.95.13:80 abc.com -I
HTTP/1.1 401 Unauthorized
$ curl -x192.168.95.13:80 -uuser:passwd abc.com -I
HTTP/1.1 200 OK
```

#### 2、针对单个文件认证
```bash
$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.abc.com
    <FilesMatch admin.php>
        AllowOverride AuthConfig
        AuthName "abc.com user auth"
        AuthType Basic
        AuthUserFile /data/.htpasswd
        require valid-user
    </FilesMatch>
    ErrorLog "logs/abc.com-error.log"
    CustomLog "logs/abc.com-access.log" common
</VirtualHost>

$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x 192.168.95.13:80 abc.com/admin.php -I
HTTP/1.1 401 Unauthorized
$ curl -x 192.168.95.13:80 abc.com/admin.php -uuser:passwd
HTTP/1.1 200 OK
```

## 7、域名跳转（域名重定向）
> 需求：将 abc.com 跳转到 www.xing.com

```bash
$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.xyz.com
    <IfModule mod_rewrite.c> //需要 mod_rewirte 模块支持
        RewriteEngine on //打开 rewrite 功能
        RewriteCond %{HTTP_HOST} !^abc.com$ //定义 rewrite 条件，主机名（域名）不是 abc.com 的时候满足条件
        RewriteRule ^/(.*)$ http://www.xing.com/$1 [R=301,L] //定义 rewrite 规则，当满足上面的条件时，本规则才会执行，L 表示只跳转一次，last
    </IfModule>
    ErrorLog "logs/abc.com-error.log"
    CustomLog "logs/abc.com-access.log" common
</VirtualHost>

$ bin/apachectl -M | grep rewrite //检测是否打开 rewrite 模块
# 打开 rewrite 模块
$ vim /usr/local/apache2.4/conf/httpd.conf
150 LoadModule rewrite_module modules/mod_rewrite.so

$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x127.0.0.1:80 www.xyz.com -I
HTTP/1.1 301 Moved Permanently // 301 永久重定向
```

## 8、Apache 访问日志
> 访问日志记录用户的每一个请求
```bash
$ vim /usr/local/apache2.4/conf/httpd.conf
283     LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
284     LogFormat "%h %l %u %t \"%r\" %>s %b" common

$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName ddd.com
    ServerAlias www.xyz.com
    <IfModule mod_rewrite.c> //需要 mod_rewirte 模块支持
        RewriteEngine on //打开 rewrite 功能
        RewriteCond %{HTTP_HOST} !^ddd.com$ //定义 rewrite 条件，主机名（域名）不是 ddd.com 的时候满足条件
        RewriteRule ^/(.*)$ http://www.test.com/$1 [R=301,L] //定义 rewrite 规则，当满足上面的条件时，本规则才会执行，L 表示只跳转一次，last
    </IfModule>
    ErrorLog "logs/abc.com-error.log"
    CustomLog "logs/abc.com-access.log" combined
</VirtualHost>

$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x 127.0.0.1:80 www.xyz.com -I
$ tail /usr/local/apache2.4/logs/abc.com-access.log
```

## 9、Apache 访问日志不记录静态文件
> 网站大多数元素为静态文件，如图片，css，js等，这些静态元素可以不用记录到日志中

```bash
$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.lll.com
    ErrorLog "logs/abc.com-error.log"
    SetEnvIf Request_URI ".*\.gif$" img
    SetEnvIf Request_URI ".*\.jpg$" img
    SetEnvIf Request_URI ".*\.png$" img
    SetEnvIf Request_URI ".*\.bmp$" img
    SetEnvIf Request_URI ".*\.swf$" img
    SetEnvIf Request_URI ".*\.js$" img
    SetEnvIf Request_URI ".*\.css$" img
    SetEnvIf Request_URI ".*\.txt$" img
    SetEnvIf Request_URI ".*\.ico$" img
    CustomLog "logs/abc.com-access.log" combined env=!img
</VirtualHost>
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x 127.0.0.1:80 www.lll.com/meinv.png -I
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x 127.0.0.1:80 www.lll.com/meinv.png -I
$ tail /usr/local/apache2.4/logs/abc.com-access.log
```

## 10、Apache 访问日志切割
> 随着日志越来越多，占用磁盘空间，必须自动切割，并删除老旧的日志

#### 1、访问日志切割
```bash
$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.lll.com
    ErrorLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-error_%Y%m%d.log 86400"
    CustomLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-access_%Y%m%d.log 86400" //最前面的那个竖线是管道符，意思是把产生的日志交给 rotatelogs，Apache 自带的切割日志的工具， -l 以当前系统日期为基准切割
</VirtualHos
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x127.0.0.1:80 www.lll.com
$ ls /usr/local/apache2.4/logs/
```

#### 2、定期删除老旧日志
```bash
$ mkdir /usr/local/crontab
$ vim /usr/local/crontab/clean_apache_logs.sh
#!/bin/bash
# 删除 apache 日志文件，保留最近30天的日志
/usr/bin/find /usr/local/apache2.4/logs -mtime +30 -exec rm -f {} \;

$ chmod 755 /usr/local/crontab/clean_apache_logs.sh
$ crontab -e
0 0 1 * * /usr/local/crontab/clean_apache_logs.sh
$ crontabl -l
```

## 11、配置静态元素的过期时间
> 浏览器访问网站的图片时会把静态文件缓存在本地电脑，这样下次访问时就不用从远程下载了，减少访问时间，加快访问速度，节省带宽，减小IO压力

```bash
# 打开 expires 模块
$ vim /usr/local/apache2.4/conf/httpd.conf
109 #LoadModule expires_module modules/mod_expires.so

$ vim /usr/local/apache2.4/conf/extra/httpd-vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.lll.com
    <IfModule mod_expires.c>
        ExpiresActive on
        ExpiresByType image/gif "access plus 1 days"
        ExpiresByType image/jpeg "access plus 24 hours"
        ExpiresByType image/png "access plus 24 hours"
        ExpiresByType text/css "now plus 2 hours"
        ExpiresByType application/x-javascript "now plus 2 hours"
        ExpiresByType application/javascript "now plus 2 hours"
        ExpiresByType application/x-shockwave-flash "now plus 2 hours"
        ExpiresDefault "now plus 0 min"
    </IfModule>
    ErrorLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-error_%Y%m%d.log 86400"
    CustomLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-access_%Y%m%d.log 86400"
</VirtualHost>
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -x127.0.0.1:80 abc.com/images/mix.jpg -I
Cache-Control: max-age=86400
Expires: Fri, 21 Sep 2018 06:30:11 GMT
```
> 在浏览器中访问  abc.com/images/mix.jpg，第一次 Status Code:200 ,再次刷新后 Status Code: 304 Not Modified

## 12、配置防盗链
> 通过限制 referer 来实现防盗链的功能
```bash
$ vim /usr/local/apache2.4/conf/extra/httpd_vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.aaa.com
    <Directory /data/wwwroot/abc.com>
        SetEnvIfNoCase Referer "http://abc.com" local_ref
        SetEnvIfNoCase Referer "http://www.aaa.com" local_ref
        SetEnvIfNoCase Referer "^$" local_ref //空的 referer
        <FilesMatch "\.(txt|doc|mp3|zip|rar|jpg|gif|png|swf)">
            Order Allow,Deny
            Allow from env=local_ref
        </FilesMatch>
    </Directory>
    ErrorLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-error_%Y%m%d.log 86400"
    CustomLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-access_%Y%m%d.log 86400"
</VirtualHost>
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
$ curl -e "http://www.aaa.com" -x 127.0.0.1:80 abc.com/cat.jpg -I
$ curl -e "http://abc.com" -x 127.0.0.1:80 abc.com/images/cat.jpg -I
HTTP/1.1 200 OK

$ curl -e "http://qq.com" -x 127.0.0.1:80 abc.com/images/cat.jpg -I
HTTP/1.1 403 Forbidden
```

## 13、访问控制 Directory
```bash
$ vim /usr/local/apache2.4/conf/extra/httpd_vhosts.conf
<VirtualHost *:80>
    DocumentRoot "/data/wwwroot/abc.com"
    ServerName abc.com
    ServerAlias www.aaa.com
    <Directory /data/wwwroot/abc.com/admin/>
        Order deny,allow
        Deny from all
        allow from 127.0.0.1
    </Directory>
    ErrorLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-error_%Y%m%d.log 86400"
    CustomLog "|/usr/local/apache2.4/bin/rotatelogs -l logs/abc.com-access_%Y%m%d.log 86400"
</VirtualHost>
$ /usr/local/apache2.4/bin/apachectl -t
$ /usr/local/apache2.4/bin/apachectl graceful
```