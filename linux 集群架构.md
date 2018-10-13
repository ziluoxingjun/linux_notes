# Linux 集群

#### 集群从功能划分两大类：高可用和负载均衡

- 高可用集群通常为两台服务器，一台工作，另一台作为冗余，当提供服务的机器宕机，另一台则接替继续提供服务。
- 负载均衡集群需要一台服务器作为分发器，它负责把用户的请求分发给后端的服务器处理，后端服务器至少为两台。

> 实现高可用的开源软件：heartbeat , keepalived

> 实现负载均衡的开源软件：LVS , keepalived , haproxy , nginx, 商业软件有 F5 , Netscaler

## 1.1 HA 集群配置（heartbeat 弃用）

HA = High Availability

在主从机上：

    iptables -F
    iptables -nvL
    setenforce 0
    getenforce
    vim /etc/hosts
     192.168.1.11 star
     192.168.1.10 xing 
    
    yum install -y heartbeat
    yum install -y libnet

配置

    cd /usr/share/doc/heartbeat-3.0.4
    cp authkeys ha.cf haresources /etc/ha.d/
    cd /etc/ha.d/
    vim authkeys
     #auth 3
     #1 crc（最简单）
     #2 sha1 HI!（最复杂）
     #3 md5 Hello!
    
    chmod 600 authkeys 
    vim haresources
     #node-name resource1 resource2 ... resourceN
     star 192.168.1.100/24/eth0:0 nginx
    
    vim ha.cf
    debugfile       /var/log/ha-debug
    logfile         /var/log/ha-log
    logfacility     local0
    keepalive       2（多长时间探测一次，需要知道是否活着，2s）
    deadtime        30（30s 不通 死掉）
    warntime        10（10s 发出警告，记入logfile）
    initdead        60（重启需要预留出时间）
    udpport         694（心跳线通信端口）
    ucast eth0      192.168.1.10
    auto_failback   on（当主再次激活的时候，备自动沉寂）
    node            star（主）
    node            xing（从）
    ping            192.168.1.1
    respawn hacluster /usr/lib64/heartbeat/ipfail（此脚本用来检测网络连通性；respawn 当此脚本死掉时，自动拉起）
    
    yum install -y openssh-clients
    scp authkeys ha.cf haresources xing:/etc/ha.d/
    在从机上 authkeys haresources 不需要改，需要改 ha.cf
    vim ha.cf
     ucast eth0      192.168.1.11
    
    测试
    iptables -A INPUT -p icmp -j DROP

同一块网卡可以有多个 IP 地址

    cd /etc/sysconfig/network-scripts/
    cp ifcfg-eth0 ifcfg-eth0\:1
     
    DEVICE=eth0:1
    #HWADDR=00:0C:29:40:CC:5F
    TYPE=Ethernet
    #UUID=760e05fd-db50-4396-99a0-c6df923b06f5
    ONBOOT=yes
    NM_CONTROLLED=yes
    BOOTPROTO=static
    IPADDR=192.168.1.111
    NETMASK=255.255.255.0
    #GATEWAY=192.168.1.1
    #DNS1=192.168.1.1
    #DNS2=8.8.8.8
    
    service network restart
    ifconfig

## 1.2 keepalived 配置 HA

> keepalived 通过 VRRP (Virtual Router Redundancy Protocol 虚拟路由冗余协议) 来实现高可用，防止单点故障；在这个协议里会将多台功能相同的路由器组成一个小组，这个小组里会有一个 master 和 n(n>=1) 个 backup

> master 会通过组播的形式向各个 backup 发送 vrrp 协议的数据包，当 backup 接收不到 master 发来的 vrrp 数据包时就会认为 master 宕机了。此时就要根据 各个 backup 的优先级来决定谁成为新的 master

#### keepalived 有三个模块：
- core：核心模块，负责主进程的启动，维护以及全局配置文件的加载和解析
- check：负责健康检查
- vrrp：实现 VRRP 协议

