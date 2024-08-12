### 一、zabbix的架构

![image-20230916221526358](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230916221526358.png)

### 二、安装zabbix-server

<font color='#ff0000'> **windows不支持部署zabbix-server，只能部署被监控端**</font>

安装过程参考官网地址：

https://www.zabbix.com/download

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230814203408576.png" alt="image-20230814203408576" style="zoom: 50%;" />

1、从Zabbix的官方仓库下载一个名为zabbix-release_6.0-4+ubuntu22.04_all.deb的文件

```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb
```

2、安装这个文件，将Zabbix的仓库信息和公钥添加到你的系统中

```bash
dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb
```

3、将Zabbix官方的源替换为阿里云的镜像源，提升下载速度

```bash
sed -i.bak 's#https://repo.zabbix.com#https://mirrors.aliyun.com/zabbix#' /etc/apt/sources.list.d/zabbix.list

#查看是否替换成功
cat /etc/apt/sources.list.d/zabbix.list
```

4、更新你的系统的软件包索引并下载zabbix相关的包

```bash
apt update&&apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2

# zabbix-server-mysql：这是Zabbix服务器的核心组件，它负责收集、存储和处理监控数据，以及发送告警和报告。它需要连接到一个MySQL或者MariaDB数据库来存储数据。

#zabbix-frontend-php：这是Zabbix的Web前端组件，它提供了一个图形化的用户界面，让用户可以配置、管理和查看监控数据。它需要PHP 7.2或者更高版本的支持。

#zabbix-nginx-conf：这是Zabbix的Web前端配置文件，它用于在Nginx Web服务器上运行Zabbix前端。下载这个包的时候nginx会被作为依赖包下载下来。

#zabbix-sql-scripts：这是Zabbix的数据库初始化脚本，它用于创建Zabbix数据库和用户，并导入初始的数据结构和数据。它支持MySQL、PostgreSQL和SQLite3三种数据库。

#zabbix-agent2：这是Zabbix代理的新版本，它用于在被监控的主机上收集本地数据，并发送给Zabbix服务器或者代理。它使用Go语言编写，支持更多的插件和功能。

#apt show zabbix-nginx-conf可以查看这个包所需要的依赖，depends那一项。
```

5、到MySQL服务器安装MySQL、创建zabbix账号并授权

```bash
apt -y install mysql-server 

vim /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address = 0.0.0.0

systemctl restart mysql.service

mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@'10.0.0.%' identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@'10.0.0.%';
#zabbix在第一次生成sql导入mysql的时候，使用了一些SUPER权限的功能，修改此项可以让普通用户使用这些功能。重启mysql此项会失效，恢复默认值。
mysql> set global log_bin_trust_function_creators = 1;
```

6、查看生成的Zabbix数据库初始化脚本

```bash
ll /usr/share/zabbix-sql-scripts/mysql/server.sql.gz
```

7、将Zabbix数据库初始化脚本导入到MySQL服务器上对应的库中

```bash
#zcat这个命令的作用是显示一个压缩文件的内容，而不需要解压缩它。它可以用来查看gzip格式的压缩文件，比如.txt.gz或者.Z的文件
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p123456 -h10.0.0.161 zabbix
```

8、修改zabbix配置文件

```bash
vim /etc/zabbix/nginx.conf

listen          80;
server_name     www.zabbix.com;

nginx -s reload
------------------------------------------------------
vim /etc/zabbix/zabbix_server.conf

#填写MySQL服务器的地址
DBPassword=123456
#填写MySQL服务器的密码
DBHost=10.0.0.161
```

9、停止apache服务并禁止开机启动

```bash
#因为在安装zabbix包的时候，nginx和apache2作为依赖被下载了，如果都设置开机启动会造成端口冲突，这里我们需要的是nginx。
systemctl disable --now apache2.service
```

10、关闭nginx默认监听80的default server

```bash
vim /etc/nginx/sites-enabled/default
将listen注释掉
或者
配置完成后访问10.0.0.160/zabbix
```

11、重启各个服务

```bash
systemctl restart zabbix-server zabbix-agent2 nginx php8.1-fpm
systemctl enable zabbix-server zabbix-agent2 nginx php8.1-fpm
```

11、安装中文语言包

```bash
apt -y install language-pack-zh-hans


#重启各个服务
systemctl restart zabbix-server zabbix-agent2 nginx php8.1-fpm

#清除浏览器缓存，重新进入zabbix的安装页面
·····
```

12、解决图形界面乱码问题

