### 一、服务治理的相关概念

单体架构到微服务架构

将不同的业务解耦成单独的微服务，将各个微服务的api接口注册到注册中心，各个微服务通过注册中心获得其它服务的api地址来获取数据

前端根据业务不同，也需要打成独立的包部署

前端调用后端微服务的接口，是通过http://api的形式调用，而微服务网关调用到的最小粒度是微服务，而api网关则能调用到具体的api

微服务网关，

api网关

![image-20230905231045042](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230905231045042.png)

如果说不考虑前端，各个微服务之前相互通过api接口调用，只需要服务注册与发现中心就能满足需求。如果考虑前端，就需要把微服务的api接口暴露给前端app，这个时候就需要引入微服务网关的概念。



### 二、K8S入门

#### 1、kubeadm快速部署集群

环境准备：

| IP         | OS          | ROLE   | HOSTNAME     | DNS              |
| ---------- | ----------- | ------ | ------------ | ---------------- |
| 10.0.0.160 | Ubuntu22.04 | master | k8s-master01 | www.master01.com |
| 10.0.0.161 | Ubuntu22.04 | node   | k8s-node01   | www.node01.com   |
| 10.0.0.163 | Ubuntu22.04 | node   | k8s-node02   | www.node02.com   |
| 10.0.0.164 | Ubuntu22.04 | node   | k8s-node03   | www.node03.com   |

| Kubernetes | Container Runtime | CRI                | CNI     |
| ---------- | ----------------- | ------------------ | ------- |
| v1.27.1    | Docker CE 23.0.5  | cri-dockerd v0.3.1 | flannel |

(1)配置域名

```bash
#所有节点都配置
vim /etc/hosts
.........
10.0.0.160 www.master01.com k8s-master01
10.0.0.161 www.node01.com   k8s-node01
10.0.0.163 www.node02.com   k8s-node02
10.0.0.164 www.node03.com   k8s-node03
........
```

(2)禁用swap设备

```bash
#关闭当前已启用的所有Swap设备
swapoff -a

#编辑/etc/fstab配置文件，注释用于挂载Swap设备的所有行
vim /etc/fstab
.......
#/swap.img      none    swap    sw      0       0
.......

#Ubuntu20.04及以后版本需要用systemctl mask禁用systemctl --type swap列出的所有设备
systemctl --type swap
systemctl mask SWAP_DEV

reboot

#如果确实需要在节点上使用swap设备，需要设置kubeam忽略Swap启用的状态错误
vim /etc/default/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```

(3)设置时钟同步

```bash
#如果有本地的时钟服务器，最好用本地的时间服务器
apt install chrony -y
systemctl start chrony.service
```

(4)禁用默认的防火墙服务

```bash
systemctl disable --now ufw
```

(5)安装docker、配置daemon.json、配置docker代理

```bash
vim docker_install.sh
#!/bin/bash
apt-get update && apt-get -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
echo | add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-get -y update
apt -y install docker-ce=5:23.0.5-1~ubuntu.22.04~jammy docker-ce-cli=5:23.0.5-1~ubuntu.22.04~jammy

#配置daemon.json优化镜像加速等
cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://si7y70hh.mirror.aliyuncs.com/",
    "https://registry.docker-cn.com"
 ],
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
  "max-size": "200m"
},
"storage-driver": "overlay2"
}
EOF

#配置docker代理
#sed -i '/\[Service\]/a \
#Environment="HTTP_PROXY=http://172.20.0.29:7980" \
#Environment="HTTPS_PROXY=https://172.20.0.29:7980"' /lib/systemd/system/docker.service

systemctl daemon-reload
systemctl restart docker.service
systemctl enable docker.service
```

(6)安装cri-dockerd

```bash
export https_proxy=172.20.0.29:7980
#下载好后用scp将包传到其它节点
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
dpkg -i dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
systemctl enable cri-docker.service
unset https_proxy
```

(7)安装kubelet、kubeadm和kubectl

```bash
#每个节点都执行
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
#apt-get install -y kubelet kubeadm kubectl默认安装最新版
apt install -y kubeadm=1.27.1-00 kubectl=1.27.1-00 kubelet=1.27.1-00
systemctl enable kubelet
```

(8)配置cri-dockerd

```bash
vim /lib/systemd/system/cri-docker.service
......
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --cni-bin-dir=/opt/cni/bin --cni-cache-dir=/var/lib/cni/cache --cni-conf-dir=/etc/cni/net.d --pod-infra-container-image registry.aliyuncs.com/google_containers/pause:3.6
......

systemctl daemon-reload && systemctl restart cri-docker.service
```

(8)配置kubelet

```bash
mkdir /etc/sysconfig
cat > /etc/sysconfig/kubelet << EOF
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=/run/cri-dockerd.sock"
EOF
```

(9)master节点拉取相关镜像文件

```bash
#查看需要拉取哪些镜像文件
kubeadm config images list

#从国内仓库拉取镜像
kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock --image-repository=registry.aliyuncs.com/google_containers
```

(10)初始化master节点

```bash
kubeadm init --apiserver-advertise-address="10.0.0.160" --kubernetes-version=v1.27.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --cri-socket unix:///run/cri-dockerd.sock  --upload-certs --image-repository=registry.aliyuncs.com/google_containers

docker pull  pause 重新tag成

#如果初始化失败需要重新执行初始化，执行下面的语句
kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock -f
rm -fr  $HOME/.kube/config
```

(11)实现kubeatl、kubeadm命令补全功能

```bash
vim .bashrc
.....
source <(kubectl completion bash)
source <(kubeadm completion bash)

source .bashrc
```

(12)Kubernetes集群管理员认证到Kubernetes集群时使用的kubeconfig配置文件

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

(13)部署网络插件

```bash
mkdir /data/kubernetes/network/flannel -pv
cd  /data/kubernetes/network/flannel
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
#查看部署flannel需要的拉取的镜像
grep image: kube-flannel.yml
#部署flannel并应用
#插件有问题可以清除再重新应用kubectl delete -f kube-flannel.yml
kubectl apply -f kube-flannel.yml
#查看2个镜像是否拉取成功
docker images
```

(14)添加节点到集群中

```bash
kubeadm join 10.0.0.160:6443 --token 7pf113.ebnjsdg948zgltz8 --discovery-token-ca-cert-hash sha256:145020abb2f29ccd1f10d1614795bf80d18a0fd4d0137055950f1564e0cf970c --cri-socket=unix:///var/run/cri-dockerd.sock
kubeadm join 10.0.0.160:6443 --token xa6pw8.bv92qztv0u5qcoa6 --discovery-token-ca-cert-hash sha256:b44fad882edd5c9743ebd5ccb21d8ebc5bc0d0dd39b5e1970b954d021fd14484 --cri-socket=unix:///var/run/cri-dockerd.sock
```

(14)验证集群节点

```bash
#列出集群中所有节点的状态，ready才是正常的
kubectl get nodes
#列出集群中所有名称空间的pod信息，running才是正常的
kubectl get pods -A

#如果不是running可以用kubectl describe查看pod的信息
kubectl describe pod pod_name -n kube-system
```

![image-20231010225856122](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231010225856122.png)

2、脚本安装k8s

```bash
vim all.sh
#!/bin/bash
sed -i '/following lines/i\
10.0.0.160 www.master01.com k8s-master01\
10.0.0.161 www.node01.com   k8s-node01\
10.0.0.163 www.node02.com   k8s-node02\
10.0.0.164 www.node03.com   k8s-node03' /etc/hosts
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
apt install chrony -y
systemctl start chrony.service
systemctl disable --now ufw
apt-get update && apt-get -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
echo | add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-get -y update
apt -y install docker-ce=5:23.0.5-1~ubuntu.22.04~jammy docker-ce-cli=5:23.0.5-1~ubuntu.22.04~jammy

#配置daemon.json优化镜像加速等
cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://si7y70hh.mirror.aliyuncs.com/",
    "https://registry.docker-cn.com"
 ],
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
  "max-size": "200m"
},
"storage-driver": "overlay2"
}
EOF
systemctl daemon-reload
systemctl restart docker.service
systemctl enable docker.service
export https_proxy=172.20.0.29:7980
#下载好后用scp将包传到其它节点
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
unset https_proxy
reboot
```



2、k8s常用命令



```bash
kubectl run my-nginx --image=nginx:latest
kubectl logs my-nginx
kubectl get pod my-nginx -o wid
kubectl run my-nginx --image=nginx:latest --dry-run=client -o yaml
kubectl create deployment my-nginx-deployment --image=nginx:latest
kubectl delete deployment my-nginx-deployment
kubectl get deployments.apps
kubectl get rs
kubectl edit deployments.apps my-nginx-deployment
kubectl scale
kubectl set
kubectl 
kubectl 
```

3、



service三种类型：

ClusterIP、NodePort、LoadBalancer



NodePort实现原理

NodePort的工作原理其实就是将service的端口映射到Node的端口上，然后通过NodeIP:NodePort 来访问Service



endpoint

endpoints controller



kube-proxy

iptables、ipvs  默认是iptables      service过多，iptables线性查找，性能不好。到达一定数量，用ipvs，ipvs是基于hash实现的，时间复杂度为O(1)。都基于基于Netfiltea





workernode -> loadblancer -> master

这个过程中，wokernode到loadblancer是做的https，loadblancer到master做的四层负载，直接让workernode和master进行认证，如果loadblancer到master做的七层负载，那么workernode和master都需要和loadblancer进行认证，十分复杂











#### 2、K8S集群架构

K8S架构图如下图，可以分为MasterNode、WorkerNode两大部分。

- master节点主要有API Server、Controller-Manager和Scheduler三个组件，以及一个用于集群状态存储的Etcd存储服务组成，它们构成整个集群的控制平面
- worker节点主要由Kubelet、Kube Proxy及容器运行时三个组件组成，它们承载运行各类应用容器

![image-20231105171935887](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231105171935887.png)

##### 2.1、API Server

提供了集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等

提供了集群其它组件之间数据交互的渠道

##### 2.2、Scheduler

负责整个集群资源的调度功能，根据特定的调度算法和策略将pod调度到最优的工作节点上面去

##### 2.3、Controller-Manager