```bash
# 准备两台机器 11 12，11作为 master，12作为 backup
$ yum install keepalived
$ yum install nginx // 11已编译安装
# yum 安装 nginx 路径：/usr/share/nginx
# 编辑配置文件
$ vim /etc/keepalived/keepalived.conf
global_defs {
   notification_email {
        admin@admin.com
   }   
   notification_email_from root@admin.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_script chk_nginx {
    script "/usr/local/sbin/check_nginx.sh"
    interval 3
}

vrrp_instance VI_1 {
    state MASTER //从为 BACKUP
    interface ens33
    virtual_router_id 51 //主从要一致
    priority 100 //从为90 权重
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass zilo>com
    }
    virtual_ipaddress {
        192.168.95.100 //主从一致
    }   
    track_script {
        chk_nginx
    }   
}
# 监控脚本
$ vim /usr/local/sbin/check_nginx.sh
#!/bin/bash
#时间变量，用于记录日志
d=`date --date today +%Y%m%d_%H:%M:%S`
#计算nginx进程数量
n=`ps -C nginx --no-heading|wc -l`
#如果进程为0，则启动nginx，并且再次检测nginx进程数量，
#如果还为0，说明nginx无法启动，此时需要关闭keepalived
if [ $n -eq "0" ]; then
    /etc/init.d/nginx start
    n2=`ps -C nginx --no-heading|wc -l`
    if [ $n2 -eq "0"  ]; then
        echo "$d nginx down,keepalived will stop" >> /var/log/check_nginx.log
        systemctl stop keepalived
    fi
fi
$ chmod 755 /usr/local/sbin/check_nginx.sh
$ systemctl start keepalived
$ vim /var/log/messages #日志
# ip add 看 vip,ifconfig 看不到
$ ip add
# 配置完成，服务启动，检查防火墙，可以在浏览器中分别访问主ip 从ip vip 

测试:
test1:关闭 master 上的 nginx 服务
test2:在 master 上增加 iptables 规则：iptables -I OUTPUT -p vrrp -j DROP
test3:关闭 master 上的 keepalived 服务
test4:开启 master 上的 keepalived 服务
```


2、LB 集群之 LVS 介绍

LB ＝ Load Balancing （负载均衡）

10000/s （10000请求量，并发） 每台机器 1000/s *10

- nginx 7层（应用层） lvs 4层（网络层 网络OSI7层模型） keeplived 的负载均衡其实就是 lvs
- lvs 这种4层的负载均衡可以分发除80外的其它端口通信，如 mysql ,haproxy 也支持 mysql 这种
- nginx 仅仅支持 https https mail
- 相比较来说 lvs 这种4层的更稳定，能承受更多的请求，而 nginx 这种7层的更加灵活，能实现更多的个性化需求

LVS ＝Linux Virtual Server 由国人章文嵩开发，流行度不亚于 httpd ，基于tcp/ip 做路由转发，稳定性和效率都很高。

LVS的三种转发模式

LVS-NAT：网络地址转换 Network address translation

LVS-DR：直接路由 Direct routing

LVS-TUN：IP隧道 IP tunneling

lvs 架构中有一个核心角色叫做分发器（load balancer），用来分发用户请求，还有诸多处理用户请求的服务器（Real Server,rs）

LVS 的算法

- 轮询 Round-Robin rr
- 加权轮询 Wight Round-Robin wrr
- 最小连接 Least-Connection lc
- 加权最小连接 Weight Least-Connection wlc
- 基于局部性的最小连接 Locality-Based Least Connecicon lblc
- 带复制的基于局部性最小连接 Locality-Based Least Connection with Replication lblcr
- 目标地址散列调度 Destination Hashing dh
- 源地址散列调度 Source Hashing sh

LVS 的 NAT 模式

