## LAMP 架构
![LAMP 架构图](https://images.gitee.com/uploads/images/2018/0911/143122_f3f75dee_922657.png)

#### httpd php Mysql 三者工作原理

1. Apache 和 php 是一个整体，php 以模块的形式和 Apache 结合在一起，Apache 不能直接和 Mysql 相互通信，必须要通过 php 模块去 mysql 拿数据，php 拿到数据后交给 Apache, Apache再交给用户，以上操作称为动态请求。
2. 访问一个网站论坛，首先登录，在浏览器里输入网址登录时，请求发给 Apache ，Apache 检查请求是静态还是动态，登录论坛需要输入用户名和密码，Apache 拿到用户名和密码需要通过 php 模块去数据库比对，比对成功 Apache 会返回一个登录的状态，这个过程为动态请求。
3. 用户请求一张图片，文件等，Apache 就会从服务器上的某个目录下拿到图片再返回给用户，这个过程没有和 Mysql 打交道，这个过程为静态请求。文件不能存放在数据库中。

#### 1、mysql/MariaDB 安装
> MariaDB 5.5 版本对应 Mysql 5.5 版本，MariaDB 10.0 版本对应 Mysql 5.6 版本

##### 1、下载
```bash
$ wget https://cdn.mysql.com//Downloads/MySQL-5.6/mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
$ wget http://mirrors.ustc.edu.cn/mariadb/mariadb-10.2.14/bintar-linux-glibc_214-x86_64/mariadb-10.2.14-linux-glibc_214-x86_64.tar.gz
```

##### 2、解压
```bash
$ tar zxvf mysql-5.6.41-linux-glibc2.12-x86_64.tar.gz
```

##### 3、创建 mysql 账号
```bash
$ useradd -s /sbin/nologin mysql
```

##### 4、将解压完的目录移动到 /usr/local/mysql
```bash
$ mv mysql-5.6.41-linux-glibc2.12-x86_64 /usr/local/mysql
$ mv mariadb-10.2.14-linux-glibc_214-x86_64 /usr/local/mariadb
```

##### 5、初始化数据库
```bash
$ cd /usr/local/mysql/
$ mkdir -p /data/mysql
$ chown -R mysql /data/mysql
$ ./scripts/mysql_install_db --user=mysql --datadir=/data/mysql
$ ./scripts/mysql_install_db --user=mysql --basedir=/usr/local/mariadb/ --datadir=/data/mariadb
```

##### 6、拷贝配置文件
```bash
$ cd support-files/
$ cp my-large.cnf /etc/my.cnf (my-fault.cnf)
$ cp my-small.cnf /usr/local/mariadb/my.cnf
$ vim /usr/local/mariadb/my.cnf（定义 basedir datadir socket）
```

##### 7、拷贝启动脚本
```bash
$ cp mysql.server /etc/init.d/mysqld
$ cp mysql.server /etc/init.d/mariadb
```

##### 8、修改启动脚本
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

##### 9、加入系统服务项，并设为开机启动，启动
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

