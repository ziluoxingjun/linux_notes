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
 JAVA_HOME=/usr/local/jdk1.8.0_111
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
> 8009：第三方服务调用端口，比如 httpd 和 tomcat 结合时会用到
>
> 8005：管理端口


## 3、配置 tomcat 监听端口为 80
```bash
$ vim /usr/local/tomcat/conf/server.xml
<Connector port="80" protocol="HTTP/1.1" //改为 80
    
$ /usr/local/tomcat/bin/shutdown.sh
$ /usr/local/tomcat/bin/startup.sh
   
$ netstat -lnp|grep java //80端口
# 浏览器 192.168.95.145 //不用输入 80
```


## 4、配置 tomcat 的虚拟主机
> <Host> 和 </Host> 之前的配置为虚拟主机配置部分，name 定义域名，appBase 定义应用的目录，Java 的应用通常是一个 jar 格式的压缩包，只要将压缩包放到  appBase 目录下即可。

```bash
$ vim /usr/local/tomcat/conf/server.xml
最后一个 Host 下 添加：
<Host name="www.xtest.com" appBase="/data/tomcatweb"
            unpackWARs="true" autoDeploy="true"
            mlValidation="false" xmlNamespaceAware="false">
    <Context path="" docBase="/data/www/test.com/" debug="0" reloadable="true" crossContext="true"/>
</Host>
    
$ curl -xlocalhost:80 www.xtest.com/1.txt（可以访问到文件）
    vim 2.jsp 
    <html>
       <body>
           <center>
               Now time is:<%=new java.util.Date() %>
           </center>
       </body>
    </html>
    
$ curl -xlocalhost:80 www.xtest.com/2.jsp
```
> appBase : 应用存放目录，需要把 war 包直接放入该目录，会自动解压为一个程序目录

> unpackWARs : 自动解压

> docBase : 定义网站的文件存放路径，和 appBase 二选一，另一个留空，如果定义了此参数就默认以该目录为主


## 5、java 网站
> 通过实例，部署一个 java 网站
```bash
$ cd /usr/local/src/
$ http -d http://dl.zrlog.com/release/zrlog-1.10.0-f3c522d-release.war
$ mv zrlog-1.10.0-f3c522d-release.war /usr/local/tomcat/webapps/
$ cd /usr/local/tomcat/webapps/
$ mv zrlog-1.10.0-f3c522d-release zrlog
# 浏览器 ip:8080/zrlog/install
$ mysql -uroot -p
mysql> create database zrlog;
mysql> grant all on arlog.* to zrlog@127.0.0.1 identified by 'password'
    
$ mv /usr/local/tomcat/webapps/zrlog/* /data/www/log.com/
# 浏览器 ip:8080 直接访问
```


## 6、tomcat 日志

    $ ls /usr/local/tomcat/logs

catalina 开头的日志为 tomcat 的综合日志，它记录 tomcat 服务相关信息，也会记录错误日志。

catalina.xxxx-xx-xx.log 和 catalina.out 内容相同，前者会每天生成一个新的日志。

host-manager 和 manager 为管理相关的日志，前者为虚拟主机的管理日志。

localhost 和 localhost_access 为虚拟主机相关日志，前者为默认虚拟主机的错误日志，后者为访问日志。

访问日志不会自动生成，需要在 server.xml 中配置一下。

    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
           prefix="log.com_access" suffix=".log"
           pattern="%h %l %u %t &quot;%r&quot; %s %b" />

prefix 访问日志的前缀

suffix 访问日志的后缀

pattern 访问日志的格式

错误日志会统一记录在 catalina.out 中，出现问题时，第一时间要想到查看它。
