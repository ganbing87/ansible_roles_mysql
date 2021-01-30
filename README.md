# 项目说明
- 通过ansbile一键自动化部署mysql服务，项目其包含两大功能：初始化系统操作（如关闭防火墙及selinux、安装jdk、修改ulimit参数、修改sysctl参数等）、安装mysql(单点模式或主从模式)。
- 适用平台centos、redhat系统。
- 本项目是基于ansible roles角色的结构来实现的。

# ansible roles说明
  角色（roles）是ansible自1.2版本开始引入的新特性，用于层次性，结构化地组织playbook.  
  
  roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。要使用roles只需要在playbook中使用include指令即可。简单的说，roles就是通过分别将变量、文件、任务、模块及处理器放置于单独的目录中、并可以便捷地include他们的一种机制。角色一般用于基于主机构建服务的场景中、但也可以是用于构建守护进程等场景中。  
  
> 关于roles，请参考：https://www.cnblogs.com/yanjieli/p/10971862.html

# 目录结构说明
```
ansible_roles_nexus/
├── group_vars      #全局变量配置
│   └── all.yml
├── inventories     #主机清单
│   └── hosts
├── nexus.yml       #执行的nexus安装文件
├── README.md
└── roles   #将执行流程分为多个role，分别完成相应功能
    ├── env_init    #系统初始化任务，将分别完成创建普通用户、安装jdk、修改dns、关闭防火墙等任务
    │   ├── createdir          #创建安装目录、安装包存放目录
    │   │   └── tasks
    │   │       └── main.yml
    │   ├── createuser         #创建普通用户
    │   │   └── tasks
    │   │       └── main.yml
    │   ├── jdk                #安装jdk
    │   │   ├── defaults
    │   │   │   └── main.yml
    │   │   └── tasks
    │   │       ├── install.yml 
    │   │       └── main.yml
    │   ├── modify_resolv.conf  #修改dns服务器信息
    │   │   └── tasks
    │   │       └── main.yml
    │   ├── python-virtualenv   #设置Python虚拟环境
    │   │   └── tasks
    │   │       └── main.yml
    │   ├── system              #关闭防火墙、selinux，配置sysctl.conf、ulimit文件
    │   │   ├── firewalld
    │   │   │   └── tasks
    │   │   │       └── main.yml
    │   │   ├── selinux
    │   │   │   └── tasks
    │   │   │       └── main.yml
    │   │   ├── sysctl
    │   │   │   └── tasks
    │   │   │       └── main.yml
    │   │   └── ulimit
    │   │       └── tasks
    │   │           └── main.yml
    │   └── tasks
    │       └── main.yml
    │── mysql         #安装mysql执行脚本
        ├── defaults
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   ├── configure.yml
        │   ├── install.yml
        │   ├── main.yml
        │   ├── master_slave_replication.yml
        │   ├── setup.yml
        │   ├── start.yml
        │   └── uninstall.yml
        └── templates
            ├── my.cnf.j2
            ├── mysqld.service.j2
            └── mysql.server.j2
```
 
***

# 安装步骤
以下安装以centos 7为例进行操作，以下操作均在ansible机器上操作.  
## 1、准备ansilbe环境
### 1.1 安装ansible
```
#备份旧yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

#下载新的CentOS-Base.repo 到/etc/yum.repos.d/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all

#安装依赖包和工具包
yum -y python-setuptools python2-pip bash-completion bind-utils net-tools iproute telnet curl wget nmap vim jq gettext e2fsprogs

#安装ansible
yum -y install ansible
```

### 1.2 安装community.general模块
```
#安装ansible通用模块，这是在线安装，其作用是后续如果用到一些额外的模块，就需要用到此模块组，这是提前把它安装准备好.
ansible-galaxy collection install community.general
```
安装完成后，会把模块生成到/root/.ansible/collections/ansible_collections目录下.  
模块安装参说明，参考相关网站：
  - https://docs.ansible.com/ansible/latest/collections/community/general/
  - https://www.cnblogs.com/lisenlin/p/10919068.html


## 2、创建安装包目录、上传软件包
```
# mkdir -p /data/packages

# ll
-rw-r--r--. 1 root root  43834194 1月  22 20:10 mysql-5.7.30-el7-x86_64.tar.gz
-rw-r--r--. 1 root root 194042837 1月  22 20:12 jdk-8u202-linux-x64.tar.gz
-rw-r--r--. 1 root root  34632262 1月  22 20:12 python-virtualenv.tar.gz
[root@localhost packages]# 
```

> 软件包请联系笔者下载（QQ 312683629）

## 3、下载此项目
```
git clone https://github.com/ganbing87/ansible_roles_consul.git
```

## 4、修改ansible.cfg
修改host_key_checking = False ，其作用是取消对此的注释以禁用SSH密钥主机检查，如果ansbile管理主机和被管理端没有做ssh_key信任，把此参数打开，可以用密码进行管理主机
```
# vim /etc/ansible/ansible.cfg 
host_key_checking = False
```