```bash
#注意！！！！！并不是所有windows下的字体都能解决此问题，试了几个都不行，最后上传的“新宋体 常规”成功了
第一步：cd /usr/share/zabbix/assets/fonts

第二步：在windows的C:\Windows\Fonts路径下找到合适的字体，将其上传到服务器上

第三步：将原来的文件备份，mv graphfont.ttf graphfont.ttf.bak

第四步：将刚上传的字体文件重命名为graphfont.ttf，mv 字体文件名 graphfont.ttf

第五步：在web图形界面中找到 User Settings----->Profile,语言选择中文然后更新即可
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230815105000747.png" alt="image-20230815105000747" style="zoom:67%;" />

### 三、zabbix-agent2与zabbix-agent

#### 1、zabbix-agent2与zabbix-agent的区别

zabbix-agent2是zabbix-agent的升级版，zabbix-agent是用C编写的，而zabbix-agent2是用GO来重构的。

#### 2、安装zabbix-agent2

(1)安装zabbix仓库

```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb

dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb

sed -i.bak 's#https://repo.zabbix.com#https://mirrors.aliyun.com/zabbix#'
/etc/apt/sources.list.d/zabbix.list

apt update
```

(2)安装zabbix-agent2

```bash
apt install zabbix-agent2 zabbix-agent2-plugin-*
```

(3)修改agent配置文件，允许zabbix-server监控

```bash
#zabbix-agent默认只允许装在本机的server监控
#修改配置文件，找到server 127.0.0.1修改为zabbix-server的ip
vim /etc/zabbix/zabbix_agent2.conf
```

(4)重启zabbix-agent2

```bash
systemctl restart zabbix-agent2

systemctl enable zabbix-agent2
```

#### 3、zabbix_get工具测试zabbix_agent是否正常

(1)在zabbix-server上安装zabbix-get工具

```bash
#注意安装包的名称是横杠不是下划线，命令才是下划线
apt -y install zabbix-get
```

(2)使用zabbix_get命令测试

```bash
zabbix_get -s 10.0.0.163 -p 10050 -k "system.cpu.load[all,avg1]"

#可能会出现的报错：
1、Get value error: cannot connect to [[10.0.0.150]:10050]: [113] No route to host
解决办法：关闭防火墙

2、Get value error: ZBX_TCP_READ() failed: [104] Connection reset by peer
  Check access restrictions in Zabbix agent configuration
解决办法：修改/etc/zabbix/zabbix_agent2.conf，将server 127.0.0.1配置为zabbix-server的ip，然后重启服务
```

### 四、监控主机与应用

#### 1、监控Linux主机

##### 1.1、创建主机

(1)配置--->主机--->创建主机

![image-20230917150616645](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230917150616645.png)

(2)填写主机的配置

主机名称：

可见的名称：

模板：

群组：

interfaces：

- 客户端：
- SNMP：
- JMX：
- IPMI：

![image-20230917150818567](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230917150818567.png)

注意：模板不点选择可以直接全量模板进行关键字搜索；点击选择进去后，需要先选择主机群组再选模板，无法通过关键字搜索。

##### 1.2、关联监控模板

**(1)监测--->主机，添加监控模板**

![image-20230919095157059](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919095157059.png)

![image-20230919095236414](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919095236414.png)

**(2)配置--->主机，添加监控模板(推荐)**

![image-20230919204639980](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919204639980.png)

##### 1.3、取消关联监控模板

![image-20230920163316486](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920163316486.png)

<font color='#ff0000'> **取消关联监控模板后，该模板的监控项将不再对之前关联的主机进行监控。但是它们仍然会存在于主机中，你会看到主机页面的监控项并不会减少 ，只是不再收集数据。这是因为这些监控项可能已经有数据，如果想移除模板并移除其所有监控项，需要手动删除这些监控项。如果你删除了一个监控项，与该监控项相关的所有历史数据也会被删除。**</font>

![image-20230920164150047](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920164150047.png)

![image-20230920164301470](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920164301470.png)

##### 1.4、修改监控项获取数据的时间间隔

生产中一个主机的监控项一般100个左右，一般间隔5分钟获取数据比较常见，如果获取数据太频繁可能会导致主机死机。

![image-20230919100950666](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919100950666.png)

![image-20230919101043531](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919101043531.png)

![image-20230919101135183](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919101135183.png)

![image-20230919101217359](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919101217359.png)

#### 2、监控windows主机

根据官方文档安装zabbix-agent，打开servers.msc，查看10050端口是否打开，去zabbix-server创建主机关联windows的模板即可。

#### 3、监控nginx服务

zabbix自带了3个监控nginx相关指标的相关模板：

- **Nginx by HTTP**：通过http协议来监控nginx，无需被监控端安装zabbix-agent，需要开启nginx的stub_status的状态页
- **Nginx by Zabbix agent**：通过在被监控端安装zabbix-agent来监控nginx，需要开启nginx的stub_status的状态页
- **NGINX Plus by HTTP**：需要使用nginx的官方收费版nginx plus才能使用此模板

![image-20230919104514210](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919104514210.png)

其中Nginx by HTTP模板和Nginx by Zabbix agent模板都依赖于nginx的http_stub_status_module模块，它提供了nginx的一个状态页，需要配置这个状态页，才能监控这俩个模板的所有监控项，比如active connections就依赖于这个状态页。

另外在配置nginx状态页的时候，location指令指定的uri的路径要与Nginx by HTTP模板和Nginx by Zabbix agent模板宏定义中的NGINX.STUB_STATUS.PATH路径一致，否则无法正确获取相关信息。

```bash
#这里的uri要模板宏定义中的path保持一致，比如都是basic_status；如果nginx中做了修改，模板的宏定义需要同样做修改
location /basic_status {
            stub_status on;
        }
