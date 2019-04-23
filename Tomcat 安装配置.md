## Tomcat 介绍
> java 程序写的网站用 tomcat + jdk 来运行

> tomcat 是一个中间件，是 webserver,依赖 jdk，真正起作用解析 java 脚本的是 jdk

> jdk(java development kit) 是整个 java 的核心，它包含了 java 运行环境和一堆 java 相关的工具以及 java 基础库

## 1、下载安装 jdk
> jdk 版本 1.6 1.7 1.8

> https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

```bash
$ cd /usr/local/src/
$ wget http://download.oracle.com/otn-pub/java/jdk/8u181-b13/96a7b8442fe848ef90c96a2fad6ed6d1/jdk-8u181-linux-x64.tar.gz?AuthParam=1538813426_02ed11d13fc1a3ea7e89c540bd596435
$ tar zxvf jdk-8u181-linux-x64.tar.gz
$ mv jdk1.8.0_181 /usr/local/jdk1.8
# $ vim /etc/profile.d/java.sh
$ vim /etc/profile
 JAVA_HOME=/usr/local/jdk1.8
 JAVA_BIN=/usr/local/jdk1.8/bin
 JRE_HOME=/usr/local/jdk1.8/jre
 PATH=$PATH:/usr/local/jdk1.8/bin:/usr/local/jdk1.8/jre/bin
 CLASSPATH=/usr/local/jdk1.8/jre/lib:/usr/local/jdk1.8/lib:/usr/local/jdk1.8/jre/lib/charsets.jar

$ source /etc/profile
# $ . /etc/profile.d/java.sh //初始化；source 也可以
$ java -version //验证是否生效
```

## 2、下载安装 tomcat
> http://tomcat.apache.org/download-80.cgi

```bash
$ wget http://mirrors.shu.edu.cn/apache/tomcat/tomcat-8/v8.5.34/bin/apache-tomcat-8.5.34.tar.gz
$ tar zxvf apache-tomcat-8.5.34.tar.gz
$ mv apache-tomcat-8.5.34 /usr/local/tomcat
$ /usr/local/tomcat/bin/startup.sh //启动

#cp -v /usr/local/tomcat/bin/catalina.sh /etc/init.d/tomcat
#chmod 755 /etc/init.d/tomcat
#chkconfig --add tomcat //服务不支持 chkconfig 配置写的不符合规范
$ vim /etc/init.d/tomcat //加入以下几行
 # chkconfig: 2345 63 37 //第63个启动，第37个关闭
 . /etc/init.d/functions
 JAVA_HOME=/usr/local/jdk1.8
 CATALINA_HOME=/usr/local/tomcat //tomcat 的家目录

$ chkconfig --add tomcat
$ chkconfig tomcat on
$ service tomcat start //不支持 restart 只能 stop 再 start
# 浏览器 192.168.1.11:8080

$ ps aux | grep java
$ netstat -lntp | grep java
```

> 3 个 端口:
>
> 8080：提供 web 服务的端口
>
> 8009：AJP端口 第三方的应用连接这个接口，和 Tomcat 结合起来，第三方服务调用端口，比如 httpd 和 tomcat 结合时会用到
>
> 8005：管理端口 shutdown


