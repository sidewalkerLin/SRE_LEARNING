### 一、docker版本前世今生

```bash
#参考文章
https://zhuanlan.zhihu.com/p/305572519
```

在容器Docker技术出现之前，Linux中已经有一个叫 docker 的工具，这个 docker 是一个窗口停靠栏程序，就像苹果MAC系统中的dock那个程序一样的工具。Docker技术出来以后，由于在Linux系统中软件名不能与 docker 重名，所以容器Docker就在软件名称上加了 io 的后缀。

这个时候在Ubuntu中Docker的包叫docker.io，在Centos中Docker的包叫docker-io。

后来又将Ubuntu和Centos中Docker的包名称进行了统一，叫做docker-engine。

再后来，Docker技术越来越火爆，在取得窗口停靠栏程序docker的作者同意的情况下，docker更名为wmdocker。这时容器Docker技术的软件包名称正式由docker-engine更名为docker。

再再后来，Docker 发展到 1.13.1 版本后，Docker 公司把 Docker 分成了两种形式：

- docker-ce 社区版，免费
- docker-ee 商业版，收费

并且版本号的命名方式也改了，在 Docker 1.13.1 之后，直接是 Docker-ce 17.03.0 版本。

目前在Ubuntu自带的只有docker.io，Centos中自带的只有podman-docker。如果需要下载docker官方的docker-ce版本需要添加docker官方的源或者国内镜像的源。

### 二、docker安装与卸载

#### 1、Ubuntu中安装docker

(1)安装一些必要的系统工具rm -rf /var/lib/docker

```bash
#apt-transport-https            允许软件包管理器通过HTTPS协议传输文件和数据
#ca-certificates                允许系统（和网页浏览器）检查安全证书
#curl                           一个用于传输数据的工具
#software-properties-common     添加了一些用于管理软件的脚本
apt-get update && apt-get -y install apt-transport-https ca-certificates curl software-properties-common
```

(2)安装GPG证书

```bash
#使用curl工具从阿里云镜像仓库下载Docker的官方GPG密钥，并使用apt-key工具将其添加到apt软件包管理器的信任列表中。这样可以让apt软件包管理器验证Docker软件包的来源和完整性，防止安装被篡改或损坏的软件
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
```

(3)写入软件源信息

```bash
#$(lsb_release -cs)是一个命令替换，用于获取当前系统的发行版代号jammy，jammy 表示这个源对应的Ubuntu发行版的代号。例如，Ubuntu 20.04的代号是focal，Ubuntu 21.04的代号是hirsute，Ubuntu 22.04的代号是jammy

add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```

(4)更新本地软件包列表

```bash
#因为前面新加入了软件源，所以需要更新本地软件包列表
apt-get -y update
```

(5)查看Docker-CE的版本

```bash
apt-cache madison docker-ce | awk -F'|' '{print $2}'
```

(6)安装指定版本的Docker-CE

```bash
apt -y install docker-ce=5:20.10.20~3-0~ubuntu-jammy docker-ce-cli=5:20.10.20~3-0~ubuntu-jammy
```

(7)验证docker版本

```bash
docker version
```

#### 2、Centos中安装docker

(1)安装yum-utils

```bash
#yum-utils 是一个yum的工具包集合，可以帮助你管理和配置 yum 源，下载和安装软件包，清理缓存和冗余的包
#比如使用yum-config-manager来添加和删除源
yum install -y yum-utils
```

(2)添加docker官方的镜像源

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

(3)将docker官方源地址改为国内清华大学开源镜像源

```bash
sed -i 's+https://download.docker.com+https://mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
```

(4)更新本地软件包列表

```bash
yum makecache
```

(5)查看Docker-CE的版本

```bash
#--showduplicate会显示软件包的所有可用版本
yum list docker-ce --showduplicates | awk 'NR>2{print $2}'
```

(6)安装指定版本的Docker-CE

```bash
#centos8上可能会报错，runc版本冲突，参考：https://www.cnblogs.com/nuccch/p/17609317.html
#解决办法：命名后面跟参数--allowerasing，或者yum remove runc
yum -y install docker-ce-20.10.24-3.el8 docker-ce-cli-20.10.24-3.el8

```

(7)启动docker

```
systemctl enable --now docker.service
```

#### 3、二进制包安装docker

(1)下载二进制包

```bash
#可以先下载好再上传到没有通网的主机
wget https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/docker-23.0.0.tgz?spm=a2c6h.25603864.0.0.6ea015acRcB3sl
```

(2)解压

```
tar xvf docker-23.0.0.tgz
```

(3)移动到PATH下

```
mv /docker/* /usr/bin/
```

(4)配置docker.service

```bash
vim /lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H unix://var/run/docker.sock
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
```

(5)检查并重载配置文件，让systemd识别新创建或修改的服务文件，让systemctl来管理docker

```bash
#检查配置文件是否有错误
systemd-analyze verify /lib/systemd/system/docker.service

#重载配置文件使其生效
systemctl daemon-reload
```

(6)设置开机启动

```bash
systemctl enable --now docker.service
```

(7)查看docker版本

```bash
docker version
```

