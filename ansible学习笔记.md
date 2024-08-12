### 一、Ansible概述

#### 1、ansible功能特性

ansible基于**ssh协议**来实现管理节点于远程节点之间的通信，只要能通过ssh登录到远程主机完成的操作，都可以通过ansible实现批量自动化操作

ansible包括以下这些功能：

- 批量执行远程命令，可以对远程的多台主机同时进行命令的执行
- 批量安装和配置软件服务，可以对远程的多台主机进行自动化的方式配置和管理各种服务
- 编排高级的企业级复杂的IT架构任务，ansible的Playbook和Role可以轻松7实现大型的IT复杂架构
- 提供自动化运维工具的开发API，有很多运维工具，如jumpserver就是基于ansible 实现自动化管理功能

#### 2、ansible优缺点

**ansible优点**：

- 部署简单方便，在对多机进行批量管量时，只需要在主控机上部署ansible服务，被管理的机器不用部署
- 默认通过SSH协议进行通信，只要保证主控端和被控机SSH通道畅通，就能保证服务可用
- 配置简单，上手快，功能强大
- 用Python开发，打开源文件，所见即所得，对二次开发的支持非常友好
- 大量常规运维操作己经内置
- 对于较复杂的需求，可以通过Playbook功能和Role功能来实现
- 还提供了操作方便，功能强大的的web管理界面

**ansible缺点**：

- 在管理的主机数量较多时，比如上千台主机，性能略差，执行效率不如saltstack高
- 不支持事务回滚

#### 3、ansible工作原理

(1)ansible加载配置读取配置并加载对应的模块文件

(2)ansible会根据命令生成一个临时的py文件，并将其传输到被管理的远程主机上

(3)给远程的py文件添加可执行文件

(4)执行改py文件并且将执行结果抓取回控制端主机显示

(5)删除远端的临时py文件

#### 4、ansible使用注意事项

- 主控端的Python版本需要在2.6及以上版本
- 被控端的Python版本需要在2.4及以上版本
- Windows主机不能作为主控端，只能作为被控端
- 被控制端如果开启SELinux，需要在被控端安装libselinux-python

### 二、Ansible安装

#### 1、在Rocky9中安装ansible

(1)安装epel源

```bash
yum install -y epel-release
```

(2)安装ansible

```bash
#也可以安装ansible-core，ansible-core只包含了ansible的核心模块
yum -y install ansible
```

(3)查看ansible的版本以及配置文件信息

```bash
ansible --version
```

(3)查看主配置文件

```bash
#需要根据配置文件中的提示生成主配置文件的模板，根据下图的提示有2种方法，在ansible 2.12版本及以上可以通过ansible-config命令生成，在低版本可以通过github上ansible仓库stable分支中去找到ansible.cfg去下载
cat /etc/ansible/ansible.cfg
```

![image-20240107153359110](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20240107153359110.png)

(3)生成主配置文件的模板

```bash
#2.12版本及以上，注意生成的文件是在当前目录下的，需要放到/etc/ansible下
ansible-config init --disabled > ansible.cfg

#2.12版本以下
wget https://raw.githubusercontent.com/ansible/ansible/stable-2.9/examples/ansible.cfg
```

#### 2、在Ubuntu中安装ansible

(1)更新apt源并安装ansible

```bash
#也可以安装ansible-core，ansible-core只包含了ansible的核心模块
apt update && apt install -y ansible
```

(2)查看ansible的版本以及配置文件信息

```bash
ansible --version
```

![image-20240107194104047](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20240107194104047.png)

(3)生成主配置文件的模板

参考rokcy中生成ansible主配置文件模板

### 三、Ansible配置文件详解

#### 1、查看ansible配置文件路径

```bash
ansible --version
```

![image-20231204224010946](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231204224010946.png)

#### 2、主配置文件生效的优先级

ansible的主配置文件可以有多个，并存放在不同的路径下