Controller Manager 就是集群内部的管理控制中心，由负责不同资源的多个 Controller 构成，共同负责集群内的 Node、Pod 等所有资源的管理。Controller 保证集群内的资源保持预期状态，而 Controller Manager 保证了 Controller 保持在预期状态。

##### 2.4、Etcd

基于raft协议，集中式存储集群状态信息

##### 2.5、Kubelet

监控节点状态如cpu、内存、磁盘空间等信息定时报告给apiserver

监控pod状态如容器的运行状等态信息定时报告给apiserver

3种探针进行监控检查，如果容器出错，kubelet要根据 pod 设置的重启策略进行处理

容器日志管理

监听apiserver发来的决策，调用cri接口，实现决策的结果

##### 2.6、Kube-Proxy

pod的特性使得它自己只具有一个临时的生命周期，一个pod可能随时会被创建或销毁，如果pod发生变化，我们很可能无法通过它的ip地址来访问它，所以我们需要一个service来提供一个vip也就是cluster ip来通过一定的负载均衡策略将流量分发到service关联的标签的pod中，这个负载均衡的过程通常是由kube-proxy来实现的

Kube-Proxy支持3种代理模式：user space、iptables、ipvs

user space模式是k8s早期版本支持的模式，由于存在严重的性能问题(内核态和用户态数据包来回拷贝)，现在已基本不再使用

iptables模式是k8s中比较常用的一种模式，它是通过添加iptables规则来实现的负载均衡功能的。并且在iptables模式下，随着service和pod数量的不断增长，iptables的规则也会不断增长且变得复杂，service或endpoints的 更新可能会导致大量的iptables规则进行更新，而iptables规则的更新会造成短暂的网络中断以及占用cpu资源，当大量的规则更新就可能造成更大的中断时间和cpu资源占用。另外在iptables模式下，kube-proxy的时间复杂度为O(n)，因为iptables规则是线性的，一个网络包需要从上到下顺序的去匹配规则，最坏的情况下可能需要找到最后一个规则才行，所以规则如果太多，性能就会变得越差。另外，iptables模式支持的负载均衡算法只有轮询和随机。

ipvs模式则是为了处理大规模service和pod而提供的，它是基于哈希表来实现的，时间复杂度为O(1)，所以它的查询、插入、删除操作是非常快的，与规则的数量无关。并且ipvs支持更多的负载均衡算法，如加权、源地址哈希、最小连接等。

**iptables模式与ipvs模式比较**

相同点：都是基于Netfilter实现的

不同点：

iptables是线性的，查询的时间复杂度为O(n) / ipvs是基于哈希表的，查询的时间复杂度为O(1)

iptables支持的负载均衡算法只有轮询和随机 / ipvs支持多种负载均衡算法

iptables在大规模环境中，规则的大量刷新会造成网络中断和占用cpu资源 / ipvs不存在这种情况



iptables中，流量进入iptables后，先根据流量的目标clusterip找到对应的service自定义chain，然后根据具体规则在分发到后端pod

##### 2.7、Container Runtime

docker、containerd、cri-o

2.8、CoreDNS

可以让pod把service的名称解析为ip地址

### 三、资源管理基础

#### 1、资源对象及API群组

api-server中内置了很多资源类型，如果按照资源的主要功能作为分类标准，可以将这些API对象分为以下5种类别：

- 工作负载(Workload)：负责应用编排
- 发现和负载均衡(Discovery & LB)：完成服务注册、发现以及流量调度
- 存储与配置(Config & Storage)：
- 集群(Cluster)
- 元素据(Metadata)

##### 1.1、工作负载型资源：

如果集群中某个workernode宕机，那么该节点上的pod状态都会变为失败，这就需要集群管理员来创建新的pod来恢复应用。而工作负载型资源就是来解决这个问题的，它的出现就是为了帮集群管理员减轻负担，让集群管理员不用再直接管理pod，而是给工作负载型资源配置控制器来保证正常状态的pod个数是正确的，与集群管理员指定的数量一致。

Kubernetes 提供若干种内置的工作负载资源：

Deployment和ReplicaSet 

StatefulSet

DaemonSet  比如agent，每个节点上运行一个就行，部署俩个没有必要；Deployment可以有可以0个

Job和CronJob

1.2、









1.3、









1.4、









1.5、

#### 2、API资源对象管理机制

##### 2.1、指令式命令

指令式命令即使用构建在kubectl命令行工具中的指令式命令直接快速创建、更新、删除Kubernetes 对象

特点：

- 直接作用于集群上的活动对象
- 适合在开发环境、测试环境一次性完成一次性的操作任务，不适用于生产环境

```bash
kubectl create deployment nginx --image nginx:latest --replicas 2
```

##### 2.2、指令式对象配置

指令式对象配置即使用kubectl命令配合yaml或json编写的对象配置文件来创建、更新、删除Kubernetes 对象

特点：

- 基于资源对象配置文件对对象进行管理
- 只能引用单个的配置清单文件，但是可以使用多个-f来指定多个配置清单文件
- 可用于生产环境
- 执行同样的操作会报错，比如执行了2次同样的命令

```bash
kubectl create -f my-nginx.yaml -f xxx.yaml .......
```

##### 2.3、声明式对象配置

声明式对象配置可以通过在一个目录中存储多个对象配置文件、并使用 kubectl apply 来递归地创建和更新对象来创建、更新和删除 Kubernetes 对象。

特点：

- 基于资源对象配置文件对对象进行管理
- 可以引用单个的配置清单文件，也可以引用目录来引用其下的所有配置清单文件
- 适用于生产环境
- 执行同样的操作不会报错

```bash
kubectl apply -f my-nginx.yaml
kubectl apply -f my-nginx/
```

#### 3、kubectl命令与资源管理

3.1、kubectl create



3.2、kubectl run



3.3、kubectl expose



3.4、kubectl set

#### 3.5、kubectl get

3.6、kubectl explain

3.7、kubectl edit

3.8、kubectl delete

3.9、kubectl rollout

3.10、kubectl scale

3.11、kubectl describe

3.12、kubectl logs

3.13、kubectl exec

3.14、kubectl apply

3.15、kubectl patch

##### 3.16、kubectl taint

```bash
#给node节点添加污点
kubectl taint nodes <node-name> hardware-type=gpu:NoSchedule
```

##### 3.17、kubectl lable

```bash
#查看pod的标签
kubectl get pods --show-labels
#添加、删除、修改名为foo的pod的标签
kubectl label pods foo unhealthy=true -n namespace_name
kubectl label pods foo unhealthy- -n namespace_name
kubectl label pods foo unhealthy=false --overwrite -n namespace_name
```

#### 4、对象类资源格式

4.1、资源配置清单

```yaml
apiVersion:        #资源群组名称和版本
kind:              #资源类型标识
metadata:          #对象元数据
name:              #对象名称标识，同类型下必须惟一
namespace:         #对象的名称的作用域
labels:            #标签
	key1: value1
	key2: value2
	...
annotations：      #注解
	key1: value1
	...
spec:              #期望状态，通过二级字段进行定义
status:            #实际状态，由Controller自动维护
```

###  四、管理Pod资源对象   

#### 1、定义Pod资源

##### 1.1、什么是Pod？

Pod是一个或多个容器的集合，所以也可以称为容器集，一个**共享Network、IPC和UTS名称空间以及存储资源**的容器集。它是Kubernetes集群调度、部署、运行应用的原子单元。

在Kubernetes集群中，可以从不同角度来对Pod的分类进行划分。

**按照Pod中的容器数量，Pod分为以下俩种**：

- 单容器Pod：单个Pod中仅包含单个容器，实际上大多数Pod中都只有一个容器

- 多容器Pod：单个Pod中包含多个"超亲密关系"的容器


如何理解超亲密关系？

俩个容器如果需要共享network、ipc、uts名称空间或共享文件系统，那么这个俩个容器具有“超亲密关系”。

**按照Pod的创建和管理方式，Pod分为以下俩种**：

- 静态Pod
- 动态Pod

##### 1.2、Pod中的pause容器

Pause 容器，又叫 Infra 容器。每一个Pod中，都运行了pause容器。那pause容器有什么作用呢？

我们都知道每个docker容器，它们的名称空间，如network、pid、uts等名称空间都是隔离的。如果一个Pod中包含多个容器的时候，它们往往是需要共享network、uts、ipc甚至pid这些名称空间。

理论上，只需要创建一个父容器就可以配置docker来管理容器之间共享名称空间的问题，当然这个父容器需要能够准确的知道如何去创建共享运行环境的容器，还能管理这些容器的生命周期。

基于这个思想，k8s中使用了pause容器来作为Pod中的所有容器的父容器。

总的来说，pause容器提供了以下者俩个功能：

- Pod中每个容器默认共享Network、UTS、IPC名称空间(PID共享需要手动开启)
- 启动init进程
- Pod中容器对于存储卷的共享是依靠pause容器实现的

在先版本的k8s中，Pod是默认是不开启PID名称空间共享的，如果需要开启需要手动进行配置。如果共享了PID名称空间，那么Pod中所有容器进程都会处于同一个PID名称空间内，并且pause容器启动的init进程会作为父进程监控其它所有容器进程，并自动处理所有的僵尸进程。如果没有共享PID空间，则每个容器需要自己处理自己的僵尸进程，这样可能会造成进程表中占用的记录太多，从而占用用内存资源以及pid号。

什么是僵尸进程？

在Linux系统中，如果一个子进程比父进程先结束，而父进程又没有回收子进程，释放子进程占用的资源，那么这个子进程就会成为一个僵尸进程。僵尸进程几乎放弃了所有的内存空间，没有任何可执行代码，也不能被调度，它仅仅在进程表中占用一条记录。一般来说，僵尸进程会由父进程来收尸，如果父进程没有处理，那么在父进程结束后将由init托管。如果父进程是一个循环，那么子进程就会保持僵尸状态。

##### 1.3、Pod资源规范



##### 1.4、Pod重启策略

Pod重启策略决定了容器终止后是否应该重启Pod。

定义Pod重启策略的字段是restartPolicy，它支持3个选项：

- **Always**：无论何种退出状态码，都要重启容器，默认重启策略
- **Onfailure**：仅在退出状态码为非0值退出时重启容器
- **Never**：无论何种exit code，都不重启容器，需要手工重启