#### 4、centos中卸载docker

这里卸载指的是官方的docker-ce版本，如果是docker.io或是podman-docker需要自行写对包名称。

在安装docker之前要保证之前的docker卸载掉。

```bash
yum remove docker-ce

rm -rf /var/lib/docker
```

#### 5、Ubuntu中卸载docker

这里卸载指的是官方的docker-ce版本，如果是docker.io或是podman-docker需要自行写对包名称。

在安装docker之前要保证之前的docker卸载掉。

```bash
apt purge docker-ce

rm -rf /var/lib/docker
```



### 三、docker优化配置

```bash
#没有docker文件就创建
vim /etc/docker/daemon.json

{
"registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://si7y70hh.mirror.aliyuncs.com/"
 ],
#开启远程:https://docs.docker.com/config/daemon/remote-access/ 或者  ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// 
"hosts": ["unix:///var/run/docker.sock", "tcp://10.0.0.10:2375"], 
"insecure-registries": ["harbor.test.org"],
#k8s中要求cgroup driver必须是systemd，在docker老版本中此项值是cgroup，可以利用下面语句修改
"exec-opts": ["native.cgroupdriver=systemd"],
"graph": "/data/docker",  #指定docker数据目录,新版24.0.0不支持，实现：ExecStart=/usr/bin/dockerd --data-root=/data/docker

"max-concurrent-downloads": 10,
"max-concurrent-uploads": 5,
"log-opts": {
   "max-size": "300m",   #指定容器日志文件的最大值
   "max-file": "2"       #指定容器日志文件的个数，循环写入日志文件，即一个日志满，会写入第二个文件
 },
 #此项为true时，重启docker服务不会影响容器的状态。为false时，重启docker会关掉容器。
"live-restore": true, 
#设置代理科学上网
  "proxies": {           #代理 https://docs.docker.com/network/proxy/
   "default": {
     "httpProxy": "http://proxy.example.com:3128",
     "httpsProxy": "https://proxy.example.com:3129",
     "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
   }
   "tcp://docker-daemon1.example.com": {
     "noProxy": "*.internal.example.net"
   }
 }
}

```

### 四、docker资源隔离、资源控制、镜像分层原理

#### 1、namespace

**(1)namespace简介**

namespace是Linux内核的一项功能，它对内核资源进行隔离，让一组进程只能看到与自己相关的一部分资源，而另一组进程看到另一组资源，使得处于不同namespace的进程拥有独立的全局系统资源。改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

**(2)namespace分类**

截止至内核5.6，Linux namespace 实现了 8 项资源隔离，基本上涵盖了一个小型操作系统的运行要素，包括主机名域名、用户权限、文件系统、网络、进程号、进程间通信、进程资源控制、进程时间。

| namespace | 系统调用参数    | 隔离内容                                                     | 内核版本 |
| --------- | --------------- | ------------------------------------------------------------ | -------- |
| uts       | CLONE_NEWUTS    | 允许单个系统对不同的进程具有不同的主机名和域名               | 2.6.19   |
| ipc       | CLONE_NEWIPC    | 将进程与 SysV 风格的进程间通信隔离开来(信号量、消息队列、共享内存) | 2.6.19   |
| pid       | CLONE_NEWPID    | 为进程提供了一组独立于其他namespace的进程 ID                 | 2.6.24   |
| network   | CLONE_NEWNET    | 用来隔离网络设备、IP地址端口等网络栈的namespace              | 2.6.29   |
| mount     | CLONE_NEWNS     | 控制挂载点（文件系统）                                       | 2.4.19   |
| user      | CLONE_NEWUSER   | 跨多组进程提供权限隔离和用户身份隔离（用户和用户组）         | 3.8      |
| cgroup    | CLONE_NEWCGROUP | 隐藏了进程所属的控制组的身份                                 | 4.6      |
| time      | CLONE_NEWTIME   | 允许不同进程看到不同的系统时间                               | 5.6      |

**UTS namespace**

UTS namespace 提供了主机名和域名的隔离，这样每个容器就拥有独立的主机名和域名了，在网络上就可以被视为一个独立的节点，在容器中对 hostname 的命名不会对宿主机造成任何影响。

**IPC namespace**

IPC namespace 实现了进程间通信的隔离，包括常见的几种进程间通信机制，如信号量，消息队列和共享内存。我们知道，要完成 IPC，需要申请一个全局唯一的标识符，即 IPC 标识符，所以 IPC 资源隔离主要完成的就是隔离 IPC 标识符。

**PID namespace**

PID namespace 完成的是进程号的隔离

**Network namespace**

Network namespace是用来隔离网络设备、IP地址端口等网络栈的namespace。每个网络接口（物理或虚拟）只存在于 1 个namespace中，并且可以在namespace之间移动。每个namespace都有一组私有 IP 地址、它自己的路由表、套接字列表、连接跟踪表、防火墙和其他与网络相关的资源。Network namespace可以让进程拥有自己独立的（虚拟的）网络设备，每个namespace内的端口都不会冲突。

