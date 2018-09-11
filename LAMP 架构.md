## LAMP 架构
![LAMP 架构图](https://images.gitee.com/uploads/images/2018/0911/143122_f3f75dee_922657.png)
（Apache 和 php 是一个整体，php 以模块的形式和 Apache 结合在一起，Apache 不能直接和 Mysql 通信，必须要通过 php 模块去 mysql 拿数据，php 拿到数据后交给 Apache,Apache再交给用户，以上操作称为动态请求。）
#### 1、mysql/MariaDB 安装
```bash
$ wget mysql-5.1.73-linux-x86_64-glibc23.tar.gz
$ wget http://mirrors.ustc.edu.cn/mariadb/mariadb-10.2.14/bintar-linux-glibc_214-x86_64/mariadb-10.2.14-linux-glibc_214-x86_64.tar.gz
```