关于退出状态码：

- 0表示成功结束，比如容器启动执行了一个前台的命令，在前台命令执行完后容器就会退出
- 1-255这种非0状态码则表示程序因为某种异常而退出，比如退出状态码为2表示因为语法错误而退出

##### 1.5、容器镜像拉取策略

k8s中容器镜像的拉取策略由字段imagePullPolicy指定，支持3个选项：

- **Always**：每次启动容器时都尝试重新拉取镜像
- **Never**：永远不会自动拉取镜像，容器只会使用本地已有的镜像
- **IfNotPresent**：如果本地有镜像就使用本地的，如果没有就拉取

需要注意的是：

- 如果没有指定容器镜像拉取策略且容器镜像tag不是latest，则默认拉取策略是IfNotPresent
- 如果没有指定容器镜像拉取策略且容器镜像tag是latest，因为latest是漂移的，则默认拉取策略是Always
- 如果指定了容器镜像拉取策略，则直接按指定的拉取策略来，不管镜像tag是否是latest

#### 2、配置Pod

##### 2.1、通过环境变量向容器传递参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wordpress
  namespace: demo
  labels:
    app: wordpress
    version: "6"
spec:
  containers:
  - name: wordpress
    image: wordpress:6-apache
    imagePullPolicy: IfNotPresent
    env:
    - name: WORDPRESS_DB_HOST
      value: mysql.demo.svc.cluster.local
    - name: WORDPRESS_DB_USER
      value: "wpuser"
    - name: WORDPRESS_DB_PASSWORD
      value: "wppass"
    - name: WORDPRESS_DB_NAME
      value: "wpdb"
  restartPolicy: Always
```

##### 2.2、Pod的监控状态监测机制以及探针配置

**(1)Pod支持3种探针类型**：

- **startup Probe(启动探针)**：用于检查容器是否正常启动，探测时间间隔一般比较大。容器在启动状态才会使用启动探针，使用启动探针时其它探针会被禁用，直到容器启动成功为止，启动探针就会退出。如果启动探针失败，那么kubelet就会杀死容器，然后容器根据重启策略进行重启。
- **liveness Probe(存活探针)**：用于检查容器是否还在运行，探测时间间隔一般比较小。在容器正常启动后，存活探针就会一直周期性的对容器进行探测，如果存活探针失败，那么kubelet就会杀死容器，然后容器会根据重启策略进行重启。
- **readiness Probe(就绪探针)**：用于检查容器是否已经准备好接受请求，探测时间间隔一般比较小。在容器正常启动后，就绪探针也会一直周期性的对容器进行探测，如果就绪探针失败，那么就会将Service的endpoints中移除这个Pod

<font color='#ff0000'> **思考一个问题：从功能上看，startup Probe和liveness Probe的功能似乎是相似的，只是一个在容器启动阶段起作用，另一个在容器启动后起作用，并且探针失败后都是根据重启策略重启，那为什么不把容器启动阶段和启动成功后的阶段组合在一起，使用一个探针探测不行吗？**</font>

举个例子，对于一些java打包成的war包，动则几百MB，这样的情况下业务容器的启动时间可能会比较长，比如几十秒、几分钟、几十分钟。那么使用liveness Probe显然是不合适的，如果将liveness Probe的探测周期设置的比较长，显然不符合逻辑，因为我们需要在短时间内进行探测，以保证业务容器的正常，就算出现问题也能及时重启，如果探测周期设置的比较长显然不合理。反过来，把startup Probe的探测周期设置的短一点行不行呢？显然也不行，前面都说了业务容器可能启动的比较慢，如果startup Probe的探测周期很短，那么业务容器在还没有启动完成就又被重启了，一直这样无限循环，那么永远也无法启动成功容器。

综上所述，所以startup Probe一般会把探测周期设置的比较长，主要针对那种启动慢的容器；而liveness Probe会把探测周期设置的比较短，以保证容器出现问题能及时重启。所以需要这俩种形式的探针都是需要的。

**(2)探针的3种监测机制**：

- **Exec Action**：根据指定命令的结果状态码判定
- **TcpSocket Action**：根据相应的TCP套接字连接建立状态判定
- **HTTPGet Action**：根据指定的HTTP/HTTPS服务URL的响应结果判定
- GRPC

**(3)探针的配置参数**：

- **initialDelaySeconds(初始延迟秒数)**：：这个参数指定容器启动后的等待时间，直到开始执行首次探测，该值用于防止容器在启动时立即执行探测。
- **periodSeconds(周期秒数)**：这个参数定义了探测器进行连续探测的间隔时间。
- **timeoutSeconds(超时秒数)**：探测动作中每一次探测请求的超时时间，如果容器在定义的秒数内没有响应，该探测会被视为失败。
- **successThreshold(成功阈值)**：连续探测成功定义的次数，容器才会被标记为健康。
- **failureThreshold(失败阈值)**：连续探测失败定义的次数，容器才会被标记为不健康。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-probe-demo
  namespace: default
spec:
  containers:
  - name: demo
    image: ikubernetes/demoapp:v1.0
    imagePullPolicy: IfNotPresent
    startupProbe:
      exec:
        command: ['/bin/sh','-c','test','"$(curl -s 127.0.0.1/livez)"=="OK"']
      initialDelaySeconds: 0
      failureThreshold: 3
      periodSeconds: 2
    livenessProbe:
      httpGet:
        path: '/livez'
        port: 80
        scheme: HTTP
      initialDelaySeconds: 3
      timeoutSeconds: 2
    readinessProbe:
      httpGet:
        path: '/readyz'
        port: 80
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 2
restartPolicy: Always
```

##### 2.4、Pod与容器安全上下文



##### 2.5、资源需求和资源限制



#### 3、Pod设计模式



#### 4、Pod状态及异常问题排查

- **Pending**

若Pod停留在Pending状态，说明该Pod不能被调度到某一个节点上。通常是由于资源依赖、资源不足、该Pod使用了hostPort、污点和容忍等原因导致集群中缺乏需要的资源

- **ImagePullBackOff和ErrImagePull**



CrashLoopBackOff



Running



Terminating



Evicted



Completed



Unknown

只要删除pod，卷的定义的被删掉的，卷是不能被服用的，复用的只能是卷存储的数据

### 五、存储卷、持久卷、存储架构以及CSI

#### 1、存储类型和相应的插件

- **In-Tree存储卷插件**

临时存储卷：emptyDir

节点本地卷：hostPath, local

网络存储卷：

​	文件系统：NFS、GlusterFS、CephFS和Cinder

​	块设备：iSCSI、FC、RBD和vSphereVolume

​	存储平台：Quobyte、PortworxVolume、StorageOS和ScaleIO

​	云存储：awsElasticBlockStore、gcePersistentDisk、azureDisk和azureFile

特殊存储卷：Secret、ConfigMap、DownwardAPI和Projected

- **Out-of-Tree存储卷插件**

经由CSI或FlexVolume接口扩展出的存储系统

#### 2、存储卷

##### 2.1、存储卷概念

存储卷定义在Pod之上，所以卷本身的生命周期与Pod相关联，或者说等同于Pod的生命周期。只要删除Pod，卷的定义也会被跟着删除，无法对卷进行复用，能复用的仅仅是卷持久化存储的数据。

##### 2.2、存储卷资源规范

存储卷的定义由俩部分构成:：spec.volumes和spec.containers.volumeMounts

```yaml
spec:
  volumes:
    #定义卷名
  - name: <volume_name>
    #定义使用何种存储卷，也就是存储介质类型
    volume_type:
      <object>
      ......
  containers:
  - image: ......
    name: ......
	.....
    volumeMounts:
      #引用上面定义的卷名
    - name: <volume_name>
      #定义容器内部的挂载路径
      mountPath: ......
      #定义卷是否为只读，true为只读，false为可读写，默认为false
      readOnly: ......
      #指定存储卷上的一个子目录挂载到容器的内部的mountPath，一般用于共享一个卷的不同部分到不同容器中，需要写对于卷的相对路径
      subpath: ......
      #可以使用环境变量来动态构建子路径名称
      subPathExpr: ......
```

##### 2.3、emptyDir

emptyDir存储卷是Pod上的一个临时目录，在Pod在启动的时候被创建，在pod对象被移除的时候一并被删除，数据无法持久保持。

适用场景：

- 同一个Pod中多个容器间文件共享
- 作为容器数据的临时存储目录用于数据缓存系统

emptyDir.medium的可选值有2个：Memory或default

- **Memory**：存储卷在内存中创建，性能会好点
- default：存储卷在本地磁盘上创建，访问速度会差点，性能会差点

**需要注意的是，当存储卷在内存创建时，需要对卷做限制，否则数据可能会占满内存。**

```yaml
#在此yaml文件中，是定义一个内存的存储卷，并且将其挂载到容器admin的/data目录下以及容器nginx的/usr/share/nginx/html目录下，所以admin的/data和nginx的/usr/share/nginx/html都共享着一块内存空间，我们只需要在admin的/data目录下新建一个index.html，那么nginx容器也能读取到这个页面
apiVersion: v1
kind: Pod
metadata:
  name: pods-with-emptydir-vol
spec:
  containers:
  - image: ikubernetes/admin-box:v1.2
    name: admin
    command: ["/bin/sh", "-c"]
    args: ["sleep 99999"]
    resources: {}
    volumeMounts:
    - name: data
      mountPath: /data
  - image: nginx:1.22-alpine
    name: nginx
    resources: {}
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    emptyDir:
      medium: Memory
      sizeLimit: 16Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

##### 2.4、hostPath

hostPath存储卷是将本地节点上的某目录当做存储卷，数据的生命周期与节点的生命周期相同。

hostPath.type支持多种挂载模式：

- **DirectoryOrCreate**：如果指定的目录不存在，则创建它并将主机上的目录挂载到容器中，权限设置为0755，与Kubelet有相同的组和属主信息
- **Directory**：要求指定的目录在主机上存在，然后将其挂载到容器中
- **FileOrCreate**：如果指定的文件不存在，则创建它并将主机上的文件挂载到容器中
- **File**：要求指定的文件在主机上存在，然后将其挂载到容器中

- **Socket**：要求指定的 Unix 套接字文件在主机上存在，并将其挂载到容器中
- **CharDevice**：要求指定的字符设备文件在主机上存在，并将其挂载到容器中
- **BlockDevice**：要求指定的块设备文件在主机上存在，并将其挂载到容器中

```yaml
#在此yaml文件中，目的是为了将redis容器的数据持久化到本地节点上。创建pod后，进入到redis容器中，适用redis-cli连接redis，再set mykey 1234，再bgsave一下，发现容器的/data目录下生成了一个dump.rdb文件，再查看此pod在哪个节点上，再到对应节点上的/data/redis目录下也存在dump.rdb文件，实现了数据的持久化
#如果此Pod删除重建并重新调度到其它节点上，则无法读取到此前的数据
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:6
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    hostPath:
      type: DirectoryOrCreate
      path: /data/redis
