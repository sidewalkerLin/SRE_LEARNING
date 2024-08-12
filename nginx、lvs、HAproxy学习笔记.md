#                                                                                                 NGINX篇

### 一、nginx编译安装

1、下载nginx需要的一些依赖包

```bash
#Ubuntu22.04和Ubuntu20.04
apt update && apt -y install gcc make libpcre3 libpcre3-dev openssl libssl-dev zlib1g-dev

#centos8
yum -y install gcc pcre-devel openssl-devel zlib-devel

#rocky8
yum -y install gcc make gcc-c++ libtool pcre pcre-devel zlib zlib-devel openssl openssl-devel perl-ExtUtils-Embed 
```

2、创建nginx用户

```bash
#-s用来指定登录的shell，/sbin/nologin表示不能登录来交互的访问系统，-M表示不创建家目录
useradd -s /sbin/nologin nginx -M
```

3、在官网下载nginx源码包并解压

```bash
#/usr/local/src/这个目录通常用于存放一些源码包，所以我放这里。可以放到其它目录下不影响。
cd /usr/local/src/
#可以到nginx官网去选其它版本，注意要偶数版的，因为是稳定版，基数版是测试版
wget http://nginx.org/download/nginx-1.18.0.tar.gz 
tar -xf nginx-1.18.0.tar.gz
```

4、进入到nginx-1.18.0下执行configure命令

```bash
cd nginx-1.18.0/

#--prefix=/apps/nginx   指定安装nginx的目录，包含nginx的二进制文件，配置文件，日志文件，模块文件。并且这个文件会自动生成，不需要手动创建。
#--user=nginx			指定以nginx这个用户来运行nginx
#--group=nginx			指定以nginx这个组来运行nginx
#--with-http_ssl_module 启用HTTP SSL模块，让nginx支持HTTPS协议，也就是加密的HTTP通信
#--with-http_v2_module  启用HTTP/2模块，让nginx支持HTTP/2协议，也就是更高效的HTTP通信
#--with-http_realip_module 启用HTTP Real IP模块，这个模块可以让nginx获取客户端的真实IP地址，而不是代理服务器的IP地址。你需要在配置文件中指定哪些代理服务器是可信的，并设置相应的头部字段
#--with-http_stub_status_module 启用HTTP Stub Status模块，这个模块可以让nginx提供一个简单的状态页面，显示一些基本的统计信息，比如连接数，请求数等。你需要在配置文件中指定一个location来访问这个状态页面，并设置相应的权限
#--with-http_gzip_static_module 启用HTTP Gzip Static模块，这个模块可以让nginx直接发送预压缩的静态文件，而不是动态压缩它们。这样可以节省CPU资源和带宽，提高响应速度。你需要事先压缩好静态文件，并在配置文件中开启gzip_static指令
#--with-pcre  PCRE库来支持正则表达式的匹配和重写功能,先安装好PCRE库，在配置文件中使用相应的指令。
#--with-stream 启用Stream Core模块，这个模块可以让nginx支持TCP/UDP代理功能。你需要在配置文件中使用stream{}块来定义代理服务器和上游服务器
#--with-stream_ssl_module 启用Stream SSL模块，这个模块可以让nginx支持加密的TCP/UDP代理功能
#-with-stream_realip_module 启用Stream Real IP模块，这个模块可以让nginx获取TCP/UDP客户端的真实IP地址，而不是代理服务器的IP地址。你需要在配置文件中指定哪些代理服务器是可信的，并设置相应的头部字段
#--with-cc-opt=-Wno-error=deprecated-declarations 表示你想给编译器传递一个参数-Wno-error=deprecated-declarations，这个参数可以忽略一些废弃函数的错误提示，比如OpenSSL 3.0中废弃了一些引擎相关的函数
./configure --prefix=/apps/nginx \--user=nginx \--group=nginx \--with-http_ssl_module \--with-http_v2_module \--with-http_realip_module \--with-http_stub_status_module \--with-http_gzip_static_module \--with-pcre \--with-stream \--with-stream_ssl_module \--with-stream_realip_module \--with-cc-opt=-Wno-error=deprecated-declarations


```

注意:

a、Makefile中保存了编译和链接的规则和命令。在执行configure脚本时，它会根据你指定的编译选项和参数，用nginx-1.18.0/objsauto/make生成一个Makefile文件，这个文件是最终用来执行make命令的。

b、objs文件夹是用来存放nginx编译过程中生成的一些文件的，比如.o文件、.so文件、.h文件、.c文件等。这些文件是用来链接成最终的nginx可执行文件的，也就是nginx.exe。

```bash
#执行configure命令后会生成Makefile和objs文件
root@Ubuntu22:/usr/local/src/nginx-1.18.0/objs# ls
autoconf.err  Makefile  ngx_auto_config.h  ngx_auto_headers.h  ngx_modules.c  src
```

5、编译以及安装

注意：要在nginx-1.18.0下执行命令，而不要在nginx-1.18.0/objs下执行。nginx-1.18.0下的Makefile会调用objs下的Makefile。

```bash
#先执行make命令，如果成功了再执行make install命令。这样可以避免在编译失败时还尝试安装。&&是一个逻辑运算符，表示如果左边的命令返回0（表示成功），才会执行右边的命令。
#make命令是用configure命令生成的Makefile来编译源代码，生成可执行文件或者库文件。
#make install命令是用来安装编译好的文件到指定的位置
make && make install

```

6、修改安装目录权限并查看相关文件

```bash
chown -R nginx.nginx /apps/nginx

#conf：保存nginx所有的配置文件，其中nginx.conf是nginx服务器的最核心最主要的配置文件，其他的.conf则是用来配置nginx相关的功能的，例如fastcgi功能使用的是fastcgi.conf和fastcgi_params两个文件，配置文件一般都有一个样板配置文件，是以.default为后缀，使用时可将其复制并将default后缀去掉即可。
#html目录中保存了nginx服务器的web文件，但是可以更改为其他目录保存web文件,另外还有一个50x的web文件是默认的错误页面提示页面。
#logs：用来保存nginx服务器的访问日志错误日志等日志，logs目录可以放在其他路径，比如/var/logs/nginx里面。
#sbin：保存nginx二进制启动脚本，可以接受不同的参数以实现不同的功能。
root@Ubuntu22:/usr/local/src/nginx-1.18.0# ll /apps/nginx/
total 24
drwxr-xr-x 6 nginx nginx 4096 Jul 30 21:01 ./
drwxr-xr-x 3 root  root  4096 Jul 30 21:01 ../
drwxr-xr-x 2 nginx nginx 4096 Jul 30 21:01 conf/
drwxr-xr-x 2 nginx nginx 4096 Jul 30 21:01 html/
drwxr-xr-x 2 nginx nginx 4096 Jul 30 21:01 logs/
drwxr-xr-x 2 nginx nginx 4096 Jul 30 21:01 sbin/
```

7、创建软连接，将/apps/nginx/sbin/下的启动脚本创建到环境变量PATH下

```bash
ln -s /apps/nginx/sbin/nginx /usr/sbin/
```

8、验证版本以及编译参数

```bash
#查看nginx版本
nginx -v
#查看nginx版本以及编译参数
root@Ubuntu22:/usr/local/src/nginx-1.18.0# nginx -V
nginx version: nginx/1.18.0
built by gcc 11.3.0 (Ubuntu 11.3.0-1ubuntu1~22.04.1) 
built with OpenSSL 3.0.2 15 Mar 2022
TLS SNI support enabled
configure arguments: --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module --with-cc-opt=-Wno-error=deprecated-declarations
```