**不同的Network namespace之间通信是通过veth pair 来实现的，veth pair 是一对虚拟设备，它们在内核中连接。让不同的Network namespace之间通信只需要创建一个 veth pair，然后将veth 设备的俩个网络接口分别放入俩个Network namespace中即可。比如将veth网卡放在宿主机中，而eth0网卡放到容器中，实现宿主机与通信的通信。**

**Mount namespace**

Mount namespace用来控制挂载点。不同namespace中的进程看到的文件系统层次也是不一样的。在mount namespace中调mount(), unmount()只会影响当前namespace内的文件系统。在创建时，当前mount namespace中的挂载点被复制到新命名空间，但之后创建的挂载点不会在namespaces之间传播（如果使用共享子树，可以在命名空间之间传播挂载点 ）。

**User namespace**

User namespace用于隔离用户和用户组 ID。在 user namespace 中，一个进程可以有一个在该 namespace 外部完全不同的用户和用户组 ID。在 user namespace 中作为 root 用户运行进程，而在外部，这个进程可能只是一个普通用户。这对于容器技术非常有用，因为它允许容器内的进程以 root 用户身份运行，而不需要给它在宿主机上的真正的 root 权限。这样做的好处是增加了安全性。即使容器内的进程被攻击者利用，攻击者也只能获取容器内的 root 权限，而不能获取宿主机的 root 权限。因此，攻击者不能控制整个系统，只能控制被攻击的容器。

**Cgroup namespace**

Cgroup namespace用于隔离、限制和审计进程组（cgroup）的资源使用情况。它的主要作用是为每个namespace提供一个独立的cgroup视图，使得在namespace内的进程只能看到与其相关的cgroup，并且不能看到或影响到其他namespace的cgroup。这样可以有效地隔离不同的进程或进程组，防止它们相互干扰。

**Time namespace**

Time namespace运行不同的进程看到不同的时间。

```bash
[14:53:12 root@rocky86 ~]#ll /proc/1402/ns
total 0
lrwxrwxrwx 1 root root 0 Sep 10 14:49 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 10 14:53 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 10 14:53 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 10 14:53 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 10 14:52 uts -> 'uts:[4026531838]'

[14:58:56 root@rocky86 ~]#ll /proc/2808/ns
total 0
lrwxrwxrwx 1 root root 0 Sep 10 14:59 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 10 14:55 ipc -> 'ipc:[4026532679]'
lrwxrwxrwx 1 root root 0 Sep 10 14:55 mnt -> 'mnt:[4026532677]'
lrwxrwxrwx 1 root root 0 Sep 10 14:55 net -> 'net:[4026532682]'
lrwxrwxrwx 1 root root 0 Sep 10 14:55 pid -> 'pid:[4026532680]'
lrwxrwxrwx 1 root root 0 Sep 10 14:59 pid_for_children -> 'pid:[4026532680]'
lrwxrwxrwx 1 root root 0 Sep 10 14:59 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 10 14:59 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 10 14:59 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 10 14:55 uts -> 'uts:[4026532678]'
```



#### 2、cgroup







#### 3、unionFS



### 五、docker镜像管理

#### 1、docker search

范例1：搜索镜像名中包含nginx的镜像

<font color='#ff0000'> **注意：docker search查出来的信息不包含具体的版本信息**</font>

![image-20230827210602764](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230827210602764.png)

#### 2、docker pull

```bash
格式：
# NAME         拉取仓库中默认标签（通常是 latest）的镜像
# NAME:TAG	   拉取仓库中指定标签的镜像
# NAME@DIGEST  拉取仓库中指定摘要的镜像
docker pull NAME 或 docker pull NAME:TAG 或 docker pull NAME@DIGEST

常用选项：
-a or --all-tags   #拉取仓库中所有标签的镜像

-q or --quiet      #禁止冗余的输出          
```

#### 3、docker images

```bash
#常用选项:  
-q or --quiet     #只显示镜像 ID
-a or --all       #出所有镜像，包括中间层镜像
--digests         #显示镜像的摘要信息
--no-trunc        #显示完整的镜像 ID

-f or --filter	  #列出符合过滤条件的镜像。过滤条件可以是 dangling,label,reference,before,since等
dangling （dangling=true或者dangling=false）；
label    （label=<key>或者label=<key>=<value>）；
before   （before=<image-name>[:<tag>], <image id> 或者<image@digest>），筛选在给定镜像之前创建的镜像；
since    （since=<image-name>[:<tag>], <image id> 或者<image@digest>），筛选在给定镜像之后创建的镜像；
reference（reference=文件名的额通配符格式，例如-f “reference=ngin*”），筛选引用与给定镜像相匹配的镜像

--format          #使用go模板来格式化输出结果，注意字母大小写
{{.Repository}}： #镜像的仓库名称
{{.Tag}}：		 #镜像的标签
{{.ID}}：		 #镜像的ID
{{.Digest}}：	 #镜像的摘要值
{{.CreatedAt}}：  #镜像的创建时间
{{.Size}}：		 #镜像的大小

```

范例1：查看本地镜像列表

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829144852672.png" alt="image-20230829144852672" style="zoom:80%;" />

