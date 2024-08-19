# 一、前置准备

## 1、环境规划

| 主机名         | IP地址     | 操作系统    |
| -------------- | ---------- | ----------- |
| k8s-master-102 | 10.0.0.102 | Ubuntu22.04 |
| k8s-master-103 | 10.0.0.103 | Ubuntu22.04 |
| k8s-master-104 | 10.0.0.104 | Ubuntu22.04 |
| k8s-node-105   | 10.0.0.105 | Ubuntu22.04 |
| k8s-node-106   | 10.0.0.106 | Ubuntu22.04 |
| k8s-node-107   | 10.0.0.107 | Ubuntu22.04 |

## 2、网络划分



## 3、所有节点配置apt源

```shell
cat > /etc/apt/sources.list.d/mirrors.tuna.tsinghua.edu.cn.list << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
EOF
```

## 4、配置master节点免密登录集群所有节点

```bash
#在一个master节点执行，会在节点的/root/.ssh下生成密钥对，id_rsa,id_rsa.pub
ssh-keygen -t rsa

#给私钥id_rsa权限
chmod 600 /root/.ssh/id_rsa

#查看公钥
cat /root/.ssh/id_rsa.pub

#将公钥的内容复制粘贴追加到集群所有节点的/root/.ssh/authorized_keys中
vim /root/.ssh/authorized_keys

#将/root/.ssh/id_rsa转发其它master节点的/root/.ssh/下
scp /root/.ssh/id_rsa root@ip:/root/.ssh/
```

## 5、配置host解析

```bash
#所有节点执行
cat >> /etc/hosts << EOF
10.0.0.102 k8s-master-102
10.0.0.103 k8s-master-103
10.0.0.104 k8s-master-104
10.0.0.105 k8s-node-105
10.0.0.106 k8s-node-106
10.0.0.107 k8s-node-107
EOF
```

## 6、禁用默认的防火墙服务

```bash
#所有节点执行
systemctl disable --now ufw
```

## 7、禁用selinux

```bash
#如果/etc/selinux/config文件不存在，无法判断selinux是否启用，安装下列工具，会生成/etc/selinux/config文件以及提供sestatus命令
apt -y install policycoreutils

#查看selinux是否启用
sestatus

#如果是启动状态，需要修改/etc/selinux/config
SELINUX=disabled

#修改完配置文件需要重启生效
reboot
```

## 8、禁用swap设备

```bash
#临死关闭swap分区
swapoff -a

#注释掉fstab文件里面swap有关的行
vim /etc/fstab
```

## 9、设定时钟同步

```bash
#所有节点执行，若节点可以直接访问互联网，则可以从配置默认的时间服务器同步时间
apt -y install chrony && systemctl enable --now chrony.service
```

## 10、内核参数调整

```bash
#所有节点执行
#修改内核参数
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp.keepaliv.probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp.max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp.max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.top_timestamps = 0
net.core.somaxconn = 16384
EOF

#内核参数立刻生效
sysctl --system
```

## 11、安装IPVS

```bash
#所有节点执行
#安装IPVS
apt -y install ipvsadm ipset sysstat conntrack libseccomp2

#开机启动会自动加载下列IPVS模块
cat > /etc/modules-load.d/ipvs.conf << EOF
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

#立即临时加载以上ipvs模块
cat /etc/modules-load.d/ipvs.conf | xargs -I {} modprobe {}
```

# 二、部署containerd

## 1、安装一些必要的工具

```bash
apt -y install apt-transport-https ca-certificates curl software-properties-common
```

## 2、添加docker源

```bash
#下载并添加了Docker源的GPG密钥，确保从Docker的APT源下载的软件包是可信的
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | apt-key add -

#添加了Docker的APT源到系统中，下载containerd依赖这个源
add-apt-repository "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```

## 3、安装containerd

```bash
apt -y install containerd.io
```

## 4、加载containerd模块

```bash
#开机启动会自动加载下列containerd模块
cat > /etc/modules-load.d/containerd.conf << EOF
overlay
br_netfilter
EOF

#立即临时加载以上containerd模块
cat /etc/modules-load.d/containerd.conf  | xargs -I {} modprobe {}
```

## 5、重新初始化containerd配置文件

```bash
containerd config default > /etc/containerd/config.toml
```

## 6、修改Cgroup的管理者为systemd组件

```bash
sed -ri 's#(SystemdCgroup = )false#\1true#' /etc/containerd/config.toml
```

## 7、修改pause容器的拉取源

```bash
sed -i 's#registry.k8s.io/pause:3.8#registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9#' /etc/containerd/config.toml
```

## 8、启动containerd

```bash
systemctl daemon-reload && systemctl enable --now containerd && systemctl status containerd
```

## 9、配置crictl工具

```bash
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

# 三、生成证书

## 1、安装证书生成工具

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl
chmod +x /usr/local/bin/cfssljson
```

### 2、根证书

根证书只需要一个，用来签发其它各个组件的证书

```bash
#创建一个文件存放证书以及密钥
mkdir -p /etc/kubernetes/pki&& cd /etc/kubernetes/pki

#此json文件用于配置证书颁发机构（CA）的签发策略
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "87600 h"
      }
    }
  }
}
EOF

#此json文件用于定义 CA 证书的签名请求（CSR）内容
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Beijing"
    }
  ]
}
EOF

#生成CA证书以及私钥
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## 3、kube-apiserver证书

```bash

```

## 4、etcd证书

```bash

```

## 5、kube-scheduler证书

```bash
cat > kube-scheduler-csr.json <<EOF
{
    "CN": "system:kube-scheduler",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "BeiJing",
        "L": "BeiJing",
        "O": "system:kube-scheduler",
        "OU": "seven"
      }
    ]
}
EOF

#生成kube-scheduler证书以及密钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

## 6、kube-controller-manager证书

```bash
cat > kube-controller-manager-csr.json <<EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "BeiJing",
        "L": "BeiJing",
        "O": "system:kube-controller-manager",
        "OU": "seven"
      }
    ]
}
EOF

#生成kube-controller-manager证书及密钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

## 7、admin客户端证书

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "seven"
    }
  ]
}
EOF

#生成admin客户端证书及私钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

## 8、kubelet客户端证书

```bash
WORKERS=("k8s-node-105" "k8s-node-106" "k8s-node-107")

for ((i=0;i<${#WORKERS[@]};i++)); do
    cat > ${WORKERS[$i]}-csr.json <<EOF
    {
      "CN": "system:node:${WORKERS[$i]}",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "L": "Beijing",
          "O": "system:nodes",
          "OU": "seven",
          "ST": "Beijing"
        }
      ]
    }
    EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${WORKERS[$i]},${WORKER_IPS[$i]} \
  -profile=kubernetes \
  ${WORKERS[$i]}-csr.json | cfssljson -bare ${WORKERS[$i]}
done
```

## 9、kube-proxy证书

```bash

```

# 四、部署etcd

1、下载etcd部署包

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.5.14/etcd-v3.5.14-linux-amd64.tar.gz
```

2、

```

```

3、

```

```

