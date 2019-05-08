## Ansible

官网 www.ansible.com

在线电子书：https://getansible.com

Ansible是一款由RedHat赞助的开源软件。它是一款可以在整个IT团队中使用的自动化语言，从系统到网络到开发。它目前已经整合了虚拟化（Vmware、RHEV、Xen等）、网络设备（思科、F5、OpenSwitch）、容器（Docker、LXC）、公有云（亚马逊云AWS、微软Azure）、DEVOPS（Gitlab、Github、Jenkins）、监控/分析（Splunk、InfluxDB）等多个领域。

## 安装配置 Ansible
> 文档：https://docs.ansible.com/ansible/latest/index.html

在 CentOS7 上安装 Ansible：
```bash
$ yum install -y epel-release
$ yum install -y ansible

$ pip install ansible
```
Ansible 因为是 agent-less，所以只有一个控制中心，其他机器无需安装任何软件包。但要想控制远程机器，还需要配置密钥认证。

1. 在控制中心生成密钥对
```bash
检查一下 ~/.ssh/，看看该目录下有没有 id_rsa 以及 id_rsa.pub 两个文件。如果没有执行如下命令
ssh-keygen -t rsa
```

2. 配置密钥认证
```bash
# 在控制中心，执行 ssh-copy-id
ssh-copy-id [user@]远程机器ip
```

3. 编辑hosts文件
```bash
# Ansible 控制中心有个 hosts 文件，用来配置它所管理的机器
$ vim /etc/ansible/hosts #格式如下
## [webservers]
## alpha.example.org
## beta.example.org
## 192.168.1.100
## 192.168.1.110

# 其中[]里面为主机组名，下面为要管理的机器IP或主机名。
```

## 测试
```bash
$ ansible all -m ping 
# 这里的all代表 hosts 文件里所有的主机，也可以指定单个
$ ansible test2 -m ping
```

#### 远程执行命令
```bash
$ ansible test -m command -a 'hostname'
#这里的test为hosts配置文件中配置的那个主机组名，-m后面指定模块名，这里的command为远程执行命令的模块，-a后面跟具体的命令

$ ansible 127.0.0.1 -m shell -a 'w' 
#除了command模块外，也可以使用shell模块实现远程执行命令。shell还支持执行远程主机上的shell脚本。
```

#### 拷贝文件或目录
```bash
$ ansible aminglinux02 -m copy  -a "src=/etc/passwd dest=/tmp/test123 owner=root group=root mode=0755"
#注意：源目录会放到目标目录下面去，如果目标指定的目录不存在，它会自动创建。如果拷贝的是文件，dest指定的名字和源如果不同，并且它不是已经存在的目录，相当于拷贝过去后又重命名。但相反，如果desc是目标机器上已经存在的目录，则会直接把文件拷贝到该目录下面。
 
$ ansible test -m copy -a "src=/etc/passwd dest=/tmp/123"
#这里的/tmp/123和源机器上的/etc/passwd是一致的，但如果目标机器上已经有/tmp/123目录，则会再/tmp/123目录下面建立passwd文件
```

#### 远程执行脚本
```bash
$ 首先创建一个shell脚本，/tmp/1.sh，内容如下
$ vim /tmp/1.sh
#!/bin/bash
echo $(date) > /tmp/ansible_test.txt
cat ansible_test.txt

# 然后把该脚本分发到各个机器上

$ ansible test -m copy -a "src=/tmp/1.sh dest=/tmp/test.sh mode=0755"
# 最后是批量执行该shell脚本

$ ansible test -m shell -a "/tmp/test.sh"
# shell模块，还支持远程执行命令并且带管道

$ ansible test -m shell -a "cat /etc/passwd|wc -l "
```

#### 管理任务计划
```bash
$ ansible test -m cron -a "name='test cron' job='/bin/touch /tmp/1212.txt'  weekday=6"

#若要删除该cron 只需要加一个字段 state=absent，前提是之前用ansible增加的那个cron没有手动修改过。 
$ ansible test -m cron -a "name='test cron' state=absent"

#其他的时间表示：分钟 minute 小时 hour 日期 day 月份 month
#示例
$ ansible aminglinux02 -m cron -a "name='test cron again' job='/bin/bash /usr/local/sbin/123.sh' minute=*/10 hour=10-20 day=5,10,15"
```