范例2：查看本地镜像的id

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829145106984.png" alt="image-20230829145106984" style="zoom:80%;" />

范例3：显示镜像的摘要信息/显示完整的镜像 ID

![image-20230829145605649](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829145605649.png)

范例4：显示符合多个条件的镜像

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829150137913.png" alt="image-20230829150137913" style="zoom:80%;" />

范例5：使用go模板来格式化输出结果

![image-20230829152324115](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829152324115.png)

#### 4、docker save

docker save 用于将一个或多个镜像镜像打包(导出)为一个tar文件

```bash
#打包单个镜像
docker save nginx:1.25.2 -o mirror.tar
docker save nginx:1.25.2 > mirror.tar

#打包多个镜像
docker save nginx:1.25.2 alpine:3.18.3 -o mirrors.tar
docker save nginx:1.25.2 alpine:3.18.3 > mirrors.tar

#如果镜像太大，可以压缩镜像,降低传输时间
docker save nginx:1.25.2 alpine:3.18.3 -o mirrors.tar && gzip mirrors.tar
docker save nginx:1.25.2 alpine:3.18.3 | gzip > mirrors.tar.gz

#打包所有镜像
docker save `docker images | awk 'NR>1{print $1":"$2}'` > all.tar
docker save `docker images --format "{{.Repository}}:{{.Tag}}"` > all1.tar
```

#### 5、docker load

docker load常用来和docker save配套使用，使用docker load来导入docker save打包的镜像

```bash
#常用选项
-i             #指定导入文件的路径
-q or --quiet  #禁止冗余的输出

docker load -i all.tar
docker load -i all.tar.gz
docker load -i < all.tar
```

#### 6、docker rmi

docker rmi 用于删除一个或多个镜像

```bash
docker rmi [options] IMAGE
#常用选项
-f or --force  #强制删除一个或多个镜像，即使它们被容器引用或者有子镜像
--no-prune     #不删除未被任何镜像引用的父镜像

#IMAGE格式
repository      #相当于repository:latest
repository:tag

```

范例1：删除所有dangling镜像

```
docker rmi `docker images -qa -f dangling=true`
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230830105215675.png" alt="image-20230830105215675" style="zoom:80%;" />

范例2：删除所有镜像

```bash
#使用-f可以强制删除容器使用的镜像，但是并不是物理删除，而是逻辑删除
docker rmi  `docker images -qa`
docker rmi  `docker images --format "{{.Repository}}:{{.Tag}}"`
docker rmi  `docker images | awk 'NR>1{print $1":"$2}'`
```

#### 7、docker image prune

docker image prune用于删除没有标签且没有被容器引用的镜像，这些镜像被称为悬空镜像(dangling镜像)

```bash
#直接使用docker image prune可以移除所有的dangling镜像，如果需要移除dangling镜像之外的镜像需要加-a
docker image prune [OPTIONS]

#常用选项
-a or --all    #移除所有的无用镜像，不只包括dangling镜像。判断是否是无用镜像？无用镜像表示尚未在容器中分配或使用它，可以用docker ps -a进行查看，如果没有任何一个容器使用过这个镜像，那么这个镜像就是无用镜像，可以被删除。反之，如果被容器使用则无法删除。

-f or --force  #删除镜像时会默认会提示确认信息，-f表示不提示确认信息。

--filter	   #提供过滤器的功能，添加过滤条件进行删除。
目前支持的过滤器有以下俩种：
1、until<timestamp> #这里需要注意timestamp的格式，支持时(h)、分(m)、秒(s),也就是说支持删除多少小时以前或多少分钟以前或多少秒以前的镜像，不支持周、月、年这种格式，但是可以通过h进行换算。另外还支持绝对的时间戳，例如2023-05-05T01:37:03的格式，可以通过go模板的{{.CreatedAt}}查看
2、label (label=<key>、label=<key>=<value>、label!=<key>或label!=<key>=<value>) 
```

范例1：删除所有的dangling镜像

```bash
#此操作只能删除dangling镜像，不会删除其它无用镜像
docker image prune -f
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829223245152.png" alt="image-20230829223245152" style="zoom:80%;" />

范例2：删除所有无用镜像

```bash
docker image prune -f -a
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829230439920.png" alt="image-20230829230439920" style="zoom:80%;" />

范例3：被容器使用的镜像无法删除

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829230857167.png" alt="image-20230829230857167" style="zoom:80%;" />

范例4：删除指定时间戳之前的镜像

注意：以下写法同样无法删除被容器使用的镜像

```bash
#删除24小时以前创建的无用镜像
docker image prune -a -f --filter "until=24h"
#删除60分钟以前创建的无用镜像
docker image prune -a -f --filter "until=60m"
#删除60秒以前创建的无用镜像
docker image prune -a -f --filter "until=60s"
#删除2周以前创建的无用镜像
docker image prune -a -f --filter "until=$(echo 24*7*2|bc)h"

