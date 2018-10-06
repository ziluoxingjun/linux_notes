## Tomcat 介绍
> java 程序写的网站用 tomcat + jdk 来运行

> tomcat 是一个中间件，真正起作用解析 java 脚本的是 jdk

> jdk(java development kit) 是整个 java 的核心，它包含了 java 运行环境和一堆 java 相关的工具以及 java 基础库

## 1、下载安装 jdk
```bash
$ wget http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz?AuthParam=1483976568_18c5c534b849303f998fc0d075e1d691
$ tar xvf jdk-8u111-linux-x64.tar.gz 
$ mv jdk1.8.0_111 /usr/local/jdk1.8
$ vim /etc/profile.d/java.sh
$ vim /etc/profile
 JAVA_HOME=/usr/local/jdk1.8
 JAVA_BIN=/usr/local/jdk1.8/bin
 JRE_HOME=/usr/local/jdk1.8/jre
 PATH=$PATH:/usr/local/jdk1.8/bin:/usr/local/jdk1.8/jre/bin
 CLASSPATH=/usr/local/jdk1.8/jre/lib:/usr/local/jdk1.8/lib:/usr/local/jdk1.8/jre/lib/charsets.jar

$ source /etc/profile
$ . /etc/profile.d/java.sh （初始化；source 也可以）
$ java -version（验证是否生效）
```
## 2、下载安装 tomcat