#### 软件、服务管理
```bash
$ ansible test -m yum -a "name=httpd" 
#在name后面还可以加上state=installed/removed

$ ansible test -m service -a "name=httpd state=started enabled=yes" 
#注意，这里的name要和上面那个name区分开，上面的是包的名字，这里的是服务名字，如果本机没有该服务，则会报错
#在CentOS7上可以使用"systemctl list-unit-files  -t service"列出所有的服务
#state定义服务是开启还是关闭，stoped为关闭
#enabled定义服务是否开机启动
```

#### Ansible文档的使用
```bash
$ ansible-doc -l   #列出所有的模块
$ ansible-doc copy  #查看指定模块的文档
```

## ansible playbook
Ansible playbook 是将要做的所有操作汇集到一个或者几个 yaml 文件中去，其实就跟我们写 shell 脚本一样，只不过这个 playbook 有它自己的语法和规则。

好处很明显：方便维护、升级；可以反复使用；将复杂的步骤逻辑化。

#### 示例1
```bash
$ vim test.yml
---
- hosts: 192.168.6.166
  remote_user: root
  tasks:
  - name: test_playbook
    shell: touch /tmp/violet.txt

# 说明：第一行需要有三个杠，hosts 参数指定了对哪些主机进行参作，如果是多台机器可以用逗号作为分隔，
# 也可以使用主机组，在 /etc/ansible/hosts 里定义；
# user 参数指定了使用什么用户登录远程主机操作；
# tasks 指定了一个任务，其下面的 name 参数同样是对任务的描述，在执行过程中会打印出来，shell 是 ansible 模块名字

$ ansible-playbook test.yml
```

#### 示例2
```bash
$ vim create_user.yml
---
- name: create_user
  hosts: 192.168.6.166
  user: root
  gather_facts: false
  vars:
  - user: "test"

  tasks:
  - name: create user
    user: name="{{ user }}"

# 说明：name 参数对该 playbook 实现的功能做一个概述，后面执行过程中，会打印 name 变量的值，可以省略；
# gather_facts 参数指定了在以下任务部分执行前，是否先执行 setup 模块获取主机相关信息，这在后面的 task 会使用到 setup 获取信息时用到  <ansible 127.0.0.1 -m setup>；
# vars 参数指定了变量，这里指定一个 user 变量，其值为 test ，需要注意的是，变量值一定要用引号引住；
# user 指定了调用 user 模块，name 是 user 模块里的一个参数，而增加的用户名字调用了上面 user 变量的值。
```

#### playbook 中的循环
```bash
$ vim while.yml
---
- hosts: test
  user: root
  tasks:
    - name: change mode for files
      file: path=/tmp/{{ item }} mode=600
      with_items:
        - 1.txt
        - 2.txt
        - 3.txt

#说明: with_items为循环的对象
```

#### playbook 中的条件判断
```bash
$ vim when.yml
---
- hosts: test
  user: root
  gather_facts: True
  tasks:
    - name: use when
      shell: touch /tmp/when.txt
      when: ansible_ens33.ipv4.address == "192.168.222.128"

#说明：这里的ansible_ens33.ipv4.address就是通过setup模块查看到的facter信息
```

#### playbook 中的 handlers
```bash
# 执行task之后，服务器发生变化之后要执行的一些操作，比如我们修改了配置文件后，需要重启一下服务
$ vim handlers.yml
---
- name: handlers test
  hosts: aminglinux02
  user: root
  tasks:
    - name: copy file
      copy: src=/etc/passwd dest=/tmp/aaa.txt
      notify: test handlers
  handlers:
    - name: test handlers
      shell: echo "111111" >> /tmp/aaa.txt

#说明，只有copy模块真正执行后，才会去调用下面的handlers相关的操作。这种比较适合配置文件发生更改后，重启服务的操作
```