## 5、修改inventories/hosts文件
```
# vim inventories/hosts
[mysql]
mysql-master ansible_host=192.168.245.129 ansible_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
#mysql-slave  ansible_host=192.168.245.130 ansible_port=22 ansible_ssh_user=root ansible_ssh_pass=123456 
```
说明：修改mysql主机清单信息（如安装consul服务器的IP、端口、ssh密码等）,如果只安装单节点mysql，则修改mysql-master主机这一行信息，如果安装mysql主从，则需要把mysql-slave这一行配置打开。只修改IP、端口、ssh用户即可，mysql-master、mysql-slave这两个词不要修改。

## 6、修改全局变量文件all.yml
```
# vim group_vars/all.yml
#--------------------Global Configuration-------------------------
install_dir: "/export/server"     # 被管理主机，安装程序软件的目录
softlink_dir: "/export"           # 被管理主机，软件程序软链接目录
local_packages_dir: "/data/packages"        # ansilbe主机，本地安装包上传目录，需要手动创建此目录
remote_packages_dir: "/export/packages"     # 被管理主机，安装包存放目录
java_home: "{{ softlink_dir }}/jdk"         # 被管理主机，java软链接目录
ssh_user: "root"                # 用于ansible主机ssh到被管理主机的用户
appuser: "zhangsan"             # 用于自动在被管理机器上创建的普通用户
limit_nofile: "65536"           # 用于下面system configuratio配置中引用的变量
limit_nproc: "131072"           # 用于下面system configuratio配置中引用的变量
domain: "ganbing.cnn"           # 定义一个测试的一级域名


#--------------------jdk configuration-------------------------
jdk8_version: "202"   # jdk的版本号，需要核实下jdk包的版本是否和这一致

#---------------------Mysql Configuration---------------------------
install_mysql: true                                 #是否安装mysql的选项，true和false
mysql_port: 3306                                    #mysql端口
mysql_data_path: /export/data/mysql/data/3306/data  #mysql数据库存放路径
mysql_log_path: "{{ softlink_dir }}/mysql/logs"     #mysql 日志存放路径
MYSQL_SOCKET_PATH: /var/lib/mysql                   #mysql scoket路径
mysql_passwd: root@123                              #设置mysql root密码
mysql_user: root                                    #默认mysql登录用户
mysql_fqdn_hostname: mysql-master           #mysql默认域名，如果把mysql配置域名，则无需关注此参数

#--------------------system configuration-------------------------
kernel_config:
   - name: net.ipv4.ip_forward
     value: 1
   - name: net.ipv4.tcp_tw_recycle
     value: 0
   - name: vm.swappiness
     value: 0
   - name: vm.overcommit_memory
     value: 1
   - name: vm.panic_on_oom
     value: 0
   - name: vm.max_map_count
     value: 262144
   - name: fs.inotify.max_user_instances
     value: 8192
   - name: fs.inotify.max_user_watches
     value: 1048576
   - name: fs.file-max
     value: 52706963
   - name: net.ipv6.conf.all.disable_ipv6
     value: 1
   - name: net.ipv6.conf.default.disable_ipv6
     value: 1
   - name: net.ipv6.conf.lo.disable_ipv6
     value: 1

ulmit_config:
  - limit_domain: "*"
    limit_type: "soft"
    limit_item: "nofile"
    limit_value: "{{ limit_nofile }}"
  - limit_domain: "*"
    limit_type: "hard"
    limit_item: "nofile"
    limit_value: "{{ limit_nofile }}"
  - limit_domain: "*"
    limit_type: "soft"
    limit_item: "nproc"
    limit_value: "{{ limit_nproc }}"
  - limit_domain: "*"
    limit_type: "hard"
    limit_item: "nproc"
    limit_value: "{{ limit_nproc }}"

```

## 7、执行ansible-playbook命令
说明：执行命令前，先检查mysql 3306端口是否被占用，如被占用，需杀掉相关进程，或者修改all.yml中的nexus端口参数.  
```
ansible-playbook -i inventories/hosts  mysql.yml  -v
```


## 8、验证运行结果
- 验证系统参数
进入consul服务器通过以下方式进行结果验证：
```
# 查看/etc/sysctl.conf文件，是否写入了相关配置
cat /etc/sysctl.conf

# 查看limits.conf文件，是否写入了相关配置
cat /etc/security/limits.conf

# 查看selinux是否禁用
getenforce

# 查看防火墙是否关闭
systemctl status firewalld
```

- 验证mysql服务是否正常运行
```
ps -ef |grep mysql
netstat -ntpl |grep 3306
mysql -uroot -proot@123
```


### 小结
如果你喜欢此项目，并且它对你确实有帮助，欢迎给我打赏一杯咖啡哈~  
您的支持将是我继续开源的动力，感谢提出宝贵的意见!  

![支付宝收款码.jpg](https://sm.ms/image/MaFUGs6yx41b32C)