- 这种模式借助 iptables 的 nat 表实现，用户的请求到分发器后，通过预设的 iptables 规则，把请求的数据包转发到后端的 rs 上。
- rs 需要设定网关为分发器的内网 ip，用户请求的数据和返回给用户的数据包全部经过分发器，所以分发器成为瓶颈。一般10台左右，不能太多，否则影响效率。
- 在 nat 模式中，只需要分发器有公网 ip 即可，所以比较节省公网 ip 资源。

    # director 内网 ip : 95.13 外网 ip : 106.128
    # rs1 ip : 95.11
    # rs2 ip : 95.12
    
    NAT
    hostname director
    $ yum install iptables-services
    $ systemctl stop firewalld
    $ systemctl disable firewalld
    $ systemctl start iptables
    $ iptables -F && services iptables save
    yum install -y ipvsadm
    vim /usr/local/sbin/lvs_nat.sh
     #! /bin/bash
     # director 服务器上开启路由转发功能：
     echo 1 > /proc/sys/net/ipv4/ip_forward
    
     # 关闭 icmp 的重定向
     echo 0 > /proc/sys/net/ipv4/conf/all/send_redirects
     echo 0 > /proc/sys/net/ipv4/conf/default/send_redirects
     # 注意网卡名字
     echo 0 > /proc/sys/net/ipv4/conf/ens33/send_redirects
     #echo 0 > /proc/sys/net/ipv4/conf/ens37/send_redirects
    
     # director 设置 nat 防火墙
     iptables -t nat -F
     iptables -t nat -X //清空所有的链
     iptables -t nat -A POSTROUTING -s 192.168.95.0/24 -j MASQUERADE //某网段的内网可以上网
    
     # director 设置 ipvsadm
     IPVSADM='/usr/sbin/ipvsadm'
     $IPVSADM -C
    # $IPVSADM -A -t 192.168.106.128:80 -s rr（rr 为算法）
    # $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.11:80 -m
    # $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.12:80 -m（-m 代表 NAT 模式）
    # $IPVSADM -A -t 192.168.106.128:80 -s wrr（rr 为算法）
    # $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.11:80 -m -w 2（w weight 权重 为 1）
    # $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.12:80 -m -w 1
      $IPVSADM -A -t 192.168.106.128:80 -s wlc -p 3 // -p 超时时间
      $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.11:80 -m -w 1
      $IPVSADM -a -t 192.168.106.128:80 -r 192.168.95.12:80 -m -w 1
    sh /usr/local/sbin/lvs_nat.sh 
    ipvsadm -ln
    
    hostname rs1
    hostname rs2
    vim /etc/sysconfig/network-scripts/ifcfg-eth0 
     GATEWAY=192.168.1.147（网关设置为 director 的 ip）

LVS 的 DR 设置（用的比较多）

- 需要有一个公共的 ip 配置在分发器上和所有 rs 上，也就是 vip
- 和 ip tunnel 不同，它会把数据包的 mac 地址改为 rs 的 mac 地址，这样数据包就到了 rs 上
- rs 接收到数据包后会还原原始数据包，里面有源 ip，直接通过公网 ip 返回到客户端，不经过分发器

    # director ip : 95.13
    # rs1 ip : 95.11
    # rs2 ip : 95.12
    # vip 95.200
    
    # DR diretor
    先清空：
    ipvsadm -C
    iptables -t nat -F
    
    将 rs1 rs1 gateway 改回
    将 eth1 down 掉
    
    需要三个 vip
    vim /usr/local/sbin/lvs_dr.sh
    
    #! /bin/bash
    echo 1 > /proc/sys/net/ipv4/ip_forward
    ipv=/usr/sbin/ipvsadm
    vip=192.168.95.200
    rs1=192.168.95.11
    rs2=192.168.95.12
    ifconfig ens33 down
    ifup ens33
    ifconfig ens33:1 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip dev ens33:1
    $ipv -C
    $ipv -A -t $vip:80 -s rr
    $ipv -a -t $vip:80 -r $rs1:80 -g -w 1 //-g 代表 dr 模式
    $ipv -a -t $vip:80 -r $rs2:80 -g -w 1
    
    sh /usr/local/sbin/lvs_dr.sh
    ipvsadm -ln
    
    # rs
    vim /usr/local/sbin/lvs_rs.sh
    #! /bin/bash
    vip=192.168.95.200
    # 把 vip 绑定在 lo 上，是为了实现 rs 直接把结果返回给客户端
    ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip lo:0
    # 以下操作为更改 arp 内核参数，目的是让 rs 顺利发送 mac 地址给客户端
    # 参考：www.cnblogs.com/lgfeng/archive/2012/10/16/2726308.html
    echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce
    
    sh /usr/local/sbin/lvs_rs.sh

LVS IP Tunnel 模式

- 需要有一个公共的 ip 配置在分发器和所有 rs 上，叫做 vip
- 客户端请求的目标 ip 为 vip，分发器接收到请求数据包后，会对数据包做一个加工，会把目标 ip 改为 rs 的 ip，数据包就到了 rs 上
- rs 接收到数据包后会还原原始数据包，里面有源 ip，直接通过公网 ip 返回到客户端，不经过分发器