```

##### 2.5、nfs

```yaml
#需要提前准备nfs-server但是不需要将nfs-server挂载到node节点上，并且每个node节点都要安装nfs-common，因为不知道会调度到哪个节点上
#因为数据持久化是到nfs-server上的，所以不同于hostPath，即使Pod删除重建并重新调度到其它节点上，Pod依然可以读取到此前持久化的数据
apiVersion: v1
kind: Pod
metadata:
  name: redis-nfs
spec:
  containers:
  - name: redis
    image: redis:6
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    nfs:
      #配置nfs-server的地址
      server: 172.29.7.1
      path: /data/redis001
```

#### 3、持久卷

笔者理解：想象一下，如果没有PV、PVC，那么卷只能定义在Pod中，那么它的生命周期是跟Pod是一样的 ，当Pod删除时那么卷的定义也同时被删除了，无法对卷进行复用。并且在定义卷的时候，还会要求操作的人对存储要足够熟悉，要知道企业中最熟悉集群配置的人应该是运维了，那么如果是一个开发需要在Pod中定义一个卷，那往往会带来麻烦，因为他不熟悉存储等配置。

基于以上场景呢，K8S提供了持久卷的概念，也就是PV和PVC来解耦在Pod之上直接定义卷的这种耦合关系。由集群管理员来将存储定义为PV，定义这个PV是块设备还是文件系统，定义PV运行的空间上限、定义PV隶属的存储类型是standard还是gp2、io1、csi、local等等。集群的用户只需要定义一个PVC来描述Pod希望使用的属性，比如哪种存储空间？希望的空间大小？然后由PV Controller自动筛选出一个合适的PV与PVC进行绑定即可。

但是这会迎来一个新的问题，如果集群管理员定义的PV不能满足用户，那么就会造成问题，可能需要用户与集群管理员进行私下的协商，然后再有集群管理员按照需求创建PV，但是这会带来一系列的沟通成本。所以k8s也支持用户根据StorageClass来充当PV的模板来为PVC动态创建PV。

##### 3.1、PV、PVC的概念

PV和PVC都是集群的标准资源类型，并且PV和PVC是一对一的关系，一个PVC只能绑定一个PV，一个PV也只能绑定一个PVC。

- PV

PV是持久化存储卷，是集群级别的资源，负责将存储空间引入到集群中，通常由管理员定义

- PVC

PVC是名称空间级别的资源，描述的是Pod所希望使用的持久化存储的属性，由用户定义，用于在空闲的PV中申请使用符合过滤条件的PV

- StorageClass

充当PV的模板，自动为PVC创建PV，支持PV的动态预配

##### 3.2、PV资源与PVC资源

**PV资源**

spec.volumeMode：

- Filesystem(默认值)：表示当前PV提供的存储空间类型为文件系统
- Block：表示当前PV提供的存储空间类型为

spec.storageClassName：填有集群管理员定义好的StorageClass的名字，PV可以不进行定义此字段

spec.accessMode：

- ReadWriteOnce：单路读写，仅允许一个节点以读写模式挂载卷
- ReadOnlyMany：多路只读，允许多个节点以只读模式挂载卷
- ReadWriteMany：多路读写， 允许多个节点同时以读写模式挂载卷
- ReadWriteOncePod：单Pod单路只读，允许单个Pod以读写方式挂载卷

spec.capacity.storage：当前PV允许使用的空间上限

spec.persistentVolumeReclaimPolicy：

- Retain：默认策略，当PVC被删除时，PV仍然存在，并保留其所有的数据，但不能再被别的新建的PVC绑定，管理员需要手动清理数据并手动重新将PV置为可用状态
- Delete：当PVC被删除时，PV及其存储的数据也会被自动删除
- Recycle：当PVC被删除时，会清空PV的数据，PV可以与其它PVC进行绑定，如果其它PVC进行绑定以后数据被恢复，会造成数据安全问题，所在此策略在新版本中会被废弃

**PVC资源**

支持按照VolumeMode → LabelSelector → StorageClassName → AccessMode → Size的顺序筛选

支持动态预配的存储类，还可以根据PVC的条件按需完成PV创建

##### 3.3、静态PV和PVC配置卷的步骤

- 第一步，集群管理员先准备好存储
- 第二步，集群管理员将存储单元定义成PV
- 第三步，集群用户声明PVC
- 第四步，PV Controller为PVC筛选一个合适的PV并将其关联起来
- 第五步，集群用户定义一个Pod并引用定义好的PVC

![image-20231116201633757](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231116201633757.png)

##### 3.4、基于NFS的静态PV

```yaml
#创建静态PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path:  "/data/redis02"
    server: 172.29.7.1
 
#创建PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
spec:
  accessModes: ["ReadWriteMany"]
  volumeMode: Filesystem
  resources:
    requests:
      storage: 3Gi
    limits:
      storage: 10Gi

#在Pod上使用PVC卷
apiVersion: v1
kind: Pod
metadata:
  name: redis-pvc
spec:
  containers:
  - name: redis
    image: redis:6
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 6379
      name: redisport
    volumeMounts:
    - mountPath: /data
      name: redis-pvc-vol
  volumes:
  - name: redis-pvc-vol
    persistentVolumeClaim:
      claimName: pvc-nfs
```

##### 3.5、StorageClass资源

StorageClass是集群级别的资源类型，它是PVC的筛选条件之一

需要存储服务提供SC的管理API，nfs是没有这个API的，所以nfs并不能动态置备PV，但是如果nfs是通过csi的接口接入存储的话，就可以进行动态PV置备

PVC和PV要么都不属于任何StorageClass，要么同时属于同一个StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
#指定用于动态卷配置的存储插件
provisioner: nfs.csi.k8s.io
#指定传递给 provisioner 的配置参数
parameters:
  #server: nfs-server.default.svc.cluster.local
  server: nfs-server.nfs.svc.cluster.local
  share: /
#指明PVC在被删除后，处理PV的策略
reclaimPolicy: Retain
#指明在PV在被创建后是否立即关联PVC，WaitForFirstConsumer表示延迟绑定
volumeBindingMode: Immediate
#指定挂载PV时应用的文件系统选项
mountOptions:
  - hard
  - nfsvers=4.1
```

##### 3.6、动态PV和PVC配置卷的步骤

第一步，集群管理员先准备好存储

第二步，集群管理员将定义StorageClass

第三步、集群用户声明PVC并引用定义好的StorageClass，k8s会自动为这个PVC创建一个PV并关联起来

第四步、集群用户定义一个Pod并引用定义好的PVC



##### 3.7、基于NFS的动态PV

```yaml
#定义StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.nfs.svc.cluster.local
  share: /
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1

#定义PVC关联StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-csi

#定义pod并引用PVC卷
apiVersion: v1
kind: Pod
metadata:
  name: redis-pvc
spec:
  containers:
  - name: redis
    image: redis:6
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 6379
      name: redisport
    volumeMounts:
    - mountPath: /data
      name: redis-pvc-vol
  volumes:
  - name: redis-pvc-vol
    persistentVolumeClaim:
      claimName: pvc-nfs-dynamic
```

#### 4、K8S存储架构及CSI

##### 4.1、存储架构

存储卷的具体的管理操作由相关的控制器向卷插件发起调用请求来完成的

- **AD Controller**

一个PVC如果想关联一个NFS的存储单元，首先需要把NFS的存储单元关联到集群节点的内核中去，由内核来进行驱动，然后才能被定义为PVC，最少再被Pod使用。一旦PVC被删除，那么NFS的存储单元将不再会关联到节点。

Attach：将设备附加到目标节点，也就是将外部存储单元关联到集群节点的内核中去

Detach：将设备从目标节点上拆除，也就是取消外部存储单元与集群节点关联的过程

- **PV Controller**

负责PV/PVC的绑定、生命周期管理，以及存储卷的Provision/Delete操作

- **Volume Manager**

在谈到AD Controller的时候，提到的都是外部存储单元都是与节点相关联，实际上我们要外部存储与Pod相关联，具体关联到Pod是由Kubelet中内置的Volume Manager的来做的，它负责将挂载到当前节点的卷关联到容器里面。

Volume Manager负责卷的Mount/Umount操作，以及设备的格式化操作等

![image-20231117172107255](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231117172107255.png)

##### 4.2、Out-of-Tree 存储

如果说kubelet内置的存储插件不满足要求，我们就需要通过CSI来进行扩展卷插件，并且我们还需要控制器来控制csi插件来做一下attach、detach等操作。遗憾的是，AD Controller和PV Controller都不支持CSI插件，它们只支持内置的卷插件，所以我们还需要安装CSI Controller来完成AD Controller和PV Controller的部分功能。

所以，如果要支持Out-of-Tree 存储，那么需要部署相应的驱动程序组件：

- CSI Controller：负责与存储服务的API通信从而完成后端存储的管理操作。由于是有状态的，所以需要StatefulSet控制器对象编排运行，如果有需要还要做多副本高可用
- Node Plugin：CSI插件，负责在节点级别完成存储卷的管理，由DaemonSet控制器对象编排运行

如果需要安装CSI插件，则每个节点上都要安装，包括master节点，因为它们底层都依赖了volume plugin。另外还需要



如果k8s想要树外的csi插件，那么在每个节点上都需要安装这个插件，并且由于k8s内置的AD Controller和PV Controller并不支持外部的csi插件，所以我们还需要安装一个csi Controller来完成AD Controller和PV Controller的功能

##### 4.3、部署NFS CSI

```
https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/README.md
```

#### 5、应用配置之ConfigMap和Secret基础

