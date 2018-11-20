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
# 保存虚拟帐号和密码的文本文件无法被系统帐号直接调用。需要生成 vsftp 可以识别的二进制 db 的库文件，如果更改了 vsftpd_login 文件，需要重新执行此命令
$ db_load -T -t hash -f /etc/vsftpd/vsftpd_login /etc/vsftpd/vsftpd_login.db // -T 允许应用程序能够将文本文件转译载入数据库。-t 使用 hash 码加密
$ mkdir /etc/vsftpd/vsftpd_user_conf //虚拟用户配置文件
$ cd !$
$ vim user1 //与 vsftpd_login 里同名文件
 local_root=/home/virftp/user1
 anonymous_enable=NO
 write_enable=YES
 local_umask=022
 anon_upload_enable=NO
 anon_mkdir_write_enable=NO
 idle_session_timeout=600
 data_connection_timeout=120
 max_clients=10
 #max_per_ip=5
 #local_max_rate=50000
$ mkdir /home/virftp/user1
$ touch /home/virftp/user1/demo.txt
$ chown -R virftp:virftp /home/virftp/user1
$ vim /etc/pam.d/vsftpd //认证方式，用虚拟用户登录，否则会用系统用户登录
auth sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vsftpd_login //上面 db_load 就和 pam_userdb.so（认证的模块）有关
account sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vsftpd_login

$ vim /etc/vsftpd/vsftpd.conf
anonymous_enable=NO
anon_upload_enable=NO
anon_mkdir_write_enable=NO
chroot_local_user=YES //限制用户只能访问家目录，不能访问其它目录
# 最下面加入
guest_enable=YES //设为 NO 虚拟用户无法登录
guest_username=virftp
virtual_use_local_privs=YES
user_config_dir=/etc/vsftpd/vsftpd_user_conf
allow_writeable_chroot=YES

# 查看日志
$ vim /var/log/secure
$ vim /var/log/messages

```

VSFTP virtual_use_local_privs 参数
 
当virtual_use_local_privs=YES时，虚拟用户和本地用户有相同的权限；
当virtual_use_local_privs=NO时，虚拟用户和匿名用户有相同的权限，默认是NO。
 
当virtual_use_local_privs=YES，write_enable=YES时，虚拟用户具有写权限（上传、下载、删除、重命名）。
 
当virtual_use_local_privs=NO，write_enable=YES，anon_world_readable_only=YES，
anon_upload_enable=YES时，虚拟用户不能浏览目录，只能上传文件，无其他权限。
 
当virtual_use_local_privs=NO，write_enable=YES，anon_world_readable_only=NO，
anon_upload_enable=NO时，虚拟用户只能下载文件，无其他权限。
 
当virtual_use_local_privs=NO，write_enable=YES，anon_world_readable_only=NO，
anon_upload_enable=YES时，虚拟用户只能上传和下载文件，无其他权限。
 
当virtual_use_local_privs=NO，write_enable=YES，anon_world_readable_only=NO，
anon_mkdir_write_enable=YES时，虚拟用户只能下载文件和创建文件夹，无其他权限。
 
当virtual_use_local_privs=NO，write_enable=YES，anon_world_readable_only=NO，
anon_other_write_enable=YES时，虚拟用户只能下载、删除和重命名文件，无其他权限。


## pure-ftp 搭建 FTP 服务