3、LVS结合keepalived配置

keeplived = HA + LB

- 一般为 4 台机器，两台 director,master slave,分别安装 keepalived，作为高可用，两台 rs1 rs2
- 但 keepalived 本身有负载均衡的功能，实验时可以只安装一台 keepalived
- keepalived 内置 ipvsadm 功能，所以不需要再安装 ipvsadm 包，也不用再编写执行 lvs_dr 脚本
- keepalived 能够在一台宕机的时候，不再把请求转发过去

第一次

    在 master 上
    ipvsadm -C
    yum install -y keepalived（主从都安装）
    vim /etc/keepalived/keepalived.conf
     vrrp_instance VI_1 {
        state MASTER #备用服务器上为 BACKUP
        interface eth0
        virtual_router_id 51
        priority 100 #优先级 备用服务器上为 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.147
        }
     }
     virtual_server 192.168.1.147 80 {
        delay_loop 6 #每隔6秒查询realserver状态
        lb_algo rr #lvs 算法 
        lb_kind DR #Direct Route 模式
        nat_mask 255.255.255.0
        persistence_timeout 0 #留存时间
        protocol TCP #用 TCP 协议检查 realserver 状态
    
        real_server 192.168.1.138 80 {
            weight 100 #权重
            TCP_CHECK { 
                connect_timeout 10 #10 秒无响应超时，超时就踢除
                nb_get_retry 3
                delay_before_retry 3
                connect_port 80
            }
        }
        real_server 192.168.1.139 80 {
            weight 100
            TCP_CHECK {
                connect_timeout 10
                nb_get_retry 3
                delay_before_retry 3
                connect_port 80
            }
        }
     }
    
    拷贝到 slave 上
    scp /etc/keepalived/keepalived.conf 192.168.1.138:/etc/keepalived/keepalived.conf 
    echo 1 > /proc/sys/net/ipv4/ip_forward （从上也需要打开端口转发）

    fconfig eth0:0 down
    ip addr（ifconfig 看不到）
    /etc/init.d/keepalived start
    ipvsadm -ln

http://ask.apelearn.com/question/8071

第二次

    $ vim /etc/keepalived/keepalived.conf
    vrrp_instance VI_1 {
        #备用服务器上为 BACKUP
        state MASTER
        #绑定vip的网卡为ens33，你的网卡和我的可能不一样，这里需要你改一下
        interface ens33
        virtual_router_id 51
        #备用服务器上为90
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass aminglinux
        }
        virtual_ipaddress {
            192.168.95.200
        }
    }
    virtual_server 192.168.95.200 80 {
        #(每隔10秒查询realserver状态)
        delay_loop 10
        #(lvs 算法)
        lb_algo wlc
        #(DR模式)
        lb_kind DR
        #(同一IP的连接60秒内被分配到同一台realserver)
        persistence_timeout 0
        #(用TCP协议检查realserver状态)
        protocol TCP
    
        real_server 192.168.95.11 80 {
            #(权重)
            weight 100
            TCP_CHECK {
            #(10秒无响应超时)
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
            }
        }
        real_server 192.168.95.12 80 {
            weight 100
            TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
            }
         }
    }
    $ ipvsadm -C //把之前的规则清空
    $ systemctl restart network //可以把之前的 vip 清空
    $ systemctl start keepalived
    $ ipvsadm -ln
    ### 两台 rs 上，依然要执行 lvs_rs.sh 脚本



4、nginx 的负载均衡集群

LVS 针对 IP 做转发，nginx 可以针对 域名做转发

    cd /etc/nginx/conf/vhosts
    vim lb.conf
     upstream aming {
            #server 192.168.1.138:80;
            #server 192.168.1.139:80;
            server 192.168.1.138:80 weight=2;
            server 192.168.1.139:80 weight=1;
    
     }
    
     server {
            listen 80;
            server_name www.123.com;
            location / {
                    proxy_pass http://aming/;
                    proxy_set_header Host $host;
            }
     }
    
    ipvsadm -C
    iptables -F
    
    curl -xlocalhost www.123.com
    
    在 192.168.1.138 上 service nginx stop
    curl -xlocalhost www.123.com

根据目录来做代理：http://ask.apelearn.com/question/920

lvs无法做到