9、创建nginx自启动文件

```bash
vim /lib/systemd/system/nginx.service

[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target
[Service]
Type=forking
#指定pid文件的目录,默认在logs目录下,可选配置
PIDFile=/apps/nginx/run/nginx.pid
ExecStart=/apps/nginx/sbin/nginx -c /apps/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
LimitNOFILE=100000
[Install]
WantedBy=multi-user.target

#[Unit]部分是用来描述nginx服务的基本信息和依赖关系的
#Description：这个参数是用来给nginx服务一个简短的描述的
#After：这个参数用来指定nginx服务在哪些其他服务之后启动的，以保证依赖的服务都已经启动。让网络、远程文件系统和域名解析等服务都启动完成后才启动nginx，以避免出现问题。
#Wants：这个参数用来指定nginx服务需要哪些其他服务启动的，如果这些服务没有启动，nginx服务也会尝试启动它们。
#[Service]部分是用来描述nginx服务的具体行为和设置的
#Type：这个参数是用来指定nginx服务的启动类型的，forking表示nginx会创建一个子进程并退出父进程，systemd会认为父进程退出后就表示服务启动成功。
#PIDFile：这个参数是用来指定nginx服务的进程ID文件的位置的，systemd会根据这个文件来监控和管理nginx服务的进程。
#ExecStart：这个参数是用来指定在启动nginx服务之前执行的命令的，-c表示启动nginx时用/apps/nginx/conf/nginx.conf作为配置文件
#ExecReload：这个参数是用来指定重新加载nginx配置文件的命令的。/bin/kill -s HUP $MAINPID 等价于nginx -s reload。$MAINPID是一个变量，表示nginx服务的主进程ID，它可以从/run/nginx.pid文件中读取。
#ExecStop：这个参数是用来指定停止nginx服务的命令的。/bin/kill -s TERM $MAINPID等价于nginx -s stop。
#[Install]部分是用来描述nginx服务如何被安装和启用的。
#WantedBy：这个参数是用来指定哪些其他单元需要或者想要启动nginx服务的，multi-user.target表示多用户模式下需要启动nginx服务。
```

10、创建pid文件的存放目录并修改配置文件

```bash
mkdir /apps/nginx/run/

vim /apps/nginx/conf/nginx.conf
pid   /apps/nginx/run/nginx.pid;
#nginx.conf里面的pid文件路径是用来告诉nginx进程自己要把pid文件写在哪里的，而nginx.service里面的PIDFile参数是用来告诉systemd服务管理器要从哪里读取pid文件的。这两个路径必须保持一致，否则systemd无法正确地监控和控制nginx服务。
```

11、重载配置文件，让systemd识别新创建或修改的服务文件，让systemctl来管理nginx服务

```bash
systemctl daemon-reload
```

12、设置开机启动

```bash
systemctl enable --now nginx
```

13、重启nginx并查看pid文件是否生成

```bash
systemctl restart nginx.service

root@Ubuntu22:~# ll /apps/nginx/run/
total 12
drwxr-xr-x  2 root  root  4096 Jul 31 21:04 ./
drwxr-xr-x 12 nginx nginx 4096 Jul 31 20:50 ../
-rw-r--r--  1 root  root     6 Jul 31 21:04 nginx.pid
```

(14)查看pid文件的进程号是否和master进程编号一致，安装完成

![image-20230731210821388](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230731210821388.png)



注意的点:

```bash
源码包
auto  CHANGES  CHANGES.ru  conf  configure  contrib  html  LICENSE  man  README  src

执行configure后
auto  CHANGES  CHANGES.ru  conf  configure  contrib  html  LICENSE  Makefile  man  objs  README  src

make以前的objs
root@Ubuntu22:/usr/local/src/nginx-1.18.0/objs# ls
autoconf.err  Makefile  ngx_auto_config.h  ngx_auto_headers.h  ngx_modules.c  src

make以后的objs
root@Ubuntu22:/usr/local/src/nginx-1.18.0/objs# ls
autoconf.err  Makefile  nginx  nginx.8  ngx_auto_config.h  ngx_auto_headers.h  ngx_modules.c  ngx_modules.o  src

```

### 二、nginx新增模块

前言：编译nginx的时候，会用configure命令来指定需要的模块。如果在编译完成之后，需要添加模块，比如第三方模块nginx-module-vts、echo-nginx-module，则需要重新编译生成二进制可执行文件。

1、下载需要的模块(手动下载需解压)

```bash
cd /usr/local/src
#github很多时候连不上，那就手动下载传上来
#这里需要注意的是，使用git下载的文件和手动下载的文件略有不同，手动下载的文件是压缩的形式并且后最是master结尾的，在执行configure的时候模块路径要写对
git clone git://github.com/vozlt/nginx-module-vts.git
git clone git://github.com/openresty/echo-nginx-module
```

2、找到编译nginx时执行的参数

```bash
root@Ubuntu22:/usr/local/src# nginx -V
nginx version: nginx/1.18.0
built by gcc 11.3.0 (Ubuntu 11.3.0-1ubuntu1~22.04.1) 
built with OpenSSL 3.0.2 15 Mar 2022
TLS SNI support enabled
configure arguments: --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module --with-cc-opt=-Wno-error=deprecated-declarations
```

3、在原参数的基础上增加新的模块，并执行configre命令

configre命令有个参数时--addmodule=/path/to/module，可以通过这个参数来添加新的模块后，进行重新编译。

```bash
cd /usr/local/src/nginx-1.18.0

./configure --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module --with-cc-opt=-Wno-error=deprecated-declarations --add-module=/usr/local/src/echo-nginx-module-master --add-module=/usr/local/src/nginx-module-vts-master
```

4、查看当前nginx程序的信息

这个时候nginx还没有被覆盖，在编译重新安装后，程序会被覆盖。

```bash
root@Ubuntu22:/usr/local/src/nginx-1.18.0# ll /apps/nginx/sbin/nginx 
-rwxr-xr-x 1 nginx nginx 5746968 Jul 31 19:57 /apps/nginx/sbin/nginx*

```

5、编译安装

```bash
make && make install
```

6、查看当前nginx程序的信息

```bash
#已被覆盖
root@Ubuntu22:/usr/local/src/nginx-1.18.0# ll /apps/nginx/sbin/nginx 
-rwxr-xr-x 1 root root 6789944 Jul 31 22:52 /apps/nginx/sbin/nginx*
```

7、重启nginx服务

```bash
#重新编译后必须重启nginx服务，并且不支持reload
systemctl restart nginx
```

注意：

新增模块也可以平滑的新增模块，利用平滑升级的模式。在源码包下编译，然后新旧master并存......，具体参考nginx的平滑升级。

### 三、nginx平滑升级

```bash
#从nginx-1.18.0升级到nginx-1.20.1
```

原理：

```bash
a、在不停掉老进程的情况下，启动新进程
b、老进程负责处理仍然没有处理完的请求，但不再接受处理请求
c、新进程接受新请求
d、老进程处理完所有请求，关闭所有连接后，停止
```

1、下载新版本的nginx源码包并解压

```bash
#这里为了方便我直接下载到/root下了
wget http://nginx.org/download/nginx-1.20.1.tar.gz
tar -xf nginx-1.20.1.tar.gz
```

2、查看旧版本的编译参数