#可以通过go模板查看镜像创建的具体时间，然后将{{.CreatedAt}}的时间格式换成2023-05-05T01:39:03这样的格式配合until来过滤
docker images --format "{{.Repository}} {{.Tag}} {{.ID}} {{.CreatedAt}}"
#删除2023-05-05T01:39:03以前创建的无用镜像，这个时间一定要比你想删除的镜像的时间要大才行
docker image prune -a -f --filter "until=2023-05-05T01:39:03"
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230829232349713.png" alt="image-20230829232349713" style="zoom:80%;" />

#### 8、docker tag

docker tag用于给镜像打标签，相当于给镜像起别名

```bash
用法：
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG] 或 docker tag SOURCE_IMAGE_ID TARGET_IMAGE[:TAG]
```

范例1：给dangling镜像打标签

dangling镜像是没有标签的，它的repository和tag都是<none>,可以通过docker tag给dangling镜像加上标签。

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230831113500406.png" alt="image-20230831113500406" style="zoom:80%;" />

范例2：给带标签的镜像打标签

给带标签的镜像打标签，标签不会被替换而是新增，俩个标签共用的一份磁盘资源(镜像ID相同)，删除一个标签不会将镜像删除

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230831141243359.png" alt="image-20230831141243359" style="zoom:80%;" />

#### 9、docker push



#### 10、docker inspect



#### 11、docker history



### 六、docker容器管理

#### 1、docker run

docker run用于新建并启动容器

```bash
docker run [选项] [镜像名] [shell命令] [参数]

#常用选项:  
-i, --interactive   #让容器的标准输入保持打开，通常和-t一起使用
-t, --tty           #让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上，通常和-i一起使用,注意对应的容器必须运行 shell才支持进入

-d, --detach        #在后台运行容器，并且打印容器id。容器默认在前台运行。
--name string       #给容器指定一个名字，没指定名字会随机生成
--h, --hostname     #指定容器的主机名

--rm                #在容器停止运行后自动删除容器。需要注意的是，在使用 --rm 选项时，所有容器的数据在容器终止时都会被删除。因此，在需要保存数据时，需要将数据挂载到主机上或使用卷。另外，--rm 选项不能与 -d 同时使用，即只能自动清理前台容器，不能自动清理后台容器。

-m                  #设置容器使用内存最大值
-v                  #绑定一个卷
--cpuset            #绑定容器到指定CPU运行
-p, --publish list  #随机端口映射，容器内部端口随机映射到主机的端口
-P, --publish-all   #指定端口映射，格式为：主机(宿主)端口:容器端口
--dns               #指定容器使用的DNS服务器，默认和宿主一致

--entrypoint        #此选项是用来指定容器运行时的入口点的。入口点是指容器启动时执行的命令或脚本。通常，一个镜像会有一个默认的入口点，比如bash或 sh。使用此选项可以覆盖镜像的默认入口点。

--privileged        #此选项是用来给容器赋予特权模式的。特权模式是指让容器拥有宿主机的所有权限，可以访问宿主机的所有设备和资源，包括内核模块、网络接口、挂载点等。这样可以让容器执行一些宿主机才能执行的操作，比如加载内核模块、修改系统时间、配置 iptables 等

-e, --env=[]        #设置环境变量
--env-file=[]       #从指定文件读入环境变量

--restart           #指定容器在退出时是否自动重启的
					#--restart=no,默认值，表示不自动重启容器。
					#--restart=on-failure[:max-retries]，表示只有当容器非正常退出时（退出状态码非零），才自动重启容器。
					#--restart=on-failure:5表示容器非正常退出时，会最多进行5次重启，直到重启成功。5次以后，不再尝试重启。
					#--restart=always，表示无论容器退出状态如何，都自动重启容器。开机也会自启。
					#--restart=unless-stopped，示无论容器退出状态如何，都自动重启容器，除非容器被手动停止。
					#--restart=for-update，表示只有当容器的镜像被更新时，才自动重启容器。
```

#### 2、docker ps

```bash
用法：
docker ps [OPTIONS]

#常用选项
-a, --all          #列出所有的容器，包括已经停止的   
-q, --quiet        #只列出容器的ID，不显示其他信息  
-s, --size         #显示每个容器的文件大小 
-f, --filter       #
-l, --latest       #列出最近创建的1个容器信息，无论它们是否在运行
-n, --last int     #列出最近创建的n个容器，无论它们是否在运行。例如-n 3，列出最近创建的3个容器。
(default -1)


--format           #按格式输出信息
{{.ID}}：		#容器的ID。
{{.Image}}：		#容器使用的映像名称。
{{.Command}}：	#容器的启动命令。
{{.CreatedAt}}：	#容器的创建时间。
{{.RunningFor}}：#容器运行的时间。
{{.Ports}}：		#容器的端口映射信息。
{{.Status}}：	#容器的状态。
{{.Size}}：		#容器的大小。
{{.Names}}：		#容器的名称。
{{.Label}}：		#容器的标签。
```

#### 3、docker rm

docker rm用于删除一个或多个镜像

```bash
docker rm [OPTIONS] CONTAINER [CONTAINER...]

#常用选项
-f, --force    #强制删除容器，包括正在运行的
l, --link      #删除容器之间的网络连接，而不删除容器本身
-v, --volumes  #删除容器挂载的匿名卷
```