```bash
#环境变量定义的主配置文件>当前目录下的主配置文件>当前用户家目录下的主配置文件>默认的主配置文件路径
ANSIBLE_CONFIG > ./ansible.cfg > ~/.ansible.cfg >  /etc/ansible/ansible.cfg
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20240107190615666.png" alt="image-20240107190615666" style="zoom:80%;" />

#### 3、主配置文件

```bash
[defaults]
inventory      = /etc/ansible/hosts         #远程主机清单
remote_tmp     = ~/.ansible/tmp             #远程主机临时文件目录
local_tmp      = ~/.ansible/tmp             #本地主机临时文件目录
keep_remote_files=False                     #是否保留远程主机上的py脚本文件，默认不保留
executable=/bin/sh                          #生成shell命令时用指定的bash执行
forks          = 5                          #并发数
ask_pass       = False                      #连接远程主机时，询问是否需要输入密码，默认不询问,走ssh key校验
host_key_checking = True                    #是否需要每次询问要不要添加HostKey，默认是true，每次询问
log_path =                                  #日志文件地址,为空表示禁用日志
module_name = command                       #默认模块
gathering = smart|implicit|explicit         #smart表示如果有缓存则使用缓存，implicit每次收集，explicit不收集除非指定
fact_caching_timeout = 86400                #远程主机信息缓存时长，默认 86400
fact_caching = memeory|jsonfile|redis       #默认缓存到内存，可以写文件和redis
fact_caching_connection = /path/to/cachedir #如果写json 文件，此处写保存路径，如果是redis此处写redis连接地址
interpreter_python=auto                     #

#连接持久化配置
[persistent_connection]
ansible_connection_path= 
command_timeout=30 
connect_retry_timeout=15                    #超时重试时长
connect_timeout=30                          #连接
control_path_dir=~/.ansible/pc              #socket 文件保存目录

#颜色配置
[colors]
changed=yellow
console_prompt=white
debug=dark gray
deprecate=purple
diff_add=green
diff_lines=cyan
diff_remove=red
error=red
highlight=white
ok=green
skip=cyan
unreachable=bright red
verbose=blue
warn=bright purple

#普通用户提权配置
[privilege_escalation] 
agnostic_become_prompt=True
become_allow_same_user=False
become=False
become_ask_pass=False #sudo 时是否提示输入密码
become_exe=
become_flags=
become_method=sudo   #以sudo方式提权
become_user=root     #默认sudo到root
```

#### 4、主机清单文件

```bash
[group1]                     #分组
10.0.0.122
k8s-master                   #主机名
192.168.168.[101:108]        #IP区间写法
192.168.168.1[01:10]
192.168.2[21:31].1[68:70]
k8s[a:d]-master[1:10]        #主机区间写法

[group2]
10.0.0.144
#alias_name是为主机10.0.0.101定义的别名，可以为任意名称，在后面引用alias_name时，它指向的主机就是10.0.0.101
alias_name  ansible_host=10.0.0.101 ansible_python_interpreter=/usr/bin/python3 ansible_ssh_password=123456

[group3:children]           #继承写法，表示group1、group2都是group3的子组，即group3包含group1、group2的所有主机
group1
group2

[group1:vars]               #用于为组定义变量
ansible_connection=ssh
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_port=222
ansible_ssh_user=simple_user 
ansible_ssh_password=123456
```

常用配置项：

```bash
ansible_python_interpreter                     #表示在目标主机上执行Python脚本的Python解释器的路径
ansible_ssh_user                               #用于指定SSH连接时使用的用户名,默认是主控端运行ansible命令的用户
ansible_ssh_password/ansible_ssh_pass          #用于定义SSH连接到目标主机时使用的密码
ansible_become_pass/ansible_sudo_pass          #用于指定提权指定的密码
ansible_ssh_port                               #用于指定SSH连接时连接的端口
ansible_host                                   #用于定义目标主机的 IP 地址
ansible_connection                             #用于指定如何连接到远程主机，值有ssh、local、docker、kubectl等
ansible_ssh_private_key_file                   #指定连接远程主机使用的ssh私钥的文件路径
ansible_shell_type                             #用于指定在远程主机上执行命令时使用的 shell 类型
```

### 四、Ansible简单使用

1、ansible命令基础用法

ansible 10.0.0.165 -m ping -k

10.0.0.165bi需要在主机清单中

https://github.com/sidewalkerLin/SRE_LEARNING/blob/main/images/image-20231205141822286.png

![image-20240811212417624](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20240811212417624.png)

```bash
#-k使用ssh密码验证，不用-k则是基于key验证
ansible "10.0.0.150 10.0.0.206" -m script -a "./test.sh" -k
ansible 10.0.0.206 -m shell -a "sleep 50;echo 123" -k
```



### 四、Ansible常用模块及简单使用

1、commmand模块



2、shell模块





3、script模块



4、copy模块

copy 模块将 ansible 主机上的文件复制到远程主机，此模块具有幂等性

5、fetch模块

从远程主机提取文件至 ansible 的主控端，copy 相反，不支持目录

6、

### 五、Playbook



### 六、Role