```bash
root@Ubuntu22:~/nginx-1.20.1# nginx -V
nginx version: nginx/1.18.0
built by gcc 11.3.0 (Ubuntu 11.3.0-1ubuntu1~22.04.1) 
built with OpenSSL 3.0.2 15 Mar 2022
TLS SNI support enabled
configure arguments: --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module --with-cc-opt=-Wno-error=deprecated-declarations
```

3、在新版本nginx包目录下执行configure命令，使用就版本的编译参数

```bash
cd /root/nginx-1.20.1

./configure --prefix=/apps/nginx \--user=nginx \--group=nginx \--with-http_ssl_module \--with-http_v2_module \--with-http_realip_module \--with-http_stub_status_module \--with-http_gzip_static_module \--with-pcre \--with-stream \--with-stream_ssl_module \--with-stream_realip_module \--with-cc-opt=-Wno-error=deprecated-declarations
```

4、编译并查看俩个版本的nginx程序

```bash
#只执行make，不要执行make install，make install会直接替换旧的nginx二进制文件。make会在objs下生成nginx程序。
make

root@Ubuntu22:~/nginx-1.20.1# ll objs/nginx /apps/nginx/sbin/nginx
-rwxr-xr-x 1 nginx nginx 5746968 Jul 31 19:58 /apps/nginx/sbin/nginx*
-rwxr-xr-x 1 root  root  5857768 Aug  1 09:22 objs/nginx*
```

5、将新的nginx程序替换旧的nginx程序，并备份旧的nginx程序

```bash
#备份
root@Ubuntu22:~/nginx-1.20.1# mv /apps/nginx/sbin/nginx /root/nginx.old

root@Ubuntu22:~/nginx-1.20.1# mv ./objs/nginx /apps/nginx/sbin/
```

6、检测新版本nginx程序和旧版本配置文件的兼容性

```bash
root@Ubuntu22:~/nginx-1.20.1# /apps/nginx/sbin/nginx -t 
nginx: the configuration file /apps/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /apps/nginx/conf/nginx.conf test is successful
```

7、向旧的master进程发送USR2信号

```bash
#SIGUSR2信号默认处理是终止进程，但SIGUSR2信号是一个用户自定义信号，nginx对其做了特殊处理，在nginx中不再用来表示终止进程，而用来实现平滑升级的功能。
kill -USR2 `cat /apps/nginx/run/nginx.pid`
#另外俩种写法也行，一个意思
#kill -SIGUSR2 `cat /apps/nginx/run/nginx.pid`
#kill -12 `cat /apps/nginx/run/nginx.pid`
```

给旧的master进程发送USR2信号会发生下面几个过程：

a、旧的master进程的pid文件会由nginx.pid更名为nginx.pid.oldbin

b、新的nginx程序会启动一个新的master进程，继承旧的master进程监听的所有 socket 描述符，开始接收新的连接请求。

c、生成一个新的nginx.pid保存新的master进程的pid

![image-20230801105139037](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230801105139037.png)

8、查看此时是谁在处理新的请求

```bash
#再另一台主机上curl下nginx服务器，发现此时仍然是旧的master下的worker进程在接收新的请求
root@Ubuntu22:~# curl -I 10.0.0.162
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Tue, 01 Aug 2023 02:58:38 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Mon, 31 Jul 2023 11:58:40 GMT
Connection: keep-alive
ETag: "64c7a1f0-264"
Accept-Ranges: bytes

```

9、向旧的master进程发送WINCH信号

```bash
kill -WINCH `cat /apps/nginx/run/nginx.pid.oldbin`
```

给旧的master进程发WINCH信号会发生下面几个过程：

a、关闭旧的master进程下的worker进程(不关闭master进程的原因是方便进行回滚)

b、新的请求都交给新的master下的worker进程来处理

10、再次查看此时是谁在处理新的请求

```bash
#发现nginx的版本已经更新到1.20.1了
root@Ubuntu22:~# curl -I 10.0.0.162
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Tue, 01 Aug 2023 03:03:00 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Mon, 31 Jul 2023 11:58:40 GMT
Connection: keep-alive
ETag: "64c7a1f0-264"
Accept-Ranges: bytes
```

<font color=#FF0000 >**思考**</font>：发送WINCH信号后，要不要等待worker进程先处理完请求再发送quit信号？

不用，quit是平滑的关闭进程，如果进程正在处理请求，它会等待进程处理完请求之后再关闭进程。

11、这里可以先停下来进行测试，确认新版本服务有没有问题。没有问题则继续升级，有问题则进行回滚。

(1)确认新版本服务没有问题，继续升级

a、向旧的master进程发送QUIT信号(先等待前面旧的worker进程处理完请求)

```bash
#QUIT信号会关闭旧的master进程，平滑的关闭
kill -QUIT信号会关闭旧的master进程 `cat /apps/nginx/run/nginx.pid.oldbin` 

root@Ubuntu22:~/nginx-1.20.1# ps aux |grep nginx
root       28034  0.0  0.6  10016  6248 ?        S    10:27   0:00 nginx: master process /apps/nginx/sbin/nginx -c /apps/nginx/conf/nginx.conf
nginx      28035  0.0  0.3  10772  3596 ?        S    10:27   0:00 nginx: worker process
root       28073  0.0  0.2   6476  2208 pts/0    S+   11:14   0:00 grep --color=auto nginx
```

b、验证

```
nginx -v
curl -I 127.0.0.1
```

(2)确认新版本服务有问题，进行回滚

a、发送HUP信号,重新拉起旧版本的worker进程

```bash
kill -HUP `cat /apps/nginx/run/nginx.pid.oldbin`
```

![image-20230801114444381](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230801114444381.png)

b、平滑关闭新版本的master进程以及worker进程

```bash
#不发送上面的HUP信号，只发送QUIT也可以拉起旧版本的worker进程
#执行QUIT信号后新的master进程不会接收请求了，由旧的master下的worker进程进程接收
kill -QUIT `cat /apps/nginx/run/nginx.pid`
```

c、将旧的nginx二进制可执行程序还原到原来的路径下

```bash
mv /root/nginx.old /apps/nginx/sbin/nginx
```

d、验证

```
nginx -v
curl -I 127.0.0.1
```

问题:

1、生产环境中，发送WINCH信号后，要不要等待worker进程先处理完请求再发送quit信号。

2、回滚的时候，新版本的worker线程正在处理请求，那后面再来的请求由谁来处理？如果还是由新的worker线程处理不是关不了了吗？

### 四、master进程与worker进程

参考下面几篇文章：

```
https://zhuanlan.zhihu.com/p/96757160
https://www.cnblogs.com/jianmingyuan/p/13960906.html
https://blog.csdn.net/qq_35733751/article/details/79832005
```

简单概括下就是：

a、master进程是用来管理nginx本身和worker进程的，真正处理用户请求的是worker进程

b、master进程listen到用户请求时，会让所有空闲状态的worker线程进行抢占(这里面涉及一些算法，锁)，抢占成功的才会去处理请求

c、在nginx.conf文件中可以定义worker线程数量worker_ processes 1，定义的值为多少，在启动nginx后worker进程的数量就是多少

d、worker进程的数量通常与cpu核心数量有关，假如服务器时4核的cpu，那么worker_ processes的值一般为4

e、上述配置为了让每个worker进程都有一个cpu可以使用，尽量避免了多个worker进程抢占同一个cpu的情况，可以将worker_ processes的值设置为auto，这样nginx会自动检测当前主机的cpu核心数,并启动对应数量的worker进程。

### 五、nginx核心模块

#### 1、主配置文件结构