#### 4、docker top

docker top用于显示容器内部进程信息的命令，它会输出每个进程的PID，用户，命令等信息

```bash
用法：
docker top CONTAINER [ps OPTIONS]
```

#### 5、docker stats

docker stats用于显示容器的实时资源使用情况的命令，它会输出每个容器的CPU占用率，内存使用量，网络流量等信息

```bash
用法：
docker stats [OPTIONS] [CONTAINER...]

#常用选项
-a or --all       #表示显示所有容器的情况，默认显示运行的容器的情况
--no-stream       #只显示一次结果
--no-trunc        #

--format string   #go模板输出
{{.Name}}         #容器名称
{{.CPUPerc}}      #CPU占用率
{{.MemUsage}}     #内存使用量
```

#### 6、docker inspect

docker inspect 可以查看docker各种对象的详细信息,包括:镜像,容器,网络等



#### 7、docker start



#### 8、docker stop



#### 9、docker kill





#### 10、docker attch





#### 11、docker exec



```bash
 用法：
 docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
 #常用选项
 -d, --detach          #在后台模式下运行命令，不显示输出结果
 --detach-keys string  #
 -e, --env list        #设置环境变量，例如-e VAR=VALUE表示设置变量VAR的值为VALUE     
 --env-file list       # 
 -i, --interactive     #保持标准输入流打开，用于交互式的命令，与t一起用    
 --privileged      
 -t, --tty             #分配一个伪终端，用于显示彩色和格式化的输出结果      
 -u, --user string     #指定运行命令的用户，例如-u root表示以root用户运行命令     
 -w, --workdir string  #指定运行命令的工作目录，例如-w /tmp表示在/tmp目录下运行命令     

```

#### 12、docker port





#### 13、docker logs



#### 14、docker cp



#### 15、docker system



```bash
用法：
docker system COMMAND

#常用选项
df       #显示Docker的磁盘使用情况，包括镜像，容器，数据卷和网络的大小和数量  
events   #显示Docker的实时事件，包括镜像，容器，数据卷和网络的创建，启动，停止，删除等操作 
info     #显示Docker的详细信息，包括版本，配置，内核，存储驱动，网络驱动等 
prune    #清理Docker的无用资源，包括停止的容器，未使用的镜像，数据卷和网络 

```

#### 16、docker export



```bash
用法：
docker export [OPTIONS] CONTAINER

#常用选项
-o, --output string   #写入文件，默认是打印到控制台
```

#### 17、docker import



#### 18、docker create



### 七、docker镜像制作

1、基于容器手动制作镜像

这可能是因为在你执行`docker commit`命令时，没有正确地保存原始容器的状态。`docker commit`命令会创建一个新的镜像，这个镜像包含了原始容器的文件系统的当前状态，但是它不会包含任何关于容器运行状态的信息，例如环境变量、运行命令（CMD）和入口点（ENTRYPOINT）等。

如果你的原始容器是使用特定的运行命令或者环境变量启动的，那么你需要在`docker commit`命令中使用`-c`选项来指定这些设置，例如：

```bash
docker commit -c "CMD ['nginx', '-g', 'daemon off;']" web01 nginx:v1
```

这将会在新的镜像中设置正确的运行命令，使得你可以像运行原始容器一样运行新的容器。

另外，你也可以在运行新的容器时使用`docker run`命令的`-e`选项来指定环境变量，或者使用`--entrypoint`选项来指定入口点。



2、基于DockerFile制作镜像





run尽量写在一行

不变的指令尽量写在前面

exec 



docker run 后面跟shell命令会替代cmd。会影响容器后台运行吗



### 八、docker数据管理





### 九、docker网络管理

#### 1、docker网络配置

##### 1.1docker默认网络配置

docker安装完后会默认生成一个docker0的网卡，且默认的IP地址是172.17.0.1/16。

在每个容器创建并运行后，都会在宿主机上会生成一个vmth的网卡，在容器内部也会生成一个eth0的网卡，这俩个网络接口是配对的，在内核相连，可以理解为通过虚拟的以太网电缆相连，任何发送到其中一个端点的数据包都会被立即 从另一个端点传出。宿主机的net namespace和容器的net namespace是通过vmth pair来实现不同网络名称空间的网络通信。

通过ethtool -S veth网卡名称可以查询到与其配对的网卡编号，可以发现网卡编号正好和容器中eth0网卡编号一致。

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230911152632475.png" alt="image-20230911152632475" style="zoom:80%;" />

还可以通过命令brctl show查看到**vmth网卡是桥接到docker0**上的

![image-20230911153115274](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230911153115274.png)

##### 1.2、禁止同一宿主机容器与容器间通信

同一宿主机上docker容器与容器之间默认是可以通信的。某个业务中会因为安全性要求来禁止容器间的通信可以使用dockerd 的 --icc=false 选项来禁止同一个宿主机的不同容器间通信。

```bash
vim /lib/systemd/system/docker.service

#在ExecStart行最后加上--icc=false即可禁止容器间通信，默认是true
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --icc=false 
```

##### 1.3、修改docker0默认网络配置

