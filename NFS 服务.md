### NFS 介绍
> NFS == Network File System
> NFS 是由 Sun 开发并发展起来的一项用于在不同机器，不同操作系统之间通过网络互相分享各自的文件。可以让你的 PC 通过网络将远端的目录 MOUNT 到自己的系统中，使用 NFS 的远端文件就像是在使用本地文件一样。
> 有 2 3 4 三个版本，4.0版本开始 Netapp 公司参与并主导开发
> NFS 数据传输基于 RPC(Remote Procedure Call 远程过程调用) 协议

## exportfs 命令

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