```

![image-20230919223609766](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919223609766.png)

另外这里还会涉及到一个问题，就是如果采用Nginx by HTTP模板来监控客户端的nginx服务(不安装zabbix-agent)，在创建主机的时候，interfaces是没有HTTP这个选项的，只能通过选客户端(选其他协议也行，是通过http来获取的，怎么选无所谓)来进行监控，这样是不影响我们监控进行监控的，照样能监控到数据，但是主机列表的zabbix图标不会绿，因为zabbix图标是判断zabbix-agent的状态来判断的。

#### 4、监控PHP-FPM服务

监控PHP-FPM服务是依赖于PHP-FPM by HTTP模板和PHP-FPM by Zabbix agent模板来进行监控的，它们的监控也类型与nginx，依赖于PHP-FPM的状态页。

![image-20230919231331830](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230919231331830.png)

(1)修改PHP-FPM的配置文件www.conf，开启PHP的状态页

```bash
#在不同的PHP-FPM版本中，www.conf的路径可以能不一样，但主要是以上俩个路径
#将pm.status_path = /status注释去掉
#将ping.path = /ping注释去掉

vim /etc/php-fpm.d/www.conf
vim /etc/php/7.x/fpm/pool.d/www.conf
vim /etc/php/8.x/fpm/pool.d/www.conf

systemctl restart php-fpm.service
```

(2)配置nginx

```bash
#这里uri要与zabbix模板中的宏路径保持一致
location ~ \.php|/status|/ping$ {
             fastcgi_pass unix:/run/php-fpm/www.sock;
             include fastcgi_params;
             fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        }
