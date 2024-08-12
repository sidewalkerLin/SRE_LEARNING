#### 一、IP内核参数

#### 1、net.ipv4.ip_forward

##### 1.1、ip_forward应用场景

net.ipv4.ip_forward用于主机IP转发的功能，也就是当主机需要充当路由的时候才需要开启此参数。

路由是怎么进行进行转发的？数据包从一个网口转入路由，然后查路由表，从另一个网口传出。所以主机如果需要使用到IP转发功能，往往是需要多块网卡，如果数据包涉及到从在俩块网卡间进行转发，那么就需要打开IP转发功能。

例如，假设主机运行有docker服务，那么存在3块网卡：lo网卡、ens33、docker0。

如果主机需要访问互联网，那么主机直接将数据包通过ens33发送给路由即可，并不会涉及到主机中俩块网卡的数据包转发，所以并不需要开启IP转发功能，主机也能访问互联网。

如果docker容器需要访问互联网，它需要将数据包由docker0网口转送给ens33网口，再由ens33发送给路由。在数据包从docker0到ens33的过程就是IP转发的过程，所以如果容器需要访问外部网络，必须要打开IP转发的功能，否则容器无法访问外部网络。

##### 1.2、临时开启ip_forward

以下是俩种临时开启ip_forward的方法：

方法一、

```bash
sysctl -w net.ipv4.ip_forward=1
```

方法二、

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

##### 1.3、永久开启ip_forward

```bash
vim /etc/sysctl.conf

net.ipv4.ip_forward=1

#从/etc/sysctl.conf中读取变量值生效
sysctl -p
```

##### 1.4、查看当前是否开启ip_forward

```bash
sysctl net.ipv4.ip_forward
```





### 二、TCP内核参数

#### 1、net.ipv4.tcp_max_syn_backlog





#### 2、net.core.somaxconn



#### 3、net.ipv4.tcp_max_tw_buckets



4、net.ipv4.tcp_fin_timeout



5、net.ipv4.tcp_tw_reuse



6、net.ipv4.tcp_tw_recycle



7、net.ipv4.tcp_syncookies



8、net.ipv4.tcp_keepalive_time



9、net.ipv4.ip_local_port_range



10、net.ipv4.tcp_synack_retries



11、net.ipv4.tcp_syn_retries





12、net.ipv4.tcp_max_orphans





13、net.core.netdev_max_backlog
