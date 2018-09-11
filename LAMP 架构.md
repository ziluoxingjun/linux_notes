## LAMP 架构
![LAMP 架构图](https://images.gitee.com/uploads/images/2018/0911/143122_f3f75dee_922657.png)

#### httpd php Mysql 三者工作原理

1. Apache 和 php 是一个整体，php 以模块的形式和 Apache 结合在一起，Apache 不能直接和 Mysql 相互通信，必须要通过 php 模块去 mysql 拿数据，php 拿到数据后交给 Apache,Apache再交给用户，以上操作称为动态请求。
2. 访问一个网站论坛，首先登录，在浏览器里输入网址登录时，请求发给 Apache ，Apache 检查请求是静态还是动态，登录论坛需要输入用户名和密码，Apache 拿到用户名和密码需要通过 php 模块去数据库比对，比对成功 Apache 会返回一个登录的状态，这个过程为动态请求。
3. 用户请求一张图片，文件等，Apache 就会从服务器上的某个目录下拿到图片再返回给用户，这个过程没有和 Mysql 打交道，这个过程为静态请求。文件不能存放在数据库中。

#### 1、mysql/MariaDB 安装
> MariaDB 5.5 版本对应 Mysql 5.5 版本，MariaDB 10.0 版本对应 Mysql 5.6 版本

```bash
$ wget mysql-5.1.73-linux-x86_64-glibc23.tar.gz
$ wget http://mirrors.ustc.edu.cn/mariadb/mariadb-10.2.14/bintar-linux-glibc_214-x86_64/mariadb-10.2.14-linux-glibc_214-x86_64.tar.gz
```