```bash
main block：主配置段，即全局配置段，对http,mail都有效
#事件驱动相关的配置
events {
 ...
}   
#http/https 协议相关配置段
http {
 ...
}          
#默认配置文件不包括下面两个块
#mail 协议相关配置段
mail {
 ...
}    
#stream 服务器相关配置段
stream {
 ...
} 
```

#### 2、main block

```bash
#user指令时用来指定以哪个用户、组的身份来运行nginx，默认是nobody。注意：在Ubuntu中，是没有nobody这个组的，这样写会报错。user nobody默认用户和组都是nobody，可以这样写来指定组user nobody root。当然正常情况下是创建一个nginx用户和组来给nginx程序使用。
user nobody;
#user nginx
```

```bash
#worker_processes指令是指定启动nginx时，fork的worker进程数量，默认是1个。正常情况下，worker线程的数量应该与服务器cpu核心数量一致，使用auto可以让nginx自动检测当前主机的cpu核心数,并启动对应数量的worker进程。
worker_processes  1;
#worker_processes auto;

#worker_cpu_affinity指令是用来设置worker进程和CPU核心的亲和性，即将worker进程和cpu之间进行绑定，worker进程处理请求时只会到被绑定的cpu上进行处理
#格式就是要绑定几个worker进程就有几位，以下分别是绑定2、4、8个worker进程的格式
#00000001---0号cpu 00000010---1号cpu 00000100---2号cpu 00001000---4号cpu 10000000---7号cpu
worker_cpu_affinity 01 10;
worker_cpu_affinity 0001 0010 0100 1000;
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
worker_cpu_affinity auto;#按顺序自动绑

问题1：worker进程绑定cpu核心前，是怎么调度cpu的？
worker收到请求可能会被调度器调度到任意一个可用的cpu上，不一定是空闲的 CPU。调度器会根据各个 CPU 的负载情况和进程的优先级等因素来决定把进程放到哪个 CPU 上。如果有多个 CPU 都可用，那么调度器会尽量选择一个距离进程上次运行的 CPU 最近的一个，以提高缓存命中率。但是这样会造成同一个worker进程在不同cpu之间切换、缓存失效、上下文切换的开销。

问题2：提高缓存命中率是什么意思？
cpu在执行worker进程的时候，需要去访问数据和指令。如果这些数据和指令在cpu缓存中，那么就可以直接使用；如果这些数据和指令不在cpu缓存中，那么就需要从内存中读取这些数据和指令，并把读取到的内容存入cpu缓存中。所以，如果能命中cpu缓存，性能肯定是更好的。

问题3：worker进程绑定cpu核心有什么用呢？
将worker进程和cpu核心进行绑定，那么以后这个worker进程都会到这个cpu核心上执行。当1个worker进程第一次在一个绑定的cpu核心上执行后，cpu会将这次读取到的数据和指令存入cpu缓存中，那么下次worker进程再来执行，就可以命中上次保存在cpu缓存中的数据和指令，大大提升了缓存命中率，提高程序的运行效率。
```

```bash
#number -20~19，worker进程优先级越低优先级越高，nginx的性能越好。
worker_priority number;
```

```bash
#路径
error_log /apps/nginx/logs/error.log error;
pid       /apps/nginx/logs/nginx.pid;
```

```bash
#所有worker进程能打开的文件数量上限,包括:Nginx的所有连接（例如与代理服务器的连接等），而不仅仅是与客户端的连接,另一个考虑因素是实际的并发连接数不能超过系统级别的最大打开文件数的限制.最好与ulimit -n 或者limits.conf的值保持一致,默认不限制
#限制的原因是如果并发连接数过大，可能导致nginx的崩溃
worker_rlimit_nofile 65536;
```

```bash
#默认值为daemon on。daemon off的意思是让nginx以前台进程的方式运行，而不是以后台守护进程的方式运行。这样可以方便调试和监控nginx的运行状态，也可以让nginx在Docker容器中正常运行。
daemon off;
```

```bash
#默认值为on，on表示开启master-worker模式，off表示只开启master，仅用于开发调试。
master_process off|on; 
```

#### 3、event模块

```bash
events {
	#设置单个工作进程的最大并发连接数，默认512，生产建议根据性能修改更大的值。
 	worker_connections  65536;
 	#使用epoll事件驱动，Nginx支持众多的事件驱动，比如:select、poll、epoll，只能设置在events模块中设置。
 	use epoll;
 	#mutex互斥为on表示同一时刻一个请求轮流由worker进程处理,而防止被同时唤醒所有worker,避免多个睡眠进程被唤醒的设置，可以避免多个 worker 进程竞争同一连接而导致性能下降,也可以提高系统的稳定性,默认为off，新请求会唤醒所有worker进程,此过程也称为"惊群"，在高并发的场景下多个worker进程可以各自同时接受多个新的连接请求，如果是多CPU和worker进程绑定,就可以提高吞吐量
 	accept_mutex on;
 	#on时Nginx服务器的每个工作进程可以同时接受多个新的网络连接，此指令默认为off，即默认为一个工作进程只能一次接受一个新的网络连接，打开后几个同时接受多个。建议设置为on
 	multi_accept on;
}   

高并发下设置accept_mutex off;可以提高性能。反之，可以设置accept_mutex on;节约资源。
```

#### 4、http模块

