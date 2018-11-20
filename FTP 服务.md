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
> www.pureftpd.org

yum 安装配置 pure-ftpd
```bash
$ yum install epel-release
$ yum install pure-ftpd
$ vim /etc/pure-ftpd/pure-ftpd.conf
126  PureDB                        /etc/pure-ftpd/pureftpd.pdb //将注释去掉
$ systemctl start pure-ftpd
$ mkdir /home/pureftp
$ useradd -u 1010 pure-ftp
$ chown -R pure-ftp:pure-ftp /home/pureftp
$ pure-pw useradd ftp_usera -u pure-ftp -d /home/pureftp
$ pure-pw mkdb
# 用法
$ pure-pw list/userdel/usermod/passwd
```

编译安装配置 pure-ftpd
```bash
$ tar -xvf pure-ftpd-1.0.44.tar.bz2
$ cd pure-ftpd-1.0.44
$ ./configure \
--prefix=/usr/local/pureftpd \
--without-inetd \
--with-altlog \
--with-puredb \
--with-throttling \
--with-peruserlimits  \
--with-tls
$ make && make install 
$ mkdir /usr/local/pureftpd/etc
$ cd configuration-file/
$ cp pure-ftpd.conf /usr/local/pureftpd/etc/pure-ftpd.conf 
$ cp pure-config.pl /usr/local/pureftpd/sbin/ //启动文件
$ chmod 755 /usr/local/pureftpd/sbin/pure-config.pl 
$ vim /usr/local/pureftpd/etc/pure-ftpd.conf //删除原来的配置，重新配置
 ChrootEveryone              yes
BrokenClientsCompatibility  no
MaxClientsNumber            50
Daemonize                   yes
MaxClientsPerIP             8
VerboseLog                  no
DisplayDotFiles             yes
AnonymousOnly               no
NoAnonymous                 no
SyslogFacility              ftp
DontResolve                 yes
MaxIdleTime                 15
PureDB                        /usr/local/pureftpd/etc/pureftpd.pdb //存放用户名和密码的 密码库文件
LimitRecursion              3136 8
AnonymousCanCreateDirs      no
MaxLoad                     4
AntiWarez                   yes
Umask                       133:022
MinUID                      100 //不能映射 uid 小于 100 的
AllowUserFXP                no
AllowAnonymousFXP           no
ProhibitDotFilesWrite       no
ProhibitDotFilesRead        no
AutoRename                  no
AnonymousCantUpload         no
PIDFile                     /usr/local/pureftpd/var/run/pure-ftpd.pid
MaxDiskUsage               99
CustomerProof              yes

$ mkdir /tmp/ftpdir/（测试文件夹）
$ chown -R user1 /tmp/ftpdir/（映射 user1 ）
$ /usr/local/pureftpd/bin/pure-pw useradd ftpuser1 -uuser1 -d /tmp/fptdir/ //在 ftp 中添加用户 ftpuser1 在 ftp 中登录的用户 -uuser1 是系统用户 -d 共享文件夹
/usr/local/pureftpd/bin/pure-pw mkdb //生成密码库文件
/usr/local/pureftpd/bin/pure-pw list
/usr/local/pureftpd/bin/pure-pw userdel ftpuser1 //删除用户
 /usr/local/pureftpd/sbin/pure-config.pl /usr/local/pureftpd/etc/pure-ftpd.conf //启动 ftp 前面是脚本，后面是配置文件

$ tail /var/log/messages
```