部署一个服务，往往会涉及到配置信息。比如安装一个mysql主从，我们需要一个具有复制权限的账号密码；比如安装nginx，我们也需要一个配置文件。

那么，在K8S集群中，在Pod内部的容器，当它需要加载一个配置的时候，如何加载呢？

在回答上面的问题前，先思考一个问题：对于一个容器而言，如何进行需要的配置呢？

- 通过环境变量env进行配置，但是镜像支持的env可能并不能满足你的需求，不如MySQL官方镜像是不支持主从复制账号的env的
- 通过自制镜像，但是每修改一次，你可能都需要去加一层镜像，代价太大
- 通过节点卷来进行配置，但是这就要求每个节点上都要有这个卷才行
- 通过网络卷来进行配置，但是这就强依赖于外部存储了，如果存储挂掉，则所有应用都受到影响

那么，K8S集群有没有自己的一个存储呢？ETCD不就是吗？我们只需要将配置信息存入ETCD，应用需要配置信息的时候从其中加载不就行了吗？但是这里需要需要注意，从ETCD里面存储数据还是读取数据，都是需要通过apiserver来实现的。而apiserver呢，它是把集群的所有信息都抽象成了资源对象来进行管理。

实际上，ConfigMap和Secret资源就是为了容器中的应用提供配置数据而抽象出来的资源对象，降低了了配置和镜像的耦合关系。

##### 5.1、ConfigMap和Secret资源概念

ConfigMap和Secret是数据承载类的资源类型，因而一般不使用spec字段，而是直接通过一级字段定义其配置

**ConfigMap**

- 所有ConfigMap类型的对象，可以被同一个namespace下的Pod，通过configMap卷插件作为卷关联并挂载使用
- ConfigMap保存应用的配置信息，扮演整个集群上所有应用的配置中心并且所有数据明文保存

**Secret**

- 所有Secret类型的对象，可以被同一个namespace下的Pod，通过secret卷插件作为卷关联并挂载使用
- Secret是ConfigMap的补充，用于保存敏感信息而且所有数据被Base64编码后保存，并结合单独的授权来确保其安全性

##### 5.2、创建ConfigMap对象的三种方法

- 第一种方法：kubectl create configmap NAME --from-literal=key1=value1 --from-literal=...

第一种方法中可以使用多个--from-literal来指定多个key-value值，并且key值是不能省略的

- 第二种方法：kubectl create configmap NAME --from-file=[key1]=/PATH/TO/FILE ...

第二种方法中可以使用多个--from-file来指定多个具体的配置文件，并且key值是可以省略的，如果省略了key值默认就是文件名

- 第三种方法：kubectl create configmap NAME --from-file=/PATH/TO/DIR/

第三种方法中使用的是用--from-file来指定一个目录，目录下的所有文件都生效，并且不能指定key值

##### 5.3、ConfigMap中定义的配置在容器生效的俩种办法

**(1)向环境变量赋值，值来源于某个configmap的某个键**

注意事项：赋值过程发生Pod创建时，因此Pod创建完成后，修改ConfigMap中配置信息并不非生效，直到重新再创建该Pod

```yaml
#普通的环境变量赋值
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: demo
spec:
  containers:
  - name: demoapp
    image: ikubernetes/demoapp:v1.0
    env:
    - name: HOST
      value: 127.0.0.1
    - name: PORT
      value: "8080"    

#使用ConfigMap的形式对环境变量赋值
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-demo
  namespace: demo
spec:
  containers:
  - name: demoapp
    image: ikubernetes/demoapp:v1.0
    env:
    - name: HOST
      valueFrom:
        configMapKeyRef:
          name: demoapp-cfg
          key: listen
    - name: PORT
      valueFrom:
        configMapKeyRef:
          name: demoapp-cfg
          key: port
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demoapp-cfg
  namespace: demo
data:
  listen: 127.0.0.1
  port: "8080"
```

**(2)使用configmap插件，基于卷接口来引用configmap中的配置**

注意事项：

容器挂载目录下的文件是两级软链接并且在挂载点下生一个以时间戳命名的隐藏目录，该目录被..data引用，如果配置发生变更，会更新挂载为新的时间戳命名的目录，但是时间会有延迟，不是立即生效的。

```yaml
#ConfigMap的定义可以由kubectl create configmap NAME --from-file=/PATH/TO/DIR/ --dry-run=client -o yaml来生成，要不然
apiVersion: v1
kind: Pod
metadata:
  name: configmaps-volume-demo
  namespace: default
spec:
  containers:
  - image: nginx:latest
    name: nginx-server
    volumeMounts:
    - name: ngxconfs
      mountPath: /etc/nginx/conf.d/
      readOnly: true
  volumes:
  - name: ngxconfs
    configMap:
      name: nginx-cfg
      optional: false
---
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: nginx-cfg
data:
  gzip.cfg: |
    gzip on;
    gzip_comp_level 5;
    gzip_proxied     expired no-cache no-store private auth;
    gzip_types text/plain text/css application/xml text/javascript;
  myserver.conf: |
    server {
        listen 8080;
        server_name www.ik8s.io;

        include /etc/nginx/conf.d/myserver-*.cfg;

        location / {
            root /usr/share/nginx/html;
        }
    }
```



### 六、Pod控制器

#### 1、ReplicaSet Controller



#### 2、Deployment Controller

##### 2.1、创建一个deployment的过程

- 客户端将创建deployment的请求发送给apiserver
- apiserver对客户端的请求进行认证、授权、准入控制、验证后，将数据持久化存储到etcd中，etcd会返回给apiserver一个创建deployment对象的事件
- Deployment Controller通过watch接口一直监听着apiserver上deployment对象的事件，当它监听到这个事件后，会向apiserver发起创建ReplicaSet的请求，同时将RS的信息存储到etcd中，etcd向apiserver返回一个创建RS的事件
- ReplicaSet Controller通过watch接口一直监听着apiserver上RS对象的事件，当它监听到这个事件后，会向apiserver发起创建Pod的请求，同时将Pod的信息存储到etcd中，etcd向apiserver返回一个创建Pod的事件
- Scheduler通过watch接口一直监听apiserver上Pod对象的事件，当它监听到这个事件后，会对nodename为空的Pod，根据调度算法为这些Pod调度到某个node节点，然后Scheduler会将信息发送给apiserver，由apiserver将数据存储到etcd中
- kubelet通过watch接口一直监听着apiserver上Pod事件，它会筛选出Pod的nodename字段与本节点名称相同的Pod，根据跟据Pod的定义来调用CRI接口拉取镜像启动容器

##### 2.2、编写yaml文件部署Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: demoapp
      release: stable
  template:
    metadata:
      labels:
        app: demoapp
        release: stable
    spec:
      containers:
      - name: demoapp
        image: ikubernetes/demoapp:v1.0
        ports:
        - containerPort: 80
          name: http
        livenessProbe:
          httpGet:
            path: '/livez'
            port: 80
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
            path: '/readyz'
            port: 80
          initialDelaySeconds: 15
```

##### 2.3、更新Deployment

**(1)更新方法：**

```bash

kubectl set image

kubectl edit deployment deployment_name

#先导出yaml文件再apply是预防中间有更新未使用原始的yaml来更新，而导出的yaml文件肯定是最新的，只需要把多余字段删除再更新即可
kubectl get deployment deployment_name -o yaml >xxx.yaml
kubectl apply -f xxx.yaml
```

**(2)更新策略：**

Deployment Controller支持2种更新策略：**滚动更新(rolling update)和重新创建(recreate)**。默认策略为滚动更新。

- **重新创建**

步骤：先删除现有的所有旧版本的Pod，然后再创建新版本的Pod，并且是在一个RS中进行的

特点：在服务的更新期间会导致服务的可不用

应用场景：也有其适用场景，比如业务要求不能同时运行俩个不同的版本或新版本不兼容旧版本的数据库等场景

- **滚动更新**

步骤：先创建一个RS，并根据更新策略设置新的RS的副本数量并创建Pod副本，这时旧的RS的Pod仍保持正常。旧的RS会按照设定的更新策略逐步删除Pod。重复上述过程，新的RS继续创建Pod,同时旧的RS继续按比例删除Pod，直到新的RS的Pod数量与旧的RS初始Pod数量一致，并且旧的RS中的Pod全部删除，那么滚动更新完成

特点：在服务更新期间不会造成服务的不可用

应用场景：最常用的更新策略，适用于无停机的业务，支持新老版本并行

注意事项：

在滚动更新中，新的RS创建pod数量和旧的RS删除Pod数量是受俩个参数控制的：**MaxSurge和MaxUnavailable**

- MaxSurge：定义了在更新期间可以超出期望副本数的最大Pod数量
- MaxUnavailable：定义了在滚动更新过程中可以不可用的最大Pod数量

假设原始副本数为5，并且将MaxSurge设置为2，将MaxUnavailable设置为1。那么在更新过程中就会要求旧的Pod数量和新的Pod数量之和要大于等于4(原始副本数-MaxUnavailable)，旧的Pod数量和新的Pod数量之和要小于等于7(原始副本数+MaxSurge)。

值得一提的是，MaxSurge和MaxUnavailable的值不能同时为0，否则会造成死锁，相当于要求旧的Pod数量和新的Pod数量之和要大于等于原始副本数又要小于等于原始副本数，那就只能等于原始副本数，则无法进行更新。

##### 2.4、回滚Deployment

```bash
#查看deployment历史版本
kubectl rollout history deploymentdeployment历史版本 deployment_name -n namespace_name
#回滚到上一版本
kubectl rollout undo deployment deployment_name -n namespace_name 
#回滚到指定版本
kubectl rollout undo deployment deployment_name --to-revision=2 -n namespace_name
```

##### 2.5、扩容与缩容

```bash
kubectl edit deployment deployment_name -n namespace_name
kubectl scale deployment deployment_name --replicas=5 -n namespace_name
```

##### 2.6、暂停与恢复

适用于需要对Deployment进行多次更新的场景，先暂停Deployment的更新，然后进行多次更新的操作，所以更新操作都做完后，再恢复Deployment的更新，这样就可以将多次更新变为一次更新，对性能会更好。

```bash
#暂停正在进行的deployment更新
kubectl rollout pause deployment deployment_name -n namespace_name