```bash
http {
	#mime.types这个文件在conf文件下，里面记载了文件后缀和mime type的一个映射，比如html映射到text/html,json映射到application/json。这个映射是国际组织规定的，大多数web服务都支持这个规范。
	#http头部报文中设置content-type字段，这个content-type的值就是mime type的值，用来告诉收到报文的对方应该用那种形式来进行解析。
	#客户端在向服务器发送http请求时，指定content-type的值，比如客户端发送给服务端的是一个json请求，那么content-type的值就是application/json，这样服务端就知道自己应该用json的解析方法来解析数据，解析完之后就知道客户端请求的内容是什么，比如客户端发送了一个json请求来请求了一张图片。ok这个时候服务端就根据客户端告诉的图片文件的后缀，到mime types文件中进行映射得到具体的mime type值，比如值image/png，将值放到http response的头部content type中，意思就是告诉客户端用对应的解析来解析我回复的数据。如果不告诉对方content-type，对方可能会用错误的解析方式来进行解析。
	#GET请求通常不需要发送请求体，因为它们只是获取资源，而不是提交数据。所以，你不需要指定content-type，也不会影响服务器的处理。如果你使用POST或PUT等方法发送请求，并且需要发送请求体，那么你就应该指定content-type，以便服务器知道你发送的数据是什么类型。
	include  mime.types;
	#如果mime.types中没有对应的文件后缀映射的mime type,那么回复的content-type就是default_type。application/octet-stream表示未知的应用程序文件，访问mime.types外其它类型时会直接下载请求的文件，浏览器一般不会自动执行或询问执行。
	default_type application/octet-stream;
	#types{}用来增加mime.types中不支持的文件后缀，也可以直接加到mime.types文件中
	types{
	    text/html  md;
	}
	
	sendfile  on;
	keepalive_timeout  65;
	
	#server_tokens off可以隐藏版本，可以放在http模块、server模块、location模块里面，生效范围不同。修改显示的nginx版本可以通过修改源码重新编译实现。
	server_tokens on | off | build | string;
	#指定字符集，正常应该由前端工程师来在html中
	charset utf-8;

	#定义虚拟服务器，可以用include指定写到子配置文件中去
	server {
		#default_server表示如果有多个虚拟主机监听80端口的时候，默认访问当前的虚拟主机
        listen     80 default_server;
        #指定虚拟服务器处理的域名的指令，可以写做多个域名，空格分开
        server_name  www.abc.com;
        #指定虚拟服务器的根目录的指令,访问静态文件时，会从此目录下方找
        #建议最后补上/表示html是个目录而不是文件
        root /apps/nginx/html/   
        
        location /new {
            #location中的root指令优先于server中的root指令生效，以location的root指令为准
            root   /data/nginx/html;
            #alias  /test/alias;
        }

}


1、详解root指令和alias指令区别
location /new {
            #root   /data/nginx/html/;
            #alias  /test/alias;
}
现在有3个文件：
/data/nginx/html/new/index.html   内容：this is /data/nginx/html/new/index.html
/test/alias/index.html			  内容：this is /test/alias/index.html
/test/aliasnew/index.html		  内容：this is /test/alias/new/index.html        
假设域名为www.abc.com
假如在location模块中使用root /data/nginx/html时：
我们访问http://www.abc.com/new时，实际上是获取的/data/nginx/html/new下的资源。也就是说在location中使用root指令，效果就是，在用户通过"域名+location后的路径"访问资源的时候，服务器端会将域名替换为root指令的地址，然后用"root指令路径+location后的路径"访问资源。
curl http://www.abc.com/new 打印的结果是----this is /data/nginx/html/new/index.html


假如在location模块中使用alias /test/alias时：
我们访问http://www.abc.com/new时，实际上是获取的/test/alias下的资源。也就是说在location中使用alias指令，效果就是，在用户通过"域名+location后的路径"访问资源的时候，服务端会将"域名+location后的路径"全部替换为alias指令的地址。所以只要用户使用"域名+location后的路径"的形式访问资源，最后都是得到的是alias指令路径下的资源。
curl http://www.abc.com/new 打印的结果是----this is /test/alias/index.html


2、location模块
#语法规则
location [ = | ~ | ~* | ^~ ] uri { ... }
=        #请求的路径要与uri精准匹配
~        #请求的路径包含uri就行,且区分大小写
~*       #请求的路径包含uri就行,且不区分大写
^~       #请求的路径以uri开头就行，不区分字符大小写
不带符号   #请求的路径以uri开头就行
\        #用于标准uri前，表示包含正则表达式并且转义字符。可以将 . * ?等转义为普通符号

#匹配优先级从高到低：
=, ^~, ~/~*, 不带符号


示例：
    server_name www.abc.com;
    location = / {                                      ----------   A
        default_type text/html;
        return 200 'location = /';
    }
    location / {                                        ----------   B
        default_type text/html;
        return 200 'location /';
    }
    #不带符号 就是这种
    location /documents/ {                              ----------   C
        default_type text/html;
        return 200 'location /documents/';
    }
    location ^~ /images/ {                              ----------   D
        default_type text/html;
        return 200 'location ^~ /images/';
    }
    location ~* \.(gif|jpg|jpeg)$ {                     ----------   E
        default_type text/html;
        return 200 'location ~* \.(gif|jpg|jpeg)';
    }
    location ~ /images/ {                               ----------   F
        default_type text/html;
        return 200 'location ~ /images/';

    }
    #location ~ /images {                               ----------   G
    #    default_type text/html;
    #    return 200 'location ~ /images/';
    #}

curl www.abc.com/    				----------满足A、B；A优先级高        
curl www.abc.com/documents/aaa    	----------满足B、C；C更准确匹配优先级高
curl www.abc.com/index.html    		----------满足B                    
curl www.abc.com/images/a.jpg    	----------满足D、F、E；D优先级高
curl www.abc.com/documents/a.jpg  	----------满足C、F；F优先级高
curl www.abc.com/imagesdahskjdask   ----------满足B、G；G的优先级高


location @重定向：

error_page的作用是当发生错误的时候能够显示一个预定义的uri。
例如：error_page 502 503 /50x.html;如果访问出现502或503就返回50x.html的内容。
同时我们也可以自己定义这种情况下的返回状态吗，比如：error_page 502 503 =200 /50x.html;
这样用户访问产生502 、503的时候给用户的返回状态是200，内容是50x.html。

当然也可以是使用location @来处理：
#location @name 这样的location不用于常规请求处理，而是用于请求重定向
error_page 502 503 @error
location @error{
    default_type text/html;
    charset utf8;
    return 200 '你访问的页面可能走丢了!';
}
```

### 六、URI、URL、URN区别

先说一下结论，URL和URN都是URI的子集。

URL：统一资源定位符。可以理解为将某个主机的资源的位置地址给记录下来，这样可以通过定位的形式来获取资源。

URN：统一资源名称。

URI：统一资源标识符。

举个例子，主机A想获取主机B上的videos/xxx.mp4的视频资源，那么通过URL的形式来访问资源https://www.B.com/videos/xxx.mp4，这就是现在互联网上最常见的访问资源的形式。但是这样也带来了问题，如果某一天这个xxx.mp4被移动到另一个文件夹movies下或者网站直接没了转移到了另外一个网站的下，那么就无法再通过原来的那个URL来访问资源了。URN就可以解决这个问题，它可以用一串独一无二的字符串来表示一个资源，比如magnet:?xt=urn:btih:bdab9b6dasjd59950dsadjka3c8cbde2669bea619sadg，那么不管这个资源往哪移我们都可以通过URN访问它，但是解析URN需要特殊的解析器来解析，应用起来会比较复杂。

而URN和URL都是URI的子集，除此之外，URI还包括URC、PURL等。

在nginx的location中的uri并不是传统定义的uri，而是nginx自己的用法，表示请求的路径部分。

### 七、nginx常见模块

#### 1、四层访问控制

模块：ngx_http_access_module

```bash
location /about {
   alias /data/nginx/html/pc;
   index index.html;
   deny  192.168.1.1;
   allow 192.168.1.0/24;
   allow 10.1.1.0/16;
   allow 2001:0db8::/32;
   deny all;              #按先小范围在前，大范围在后排序
 }
```

#### 2、账户认证功能

模块：ngx_http_auth_basic_module

```bash
www.linqixin.com.conf配置：
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;

    location / {
    	#显示在对话框中做提示用，但是新版本的浏览器都不支持，所以看不见，可以f12响应标头看见
        auth_basic    "www.linqixin.comdsahkjdas";
        #存放用户密码的文件路径
        auth_basic_user_file     /apps/nginx/conf/conf.d/htpasswd;
    }
}

用户密码需要加密，加密使用htpasswd命令。这个命令在不同的操作系统，在不同的包中。
#CentOS安装包
[root@centos8 ~]#yum -y install httpd-tools
#Ubuntu安装包
[root@Ubuntu ~]#apt -y install apache2-utils

#-c是创建password-file这个文件，如果文件已经存在会被覆盖。所以如果要加第二个用户密码的时候，不要加-c，要不然会被覆盖。
#-b是从命令行参数中读取密码
htpasswd -cb /path/to/password-file username password
htpasswd -b /path/to/password-file username1 password

以上配置好后，去浏览器进行测试，如果报错500，查看日志可能是权限问题
#这里配置的属主属组要按nginx的worker进程的属主属组来配置
chown nginx.nginx /path/to/password-file
chmod 600 /path/to/password-file
```

#### 3、自定义错误页面

自定义错误页，同时也可以用指定的响应状态码进行响应, 可用位置：http, server, location, if in  location