```

<font color='#ff0000'> **注意如果使用PHP-FPM by HTTP模板，相比于Nginx by HTTP模板，要多修改一个配置：**</font>

![image-20230920160011978](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920160011978.png)

如果不修改此项，zabbix-server会监控本机localhost上的php-fpm服务；而nginx by http模板没有ip的宏定义，它默认取得监控主机的ip。

#### 5、监控java程序



#### 6、监控网络设备SNMP





### 五、zabbix常用功能

#### 1、自定义模板与监控项

##### 1.1、自定义监控项

(1)给监控项取名，并验证是否冲突

```bash
#在zabbix-server上使用zabbix_get来测试监控项名是否与zabbix已有的监控项名冲突,如果监控项名已存在则不能用来命名新创建的监控项
zabbix_get -s 10.0.0.150 -p 10050 -k "login_users"
```

![image-20230920180136647](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920180136647.png)

(2)自定义监控的配置文件

```bash
#zabbix-agent
vim /etc/zabbix/zabbix_agent2.conf
vim /etc/zabbix/zabbix_agent2.d/*.conf   ----推荐，便于管理

#zabbix-agent2
vim /etc/zabbix/zabbix_agentd.conf
vim /etc/zabbix/zabbix_agentd.d/*.conf   ----推荐，便于管理
```

(3)在配置文件中定义UserParameter并重启zabbix-agent2

```bash
vim /etc/zabbix/zabbix_agent2.d/uptime.conf
UserParameter=login_users,uptime | awk '{print $4}'

systemctl restart zabbix-agent2.service 
```

(4)测试新建的监控项脚本是否能获取到数据

```bash
#zabbix-agent2
zabbix_agent2 -t login_users

#zabbix-agent
zabbix_agentd -t login_users
```

![image-20230920181517921](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920181517921.png)

(5)创建一个模板，并创建此监控项与模板关联起来，参考下面自定义模板

##### 1.2、自定义模板

(1)创建空模板

模板：可以关联其它模板，关联上其它模板都就相当于拥有了那个模板的所有监控项，选填

群组：zabbix要求每个模板至少关联一个群组，这样就可以通过群组来找到模板，相当于模板的一个分类，便于管理，必填

![image-20230920172349350](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920172349350.png)

![image-20230920172419540](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920172419540.png)

(2)关联新创建的监控项

![image-20230920193040979](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920193040979.png)

![image-20230920193118888](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920193118888.png)

![image-20230920203110276](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920203110276.png)

![image-20230920203316027](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920203316027.png)

![image-20230920202728311](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920202728311.png)

1.3、监控项脚本优化

如果要监控cpu的1分钟、5分钟、10分钟平均负载，如何编写配置脚本？

(1)传统方法编写配置脚本

```bash
vim /etc/zabbix/zabbix_agent2.d/uptime.conf
UserParameter=cpu_avg1,uptime | awk -F: '{print $NF}' | awk -F, '{print $1}'
UserParameter=cpu_avg1,uptime | awk -F: '{print $NF}' | awk -F, '{print $2}'
UserParameter=cpu_avg1,uptime | awk -F: '{print $NF}' | awk -F, '{print $3}'
```

(2)使用[*]编写配置脚本

```bash
#假如脚本名后面跟的是1，执行1)下的命令；假如脚本名后面跟的是5，执行5)下的命令.....
vim /etc/zabbix/zabbix_agent2.d/cpu_avg.sh

case $1 in
1)
    uptime | awk -F: '{print $NF}' | awk -F, '{print $1}'
    ;;
5)
    uptime | awk -F: '{print $NF}' | awk -F, '{print $2}'
    ;;
15)
    uptime | awk -F: '{print $NF}' | awk -F, '{print $3}'
    ;;
esac

chomd +x /etc/zabbix/zabbix_agent2.d/cpu_avg.sh
```

```bash
#[*]是用来表示参数，可以在请求监控项的时候传递参数。这个*的值无论是多少，有多少个值，都会传给cpu_avg.sh这个脚本，但是cpu_avg.sh取那个值由后面的位置变量来决定。
#如下所示，如果zabbix请求的是cpu.avg[1，5]，那么1和5都会背传给cpu_avg.sh这个脚本。但是，因为位置变量是$1,所以脚本只会将第一个变量，也就是1传入脚本内部执行，最后zabbix得到的结果就是cpu_avg.sh 1的数据。如果是$2,得到的就是cpu_avg.sh 5得数据。如果$1、$2都有，得到的就是cpu_avg.sh 1 5的数据，但是在cpu_avg.sh只定义了$1,所以最后得到的还是cpu_avg.sh 1的数据。
vim /etc/zabbix/zabbix_agent2.d/uptime.conf
UserParameter=cpu.avg[*],/etc/zabbix/zabbix_agent2.d/cpu_avg.sh $1

systemctl restart zabbix-agent2.service 
```

![image-20230920231341567](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230920231341567.png)

#### 2、值映射

<font color='#ff0000'> **值映射就是将获取的到数据映射为自己想显示的文本，比如我们可以通过监听客户端3306端口来判断MySQL服务的状态，但是zabbix获取的到值只能是0或1并将其显示，或许我们可以自己判断：0是端口关闭，1是端口打开，但是这样终归是不好的，万一记错了呢？我们可以通过值映射的方式，将获取到的数据映射到一串文本，比如0代表端口关闭，在web页面显示获取到的数据不是0，而是显示“MYSQL端口关闭”这串文本。**</font>

(1)配置--模板--进入对应的模板

![image-20230921101748396](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230921101748396.png)

(2)点击值映射，添加相应的值映射然后更新保存

![image-20230921102249247](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230921102249247.png)

(3)进入监控项配置，将刚刚配置的值映射配置到相应的监控项下，建立监控项和值映射的关联关系

![image-20230921102059824](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230921102059824.png)

![image-20230921102134886](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230921102134886.png)

(4)结果演示

![image-20230921102337643](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230921102337643.png)

#### 3、触发器





4、



六、



zabbix监控nginx、tomcat、mysql、keepalived的vip漂移、redis

mysqladmin ping  

redis-cli ping  判断是否监控

七、
