## FTP 服务介绍
> FTP(File Transfer Protocol) 文件传输协议，用于在 Internet 上控制文件双向传输

## vsftpd 搭建 FTP 服务
```bash
$ yum install vsftpd
$ useradd -s /sbin/nologin virftp
$ vim /etc/vsftpd/vsftpd_login //虚拟用户密码文件
用户名
密码
user1
passwd1
$ chmod 600 /etc/vsftpd/vsftpd_login
# 保存虚拟帐号和密码的文本文件无法被系统帐号直接调用。我们需要使用 db_load 命令生成 二进制 db 文件
$ db_load -T -t hash -f /etc/vsftpd/vsftpd_login /etc/vsftpd/vsftpd_login.db // -T 允许应用程序能够将文本文件转译载入数据库。-t 使用 hash 码加密


```

## pure-ftp 搭建 FTP 服务


