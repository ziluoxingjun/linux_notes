
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