docker启动后会默认生成一个docker0的网桥，并且使用的ip是172.17.0.1/16，可能和宿主机的网段发生冲突，可以将其修改为其它网段的地址。

需要注意的是，在修改之后，再修改回来，使用的ip还是上一次配置的ip，而不是默认的172.17.0.1/16，如果需要使用172.17.0.1/16，需要明确指定。

<font color='#ff0000'> **注意：方法一和方法二不能同时配置，会冲突**</font>

方法一：修改daemon.json

```bash
vim /etc/docker/daemon.json

{
    "bip": "192.168.100.1/24"
}
```

方法二：修改docker.service

```bash
vim /lib/systemd/system/docker.service

#在ExecStart行尾加上绑定的ip地址段
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --bip=192.168.100.1/24
```

##### 1.4、使用自定义网桥

```bash
#新增一个网桥docker1
brctl addbr docker1

ip a a 192.168.100.1/24 dev docker1

vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -d docker1


#如需删除docker1
ip link set docker1 down
brctl delbr docker1
```

#### 2、容器名称互联

##### 2.1、为什么要容器名称互联？

docker容器在创建和运行后，会自动分配容器名称，容器ID和IP地址。容器在每次启动后，容器的ip地址都可能会改变，如果容器之间需要使用ip地址来进行通信，ip地址变来变去是极为不方便的。在容器创建启动后，容器名就不会再自动变化了，可以通过将容器名称绑定ip的形式，容器间通过固定的名称来访问即可。

##### 2.2、通过容器名称互联

```bash
#先创建名为web01的一个容器
docker run -d --name web01 nginx
#如果想通过容器名称和web01通信，在容器创建的时候加入--link web01即可，实际上的原理是在web02的/etc/hosts中加入了web01的解析，web01的ip如果变了，/etc/hosts也会跟着变
docker run -d --link web01 --name web02 nginx
```

##### 2.3、通过自定义容器别名互联

容器名称可能在后期会发生变化，如果发生变化的话通过容器名称来访问就会出现问题，这种情况下就可以给容器添加一个别名，这样主要保证别名不改变就行，容器名称变化不再影响容器间访问。

```bash
docker run -d --name web01 nginx

#给web01定义别名nginxweb01，通过nginxweb01也可以访问web01
docker run -d --link web01:"nginxweb01" --name web02 nginx
```

#### 3、dockers网络连接模式

##### 3.1、网络连接模式介绍

docker支持5种网络模式：Bridge、Host、None、Container、自定义网络连接

docker network ls可以列出当前docker配置的网络：

```bash
#默认创建好的网络模式有3种，另外俩种模式需要自己手动配置
root@Ubuntu22:~# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
886e187bc3c3   bridge    bridge    local
72276fc8541e   host      host      local
bde9595d54eb   none      null      local
```

##### 3.2、网络模式指定

默认新建的容器使用Bridge模式，创建容器时，docker run 命令使用以下选项指定网络模式。

```bash
docker run --network <mode> 或者 docker run --net=<mode>

<mode>值：
bridge
host
none
container:<容器名或容器ID>
<自定义网络名称>
```

##### 3.3、docker network 

```bash
用法：
docker network COMMAND

COMMAND：
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks
 
```

##### 3.4、Bridge模式

Bridge模式是docker默认的网络连接模式，每运行一个容器会在宿主机生成一个vmth网卡，这些网卡都会桥接在docker0网桥上。此外，每个容器还会被分配自己的网络ip，容器内的eth0网卡和宿主机对应的vmth网卡在内核相连，借此进行容器之间的通信。

可以和外部网络之间进行通信，通过SNAT访问外网，使用DNAT可以让容器被外部主机访问，所以此模式也称为NAT模式。

**此模式宿主机需要启动ip_forward功能**。

##### 3.5、Host模式

如果指定host模式启动的容器，那么新创建的容器不会创建自己的虚拟网卡，而是直接使用宿主机的网卡和IP地址，因此在容器里面查看到的IP信息就是宿主机的信息，访问容器的时候直接使用宿主机IP+容器端口即可，不过容器内除网络以外的其它资源。

此模式无法使用宿主机和容器的端口映射，使用的是宿主机的资源。

```bash
docker run --network host --name web01 -d nginx
```

##### 3.6、None模式

在使用 none 模式后，Docker 容器不会进行任何网络配置，没有网卡、没有IP也没有路由，因此默认无法与外界通信，需要手动添加网卡配置IP等，所以极少使用。

```bash
docker run -d --network none -p 8001:80 --name web02 nginx
```

##### 3.7、Container模式

使用此模式创建的容器需指定和一个已经存在的容器共享一个网络，而不是和宿主机共享网络，新创建的容器不会创建自己的网卡也不会配置自己的IP，而是和一个被指定的已经存在的容器共享IP和端口范围，也就是共享一个Network namespace，因此这个容器的端口不能和被指定容器的端口冲突，除了网络之外的文件系统、进程信息等仍然保持相互隔离，两个容器的进程可以通过lo网卡进行通信。