#恢复deployment更新
kubectl rollout resume deployment deployment_name -n namespace_name
```

#### 3、StatefulSet Controller

##### 3.1、编写yaml文件部署statefulset

```yaml
kind: StatefulSet
metadata:
  name: sts-demo
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      #partition为3，更新镜像的话，只有sts-demo-3这个pod的版本更新；partition为0，那么更新镜像，则所有pod，逆序一次更新版本。具有大于或等于分区序数的所有 Pod 将被更新。
      partition: 3
  serviceName: demoapp-sts
  replicas: 2
  selector:
    matchLabels:
      app: demoapp
      controller: sts-demo
  template:
    metadata:
      labels:
        app: demoapp
        controller: sts-demo
    spec:
      containers:
      - name: demoapp
        image: ikubernetes/demoapp:v1.0
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: appdata
          mountPath: /app/data
  volumeClaimTemplates:
  - metadata:
      name: appdata
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-csi"
      resources:
        requests:
          storage: 2Gi
```

##### 3.2、StatefulSet更新策略

StatefulSet支持2种更新策略：OnDelete和RollingUpdate，并且自Kubernetes1.7版本开始，StatefulSet默认的更新策略为RollingUpdate

- **OnDelete**

在这种更新策略下，修改StatefulSet资源Pod模板的字段后，StatefulSet控制器并不会自动更新pod，需要对Pod进行手动删除，StatefulSet控制器才会按照新的pod模板创建新的Pod

- **RollingUpdate**

默认策略，在这种更新策略下，修改StatefulSet资源pod模板的字段后，StatefulSet控制器会以pod启动的顺序的逆序的形式依次更新Pod

另外RollingUpdate策略还支持分段更新，通过spec.updateStrategy.rollingUpdate.partition字段来控制更新哪些Pod，partition: 3表示分区大于等于3的才会被更新，比如demo-3,demo-4.....，并且是逆序更新，先更新demo-4，再更新demo-3，这样就实现了一个灰度发布的功能

##### 3.3、删除StatefulSet

删除StatefulSet有俩种方式：级联删除和非级联删除

- **级联删除**

使用级联删除StatefulSet时，StatefulSet中的Pod也会被删除

```bash
kubectl delete sts statefulset_name
kubectl delete -f xxx.yaml
```

- **非级联删除**

使用非级联删除StatefulSet时，StatefulSet中的Pod不会被删除

```bash
kubectl delete sts statefulset_name --cascade=false
kubectl delete -f xxx.yaml --cascade=false
```

##### 3.4、使用StatefulSet编排运行redis



##### 3.5、使用StatefulSet编排运行mysql



##### 3.6、使用StatefulSet编排运行nacos



#### 4、DaemonSet Controller

##### 4.1、DaemonSet的编排机制

- DaemonSet与Deployment相似，是基于标签选择器来管控一组Pod
- DaemonSet只会的匹配的节点上运行一个Pod，并且如果后面有新的节点加入集群并且符合条件的话，此节点也会新建Pod
- DaemonSet是否会在节点上创建Pod与定义的节点选择器、污点与容忍、节点亲和性、资源限制、节点状态均有关
- DaemonSet一般不会在master节点上创建Pod，只会在集群的节点上创建Pod，因为master节点默认带有污点

##### 4.2、DaemonSet更新策略

DaemonSet支持2种更新策略：OnDelete和RollingUpdate

- **OnDelete**

需要手动删除已经存在的Pod，新的Pod才会被创建更新

- **RollingUpdate**

集群会自动更新Pod

##### 4.3、使用DaemonSet编排运行fluentd



##### 4.4、使用DaemonSet编排运行node-exporter

#### 5、Job Controller与CronJob Controller

5.1、



5.2、



### 七、Service与服务发现

#### 1、service资源基础概念

##### 1.1、service资源

service是K8S集群API资源类型之一，它主要有2个功能：

- 为动态的Pod资源提供流量入口

服务发现：通过标签选择器来筛选同一名称空间下的Pod的资源标签。但其实实际上是由与Service同名的Endpoint或EndpointSlice资源及控制器完 成

流量调度：由运行各工作节点的kube-proxy根据配置的模式生成相应 的流量调度规则

- 持续的监视着service相关Pod资源的变动，并实时反馈至相应的流量调度规则之上

##### 1.2、service客户端负载均衡

在service眼中，K8S集群中每个节点都是动态可配置的负载均衡器

客户端负载均衡，首先service要通过标签选择器与相关的pod关联起来，然后客户端可以通过coredns对service的名称进行解析得到ip，通过service发现其后端pod地址，然后客户端所在节点的内核将流量转发到对应的pod中，客户端节点的内核就充当了负载均衡器，当然内核流量转发的iptables或ipvs规则是依靠kube-proxy来实现的

##### 1.3、service资源示例

curl clusterip 

host -t  A svc_name.namespace.svc.cluster.local

节点上是无法解析svc_name.namespace.svc.cluster.local的，因为节点的dns是指向外网的，要解析svc_name.namespace.svc.cluster.local需要到pod中去，因为pod中的dns解析是指向

#### 2、service类型

##### 2.1、ClusterIP

用于在集群内部互相访问的场景，只有集群中资源可以访问ClusterIP

##### 2.2、NodePort

用于从集群外部访问的场景，将Pod的端口映射到节点上的端口，将集群外部的流量接入service

端口范围：30000-32767

```
kind: Service
apiVersion: v1
metadata:
  name: demoapp-nodeport-svc
spec:
  type: NodePort
  #clusterIP: 10.97.56.1
  selector:
    app: demoapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
    #nodePort: 31398
```

##### 2.3、LoadBalancer

用于从集群外部访问的场景，其实是nodeport的一种扩展，过一个特定的LoadBalancer访问Service，这个LoadBalancer将请求转发到节点的NodePort，而外部只需要访问LoadBalancer

##### 2.4、ExternalName

将集群外部的服务引入到进群内部，从而使得集群的内部的客户端访问集群外部的service像访问集群内部的service一样

Pod 请求连接到 externalname-redis-svc；CoreDNS 收到请求并返回redis.ik8s.io作为结果；pod会向coreDNS申请解析redis.ik8s.io这个地址，coreDNS不能解析则会发给上游的DNS服务器进行解析的，解析到mysql的ip

```
kind: Service
apiVersion: v1
metadata:
  name: externalname-redis-svc
  namespace: default
spec:
  type: ExternalName
  externalName: redis.ik8s.io
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
    nodePort: 0
```

##### 2.5、Headless Service

用于Pod间的互相发现，该类型的Service并不会分配单独的ClusterIP， 而且集群也不会为它们进行负载均衡和路由。您可通过指定spec.clusterIP字段的值为“None”来创建Headless Service

#### 3、service流量转发

##### 3.1、service流量策略

internalTrafficPolicy：{Cluster|Local}
externalTrafficPolicy：{Local|Cluster} 

3、service名称解析





### 八、Ingress





### 九、认证体系与Service Account

#### 1、Kubernetes的访问控制体系

##### 1.1、API Server及其各客户端的通信模型



##### 1.2、API Server内置的访问控制机制

Kubernetes主要通过API Server对外提供服务，Kubernetes对于访问API的用户提供了相应的安全控制：认证和授权，认证解决用户是谁的问题，授权解决用户能做什么的问题，只有通过合理的权限控制，才能够保证整个集群系统的安全可靠

API访问需要通过以下三个步骤：

- 认证：核验请求者身份的合法性
- 授权：核验请求的操作是否获得许可
- 准入控制：检查操作内容是否合规

![image-20231206141815254](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231206141815254.png)



#### 2、API的身份验证

##### 2.1、Kubernetes上的用户



2.2、



### 十、鉴权体系与RBAC

#### 1、Kubernetes鉴权体系



创建一个serviceaccount

根据需要访问的资源是名称空间资源还是集群的资源，创建一个role或clusterrole

然后创建一个rolebinding或clusterrolebinding资源，将serviceaccount与role或clusterrole绑定在一起

然后再在delopyment中配置serviceaccount



当你创建一个 `ServiceAccount` 时，Kubernetes 会自动创建一个类型为 `kubernetes.io/service-account-token` 的 Secret。这个 Secret 包含用于访问 API 服务器的令牌。然后k8s会将这个Secret自动挂载到serviceaccount的pod下，pod与apiserver进行通信时就能拿这个令牌进行验证



ClusterRole：可以对集群的所有资源进行操作

ClusterRoleBinding：允许赋予别人权限，让别人有集群所有资源的操作权限

Role：可以对对应的namespace资源进行操作

RoleBinding：允许赋予别人权限，让别人有对应的namespace资源的操作权限

### 十一、Kubernetes调度体系

#### 1、nodeName调度

```yaml
#将pod调度到k8s-worker-2这个节点上
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  namespace: demo
  labels:
    app: webapp
spec:
  nodeName: 'k8s-worker-2'
  containers:
    - name: webapp
      image: nginx
      ports:
        - containerPort: 80
```

#### 2、nodeSelector调度

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  namespace: demo
  labels:
    app: webapp
spec:
  nodeSelector:
    # 选择调度到具有这个label的节点
    "special-app": "specialwebapp"
  containers:
    - name: webapp
      image: nginx
      ports:
        - containerPort: 80
```

#### 3、污点与容忍



#### 4、节点亲和

4.1、



4.2、



#### 5、Pod亲和



### 十二、污点与容忍

node节点上打污点，在pod设定容忍，不同容忍node污点的pod调度到其它node节点上



NoExecute：如果 Pod 没有忽略该污点，并且已经运行在节点上，它将被驱逐出该节点；如果 Pod 没有忽略该污点，它也不会被调度到该节点

NoSchedule：如果 Pod 没有忽略该污点，它将不会被调度到该节点

PreferNoSchedule：集群将尽量避免将 Pod 调度到该节点，但这不是强制性的



kubectl taint nodes <node-name> gpu=true:NoSchedule