## 4、tomcat 启动脚本
```bash
$ vim /etc/init.d/tomcat
$ chmod 755 /etc/init.d/tomcat
$ chkconfig --add tomcat
$ chkconfig tomcat on
```
```bash
#!/bin/bash 
# 
# tomcat startup script for the Tomcat server 
# 
# chkconfig: 345 80 20 
# description: start the tomcat deamon 
# 
# Source function library 
. /etc/init.d/functions 
prog=tomcat 
JAVA_HOME=/usr/local/jdk1.8
export JAVA_HOME 
CATALINA_HOME=/usr/local/tomcat/
export CATALINA_HOME 
case "$1" in 
start) 
  echo "Starting Tomcat..." 
  $CATALINA_HOME/bin/startup.sh 
  ;; 
stop) 
  echo "Stopping Tomcat..." 
  $CATALINA_HOME/bin/shutdown.sh 
  ;; 
restart) 
  echo "Stopping Tomcat..." 
  $CATALINA_HOME/bin/shutdown.sh 
  sleep 2 
  echo 
  echo "Starting Tomcat..." 
  $CATALINA_HOME/bin/startup.sh 
  ;; 
*) 
  echo "Usage: $prog {start|stop|restart}" 
  ;; 
esac 
exit 0
```
```bash
#!/bin/bash
#
# chkconfig: - 95 15
# description: Tomcat start/stop/status script
  
JAVA_HOME=/usr/local/jdk1.8
CATALINA_HOME=/usr/local/tomcat
TOMCAT_USAGE="Usage: $0 {\e[00;32mstart\e[00m|\e[00;31mstop\e[00m|\e[00;32mstatus\e[00m|\e[00;31mrestart\e[00m}"

#SHUTDOWN_WAIT is wait time in seconds for java proccess to stop
SHUTDOWN_WAIT=20

tomcat_pid() {
    echo `ps -ef | grep $CATALINA_HOME | grep -v grep | tr -s " "|cut -d" " -f2`
}

start() {
    pid=$(tomcat_pid)
    if [ -n "$pid" ];then
        echo -e "\e[00;31mTomcat is already running (pid: $pid)\e[00m"
    else
        echo -e "\e[00;32mStarting tomcat\e[00m"
        $CATALINA_HOME/bin/startup.sh
    fi
    return 0
}

status(){
    pid=$(tomcat_pid)
    if [ -n "$pid" ];then
        echo -e "\e[00;32mTomcat is running with pid: $pid\e[00m"
    else
        echo -e "\e[00;31mTomcat is not running\e[00m"
    fi
}

stop() {
    pid=$(tomcat_pid)
    if [ -n "$pid" ];then
        echo -e "\e[00;31mStoping Tomcat\e[00m"
        $CATALINA_HOME/bin/shutdown.sh

        let kwait=$SHUTDOWN_WAIT
        count=0;
        until [ `ps -p $pid | grep -c $pid` = '0' ] || [ $count -gt $kwait ]
        do
            echo -n -e "\e[00;31mwaiting for processes to exit\e[00m\n";
            sleep 1
            let count=$count+1;
        done

        if [ $count -gt $kwait ];then
            echo -n -e "\n\e[00;31mkilling processes which didn't stop after $SHUTDOWN_WAIT seconds\e[00m"
            kill -9 $pid
        fi
    else
        echo -e "\e[00;31mTomcat is not running\e[00m"
    fi
    return 0
}

case $1 in
    start)
        start
    ;;

    stop)
        stop
    ;;

    restart)
        stop
        start
    ;;
 
    status)
        status
    ;;

    *)
        echo -e $TOMCAT_USAGE
    ;;
esac
exit 0
```

## 4、配置 tomcat 监听端口为 80
> 如果使用 nginx 代理 tomcat 监听端口可不用修改
```bash
$ vim /usr/local/tomcat/conf/server.xml
69 <Connector port="80" protocol="HTTP/1.1" //改为 80
    
$ /usr/local/tomcat/bin/shutdown.sh
$ /usr/local/tomcat/bin/startup.sh
   
$ netstat -lnp|grep java //80端口
# 浏览器 192.168.95.145 //不用输入 80
```


## 5、配置 tomcat 的虚拟主机

> \<Host\> 和 \<\/Host\> 之间的配置为虚拟主机配置部分，name 定义域名，appBase 定义应用的目录，Java 的应用通常是一个 jar 格式的压缩包，只要将压缩包放到  appBase 目录下即可。

```bash
$ vim /usr/local/tomcat/conf/server.xml
# 最后一个 Host 下 添加：
<Host name="www.ccc.com" appBase=""
      unpackWARs="true" autoDeploy="true"
      xmlValidation="false" xmlNamespaceAware="false">
    <Context path="" docBase="/data/wwwroot/ccc.com/" debug="0" reloadable="true" crossContext="true"/>
</Host>
    
$ vim /data/wwwroot/ccc.com/1.jsp
<html>
    <body>
        <center>
            Now time is:<%=new java.util.Date() %>
        </center>
    </body>
</html>
$ curl -xlocalhost:80 www.ccc.com/1.txt //可以访问到文件
$ curl -xlocalhost:80 www.ccc.com/1.jsp
```
> appBase : 应用存放目录，需要把 war 包直接放入该目录，会自动解压为一个程序目录

> unpackWARs : 自动解压

> docBase : 定义网站的文件存放路径，如果定义了此参数就默认以该目录为主,如果不定义默认是在 appBase/ROOT 下，docBase,appBase 二选一，另一个留空，也可以一样。


## 6、java 网站
> 通过实例，部署一个 java 网站
```bash
$ cd /usr/local/src/
$ wget http://dl.zrlog.com/release/zrlog-2.0.1-71c552b-release.war
$ mv zrlog-2.0.1-71c552b-release.war /usr/local/tomcat/webapps/
$ cd /usr/local/tomcat/webapps/
$ mv zrlog-2.0.1-71c552b-release zrlog
# 浏览器 ip:8080/zrlog/install
$ mysql -uroot -p
mysql> create database zrlog;
mysql> grant all on zrlog.* to 'zrlog'@127.0.0.1 identified by 'password'
$ mysql -uzrlog -ppassword -h127.0.0.1
$ mv /usr/local/tomcat/webapps/zrlog/* /data/www/ccc.com/
# 浏览器 www.ccc.com
```