```bash
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html1/;
    
    error_page 404 /40x.html;
    location /40x.html {
        root /data/nginx/html2/;
    }
}

#当客户端请求服务端某个资源报错404时，原来的url会变成www.linqixin.com/40x.html,location中root指令优先级比server高，那其实请求资源的真是地址是/data/nginx/html2/40x.html。

#


注意！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
charset utf-8;
server {

    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    error_page 404 =302 /30x.html;
    #error_page 404 =302 30x.html;


    location /30x.html {
        #charset utf-8;
        root /data/nginx/html1;
    }
}

浏览器访问一个不存在的页面http://www.linqixin.com/30xssssssssss.html时。
如果error_page 404 =302 /30x.html;
url不会变化，还是http://www.linqixin.com/30xssssssssss.html，但是会显示/data/nginx/html1/30x.html的内容，且内容乱码。

如果error_page 404 =302 30x.html;
url会变化为http://www.linqixin.com/30x.html，会显示/data/nginx/html1/30x.html中的内容，但内容不乱码。

```

#### 4、自定义错误日志

```bash
Syntax: error_log file [level];
Default: 
#默认是error及error以上的level
error_log logs/error.log error;
Context: main, http, mail, stream, server, location
level: debug, info, notice, warn, error, crit, alert, emerg

```

```bash
#如果nginx配置了多虚拟主机，那么所有server的日志都会记录到access.log和error.log中，这样不好，所以需要将它们的日志分开存放。
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    #error_log /dev/null;关闭错误日志
    access_log    /apps/nginx/logs/linqixin.com_access.log;
    error_log     /apps/nginx/logs/linqixin.com_error.log;    
} 
```

#### 5、检测文件是否存在

```bash
#url不会变化
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
        try_files $uri  $uri.html $uri/index.html /default.html;
        #try_files $uri $uri/index.html $uri.html =489;
     }
}

访问http://linqixin.com/abc时，现根据本身的uri进行匹配，找不到继续匹配abc.html,找不到再继续匹配abc/index.html,找不到再继续找root指令路径下的default.html。至少保证有一个文件是存在的，要不然会报500错误。
```

#### 6、长连接配置

```bash
keepalive_time：这个指令是用来设置nginx与上游服务器之间的长连接的最大空闲时间，也就是说，如果在这个时间内没有新的请求通过这个连接发送，那么nginx就会关闭这个连接。这个指令只能在upstream模块中使用，它的默认值是60秒。如果设置为0，表示禁用长连接。

keepalive_timeout：这个指令是用来设置nginx与客户端之间的长连接的最大空闲时间，也就是说，如果在这个时间内客户端没有发送任何数据，那么nginx就会关闭这个连接。这个指令可以在http、server或location模块中使用，它的默认值是75秒。如果设置为0，表示禁用长连接。这个指令还可以有一个可选的参数，用来设置在HTTP响应头中返回"Keep-Alive:timeout=60"字段，这个字段可以告诉客户端服务器端的长连接超时时间。

 keepalive_requests：这个指令是用来设置nginx与客户端或上游服务器之间的长连接最大交互请求数，也就是说，如果通过一个连接发送了超过这个数目的请求，那么nginx就会关闭这个连接。这个指令可以在http、server、location或upstream模块中使用，它的默认值是100。
```

#### 7、作为下载服务器配置

```bash
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
        autoindex on;
    }
}
如果这样配置server会出现2个问题：
1、访问http://www.linqixin.com时，index指令会默认补齐/data/nginx/html/index.html,最后显示的时index.html的页面。而我们想得到的是页面是文件目录，如何解决？
解决方案：在location中使用index off;来关闭index的功能。

2、/data/nginx/html/下的文件如果是有后缀的，比如.html、.pdf，那么mime_types会映射相应的context-type回复客户端，这样客户端就会用相应的content-type来解析这些文件，如果我们点击这些文件，浏览器会直接展示出来。而我们想要的效果是点击文件直接下载，如何解决？
解决方案：在location中使用types{}来覆盖mime.types中的设置，再指定默认的default_type application/octet-stream。大多数浏览器会将application/octet-stream视为二进制文件，这样的话浏览器就不会去渲染它，而是直接下载它。


正确方案：
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
        #add_header Content-Type application/octet-stream; 会导致响应中有两个content-type,一个是text/html，另一个是application/octet-stream。所以不用这个方法。
        #types{}会将mime.types中的设置覆盖为空
        types {}
        #将回复客户端的context-type设置为application/octet-stream
        default_type application/octet-stream;
        index off;
        #自动文件索引功能，设置后展示文件目录
        autoindex on;
        #计算文件确切大小，单位bytes。off显示大概大小，单位mb、kb，默认为on。
        autoindex_exact_size  off;
        #on显示本机时间，off显示格林威治时间，默认off
        autoindex_localtime   on;
        charset utf-8;
        #限制传输效率，单位b/s,不限速设置为0
        limit_rate           10k;
        #set $limit_rate 4k;
    }
}        
```

#### 8、作为上传服务器配置

```bash
#设置允许客户端上传单个文件的最大值，默认值为1m,上传文件超过此值会出413错误
client_max_body_size 1m; 
#用于接收每个客户端请求报文的body部分的缓冲区大小;默认16k;超出此大小时，其将被暂存到磁盘上的由client_body_temp_path指令所定义的位置
client_body_buffer_size size; 
#设定存储客户端请求报文的body部分的临时存储路径及子目录结构和数量，目录名为16进制的数字，使用hash之后的值从后往前截取1位、2位、2位作为目录名
client_body_temp_path path [level1 [level2 [level3]]];
```

#### 9、限流限速

为什么要限流限速？

限制某个用户在一定时间内能够产生的Http请求或者限制某个用户的下载速度；

防止个别用户对资源消耗过多,导致其它用户受影响；

启用限速后,如果超过指定的阈值,则默认提示503过载保护



限制下载速度：

```bash
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    
    #下载达到100mb后开始限速
    limit_rate_after     100m;
    #限制下载速度，单位b/s
    limit_rate           10k;
}


```

限制请求数(限制同一个ip同一时间发起的最大请求数，注意限制的是rate速率)：

```bash
Syntax: limit_req_zone key zone=name:size rate=rate [sync];
Default: —
Context: http

Syntax: limit_req zone=name [burst=number] [nodelay | delay=number];
Default: —
Context: http, server, location

Syntax: limit_req_status code;
Default: 
limit_req_status 503;
Context: http, server, location

#参数说明
#注意:$remote_addr和$binary_remote_addr有所不同，如果ipv6地址前者会占用更多空间，且后者更安全，所以用后者

limit_req_zone
#第一个参数：$binary_remote_addr表示以客户端IP地址的二进制形式为限流依据的key，通过这个来做限制，限制同一客户端ip地址。
#第二个参数：zone=req_one:10m表示生成一个大小为10M，名为req_one的共享内存区域用来存储访问状态的。访问状态包括了 IP(准确的说应该是key) 的访问次数、访问频率、访问延迟等信息，这些信息是用来判断 IP 是否超过了限制速率的依据。8000个IP地址的状态信息约占用内存空间1MB，10m可以存储80000个IP地址，也就是说10m可以存储8w个ip地址的访问状态。如果 zone 存储空间耗尽，那么 nginx 会移除最近最少使用（LRU）的 IP 状态，或者直接返回 503 错误。

原理！！！！！！！！！！！
zone作为一个共享内存区域，worker进程都可以从中读取其中的数据。worker进程读取了ip的信息之后就能判断出应该执行limit_req还是limit_conn 指令。当然这就造成可能存在多个worker进程会读取同一个ip的信息并进行修改，这会导致一些并发问题，nginx采用了读写锁和自旋锁解决了此问题。

#第三个参数：rate=1r/s表示允许相同标识的客户端的访问频次，每秒最多处理1个请求，超过此速率的请求放入burst队列做延迟处理。

limit_req
#第一个参数：zone=req_one 设置使用哪个配置区域来做限制，引用上面limit_req_zone定义的区域的name。
#第二个参数：burst=10，设置一个大小为10的缓冲区，当有大量请求过来时，超过了访问频次限制的请求可以先放到这个缓冲区内。
#第三个参数：nodelay，超过访问频次并且缓冲区也满了的时候，则会返回503，如果没有设置，则所有请求会等待排队。

limit_req_zone的zone 是用来存放 IP 的访问状态的，而不仅仅是访问次数。访问状态包括了 IP 的访问次数、访问频率、访问延迟等信息，这些信息是用来判断 IP 是否超过了限制速率的依据。
```

