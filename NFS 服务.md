## NFS 介绍
> NFS == Network File System

> NFS 是由 Sun 开发并发展起来的一项用于在不同机器，不同操作系统之间通过网络互相分享各自的文件。可以让你的 PC 通过网络将远端的目录 MOUNT 到自己的系统中，使用 NFS 的远端文件就像是在使用本地文件一样。

> 有 2 3 4 三个版本，4.0版本开始 Netapp 公司参与并主导开发

> NFS 数据传输基于 RPC(Remote Procedure Call 远程过程调用) 协议

> NFS　应用场景：Ａ,Ｂ,Ｃ 三台机器上需要保证被访问到的文件是一样的，Ａ共享数据出来，Ｂ和Ｃ分别挂载Ａ共享的数据目录，从而实现Ｂ和Ｃ访问的数据和Ａ一样

![NFS 原理](https://images.gitee.com/uploads/images/2018/1118/180948_370cb8f0_922657.png "屏幕截图.png")

早期版本叫做 portmap ，后面版本叫做 rpcbind 其实是一个东西，默认监听 111 端口

## NFS 服务端安装配置
```bash
$ yum install nfs-utils //同时会安装好 rpcbind
$ vim /etc/exports
/home/nfs 192.168.95.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)
$ mkdir -p /home/nfs
$ chmod 777 /home/nfs
$ systemctl start rpcbind
$ systemctl start nfs
$ systemctl enable rpcbind
$ systemctl enable nfs
```
## 客户端安装
```bash
$ yum install nfs-utils
$ showmount -e 192.168.95.191 //NFS 服务端 ip
$ mount -t nfs 192.168.95.191:/home/nfs /mnt
$ df -h
$ touch /mnt/test
$ ll /mnt/test //可以看到属主属组都为 1000
```

## NFS 配置选项
> rw：可读写的权限

> ro：只读的权限

> sync：内存数据实时写入磁盘

> async：非同步模式，数据先会存入内存中，不会写入磁盘

> no_root_squash：客户端挂载 NFS 共享目录后，root 用户不受约束，权限很大，此参数很不安全，建议不要使用

> root_squash：与上面选项相对，客户端上的 root 用户受到约束，会被限定为某个普通用户

> all_squash：客户端上所有用户在使用 NFS 共享目录时都被限定为一个普通用户

> anonuid/anongid：定义被限定用户的 uid gid,uid 必须存在于 /etc/passwd 中，和上面几个选项搭配使用

## exportfs 命令
exportfs 命令可以使 NFS 配置立刻生效而不用重启 NFS 服务

常用选项：
- -a 全部挂载或者全部卸载
- -r 重新挂载
- -u 卸载某一个目录
- -v 显示共享目录

在服务端上
```bash
$ vim /etc/exports //增加一行
/tmp/ 192.168.95.190(rw,sync,no_root_squash)
$ exportfs -arv //不用重启 nfs 服务，配置就会生效
```

## 客户端问题

1. 客户端挂共享目录后，不管是 root 还是普通用户，创建新文件时属主和属组都为 nobody
2. nfs 4 版本上会有该问题

解决方案1：
在客户端挂载时执行以下命令
```bash
$ mount -t nfs -oremount,nfsvers=3 192.168.95.191:/tmp /mnt //remount 不用卸载，重新挂载，nfsvers 指定版本为3
```

解决方案2：
在服务端和客户端修改 Domain 后，重启 idmapd(rpcbind) 服务
```bash
$ vim /etc/idmapd.conf
5 #Domain = local.domain.edu //去掉 # 号或者改为下面
6 Domain = xxx.com //随意定义一个域名
```