## 7、tomcat 日志
```bash
$ ls /usr/local/tomcat/logs
catalina.2018-10-06.log      localhost.2018-10-06.log
catalina.out                 localhost_access_log.2018-10-06.txt
host-manager.2018-10-06.log  manager.2018-10-06.log
```

> catalina 开头的日志为 tomcat 的核心日志，它记录 tomcat 服务相关信息，也会记录错误日志。

> catalina.xxxx-xx-xx.log 和 catalina.out 内容相同，前者是 catalina 引擎相关日志，会每天生成一个新的日志。

> host-manager 和 manager 为管理相关的日志，前者为虚拟主机的管理日志。

> localhost 和 localhost_access 为虚拟主机相关日志，前者为默认虚拟主机的错误日志，后者为访问日志。主要是应用初始化(listener, filter, servlet)未处理的异常最后被 tomcat 捕获而输出的日志

> 访问日志不会自动生成，需要在 server.xml 中配置一下。

```bash
$ vim /usr/local/tomcat/conf/server.xml
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="ccc.com_access_log" suffix=".log"
       pattern="%h %l %u %t &quot;%r&quot; %s %b" />
```
> prefix 访问日志的前缀  
> suffix 访问日志的后缀  
> pattern 访问日志的格式  
> 错误日志会统一记录在 catalina.out 中，出现问题时，第一时间要想到查看它。  
> 日志配置文件 onf/logging.properties，主要定义了非访问日志的属性，比如日志路径、哪些日志记录到哪个文件（名字）、日志级别、存储周期等信息，这个配置文件我们一般都不会更改，保持默认即可。

### tomcat 日志切割
```bash
$ vim /etc/logrotate.d/tomcat  //写入如下内容
/usr/local/tomcat/logs/catalina.out  {
    copytruncate    # 创建新的catalina.out副本后，截断源catalina.out文件
    daily    # 每天进行catalina.out文件的轮转
    rotate 7     # 至多保留7个副本
    missingok    # 如果要轮转的文件丢失了，继续轮转而不报错
    compress     # 使用压缩的方式（节省硬盘空间）
    size 16M     # 当catalina.out文件大于16MB时，就轮转
}

# 定时清空日志
$ crontab -e
0 0 */5 * * echo "" > /usr/local/tomcat/logs/catalina.out
```


## 8、Nginx 反向代理 Tomcat
为什么要为Tomcat配置反向代理？
1. 同一台机器同时开启 Nginx 和 Tomcat，则会产生端口冲突。
2. 需要把 8080 端口变成 80 端口。
3. Nginx 对于静态的请求速度上要优于 Tomcat，Tomcat 不擅长做高并发的静态文件请求处理。
```bash
$ vim /usr/local/nginx/conf/vhosts/zr.org.conf
server {
    server_name zr.org;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
# 在 windows hosts 里配置 192.168.6.165 zr.org 即可在浏览器访问 zr.org/zrlog
```

## 9、Tomcat的管理功能
### host-manager //管理虚拟主机管理
> 这个功能主要用来管理虚拟主机的，可以通过这个WEB界面，来停止、启动以及增加虚拟主机。
> 在浏览器访问 192.168.6.165,右方灰色按钮：Host Manager 或者 http://192.168.6.165/host-manager/html
> 进入页面会显示 403 Access Denied，需要修改配置文件
```bash
$ vim conf/tomcat-user.xml
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<user username="tomcat" password="s3cret" roles="admin-script,admin-gui"/>

$ vim webapps/host-manager/META-INF/context.xml 
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.6\.\d+*" //在allow那一行增加白名单IP，如果是网段用*表示，例如192.168.6.*
```
> admin-gui 和 admin-script 是 Tomcat 内置的角色，host-manager 功能需要这两个角色的支持。
- admin-gui - allows access to the HTML GUI
- admin-script - allows access to the text interface

> Tomcat 有一个安全设置，默认不允许这个客户端 IP 访问 host-manager 页面

> 增加 virtualhost，会在 conf/Catalina/ 目录下生成一个目录，目录永久存在，但 virtualhost 临时的，重启服务后会消失，如让其永久保存到server.xml，需要在 server.xml 里增加配置
```bash
$ vim conf/server.xml
33 <Listener className="org.apache.catalina.storeconfig.StoreConfigLifecycleListener"/>
```

### Manager //部署
> 在浏览器访问 192.168.6.165,右方灰色按钮：Server Status 或者 http://192.168.6.165/manager/status  
> 进入页面会显示 403 Access Denied，需要修改配置文件
```bash
$ vim conf/tomcat-user.xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>
<user username="tomcat" password="s3cret" roles="manager-gui,manager-script,manager-jmx,manager-status"/>

$ vim webapps/manager/META-INF/context.xml
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.6\.*" />
```
- manager-gui - allows access to the HTML GUI and the status pages
- manager-script - allows access to the text interface and the status pages
- manager-jmx - allows access to the JMX proxy and the status pages
- manager-status - allows access to the status pages only