限制并发连接数(限制同一个IP的同时发起的最大并发连接数):

```bash
#用法和limit_req_zone一样，但是这里的zone是存储存储不同 key 值对应的连接状态信息。包括连接数等。。
Syntax: limit_conn_zone key zone=name:size;
Default: —
Context: http

#number就是限制的连接数
Syntax: limit_conn zone number;
Default: —
Context: http, server, location

```

弄清楚连接和请求的概念！

综合应用：

```bash
#定义了req_zone这个共享内存区域可以存放80000个ip地址，且每秒最多处理1个请求
limit_req_zone $binary_remote_addr zone=req_zone:10m rate=1r/s;
limit_conn_zone $binary_remote_addr zone=conn_zone:10m;
server {
    listen 80;
    server_name mirrors.wang.org;
    root /data/mirrors/;
    charset utf8;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    limit_req zone=req_zone burst=5 nodelay;
    limit_conn conn_zone 2;
    limit_rate_after 100m;
    limit_rate 500k;
    error_page 503 @error_page;
    location @error_page {
        default_type text/html;
        charset utf8; 
        return 200 '温馨提示:请联系管理员进行会员充值!';
        #return https://pan.baidu.com/buy/center?tag=8&from=loginpage#/svip ;
    }
}
```

#### 10、nginx状态页

```bash
#基于ngx_http_stub_status_module
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
    	#状态页用于输出nginx的基本状态信息
        stub_status;
        index off;
    }
}

#输出信息示例：
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
 上面三个数字分别对应accepts,handled,requests三个值
Reading: 6 Writing: 179 Waiting: 106
Active connections： #当前处于活动状态的客户端连接数，包括连接等待空闲连接数
=reading+writing+waiting
accepts：#统计总值，Nginx自启动后已经接受的客户端请求连接的总数。handled：#统计总值，Nginx自启动后已经处理完成的客户端请求连接总数，通常等于accepts，除非有因worker_connections限制等被拒绝的失败连接,即失败连接数=accepts-handled
requests：#统计总值，Nginx自启动后客户端发来的总的请求数。因为长连接的原因此值大于上面的accept数
Reading：#当前状态，正在读取客户端请求报文首部的连接的连接数,数值越大,说明排队现象严重,性能不足
Writing：#当前状态，正在向客户端发送响应报文过程中的连接数,数值越大,说明访问量很大
Waiting：#当前状态，正在等待客户端发出请求的空闲连接数，开启keep-alive时,Waiting+reading+writing=active connections
```

![image-20230808152137681](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230808152137681.png)

#### 11、nginx第三方模块

(1)nginx-module-vts 模块实现流量监控

```bash
可以展示链接数，qps，1xx、2xx,、3xx、4xx、5xx的响应数，响应耗时，响应时间分布，访问用户国家分布；甚至是基于各种状态的流量控制统统能满足你的需求。

vhost_traffic_status_zone;
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
        vhost_traffic_status_display;
        #如果不加这行，默认展示json格式
        vhost_traffic_status_display_format html;
        index off;
    }
}
#这个模块有参数支持写shell来进行自定义和过滤
```

![image-20230808164441201](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230808164441201.png)

(2) echo 模块实现信息显示

```bash
#echo_reset_timer: 这个指令用于重置一个内部计时器，它可以记录从执行这个指令开始到输出结束的时间。这个时间可以通过变量$echo_timer_elapsed来获取。这个指令通常用于测试性能或者延迟。

#echo_location: 这个指令用于同步地访问另一个location，并将其输出内容添加到当前输出中。它可以接受一个可选的参数来指定URL参数。这个指令会阻塞当前请求，直到访问的location返回结果。

#echo_sleep: 这个指令用于延迟输出一段时间，单位是秒。它可以接受一个小数或者变量作为参数。这个指令会阻塞当前请求，直到延迟时间结束。
server {
    listen 80;
    server_name www.linqixin.com;
    root /data/nginx/html;
    location / {
        index index.html;
        default_type text/html;
        echo "hello world,main-->";
        echo $remote_addr ;
        echo_reset_timer;        
        echo_location /sub1;
        echo_location /sub2;
        echo "took $echo_timer_elapsed sec for total.";
    }
    location /sub1 {
        echo_sleep 1;
        echo sub1;
    }
    location /sub2 {
        echo_sleep 1;
        echo sub2;
    }
}
```

#### 12、nginx内置变量

```bash
$remote_addr; #存放了客户端的地址，注意是客户端的公网IP
$proxy_add_x_forwarded_for
#此变量表示将客户端IP追加请求报文中X-Forwarded-For首部字段,多个IP之间用逗号分隔,如果请求中没
有X-Forwarded-For,就使用$remote_addr
the “X-Forwarded-For” client request header field with the $remote_addr variable 
appended to it, separated by a comma. If the “X-Forwarded-For” field is not 
present in the client request header, the $proxy_add_x_forwarded_for variable is 
equal to the $remote_addr variable.
$args; #变量中存放了URL中的所有参数，例如:http://www.wang.org/main/index.do?id=20190221&partner=search，返回结果为: id=20190221&partner=search
$is_args
#如果有参数为? 否则为空
“?” if a request line has arguments, or an empty string otherwise
$document_root; #保存了针对当前资源的请求的系统根目录,例如:/apps/nginx/html。
$document_uri;
#和$uri相同,保存了当前请求中不包含参数的URI，注意是不包含请求的指令，比
如:http://www.wang.org/main/index.do?id=20190221&partner=search会被定义
为/main/index.do 
#返回结果为:/main/index.do
$host; #存放了请求的host名称
limit_rate 10240;
echo $limit_rate;
#如果nginx服务器使用limit_rate配置了显示网络速率，则会显示，如果没有设置， 则显示0
$remote_port;#客户端请求Nginx服务器时随机打开的端口，这是每个客户端自己的端口
$remote_user;#已经经过Auth Basic Module验证的用户名
$request_body_file;#做反向代理时发给后端服务器的本地资源的名称
$request_method;#请求资源的方式，GET/PUT/DELETE等
$request_filename;
#当前请求的资源文件的磁盘路径，由root或alias指令与URI请求生成的文件绝对路径，
如:/apps/nginx/html/main/index.html
$request_uri;
#全路径，包含请求参数的原始URI，包含查询字符串,不包含主机名，相当于:$document_uri?$args,例
如：/main/index.do?id=20190221&partner=search 
$scheme;#请求的协议，例如:http，https,ftp等
$server_protocol;#保存了客户端请求资源使用的协议的版本，例如:HTTP/1.0，HTTP/1.1，HTTP/2.0等
$server_addr;#保存了服务器的IP地址
$server_name;#请求的服务器的主机名
$server_port;#请求的服务器的端口号
$http_user_agent;#客户端浏览器的详细信息
$http_cookie;#客户端的所有cookie信息
$cookie_<name>
#name为任意请求报文首部字部cookie的key名
$http_<name>
#name为任意请求报文首部字段,表示记录请求报文的首部字段，ame的对应的首部字段名需要为小写，如果有横线需要替换为下划线
arbitrary request header field; the last part of a variable name is the field name converted to lower case with dashes replaced by underscores #用下划线代替横线
#示例: 
echo $http_user_agent; 
echo $http_host;
$sent_http_<name>
#name为响应报文的首部字段，name的对应的首部字段名需要为小写，如果有横线需要替换为下划线,此变量
有问题
echo $sent_http_server;
$arg_<name>#此变量存放了URL中的指定参数，name为请求url中指定的参数
echo $arg_id;
$uri;#和$document_uri相同
```