```
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda-container
    image: cuda-image
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### 十三、Helm

1、

### 十四、CNI网络插件

#### 1、Pod的网络解决方案



#### 2、Flannel

##### 2.1、Flannel简介

Flannel由CoreOS研发，兼容CNI插件API,支持Kubernetes、OpenShift、Cloud Foundry、Mesos、Amazon ECS、 Singularity和OpenSVC等平台

##### 2.2、Flannel地址分配

Flannel在地址分配方面默认是使用10.244.0.0/16划分节点网络，再在节点网络的基础上划分Pod地址

每个节点的地址是10.244.x.0/24，x的范围是0~2^8-1,也就是0-255，也就说最多可以有256个节点，节点超出这个数无法分配ip地址

每个Pod的地址则是10.244.x.y，由于全0是网络地址，全1是广播地址，像10.244.0.0、10.244.0.255到10.244.255.0、10.244.255.255的IP地址是无法使用的，所以y的范围是1~2^8-1-1，也就是1~254，也就是说每个节点最多有254个Pod

思考一个问题：如果将默认的专有网络10.244.0.0/16改为10.244.0.0/14或10.244.0.0/18呢？

##### 2.3、Flannel实现原理

Flannel会给每个Pod创建一对veth pair，其中一端是Pod中的eth0网口，一端则是宿主机上的cni0网桥的端口，Pod中从网卡eth0发出的流量都会发送到Cni0网桥上，然后再根据目标ip地址，再决定将报文发送到宿主机物理网口还是同节点Pod的虚拟网口，Cni0设备获得的ip地址是该节点分配到的网段的第一个地址

另外，Flannel还支持配置backend定义Pod间的通信方式，支持基于UDP、VXLAN的Overlay隧道网络以及基于3层路由HOST-GATEWAY的Underlay承载网络

##### 2.4、Flannel之VXLAN模式

VXLAN模式会在宿主机上生成一个VXLAN虚拟网络接口flannel.1，Pod中从网卡eth0发出的报文都会被送到Cni0网桥上

- 如果目标ip地址是同节点Pod的地址，则直接通过Cni0网桥路由到目标Pod的虚拟网络接口
- 如果目标ip地址是不同节点Pod的地址，则报文会被送到VXLAN虚拟网络接口flannel.1，并且在外部封装一层VXLAN头，其中包含了目标Pod所在节点的地址信息，封装好的VXLAN数据包会被发送到宿主机的物理网口并传输到目标节点，然后再由目标节点的物理网口将数据包发送给节点上的VXLAN虚拟网络接口flannel.1，进行解封装，然后再由Cni0网桥将数据包路由到对应的Pod
- 如果目标ip地址是集群外部的ip地址，则报文会被送到网络栈中做SNAT，将源Pod地址转换为公网IP地址，然后发送到宿主机的物理网络接口发出去

并且VXLAN模式还支持直接路由模式，如果启用该模式则处于同一二层网络内的节点的Pod间的通信可以直接通过路由的通信，而无需通过隧道网络的封装，因为封装一层VXLAN头始终会有性能的损失，而如果俩个节点垮了路由器，则无法通过直接路由的形式进行Pod通信，因为直接路由是通过查存储在etcd中的网络信息来实现的，而路由器并没有集群的网络信息，所以这个时候只能通过封装隧道网络来实现

总结一下就是VXLAN模式，不仅支持隧道网络的形式还支持直接路由的形式实现Pod间的通信，这需要开启直接路由模式，这样Pod的网络通信会自动选择是通过直接路由还是通过隧道网络实现

##### 2.5、Flannel之UDP模式

UDP模式与VXLAN模式不同的是，在UDP模式下，当一个Pod需要与另一个位于不同节点的Pod通信时，数据包会被发送到Cni0网桥上，然后被发送到flannel.0网口上，然后由flannel.0将数据包发送到flanneld进程中，flanneld会从etcd中找到Pod所在节点的地址信息，将目标节点IP地址封装在一个UDP数据包内由宿主机物理网卡发送到对应节点

flannel.0是一个TUN设备，作用是在操作系统内核和用户应用程序之间传递 IP 包

因为UDP模式存在数据包从用户态到内核态数据的来回拷贝的过程，也就是flanneld是运行在用户态的，数据包从发到flanneld又由flanneld发出，会经历2次数据包从用户态到内核态数据的来回拷贝，所以UDP模式的性能非常差

```bash
https://www.cnblogs.com/zlw-xyz/p/14968730.html
```

![image-20231216194722929](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231216194722929.png)

![image-20231216201350639](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231216201350639.png)

##### 2.6、Flannel之HOST-GATEWAY模式

HOST-GATEWAY模式类似于VXLAN模式中的直接路由模式，将Pod虚拟网络接口发出的数据包到Cni0网桥后，通过查找存储在etcd中的网络信息动态在节点生成路由表，直接找到目标Pod所在的节点地址，直接发给宿主机的物理网卡，然后发送到目标节点上，省去了封装与解封装的过程，性能肯定是好的，但是这个模式要求集群的所有节点都在同一个二层网络内部

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231216170241080.png" alt="image-20231216170241080" style="zoom:80%;" />

#### 3、Calico

##### 3.1、Calico简介



##### 3.2、Calico地址分配

192.168.0.0/16 划分子网192.168.x.y/26

节点数0-1023，每个节点的pod数量是2的6次方-2个，也就是62个



##### 3.3、Calico实现原理

CNI分为管理虚拟网络的插件、给容器或pod分配地址的插件



给ip首部帧首部再封装一层ip首部帧首部

外层封装的源ip是自身节点物理网卡的，目标ip是对端pod所在节点的物理网卡的

内层封装的源ip是自身pod的ip地址，目标ip是对端pod的ip地址

叠加网络、隧道网络

![image-20231213220643665](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231213220643665.png)

### 十五、Prometheus

#### 1、Prometheus简介

Prometheus 最开始是由 SoundCloud开发的开源监控告警系统，是Google BorgMon监控系统的开源版本。在 2016 年，Prometheus 加入CNCF，成为继Kubernetes之后第二个被CNCF托管的项目。随着Kubernetes在容器编排领头羊地位的确立，Prometheus也成为 Kubernetes容器监控的标配

##### 1.1、Prometheus特性

- 通过指标名称和标签(key/value对）区分的多维度、时间序列数据模型
- 灵活的查询语法 PromQL
- 不需要依赖额外的存储，一个服务节点就可以工作
- 利用http协议，通过pull模式来收集时间序列数据
- 需要push模式的应用可以通过中间件gateway来实现
- 监控目标支持服务发现和静态配置
- 支持各种各样的图表和监控面板组

##### 1.2、Prometheus架构



![image-20231130155113913](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231130155113913.png)

##### 1.3、Prometheus工作流程



#### 2、Prometheus的安装

##### 2.1、二进制安装

##### 2.2、容器安装

##### 2.3、Helm

##### 2.4、Prometheus Operator

##### 2.5、Kube-Prometheus Stack

Kube-Prometheus 项目地址：https://github.com/prometheus-operator/kube-prometheus/

(1)选择合适的分支版本克隆到k8s集群中

```bash
git clone -b [分支版本] https://github.com/prometheus-operator/kube-prometheus.git
git clone -b release-0.13 https://github.com/prometheus-operator/kube-prometheus.git

#切换到此目录下
cd kube-prometheus/manifests
```

(2)安装Prometheus Operator

```bash
kubectl create -f setup/
```

(3)查看Operator 容器的状态是否running

```bash
kubectl get po -n monitoring
```

(4)安装Prometheus Stack

```bash
#有些gcr的镜像地址需要修改
kubectl create -f .
```

(5)查看Prometheus容器状态是否正常

```bash
kubectl get po -n monitoring
```

#### 3、Prometheus监控

##### 3.1、Prometheus如何进行监控k8s集群？

Prometheus监控k8s集群一般是从云原生应用和非云原生应用俩个维度进行监控的

- **云原生应用**

包括节点、Pod、Service等资源，Prometheus通过metrics接口获取

- **非云原生应用**

包括MySQL、Redis、MongoDB等非云原生应用，Prometheus可以通过exporter来获取数据

##### 3.2、ServiceMonitor

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                              

##### 3.3、云原生应用Etcd监控

curl --cert /path/to/client.crt --key /path/to/client.key --cacert /path/to/ca.crt https://10.0.0.160:2379/metrics

etcd应该使用ssd硬盘，维持在10ms以下，50ms有点多

##### 3.4、非云原生应用MySQL监控



##### 3.5、黑盒监控

在监控体系中，通常可以将监控分为白盒监控和黑盒监控

- **白盒监控**

主要关注的原因，关注的是内部暴露的一些指标

- **黑盒监控**

主要关注的现象，站在用户角度能看到感知到的故障

在Prometheus中，黑盒监控主要是指在程序外部通过探针的方法进行模拟访问，来获取程序的响应指标来监控程序状态、如请求时间状态码等

Prometheus通过**Blackbox exporter**，来实现以HTTP、HTTPS、DNS、ICMP的方式探测目标

##### 3.6、Prometheus静态配置



##### 3.7、Prometheus监控外部Windows主机



#### 4、Prometheus高可用方案



#### 5、Prometheus联邦集群



#### 6、PromQL

##### 6.1、PromQL入门



6.2、



6.3、



#### 7、Alertmanager

##### 7.1、Alertmanager工作机制

在Prometheus监控系统中，数据的采集与警报是分离的，也就是说它们是属于2个不同模块的功能。其中数据的采集是由Prometheus Sever来完成的，并且可以通过定义于的PromQL 表达式的警报规则，当采集的数据满足条件或者说超过阈值，那么Prometheus Sever就会生成警报并且发送给Alertmanager。Alertmanager再对收到的警报进行分组、抑制和沉默，再通过定义好的路由规则转发到接收器，如钉钉、企业微信等，通过接收器将警报通知到对应的人

##### 7.2、Alertmanager配置文件详解

- Alertmanager的配置文件主要由以下模块组成：
- global(全局配置)：用于定义一些全局的公共参数，如全局的SMTP配置，Slack配置等内容
- templates(模板)：用于定义告警通知时的模板，如HTML模板，邮件模板等
- route(告警路由)：定义了如何处理传入的警报，包括分组、去重和沉默规则
- inhibit_rules(抑制规则)：抑制规则用于抑制某些警报，如果已经触发了其他更高优先级的警报，防止在出现大规模问题时收到大量重复的警报
- receivers(接收人)：定义了如何发送警报，每个接收人都指定了一种或多种通知方式，如电子邮件、Slack、Webhook 等

这些模块提供了分组、静默、抑制

```yaml
# global块配置下的配置选项在本配置文件内的所有配置项下可见
global:
  # 在Alertmanager内管理的每一条告警均有两种状态: "resolved"或者"firing". 在altermanager首次发送告警通知后, 该告警会一直处于firing状态,设置resolve_timeout可以指定处于firing状态的告警间隔多长时间会被设置为resolved状态, 在设置为resolved状态的告警后,altermanager不会再发送firing的告警通知.
  resolve_timeout: 1h	

  # 邮件告警配置
  smtp_smarthost: 'smtp.exmail.qq.com:25'
  smtp_from: 'dukuan@xxx.com'
  smtp_auth_username: 'dukuan@xxx.com'
  smtp_auth_password: 'DKxxx'
  # HipChat告警配置
  # hipchat_auth_token: '123456789'
  # hipchat_auth_url: 'https://hipchat.foobar.org/'
  # wechat
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
  wechat_api_secret: 'JJ'
  wechat_api_corp_id: 'ww'

  # 告警通知模板