共享Network namespace意味着共享ip和端口，比如俩个容器只能有1台打开80端口。<font color='#ff0000'> **虽然俩个容器共享了Network namespace，但这并不意味着两个容器可以访问对方的进程，只有监听在网络接口的进程才能被对方访问**</font>。比如一个容器的nginx进程监听在80端口，那么另一台容器可以通过127.0.0.1:80来访问对方的nginx进程，而对方的其它进程无法访问。

注意：

第一个容器的网络可能是bridge，或none，或者host，而第二个容器模式依赖于第一个容器，它 们共享网络；

如果第一个容器停止，将导致无法创建第二个容器；

第二个容器可以直接使用127.0.0.1访问第一个容器；

默认不支持端口映射；

```bash
这里想想一个问题：wordpress+mysql架构中，是mysql来共享wordpress的容器网络？还是wordpress来共享mysql的容器网络？
#首先指定了container模式的容器无法做端口映射，如果wordpress的容器来指定container模式的话，wordpress就无法通过端口访问了
docker run -d -p 80:80 --name wordpress --restart=always wordpress:php7.4-apache

docker run --network container:wordpress -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=wordpress -e MYSQL_USER=wordpress -e MYSQL_PASSWORD=123456 --name mysql -d --restart=always mysql:8.0.29-oracle
```

##### 3.8、自定义网络连接模式

除了以上的4种网络模式，也可以自定义网络，使用自定义的网段地址，网关等信息。可以使用自定义网络模式，实现不同集群应用的独立网络管理，而互不影响，而且在网一个网络内，可以直接利用容器名相互访问，非常便利。可以把不同业务的项目划分到不同的网段中。

<font color='#ff0000'> **注意：自定义网络内的容器可以直接通过容器名进行相互的访问,而无需使用 --link**</font>，访问的时候尽量不要通过ip来访问，因为ip也是自动分配的，使用容器名来访问即可。

第一步：创建自定义网络

```bash
#宿主机上会生成一块br×××××的网卡
docker network create --subnet 172.27.0.0/24 --gateway 172.27.0.100 mynetwork
```

第二步：引用自定义网络

```bash
#使用自定义网络的时候，vmth网卡会桥接到自定义网络的网桥上，而不再docker0上
docker run --network mynetwork -d --name web01 nginx

#查看容器ip是否是自定义网络的网段
docker exec web01 ip a
#查看容器路由是否是自定义网络设置的网关
docker exec web01 ip route
#查看容器网络能否连接互联网
docker exec web01 ping www.baidu.com
```

另外可以使用以下命令来查看自定义网络的信息和删除自定义网络：

```bash
#查看自定义网络的信息
docker network inspect <自定义网络名称>
#删除自定义网络
docker network rm <自定义网络名称>
```

##### 3.9、同一宿主机不同网络的容器通信

前置条件：

条件一、先创建2个网络

```
docker network create --subnet 192.168.1.0/24 --gateway 192.168.1.1 net1
docker network create --subnet 192.168.2.0/24 --gateway 192.168.2.1 net2
```

条件二、创建2个容器分别使用2个网络

```bash
docker run -d --name web01 --network net1 nginx sleep 1000
docker run -d --name web02 --network net2 nginx sleep 1000
```

测试俩个容器中是否能ping通对方：

```bash
#ping不通，实际上docker是通过iptables规则实现的
docker exec web01 ping 192.168.2.2
```

**(1)修改iptables规则实现同一宿主机不同网络的容器通信**

![image-20230911230421781](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230911230421781.png)



```
iptables save > backup.rule
iptables restore < backup.rule
```

**(2)使用docker network connect实现同一宿主机不同网络的容器通信**

```bash
#将web01连入net2网络中
docker network connect net2 web01
#可以ping通了
docker exec web01 ping 192.168.2.2
#反过来无法ping通，因为我们只是单向的将web01加入到net2中，而没有将web02加入到net1中
docker exec web02 ping 192.168.1.2
```

docker network connect实现原理：

在上述过程中，执行docker network connect net2 web01，实际上其实是在web01这个容器中加了一块同网段的网卡。

![image-20230912120212182](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230912120212182.png)

##### 3.10、跨宿主机间的容器通信

(1)利用网桥实现跨宿主机的容器通信

在俩个宿主机创建一个自定义网络(即网桥),将ens33桥接到上面即可。

(2)静态路由实现

```bash
#不用自定义网络，用docker0也行
A主机：ens33 10.0.0.151
docker network create --subnet 192.168.1.0/24 --gateway 192.168.1.100 net2
docker run -d --name web02 --network net2 nginx sleep 1000
#添加到对方的静态路由
ip route add 192.168.2.0/24 via 10.0.0.152 dev ens33
docker exec web02 ping 容器B的ip

B主机：ens33 10.0.0.152
docker network create --subnet 192.168.2.0/24 --gateway 192.168.2.100 net10
docker run -d --name web10 --network net10 nginx sleep 1000
#添加到对方的静态路由
ip route add 192.168.1.0/24 via 10.0.0.151 dev ens33
docker exec web10 ping 容器A的ip

#ip route del 192.168.1.0/24 via 10.0.0.151 dev ens33
```