#### 13、nginx自定义变量

```bash
语法格式：
Syntax: set $variable value;
Default: —
Context: server, location, if


范例：
#内置变量不可以赋值，$my_port不是内置变量
set $name wang;
echo $name;
set $my_port $server_port;
echo $my_port;
echo "$server_name:$server_port";
```

#### 14、不记录访问日志

一个网站会包含很多元素，尤其是有大量的images、js、css等静态资源。这样的请求可以不用记录日志。

```bash
#location的uri下的资源访问不记录资源
location ~* .*\.(gif|jpg|png|css|js)$ {
   access_log /dev/null;
   #access_log off;
}
```

#### 15、自定义访问日志

ngx_http_log_module

访问日志是记录客户端即用户的具体请求内容信息，而在全局配置模块中的error_log是记录nginx服务 器运行时的日志保存路径和记录日志的level，因此两者是不同的，而且Nginx的错误日志一般只有一 个，但是访问日志可以在不同server中定义多个，定义一个日志需要使用access_log指定日志的保存路 径，使用log_format指定日志的格式，格式中定义要保存的具体日志内容。

格式：

```bash
#必须放在http模块里面
log_format main  '$remote_addr - $remote_user [$time_local] "$request" '
                 '$status $body_bytes_sent "$http_referer" '
                 '"$http_user_agent" "$http_x_forwarded_for"';
#注意:如果放在http模块里面，此指令一定要在放在log_format命令后
access_log logs/access.log   access_log_format;
```

(1)自定义默认格式日志：

```bash

log_format access_log_format  '$remote_addr - $remote_user [$time_local] "$request" '
                              '$status $body_bytes_sent "$http_referer" '
                              '"$http_user_agent" "$http_x_forwarded_for"'
                              '$server_name:$server_port';
```

(2)自定义json格式日志

```bash
   log_format access_json '{"@timestamp":"$time_iso8601",'
        '"host":"$server_addr",'
        '"clientip":"$remote_addr",'
        '"size":$body_bytes_sent,'
        '"responsetime":$request_time,' #总的处理时间
        '"upstreamtime":"$upstream_response_time",' #后端应用服务器处理时间
        '"upstreamhost":"$upstream_addr",'   
        '"http_host":"$host",'
        '"uri":"$uri",'
        '"xff":"$http_x_forwarded_for",'
        '"referer":"$http_referer",'
        '"tcp_xff":"$proxy_protocol_addr",'
        '"http_user_agent":"$http_user_agent",'
        '"status":"$status"}';
```

#### 16、nginx压缩功能

 ngx_http_gzip_module

```bash
#启用或禁用gzip压缩，默认关闭
gzip on | off; 
#压缩比由低到高从1到9，默认为1
gzip_comp_level level;
#禁用IE6 gzip功能
gzip_disable "MSIE [1-6]\."; 
#gzip压缩的最小文件，小于设置值的文件将不会压缩
gzip_min_length 1k; 
#启用压缩功能时，协议的最小版本，默认HTTP/1.1
gzip_http_version 1.0 | 1.1; 
#指定Nginx服务需要向服务器申请的缓存空间的个数和大小,平台不同,默认:32 4k或者16 8k;
gzip_buffers number size;  
#指明仅对哪些类型的资源执行压缩操作;默认为gzip_types text/html，不用显示指定，否则出错
gzip_types mime-type ...; 
#如果启用压缩，是否在响应报文首部插入“Vary: Accept-Encoding”,一般建议打开
gzip_vary on | off; 
#预压缩，即直接从磁盘找到对应文件的gz后缀的式的压缩文件返回给用户，无需消耗服务器CPU
#注意: 来自于ngx_http_gzip_static_module模块
gzip_static on | off;

```











### 十、nginx反向代理

1、http协议反向代理

Nginx 可以基于ngx_http_proxy_module模块提供http协议的反向代理服务

```bash
#在下面的示例中，proxy_pass后跟的地址尾部是否加/有很大区别。
#如果不加/,那么nginx代理在向后端服务器转发请求的时候，请求资源的地址是proxy_pass+location中的uri，类似于root指令的效果
#proxy_pass http://10.0.0.18:8080; ==> http://10.0.0.18:8080/web/index.html
#如果加/,那么nginx代理在向后端服务器转发请求的时候，请求资源的地址就是proxy_pass，类似于alias指令的效果
#proxy_pass http://10.0.0.18:8080; ==> http://10.0.0.18:8080/index.html
location /web {
    index index.html;
    #proxy_pass http://10.0.0.18:8080/;
    proxy_pass http://10.0.0.18:8080; 
}
```





以下是客户端请求Nginx代理再到后端服务器的详细过程：

1. 客户端发起TCP连接请求：客户端向Nginx代理服务器发起一个TCP连接请求。这个过程包括TCP的三次握手过程，即SYN、SYN+ACK和ACK。
2. Nginx接受TCP连接：Nginx代理服务器接收到客户端的TCP连接请求后，会创建一个新的TCP连接。
3. 客户端发送HTTP请求：一旦TCP连接建立，客户端就可以通过这个连接发送HTTP请求到Nginx代理服务器。HTTP请求包括请求行（包括请求方法、URL和HTTP版本）、请求头和请求体。
4. Nginx解析HTTP请求：Nginx代理服务器接收到HTTP请求后，会解析请求，包括请求方法、URL和请求头等信息。
5. Nginx向后端服务器发起TCP连接：根据解析出的HTTP请求信息，Nginx代理服务器会选择一个后端服务器，然后向这个后端服务器发起一个新的TCP连接请求。
6. 后端服务器接受TCP连接：后端服务器接收到Nginx的TCP连接请求后，会创建一个新的TCP连接。
7. Nginx向后端服务器发送HTTP请求：一旦与后端服务器的TCP连接建立，Nginx就会通过这个连接将客户端的HTTP请求转发到后端服务器。
8. 后端服务器处理HTTP请求：后端服务器接收到HTTP请求后，会处理这个请求，然后生成一个HTTP响应。
9. 后端服务器向Nginx发送HTTP响应：后端服务器将HTTP响应发送回Nginx代理服务器。
10. Nginx向客户端发送HTTP响应：Nginx代理服务器接收到后端服务器的HTTP响应后，会将这个响应转发回客户端。
11. 客户端接收HTTP响应：客户端接收到HTTP响应后，会解析响应，然后根据响应内容进行相应的处理。

以上就是客户端请求Nginx代理再到后端服务器的详细过程。





nginx调优

并发连接数、压缩、



1、让nginx支持高并发要修改哪些参数？pam模块要了解下



# LVS篇

一、





































# HAproxy篇

一、
