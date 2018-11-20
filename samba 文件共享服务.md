## samba 介绍
> www.samba.org

> 1987年,微软和英特尔共同制定了 SMB 协议（Server Messages Block，服务器消息块），旨在解决局域网内的文件或打印机等资源的共享问题。

> 1991年，Tridgwell 为了解决 Linux 与 Windows 之间的文件共享问题，基于 SMB 协议开发出了开源文件共享软件 samba

## 安装配置在 windows 下访问 samba
```bash
$ yum install samba
$ vim /etc/samba/smb.conf
workgroup = WORKGROUP
security = user
# 在最后增加
[star]
comment = share all
path = /home/smbdir
browseable = yes //为 no 的话，在电脑中看不到，需要从指定地址访问
public = yes //是否允许 guest 账户访问
writable = no
$ useradd smbuser1
$ pdbedit -a -u smbuser1 //建立 samba 用户
$ smbpasswd -a smbuser1 //同上
$ pdbeidt -Lv //查看 samba 用户
$ mkdir /home/smbdir
$ chown -R smbuser1:smbuser1 /home/smbdir
$ systemctl start smb

在 windows 下 win/win + r 输入 \\192.168.95.191 即可
```


## 配置在 linux 下访问 samba
```bash
$ yum install samba-client
$ smbclient -L //192.168.1.11/xing

# 挂载使用
$ yum install cifs-utils
$ mount -t cifs //192.168.1.11/xing /mnt
$ df -h

# 设置输入用户名和密码的，可读可写的共享目录
$ vim /etc/samba/smb.conf
 security = user
 [star]
        comment = share for users
        path = /home/smbdir
        browseable = yes
        writable = yes
        public = no

$ useradd smbuser1 //不用设密码，因为登录 samba 不是用系统用户，而是虚拟用户
$ pdbedit -a smbuser1 //-a add
# 输入密码，在 linux 下访问
$ smbclient -Usmbuser1 //192.168.1.11/star

$ mount -t cifs -o username=smbuser1,password=123456 //192.168.95.191/xing /opt
```

## pdbedit 命令
> pdbedit 命令用于管理 samba 服务程序的用户信息数据库
```bash
$ pdbedit -a username 建立 samba 用户
$ pdbedit -x usernmae 删除 samba 用户
$ pdbedit -L 列出用户列表
$ pdbeidt -Lv 列出详细信息的列表
```
