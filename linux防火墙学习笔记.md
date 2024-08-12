### 一、虚拟机网络

首先，在安装VMware Workstation后，宿主机会生成俩个虚拟网卡：VMNET1和VMNET8。其中，VMNET1用于仅主机网络，VMNET8用于NAT网络。除了这俩个虚拟网卡外，宿主机还有1个物理网卡，连接通往互联网的路由。

![image-20230919145254672](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919145254672.png)

#### 1、桥接网络

桥接网络，即将虚拟机的网卡桥接到宿主机的物理网卡上。这样，虚拟机的网卡的ip地址就和宿主机物理网卡的ip地址属于同一网段。

桥接，在虚拟的概念中，是一个网口桥接到另一个网口。但实际意义上的桥接，就是将俩个主机连到一个路由或交换机上。

这样桥接到宿主机物理网卡的虚拟机，就不需要通过宿主机来做SNAT访问互联网，而是和宿主机访问的方式别无二致，都是通过宿主机的物理网卡将数据包转发到网关，然后在网关做SNAT访问互联网。

#### 2、NAT网络

NAT网络，即将虚拟机的网卡桥接到宿主机的VMNET8网卡上。这样，虚拟机的网卡的ip地址就和宿主机VMNET8网卡的ip地址属于同一网段。

在虚拟机访问互联网的时候，虚拟机会通过VMNET8卡将数据包发送给宿主机，并且VMNET8会对数据包做一个SNAT转换，将源地址转换为宿主机物理网卡的ip地址。然后再通过宿主机的物理网口，将数据包转发给连接互联网的网关，然后网关再次做一次SNAT，将源地址改为网关的公网地址，发给到互联网上。

#### 3、仅主机网络

仅主机网络，即将虚拟机的网卡桥接到宿主机的VMNET1网卡上，而VMNET1不像VMNET8这样可以做NAT转换，所以桥接在VMNET1上的虚拟机，只能和宿主机与同样桥接在VMNET1网卡上的虚拟机通信，不能访问外部网络。

你要问为什么VMNET8可以做NAT转换而VMNET1不可以的话？只能说设计就是这么设计的，用来模拟这两种模式而已。

### 二、SNAT与DNAT

#### 1、环境准备

nat网卡模拟内网环境，仅主机模拟互联网环境，中间个主机模拟为网关，需要打开ip_forward

![image-20230919192607970](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919192607970.png)

#### 2、SNAT

将中间个中间模拟成网关，内网主机访问外网时，通过网关来做SNAT，外网主机只会收到公网ip的包，而内网主机可以看见外网主机的回包。

```bash
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 ! -d 10.0.0.0/24 -j SNAT --to-source 192.168.10.8
```

内网主机抓包结果：

![image-20230919193903881](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919193903881.png)

外网主机抓包结果：

![image-20230919194034809](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919194034809.png)

网关主机抓包结果：

网关看不到发向自己包的原因是，网关只会是目标mac地址而不是目标ip地址，抓包显示的是三层

![image-20230919194054952](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919194054952.png)

#### 3、DNAT