templates:
- '/etc/alertmanager/config/*.tmpl'

# route: 根路由,该模块用于该根路由下的节点及子路由routes的定义. 子树节点如果不对相关配置进行配置，则默认会从父路由树继承该配置选项。每一条告警都要进入route，即要求配置选项group_by的值能够匹配到每一条告警的至少一个labelkey(即通过POST请求向altermanager服务接口所发送告警的labels项所携带的<labelname>)，告警进入到route后，将会根据子路由routes节点中的配置项match_re或者match来确定能进入该子路由节点的告警(由在match_re或者match下配置的labelkey: labelvalue是否为告警labels的子集决定，是的话则会进入该子路由节点，否则不能接收进入该子路由节点).
route:
  # 假如存在多个警报的标签job、altername、cluster、service、severity相应的值相同，那么将这些警报分在一组
  group_by: ['job', 'altername', 'cluster', 'service','severity']
  # 若一组新的告警产生，则会等group_wait后再发送通知，在group_wait时间内如果group_by里面定义的标签值相同则合并一起告警
  group_wait: 30s
  # 告警过一次警报组后，再次告警该时间间隔
  group_interval: 5m
  # 如果一条告警通知已成功发送，且在间隔repeat_interval后，该告警仍然未被设置为resolved，则会再次发送该告警通知
  repeat_interval: 12h
  # 默认告警通知接收者，凡未被匹配进入各子路由节点的告警均被发送到此接收者
  receiver: 'wechat'
  # 上述route的配置会被传递给子路由节点，子路由节点进行重新配置才会被覆盖

  # 子路由树
  routes:
  # 该配置选项使用正则表达式来匹配告警的labels，以确定能否进入该子路由树
  # match_re和match均用于匹配labelkey为service,labelvalue分别为指定值的告警，被匹配到的告警会将通知发送到对应的receiver
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: 'wechat'
    # 在带有service标签的告警同时有severity标签时，他可以有自己的子路由，同时具有severity != critical的告警则被发送给接收者team-ops-mails,对severity == critical的告警则被发送到对应的接收者即team-ops-pager
    routes:
    - match:
        severity: critical
      receiver: 'wechat'
  # 比如关于数据库服务的告警，如果子路由没有匹配到相应的owner标签，则都默认由team-DB-pager接收
  - match:
      service: database
    receiver: 'wechat'
  # 我们也可以先根据标签service:database将数据库服务告警过滤出来，然后进一步将所有同时带labelkey为database
  - match:
      severity: critical
    receiver: 'wechat'
# 抑制规则，当出现critical告警时 忽略warning
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  #   equal: ['alertname', 'cluster', 'service']
  #
# 收件人配置
receivers:
- name: 'team-ops-mails'
  email_configs:
  - to: 'dukuan@xxx.com'
- name: 'wechat'
  wechat_configs:
  - send_resolved: true
    corp_id: 'ww'
    api_secret: 'JJ'
    to_tag: '1'
    agent_id: '1000002'
    api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
    message: '{{ template "wechat.default.message" . }}'
#- name: 'team-X-pager'
#  email_configs:
#  - to: 'team-X+alerts-critical@example.org'
#  pagerduty_configs:
#  - service_key: <team-X-key>
#
#- name: 'team-Y-mails'
#  email_configs:
#  - to: 'team-Y+alerts@example.org'
#
#- name: 'team-Y-pager'
#  pagerduty_configs:
#  - service_key: <team-Y-key>
#
#- name: 'team-DB-pager'
#  pagerduty_configs:
#  - service_key: <team-DB-key>
#  
#- name: 'team-X-hipchat'
#  hipchat_configs:
#  - auth_token: <auth_token>
#    room_id: 85
#    message_format: html
#    notify: true 
```

##### 7.3、Alertmanager高可用



coredns

```

```







本人2年运维工作经验，工作内容涉及各种中间件，如nginx、tomcat、redis、kafka等中间件的部署调优，有k8s集群搭建及使用的经验，有兴趣欢迎私聊



1、配置startup探针和liveness探针有什么用，最后都是通过重启策略来重启，那么直接配置重启策略不就行了吗？



多容器pod：超亲密关系的容器，必须要共享network、ipc、uts的容器才行或者需要共享文件系统

比如在tomcat上安装filebeat收集日志给elastic，如果filebeat不在tomcat可能会有问题







创建一个pod，先kubectl create 或者 kubectl apply，apiserver接收到pod的创建信息后，将信息存储到etcd；

这个时候nodename字段是空的，scheduler从apiserver那里监听到pod信息的变化，nodename字段是空的，然后他给pod调度到一个node节点上去并且将nodename这个填充进去。

然后对应nodename节点的kubelet从apiserver那里监听到pod的信息变化，就通过cri调用容器运行时来拉取镜像创建容器，最后kubelet将pod的信息回存到apiserver中，就是status字段中的信息。

如果pod宕机，kubelet会监控到，然后是否重启取决于重启策略，pod级别

容器的镜像镜像的拉取策略：

ifnotpresent：如果目标wokenode上不存在镜像时才去拉取。如果镜像的tag是latest时，就算策略是ifnotpresent，也会always拉取镜像

always：总是拉取新的镜像。  

none：不去自动拉取镜像，需要手动提前拉取



nodename填值，可以调度到指定的workernode上，但是在大多数时候交给scheduler调度才是最优选择。也可以调度到一个范围的node节点上，可以给这些node打标签，然后使用标签选择器就可以



pod相位

pending 没有调度完成



kubelet每隔几分钟都会报告一次pod的状态



pod里面定义环境变量(如果pod中指定的容器镜像支持通过环境变量来进行传值的话)

pod.spec.containers



```yaml
#pod配置环境变量示例
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace: demo
  labels:
    app: mysql
    version: "8"
spec:
  containers:
  - name: mysql
    image: mysql:8
    imagePullPolicy: IfNotPresent
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "magedu.com"
    - name: MYSQL_USER
      value: "wpuser"
    - name: MYSQL_PASSWORD
      value: "wppass"
    - name: MYSQL_DATABASE
      value: "wpdb"
  restartPolicy: Always
```

service与pod的namespace要一样

service_name.service_namespace_name.svc.cluster.local = service_name.service_namespace_name = service_ip

mysql只用于内部应用访问，所以service用clusterip类型，而wordpress用外部访问 所以nodeport



有状态应用：有状态应用在处理请求时保存了一些信息，它们依赖于之前的交互或事务。它们可能会保存用户会话信息、有关用户的个人设置、事务历史、数据库的内部状态等。

无状态应用：无状态应用不保存客户端的任何数据。当它们处理请求时，不需要知道之前发生了什么，也就是说，它们不保持任何历史状态。每个请求都被当作全新的，不依赖于之前的请求。



有个k8s经常被问到的问题？为什么你们公司用的master节点是3个，不是2个、4个？ 

首先master组件中的etcd组件是基于raft协议来实现的，raft要求存活的节点数量要大于n/2才能选举出leader。

如果etcd节点是2个，如果坏掉1个节点，剩下1个节点并没有大于n/2，所以并不能选出leader。

如果etcd节点是3个，如果坏掉1个节点，剩下2个几点大于n/2，可以选出leader；如果坏掉2个节点，那就选不出leader了。

如果etcd节点是4个，如果坏掉1个节点，剩下3个几点大于n/2，可以选出leader；如果坏掉2个节点，剩下2个节点并没有大于n/2，无法选出leader。

综上所述，2个节点得容错是0个不满足可用性；3个节点4个节点得容错都是1个，4个节点显然更浪费资源，所以3个较为合理。至于k8s得master节点其它组件中，api-server不涉及选举，几个节点都可以。而scheduler和controller manager虽然是需要选举，但是它们是基于etcd进行选举，也没有要求像etcd一样要大于n/2啥的，所以2、3、4个我认为都可以。然后我是基于kubeadm安装的，那么其它组件和etcd是捆绑安装，所以数量保持一致。

Kubernetes的scheduler和controller manager使用了基于etcd的领导者选举机制来确保在任何时候，每种类型的控制器只有一个实例在运行。

这个scheduler或controller manager领导者选举机制的基本工作原理如下：

1. 每个scheduler或controller manager实例都会尝试在etcd中创建一个带有TTL（Time-To-Live，生存时间）的锁。这个锁通常是一个特定的键值对。
2. 如果创建成功，那么这个实例就成为领导者，并开始执行调度或控制器管理的任务。
3. 如果创建失败（因为锁已经存在），那么这个实例就成为跟随者，不执行任务，只是等待领导者失效或者放弃锁。
4. 领导者需要定期更新锁的TTL，以防止锁自动过期。如果领导者由于某种原因（如故障或网络问题）无法更新TTL，那么锁就会过期，其他的实例就可以尝试获取锁，成为新的领导者。
5. 当领导者失效或者放弃锁时，所有的跟随者都会尝试获取锁，成为新的领导者。这个过程可能会有竞争，但是由于etcd的原子性操作，只有一个实例能够成功获取锁。

通过这种方式，scheduler和controller manager可以确保在任何时候，每种类型的控制器只有一个实例在运行，避免了脑裂（split-brain）现象。









```bash
#!/bin/bash    bbymy
```

