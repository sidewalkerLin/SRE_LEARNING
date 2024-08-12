### 一、安装keepalived

#### 1、包安装keepalived

##### 1.1、Ubuntu22.04中包安装keepalived

**(1)apt安装keepalived**

```bash
#查看keepalived包版本
apt list keepalived

apt -y install keepalived
```

**(2)配置keepalifed.conf文件**

启动keepalived需要/etc/keepalived/keepalived.conf，Ubuntu中下载keepalived默认不自带此文文件，所以keepalived无法启动。

```bash
#/usr/share/doc/keepalived/samples下有很多keepalived.conf配置文件实例，拷贝一份到/etc/keepalived/下即可
cp /usr/share/doc/keepalived/samples/keepalived.conf.sample /etc/keepalived/keepalived.conf
```

**(3)检查配置文件是否存在格式错误**

```bash
#注意keepalived.conf.sample中默认的网卡名称是eth0，如果与主机的网口名不一致需要修改,比如修改为ens33
keepalived -t -f /etc/keepalived/keepalived.conf
```

**(4)重启keepalived.service**

```bash
systemctl restart keepalived.service
```

##### 1.2、rokcy8.6中包安装keepalived

**(1)yum安装keepalived**

```bash
#查看keepalived包版本
yum list keepalived

yum -y install keepalived
```

**(2)检查配置文件是否存在格式错误**

```bash
[16:46:49 root@server2 ~]#keepalived -t -f /etc/keepalived/keepalived.conf 
(/etc/keepalived/keepalived.conf: Line 15) number '0' outside range [1e-06, 4294]
(/etc/keepalived/keepalived.conf: Line 15) vrrp_garp_interval '0' is invalid
(/etc/keepalived/keepalived.conf: Line 16) number '0' outside range [1e-06, 4294]
(/etc/keepalived/keepalived.conf: Line 16) vrrp_gna_interval '0' is invalid
(/etc/keepalived/keepalived.conf: Line 21) WARNING - interface eth0 for vrrp_instance VI_1 doesn't exist
Non-existent interface specified in configuration
keepalived -t -f /etc/keepalived/keepalived.conf

#解决上述报错即可，可以复制/usr/share/doc/keepalived/keepalived.conf.sample，然后改下网口名称即可
```

**(3)重启keepalived.service**

```bash
systemctl restart keepalived.service
```

#### 2、编译安装keepalived

**(1)安装相关的包**

```bash
#Ubuntu22.04
apt update && apt -y install make gcc ipvsadm build-essential pkg-config automake autoconf libipset-dev libnl-3-dev libnl-genl-3-dev libssl-dev libxtables-dev libip4tc-dev libip6tc-dev libipset-dev libmagic-dev libsnmp-dev libglib2.0-dev libpcre2-dev libnftnl-dev libmnl-dev libsystemd-dev 

#红帽系列
yum -y install gcc curl openssl-devel libnl3-devel net-snmp-devel make
```

**(2)下载并加压源码包**

```bash
wget https://keepalived.org/software/keepalived-2.0.20.tar.gz
tar xvf keepalived-2.0.20.tar.gz -C /usr/local/src

```

**(3)编译**

```
cd /usr/local/src/keepalived-2.0.20/
./configure --prefix=/usr/local/keepalived
make && make install
```

**(4)创建path软连接并查看版本**

```bash
ln /usr/local/keepalived/sbin/keepalived /usr/local/sbin/keepalived
keepalived -v
```

**(5)拷贝自动生成的unit文件到正确路径下**

```bash
cp /usr/local/src/keepalived-2.0.20/keepalived/keepalived.service /lib/systemd/system/
```

**(6)创建配置文件**

