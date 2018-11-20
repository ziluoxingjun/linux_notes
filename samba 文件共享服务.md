## samba 介绍
> www.samba.org

> 1987年,微软和英特尔共同制定了 SMB 协议（Server Messages Block，服务器消息块），旨在解决局域网内的文件或打印机等资源的共享问题。

> 1991年，Tridgwell 为了解决 Linux 与 Windows 之间的文件共享问题，基于 SMB 协议开发出了开源文件共享软件 samba

## 安装配置在 windows 下访问 samba
```bash
$ yum install samba
$ vim /etc/samba/smb.conf
workgroup = WORKGROUP
security = share //修改为 share
# 在最后增加,设置不用输入用户名密码，只读的共享目录
[star]
comment = share all
path = /tmp/sambadir
browseable = yes //为 no 的话，在电脑中看不到，需要从指定地址访问
public = yes //是否允许 guest 账户访问
writable = no
$ mkdir /tmp/sambadir
$ systemctl start smb

在 windows 下 ctrl + r 输入 \\192.168.1.11 即可
```


## 配置在 linux 下访问 samba
```bash
$ smbclient //192.168.1.11/star

# 挂载使用
$ yum install cifs-utils
$ mount -t cifs //192.168.1.11/star /opt
$ df -h

# 设置输入用户名和密码的，可读可写的共享目录
$ vim /etc/samba/smb.conf
 security = user
 [star]
        comment = share for users
        path = /tmp/sambadir
        browseable = yes
        writable = yes
        public = no

$ useradd smbuser1 //不用设密码，因为登录 samba 不是用系统用户，而是虚拟用户
$ pdbedit -a smbuser1（-a add）
# 输入密码，在 linux 下访问
$ smbclient -Usmbuser1 //192.168.1.11/star

$ mount -t cifs -o username=smbuser1,password=123456 //192.168.1.11/star /opt
```