```
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

**(7)检查配置文件并处理报错**

```bash
keepalived -t -f /etc/keepalived/keepalived.conf
```

**(8)启动keepalived.service并开机自启**

```bash
systemctl enable --now keepalived.service
```

### 二、keepalived原理与架构

#### 1、基于VRRP协议的理解

keepalived是基于VRRP协议实现的，VRRP协议全称为`Virtual Router Redundancy Protocol`，即虚拟路由冗余协议。

虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master 路由器和多个backup路由器，master上面有一个对外提供服务的VIP以及虚拟mac地址。master会发向指定的多播地址发送VRRP消息，而backup会监听这个多播地址，当 backup在预定的时间内未收到vrrp包时就认为master宕掉了，这时就需要根据 VRRP 的优先级来选举一个 backup当master，这样就保证路由高可用了

#### 2、基于TCP/IP协议的理解

keepalived工作在TCP/IP协议栈的IP层，TCP层，及应用层。

IP层(Layer3)：

Keepalived使用Layer3的方式工作时，Keepalived会定期向服务器群中的服务器发送一个ICMP的数据包，如果发现IP不通，Keepalived 便报告这台服务器失效，并将它从服务器群中剔除，这种情况的典型例子是某台服务器被非法关机。Layer3 的方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。

TCP层(Layer4)：

Layer4主要以TCP 端口的状态来判断服务器工作正常与否。如 web server 的服务端口一般是80，如果 Keepalived 检测到服务器没有启动80端口，则 Keepalived 将把这台服务器从服务器群中剔除。

应用层(Layer7)：

keepalived将根据用户的设定，检查服务器程序的运行是否正常，如果不是设定值，则将其剔除集群。

#### 3、keepalived架构模型

![image-20230915110204216](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230915110204216.png)

tcpdump -i ens33 -nn host 224.6.6.6

#### 4、keepalived选举原理

keepalived选举master默认是从所有keeplived节点选择权重最大的节点作为主节点。

主节点会向多播地址种发送Advertisement，告诉同一个路由器的所有节点，自己的权重是多少。如果其它节点发现主节点的权限低于自己的权重，那么这个节点就会进行抢占并成为新主。

实际上，如果停掉主节点的keepalived服务，那么主节点会向多播地址中发送自己权重为0的Advertisement，然后从其它节点选出一个权重最高的节点作为新主。当旧的主节点重新启动keepalived服务，又会将主节点的位置抢回来。

### 三、配置文件说明

#### 1、全局配置

```bash
global_defs {
    router_id ka2    #每个keepalived主机的唯一标识，每个keepalived主机不要重名                 
    vrrp_mcast_group4 224.6.6.6 #指定组播地址，范围可以是224.0.0.0~239.255.255.255
    vrrp_garp_interval 0  #gratuitous ARP messages 报文发送延迟，0表示不延迟
    vrrp_gna_interval 0   #unsolicited NA messages （不请自来）消息发送延迟
}
```

#### 2、虚拟路由器模块配置

```bash
#配置虚拟路由器，keepalived的原理就是将多个keepalived节点虚拟为一个虚拟路由器
vrrp_instance VI_1 {      #VI_1是虚拟路由器名称，一般命名跟业务相关，可以有多台虚拟路由器
    state MASTER          #指定当前节点的路由器角色，实际上不起作用，主要看优先级权重，如果权重一样，则mater角色获取vip
    interface ens33       #指定了keepalived实例监听的网卡，心跳检测的网卡，可以和vip绑定的网卡不是同一个
    virtual_router_id 50  #虚拟路由器id，这个虚拟路由器里面包含的所有keepalived节点，id都必须相同。范围0~255。
    priority 70           #配置当前keepalived节点的优先级权重，范围1~254
    advert_int 1          #vrrp通告的时间间隔，默认1s
    authentication {      #认证机制，推荐PASS，每个keepalived节点密码必须一样
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {   #配置虚拟ip地址，生产环境可能有上百个虚拟ip
        10.0.0.10/24      #虚拟ip地址，默认绑定到interface指定的网卡上。记得指定网段，要不然默认是/32
        10.0.0.11/24 dev eth0  #将虚拟ip绑定到指定网卡
        10.0.0.12/24 dev eth1 label eth1:1  #指定vip的网卡lable
    }
}
```

#### 3、配置keepalived日志功能







### 四、keepalived实现LVS高可用





### 五、keepalived实现nginx高可用



### 六、keepalived实现MySQL高可用

```bash
#master的keepalived配置

```



### 七、keepalived实现haproxy高可用

### 

### 八、keepalived实现zabbix高可用

