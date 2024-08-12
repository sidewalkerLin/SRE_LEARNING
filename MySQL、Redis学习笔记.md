# MySQL篇

## 一、MYSQL的安装

### 1、包管理器安装MYSQL

#### 1.1、centos7+

在centos7+系统中，默认自带的MariaDB，需要增加国内镜像源以及卸载MariaDB相关包

**(1)卸载MariaDB**

```bash
#查看版本，有则需要卸载
rpm -qa|grep mariadb
#卸载所有与MariaDB相关的包
rpm -e --nodeps `rpm -qa|grep mariadb`
#确认是否干净了
rpm -qa|grep mariadb
```

![image-20230923174254200](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230923174254200.png)

**(2)增加国内镜像源**

清华大学镜像源地址：

```bash
https://mirrors.tuna.tsinghua.edu.cn/help/mysql/
```

```bash
#安装过程中被官方坑了下，官方的gpgkey没有加2022，导致key验证不对，因为mysql的gpgkey更新了，我加了2022，用的以前的key
#另外如果以脚本的形式写入，$basearch要转义，要不然会识别为变量变为空，如果直接vim不用加转义字符
cat > /etc/yum.repos.d/mysql-community.repo << EOF
[mysql-connectors-community]
name=MySQL Connectors Community
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-connectors-community-el7-\$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

[mysql-tools-community]
name=MySQL Tools Community
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-tools-community-el7-\$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

[mysql-8.0-community]
name=MySQL 8.0 Community Server
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-8.0-community-el7-\$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
EOF
```

**(3)更新软件包信息**

```bash
yum makecache
```

**(4)安装MySQL**

```bash
yum -y install mysql-community-server
```

**(5)启动MySQL并设置开机启动**

```bash
systemctl enable --now mysqld.service
```

**(6)查找MySQL密码**

```bash
#直接登录mysql会报错，因为mysql会生成随机，在日志中可找到
[root@centos7 ~]# mysql
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
[root@centos7 ~]# cat /var/log/mysqld.log | grep pass
2023-09-23T12:47:40.426510Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: Xu%IUvLFs2qx
```

**(6)登录MySQL并修改密码**

```bash
 #登录之后无任何权限，需要先修改密码
 mysql -uroot -pXu%IUvLFs2qx

#有默认的密码策略，需要改一个复杂密码
alter user 'root'@'localhost' identified by 'lqxLQX--%213';
```

### 2、二进制包安装MYSQL

**(1)下载MySQL的二进制包**

```bash
#MySQL官方下载地址
https://dev.mysql.com/downloads/mysql/

wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.34-linux-glibc2.28-x86_64.tar.gz
```

![image-20230923215413046](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230923215413046.png)

![image-20230923215446404](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230923215446404.png)

![image-20230924130325008](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230924130325008.png)

**(2)安装相关依赖**

```bash
#Ubuntu22.04
apt update && apt -y install libaio1 libaio-dev

#rocky8.6
yum -y install libaio numactl-libs ncurses-compat-libs
```

**(3)创建用户和组**

```bash
#后面的步骤似乎是用root运行mysql，如果要以mysql用户运行，/data/mysql要修改属组属主权限
groupadd -r mysql
useradd -r -g mysql -s /sbin/nologin mysql
```

**(3)解压包到指定目录**

```bash
tar -xf mysql-8.0.34-linux-glibc2.28-x86_64.tar.gz -C /usr/local/

cd /usr/local/
```

**(4)创建软连接**

```bash
ln -s mysql-8.0.34-linux-glibc2.28-x86_64/ mysql
```

**(5)添加属主属组权限**

```bash
chown -R root.root mysql
```

**(6)配置环境变量并在当前shell生效**

```bash
echo 'PATH=/usr/local/mysql/bin:$PATH' > /etc/profile.d/mysql.sh

#在当前shell中执行，使环境变量生效
. /etc/profile.d/mysql.sh
```

**(7)创建主配置文件**

```bash
#可以通过mysql  --help --verbose | grep -A 1 "Default options"查MySQL配置文件支持的路径
cat > /etc/my.cnf << EOF
[mysqld]
datadir=/data/mysql
skip_name_resolve=1
socket=/data/mysql/mysql.sock
log-error=/data/mysql/mysql.log
pid-file=/data/mysql/mysql.pid
[client]
socket=/data/mysql/mysql.sock
EOF
```

**(8)创建数据目录**

```bash
#如果要以mysql用户运行，/data/mysql要修改属组属主权限
mkdir -pv /data/mysql
```

**(9)初始化数据目录，会生成的随机密码**

```ini
mysqld --initialize --user=mysql --datadir=/data/mysql
```

**(10)配置unit文件方便systemd管理**

```bash
#如果要以mysql用户运行，/data/mysql要修改属组属主权限
cat > /lib/systemd/system/mysqld.service << EOF
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql
Type=notify
TimeoutSec=0
ExecStart=/usr/local/mysql/bin/mysqld --user=mysql
LimitNOFILE = 10000
Restart=on-failure
RestartPreventExitStatus=1
Environment=MYSQLD_PARENT_PID=1
PrivateTmp=false

EOF
```

**(11)重新加载systemd配置文件**

```bash
systemctl daemon-reload
```

**(12)启动mysql并配置开机启动**

```bash
systemctl enable --now mysqld.service
```

**(13)查看初始化生成的随机密码并修改MySQL密码，否则无任何权限**

```bash
root@Ubuntu22:/usr/local# grep password /data/mysql/mysql.log
2023-09-24T06:27:59.795973Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: yjSu)LhAH8LC
```

```bash
#登录mysql，无任何权限要先修改密码
mysql -uroot -p'yjSu)LhAH8LC'

#有默认的密码策略，需要改一个复杂密码
alter user 'root'@'localhost' identified by 'lqxLQX--%213';
```

## 二、SQL语言

### 1、SQL语句分类

| SQL语句类型 | 说明         | 具体语句                           |
| ----------- | ------------ | ---------------------------------- |
| DDL         | 数据定义语言 | create、drop、alter                |
| DML         | 数据操作语言 | update、insert、delete             |
| DQL         | 数据查询语言 | select                             |
| DCL         | 数据控制语言 | grant、revoke                      |
| TCL         | 事务控制语言 | begin、commit、rollback、savepoint |

2、常用SQL语句







6、sql优化







## 三、MySQL架构

### 1、存储引擎



### 2、缓冲池Buffer Pool



### 3、服务器配置与状态



### 4、索引



### 5、事务

#### 5.1、事务的特性

事务具有 ACID 四个特性

**原子性( Atomicity )：**原子性是指事务是一个不可分割的工作单元，事务中的操作要么全部完成，要么全部不完成。如果事务成功地完成，那么事务的更改将被永久保存到数据库中。如果在事务中的任何点发生错误，或者如果事务被中断，所有的更改都将被回滚，数据库将回到事务开始前的状态。

**一致性( Consistency )：**一致性是指事务必须使数据库从一个一致性状态转变到另一个一致性状态。也就是说，在事务执行前后，数据都是正确的，不存在矛盾。例如银行转账，从一个账户扣款和向另一个账户存款，这个事务必须满足一致性规则，即转账前后两个账户的总金额必须是一样的。

**隔离性( Isolation )：**隔离性指的是并发执行的多个事务之间的相互隔离程度，防止多个事务并发执行时的交叉执行而导致的数据不一致。事务的隔离分为不同的级别：读未提交、读提交、可重复读、串行化。

**持久性( Durability )：**事务执行成功后，其对于数据的修改会永久保存于数据库中。

#### 5.2、事务的隔离级别

**读未提交**：这是最低的隔离级别，允许一个事务读取其他事务未提交的数据。这会导致脏读、不可重复读、幻读的问题，一般很少使用此隔离级别。

**读提交**：在该隔离级别下，一个事务只能读取其他事务已经提交的数据。这样可以避免脏读的问题，但可能会导致不可重复读、幻读的问题，一般很少使用此隔离级别。

**可重复读**：在该隔离级别下，一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的。事务在启动时创建了一个快照来读取数据，并确保在事务执行期间，其他事务对数据所做的修改不会影响到已经读取过的数据。这可以解决不可重复读、脏读的问题，但可能会导致幻读的问题，即在同一事务中执行相同的查询，但返回的结果集却发生了变化。

**串行**：这是最高的隔离级别，通过加读锁和写锁实现，它通过对事务强制执行串行化顺序来避免任何并发问题。每个事务会依次执行，如同在一个隔离的环境中一样，避免了脏读、不可重复读和幻读的问题。但它也会降低并发性能，因为事务必须按顺序一个接一个地执行。



以上四种隔离级别，从上往下，隔离强度逐渐增强，性能逐渐变差，需要消耗的 MySQL 的资源越多， 所以并不是隔离强度越高越好，采用哪种隔离级别要根据系统需求权衡决定，MySQL 中默认的隔离级 别是可重复读

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

**脏读**：指一个事务读取了另一个事务尚未提交的数据，当尚未提交的事务最终回滚时，读取到的数据就是无效的或错误的。

**不可重复读**：指在同一事务内，对于某个数据进行多次读取时，由于另一个事务对该数据进行了修改并提交，导致得到的结果不一致。

**幻读**：是指一个事务在两次查询之间，另一个事务插入了新的数据行，导致第一个事务第二次查询时发现了之前不存在的数据行。例如，事务A在执行读取操作，需要两次统计数据的总量，前一次查询数据总量后，此时事务B执行了新增数据的操作并提交后，这个时候事务A读取的数据总量和之前统计的不一样，就像产生了幻觉一样，平白无故的多了几条数据，成为幻读。

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

<font color='#ff0000'> **问题1：不可重复读和幻读有什么区别？**</font>

(1) 不可重复读是读取了其他事务更改的数据，**针对update操作**

解决办法：使用行级锁，锁定该行，事务A多次读取操作完成后才释放该锁，这个时候才允许其他事务更改刚才的数据。

(2) 幻读是读取了其他事务新增的数据，**针对insert和delete操作**

解决办法：使用表级锁，锁定整张表，事务A多次读取数据总量之后才释放该锁，这个时候才允许其他事务新增数据。

这时候再理解事务隔离级别就简单多了呢。

<font color='#ff0000'> **问题2：为什么可重复读解决了不可重复读，而没有解决幻读？**</font>

因为可重复读采取了共享锁。

共享锁允许多个事务同时读取同一数据项，但不允许任何事务修改（包括更新和删除）这个数据项。

共享锁也不会阻止其它的事务插入行。

所以可重复读解决了不可重复读的问题，而没有解决幻读的问题。

#### 5.3、管理事务



### 6、锁机制





## 四、日志管理

### 1、事务日志

#### 1.1、redo log 重做日志

redo log是Innodb 存储引擎层生成的日志，它记录的是数据页的变动情况，是物理日志(记录的物理磁盘上数据的变动)，记录的是一些二进制数据。它实现了事务中的**持久性**，用于掉电等故障恢复。

redo log 是物理日志，记录了某个数据页做了什么修改，比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志。

为了防止断电导致数据丢失的问题，当有一条记录需要更新的时候，InnoDB 引擎就会先更新内存（同时标记为脏页），然后将本次对这个页的修改以 redo log 的形式记录下来，后面再由后台线程将缓存在 Buffer Pool 的脏页刷新到磁盘里，这就是 **WAL （Write-Ahead Logging）技术**。

在事务提交时，只要先将 redo log 持久化到磁盘即可，可以不需要等到将缓存在 Buffer Pool 里的脏页数据持久化到磁盘。当系统崩溃时，虽然脏页数据没有持久化，但是 redo log 已经持久化，接着 MySQL 重启后，可以根据 redo log 的内容，将所有数据恢复到最新的状态。

<font color='#ff0000'> **redo log是用来恢复事务commit后，因为系统崩溃等原因，脏页没有落盘的那部分数据的。如果系统崩溃时还没有commit，那在系统重启以后执行的应该是undo log而不是redo log。**</font>

过程如下图：

(1)执行SQL

(2)执行SQL时，Innodb 存储引擎会检查数据页是否在Buffer Pool中，如果不在，则将磁盘中的数据页加载到Buffer Pool中

(3)根据SQL更新数据页并将其标记为脏页

(4)Innodb 存储引擎会将修改记录写入到redo log buffer中(图上未提及)，然后在适当的时机（如事务提交时，或者redo log buffer满时）将redo log buffer中的内容写入到redo log，也就是持久化到硬盘中

(5)然后Innodb 存储引擎会在适当的时机（如Buffer Pool空间不足，或有checkpoint操作）将修改后的数据页从Buffer Pool写回到磁盘

![image-20230925195338917](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230925195338917.png)

#### 1.2、undo log 回滚日志

undo log是 Innodb 存储引擎层生成的日志，实现了事务中的**原子性**，主要**用于事务回滚和 MVCC**。

undo log主要用于实现事务的原子性。如果一个事务在执行过程中发生错误或被用户手动回滚，数据库系统需要能够撤销这个事务已经执行的所有操作，将数据恢复到事务开始前的状态。这就是undo log的作用。undo log记录了事务执行的所有修改操作，但是与redo log不同，undo log记录的是如何撤销这些操作。

每当 InnoDB 引擎对一条记录进行操作（修改、删除、新增）时，要把回滚时需要的信息都记录到 undo log 里，比如：

- 在**插入**一条记录时，要把这条记录的主键值记下来，这样之后回滚时只需要把这个主键值对应的记录**删掉**就好了；
- 在**删除**一条记录时，要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录**插入**到表中就好了；
- 在**更新**一条记录时，要把被更新的列的旧值记下来，这样之后回滚时再把这些列**更新为旧值**就好了

在发生回滚时，就读取 undo log 里的数据，然后做原先相反操作。比如当 delete 一条记录时，undo log 中会把记录中的内容都记下来，然后执行回滚操作的时候，就读取 undo log 里的数据，然后进行 insert 操作。

不同的操作，需要记录的内容也是不同的，所以不同类型的操作（修改、删除、新增）产生的 undo log 的格式也是不同的，具体的每一个操作的 undo log 的格式我就不详细介绍了，感兴趣的可以自己去查查。

一条记录的每一次更新操作产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：

- 通过 trx_id 可以知道该记录是被哪个事务修改的；
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为版本链；

![image-20230925093514075](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230925093514075.png)

#### 1.3、redo log 与 undo log 不同之处

初学redo log和undo log之时，我一直在思考一个问题，它们都是记录的操作日志，用来恢复数据，功能不是一样吗？有什么不同呢？

**实际上redo log是用来恢复数据完成后的状态，而undo log则是用来恢复开始前的状态**。怎么理解这段话呢？

首先，先从它们的格式上下手。

redo log记录的日志是这样的：**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**。如果把它逻辑化是这样的：update xxx set xxx = xxx。如果我们它把再执行一遍，只能达到重新修改值得目的。

undo log记录得日志则是这样的：每一次的操作都是生成一条记录，由当前的记录指向修改前的记录，这样就可以通过指针找到修改前的记录，从而达到回滚的效果。

例如，执行一个事务还未commit时，数据库崩溃了，重启数据库之后，数据库会读取undo中的日志进行回滚(不会读取redo log，因为没有commit)。反之，执行一个事务并commit后，数据库崩溃了，那么可能内存中的数据页只有部分刷入了磁盘中，那么在重启数据库后，数据库会读取redo log日志中的记录，如果脏页已经写入磁盘则跳过这部分记录，如果脏页未写入磁盘，则执行这部分redo log。	

**总结一下，redo log是用于commit后，脏页数据未落盘，来恢复这部分脏页数据的；undo log是用于commit之前，事务执行失败，回滚当事务执行完之前**。

#### 1.4、redo log和undo log的存储

```bash
#虽然在MySQL查到的变量数据如下，但是在MySQL8.0.30以后，redo日志的数量和位置都有变化，与查到的不一致
mysql>  show variables like '%innodb_log%';
+------------------------------------+----------+
| Variable_name                      | Value    |
+------------------------------------+----------+
| innodb_log_buffer_size             | 16777216 |
| innodb_log_checksums               | ON       |
| innodb_log_compressed_pages        | ON       |
| innodb_log_file_size               | 50331648 |#redo日志的大小，48M
| innodb_log_files_in_group          | 2        |#redo日志的数量
| innodb_log_group_home_dir          | ./       |#redo日志的路径，相对于datadir
| innodb_log_spin_cpu_abs_lwm        | 80       |
| innodb_log_spin_cpu_pct_hwm        | 50       |
| innodb_log_wait_for_flush_spin_hwm | 400      |
| innodb_log_write_ahead_size        | 8192     |
| innodb_log_writer_threads          | ON       |
+------------------------------------+----------+
11 rows in set (0.00 sec)

#MySQL8.0.30以前redo日志的路径及数量2个
[root@m52 ~]# ll /var/lib/mysql/ib_logfile* -h
-rw-rw---- 1 mysql mysql 48M Dec 29 16:05 /var/lib/mysql/ib_logfile0
-rw-rw---- 1 mysql mysql 48M Dec  2 21:21 /var/lib/mysql/ib_logfile1

#MySQL8.0.30以后redo日志的路径及数量32个
root@Ubuntu22:~# ll /var/lib/mysql/'#innodb_redo' -h
total 101M
drwxr-x--- 2 mysql mysql 4.0K Sep 25 22:21  ./
drwx------ 8 mysql mysql 4.0K Sep 25 22:21  ../
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:23 '#ib_redo203'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo204_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo205_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo206_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo207_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo208_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo209_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo210_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo211_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo212_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo213_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo214_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo215_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo216_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo217_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo218_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo219_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo220_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo221_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo222_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo223_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo224_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo225_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo226_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo227_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo228_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo229_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo230_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo231_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo232_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo233_tmp'
-rw-r----- 1 mysql mysql 3.2M Sep 25 22:21 '#ib_redo234_tmp'

#undo日志路径
root@Ubuntu22:~# ll -h /var/lib/mysql/undo*
-rw-r----- 1 mysql mysql 16M Sep 25 22:23 /var/lib/mysql/undo_001
-rw-r----- 1 mysql mysql 16M Sep 25 22:23 /var/lib/mysql/undo_002
```

### 2、错误日志

#### 2.1、错误日志主要内容

(1)mysqld启动和关闭过程中输出的事件信息

(2)mysqld 运行中产生的错误信息

(3)运行任何一个事件产生的日志信息，包括一条SQL语句、一个存储过程、一个触发器或任何可执行的代码块

(4)在主从复制架构中的从服务器上启动从服务器线程时产生的信息

#### 2.2、查询错误日志路径

```bash
mysql> select @@log_error;
+--------------------------+
| @@log_error              |
+--------------------------+
| /var/log/mysql/error.log |
+--------------------------+
1 row in set (0.00 sec)

root@Ubuntu22:~# ll /var/log/mysql/error.log
-rw-r----- 1 mysql adm 25565 Sep 25 22:21 /var/log/mysql/error.log
```

#### 2.3、配置错误日志路径

```bash
#Ubuntu中MySQL配置文件路径
vim /etc/mysql/mysql.conf.d/mysqld.cnf

#Redhat中MySQL配置文件路径
vim /etc/my.cnf

[mysqld]
#定义错误日志路径，如果自定义的路径注意权限问题，Ubuntu中注意apparmor.service
log-error=/data/mysqld.log

systemctl restart mysql.service
```

#### 2.4、定义错误日志记录级别

**(1)log_warnings**

log_warnings在MySQL 5.7及更早版本中使用，用于控制错误日志中记录的信息的详细程度，在参数已在MySQL8.0.3中被移除了。

- log_warnings为0：表示不记录告警信息
- log_warnings为1：表示告警信息写入错误日志
- log_warnings大于1：表示各类告警信息，例如有关网络故障的信息和重新连接信息写入错误日志

**(2)log_error_verbosity**

log_error_verbosity在MySQL5.7.2及后续版本中使用，用于替代log_warnings，用于控制错误日志中记录的信息的详细程度。

| log_warnings值 | 日志记录的内容                         |
| -------------- | -------------------------------------- |
| 1              | ERROR                                  |
| 2              | ERROR，WARNING                         |
| 3              | ERROR，WARNING，note message(通知信息) |

### 3、通用日志

#### 3.1、通用日志主要内容

通用日志，又称通用查询日志，用来记录对数据库的所有操作，包括启动和关闭 MySQL 服务、更新语句和查询语句等。

#### 3.2、通用日志相关配置

默认情况下，通用查询日志功能是关闭的。可以通过配置开启此日志，并决定将日志存储到文件还是数据表中。如果选择记录到数据表中，则具体的表是 mysql.general_log。

```bash
#通用日志路径
mysql> select @@general_log,@@general_log_file,@@log_output;
+---------------+-----------------------------+--------------+
| @@general_log | @@general_log_file          | @@log_output |
+---------------+-----------------------------+--------------+
|             0 | /var/lib/mysql/Ubuntu22.log | FILE         |
+---------------+-----------------------------+--------------+
1 row in set (0.00 sec)

#临时开启通用日志
SET GLOBAL general_log = 'ON';
#临时关闭通用日志
SET GLOBAL general_log = 'OFF';
#永久开启通用日志
[mysqld]
general_log = 1
```

**general_log**：

- 值为0，不开启通用日志
- 值为1，开启通用日志

**general_log_file**：通用日志的生成路径

**log_output**：

- 值为FILE，记录到文件中
- 值为TABLE，通用日志到表mysql.general_log中，慢查询日志记录到表mysql.slow_log中
- 值为NONE，不记录任何日志信息

### 4、慢查询日志

#### 4.1、慢查询日志主要内容

慢查询日志用来记录在 MySQL 中执行时间超过指定时间的查询语句。

通过慢查询日志，可以查找出哪 些查询语句的执行效率低，以便进行优化。慢查询日志默认不开启。

#### 4.2、慢查询日志相关配置

```bash
#slow_query_log值为0，不开启慢查询日志；值为1，开启慢查询日志
#slow_query_log_file慢查询日志的生成路径
#long_query_time对慢查询的定义，默认10S，也就是说一条SQL查询语句，执行时长超过10S，将会被记录
#log_queries_not_using_indexes值为1会将未使用索引的查询语句记录到慢查询日志中，不管语句执行时长是否超过阈值
#log_output值为TABLE会将慢查询日志记录到表mysql.slow_log中，同时也会将通用日志到表mysql.general_log中
mysql> select @@slow_query_log,@@slow_query_log_file,@@long_query_time,@@log_queries_not_using_indexes;
+------------------+----------------------------------+-------------------+---------------------------------+
| @@slow_query_log | @@slow_query_log_file            | @@long_query_time | @@log_queries_not_using_indexes |
+------------------+----------------------------------+-------------------+---------------------------------+
|                0 | /var/lib/mysql/Ubuntu22-slow.log |         10.000000 |                               0 |
+------------------+----------------------------------+-------------------+---------------------------------+
1 row in set (0.00 sec)
```

#### 4.3、慢查询日志分析工具

```bash
mysqldumpslow [ OPTS... ] 慢查询日志路径

#常用选项
-v|--verbose #显示详细信息
-d|--debug   #调试模式
--help     #显示帮助      
-s ORDER   #排序
 al #根据平均加锁时长排序
 at #根据平均查询记录行数排序
 ar #根据平均查询时间排序
 c #根据查询结果行数排序
 l #根据加锁时长排序
 r #根据查询行数排序
 t #根据查询时长排序
-r         #根据排序规则反转
-t N #只果看日志中前N行记录  
-a         #显示具体内容，不以抽象形式显示
-g PATTERN #使用 grep 过滤
-h HOSTNAME #根基主机名匹配日志文件
-l         #不从总时间中忽略锁时间

#使用mysqldumpslow报错：Can't determine basedir from 'my_print_defaults mysqld'，这是因为MySQL在安装的时候没有设置basedir参数。
#解决办法：使用mysqldumpslow时后面跟上慢查询日志路径，否则mysqldumpslow找不到，除非你设置了basedir参数
```

### 5、二进制日志

#### 5.1、binlog主要内容

二进制日志中记录了所有对数据库中数据的修改，包括所有DML、DDL语句，但不包含查询操作语句，因为查询语句并 不会改变数据库中的数据。**binlog是在事务commit之后才写入磁盘的，也就是说事务如果rollback是不会计入binlog的。**

```bash
#参考文章
https://www.cnblogs.com/wkynf/p/15855073.html#2.2%EF%BC%9A%E4%BA%8B%E5%8A%A1%E5%9B%9E%E6%BB%9A%E5%AF%B9binlog%E7%9A%84%E5%BD%B1%E5%93%8D
https://www.cnblogs.com/ZhuChangwu/p/14128460.html
```



#### 5.2、binlog文件格式

binlog文件格式分为三种：Statement、Row、Mixed。

**(1)Statement**

基于语句的记录模式，日志中会记录原生执行的 SQL 语句，对于某些函数或变量，不会替换。

**(2)Row**

基于行的记录模式，会将 SQL 语句中的变量和函数进行替换后再记录。

**(3)Mixed**

混合记录模式，在此模式下，MySQL 会根据具体的 SQL 语句来分析采用哪种模式记录日志。

```bash
#原始sql
insert into t1 (age,add_time)values(10,now());

#Statement
#如果以这种模式记录binlog，在执行binlog的时候就会取执行的时间，和实际上sql执行的时间有差别
insert into t1 (age,add_time)values(10,now());
#Row
#相比之下，更消耗空间
insert into t1 (age,add_time)values(10,'2023-09-26 16:02:34');
----------------------------------------------------------------------------------------
#原始sql
update t1 set age=20 where id<5;

#Statement
update t1 set age=20 where id<5;
#Row
update t1 set age=20 where id=4；
update t1 set age=20 where id=3；
update t1 set age=20 where id=2；
update t1 set age=20 where id=1；
```

#### 5.3、binlog相关配置

```bash
sql_log_bin=0或1          #与log_bin共同决定是否开启二进制日志，俩者都开启，二进制日志才会开启，是系统变量而非服务器选项
log_bin=/path/file_name             #与sql_log_bin共同是否开启二进制日志，并指定二进制日志生成路径，是服务器选项
binlog_format=STATEMENT|ROW|MIXED   #binlog记录日志的格式
max_binlog_size=1073741824          #设置单个binlog文件的大小，默认为1GB，用完了或重启会生成新的文件
expire_logs_days=N                  #二进制日志保存天数，超出则删除，默认为0不删除
```

#### 5.4、清理binlog

**(1)删除某个binlog之前的二进制日志文件**

```bash
mysql> show master logs;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000001 |       499 | No        |
| binlog.000002 |       157 | No        |
| binlog.000003 |       201 | No        |
| binlog.000004 |       157 | No        |
| binlog.000005 |       201 | No        |
| binlog.000006 |       180 | No        |
| binlog.000007 |       201 | No        |
| binlog.000008 |       180 | No        |
| binlog.000009 |       157 | No        |
+---------------+-----------+-----------+
9 rows in set (0.01 sec)

mysql> purge binary logs to 'binlog.000005';
Query OK, 0 rows affected (0.00 sec)

mysql> show master logs;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000005 |       201 | No        |
| binlog.000006 |       180 | No        |
| binlog.000007 |       201 | No        |
| binlog.000008 |       180 | No        |
| binlog.000009 |       157 | No        |
+---------------+-----------+-----------+
5 rows in set (0.00 sec)
```

**(2)清理某个时间点之前的binlog**

```bash
#查日志时间用ll命令
PURGE BINARY LOGS BEFORE '2023-09-26 11:34:00';
```

#### 5.5、binlog应用场景

(1)基于时间点的数据备份和恢复

(2)由于误操作删除或修改了数据，也可以利用二进制日志反向恢复

(3)实现MySQL高可用架构的基础

## 五、MySQL备份与恢复

### 1、备份类型

| 备份类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 完全备份 | 备份整个数据库全部数据                                       |
| 部分备份 | 只备份部份数据子集，例如部份库或表                           |
| 增量备份 | 仅备份最近一次完全备份或增量备份（如果存在增量）以来变化的数据，备份较快，还原复杂 |
| 差异备份 | 仅备份最近一次完全备份以来变化的数据，备份较慢，还原简单     |
| 冷备份   | 读、写操作均不可进行，数据库停止服务                         |
| 温备份   | 可读不可写                                                   |
| 热备份   | 读、写操作均可执行，MyISAM 只支持温备，不支持热备，InnoDB都支持 |
| 物理备份 | 直接复制数据文件进行备份，与存储引擎有关，占用较多的空间，速度快 |
| 逻辑备份 | 从数据库中"导出"数据另存而进行的备份，与存储引擎无关，占用空间少，速度慢，可 能丢失精度 |

### 2、冷备份实现

冷备份即将MySQL服务停止，使用rsync命令将数据目录备份到远端服务器。

如果不将服务停止，在备份过程有写操作干扰备份进行，导致备份的数据不一致。

```bash
rsync -a /var/lib/mysql root@10.0.0.183:/root/data/
```

在生产环境中，不可能这么干，所以了解一下即可。

### 3、温备份

温备份即在数据库进行备份时, 数据库的读操作可以执行, 但是不能执行写操作。

这种方式一般是通过FLUSH TABLES WITH READ LOCK给表加读锁，然后通过rsync、scp、mysqldump等工具进行备份。

### 4、热备份之mysqldump逻辑备份

#### 4.1、mysqldump简介

mysqldump是MySQL自带的客户端备份工具。它的备份原理是通过协议连接到 MySQL数据库，将需要备份的数据查询出来，将查询出的数据转换成对应的insert 语句，当我们需要还原这些数据时，只要执行这些 insert 语句，即可将对应的数据还原。

#### 4.2、mysqldump保证数据一致性

在默认情况下，mysqldump在导出数据时，会调用FLUSH TABLES WITH READ LOCK给所有的表加一个读锁，它会阻塞所有的写操作，影响应用的正常运行，在导出结束后才会解锁表。

在这种情况下，如果要解决这个问题，可以加一个**-single-transaction**参数，当然这是针对InnoDB引擎而言，InnoDB引擎是支持事务的，而MyISAM引擎是不支持事务的。-single-transaction可以在不锁表的情况下，完成备份。

```bash
#MySQLdump执行过程原理如下，暂时整不明白，放这里
FLUSH TABLES WITH READ LOCK
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT
commit
参考：
https://blog.csdn.net/qq_34556414/article/details/106781973
http://dbaselife.com/doc/1075/
```

#### 4.3、mysqldump使用方法

```bash
mysqldump [OPTIONS] database [tables...]                    #导出一个库的一个或多个表
mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]  #导出一个或多个数据库
mysqldump [OPTIONS] --all-databases [OPTIONS]               #导出所有数据库

#常用选项
-u|--user=val        #指定用户名
-P|--port=N          #指定端口，默认3306
-p|--password[=val]  #指定密码
-A|--all-databases   #备份所有数据库(创建数据库语句)
-Y|--all-tablespaces #备份所有表空间(建表语句)
-y|--no-tablespaces  #不备份表空间
-a|--create-options  #在create table语句中保留所有mysql特性选项，默认项，可用 --skip-create-options 去掉
-B|--databases db1 db2 #导出指定数据库
-E|--events          #导出事件
-R|--routines        #导出存储过程以及自定义函数
-c|--complete-insert #导出完整的insert语句，包含列名和值，但可能会受到max_allowed_packet参数影响而导致插入失败
-e|--extended-insert #导出多个VALUES列的INSERT语法,默认项，可使用 --skipextended-insert 关闭
-C|--compress        #在客户端和服务端启用压缩传输数据
-d|--no-data         #只备份表结构,不备份数据,即只备份create table
-t|--no-create-info  #只备份数据，不备份表结构
-n|--no-create-db    #不备份create database，可被-A或-B覆盖
-f|--force           #忽略错误
-S|--socket=val      #指定socket文件地址 
F|--flush-logs       #开始导出之前刷新日志，如果使用了 --databases|--alldatabases 导出多个库，会逐个刷新；如果使用了--lock-all-tables|--master-data，日志将会被刷新一次且会锁表

--single-transaction  #不锁表导出数据
--triggers           #导出触发器，默认项，可用 --skip-triggers 关闭
--default-character-set=val #设置默认字符集，默认utf8
--delete-source-logs #备份前刷新二进制日志，相当于执行 flush logs,备份后重置，相当于purge logs，需要配合 --source-data 选项一起使用
--delete-master-logs #备份后删除日志，需要配合 --master-data 选项一起使用
--master-data=N      #此配置在新版中被废弃，使用 --source-data 选项代替
--source-data=N      #将binlog的pos位置和文件名追加到输出文件中，取值为1|2
 #1会保留 CHANGE MASTER TO 语句，适合于集群使用
#2也会有 CHANGE MASTER TO 语句，但会被注释，适合于单机使用
#该选项将打开--lock-all-tables 选项，除非--singletransaction也被指定
 
--default-character-set=val #设置默认字符集，默认值为utf8
--add-drop-database #每个数据库创建语句之前添加 drop database 语句
--add-drop-table #每个数据表创建语句之前添加 drop table 语句
--add-drop-trigger #每个触发器创建语句之前添加 drop trigger 语句
--add-locks #在每个表导出之前lock table，导出后unlock table,默认项，可
使用 --skip-add-locks 去掉
--compatible=val #让备份数据兼容其它相间库类型，可选值包括 
 
#ansi|mysql323|mysql40|postgresql|oracle|mssql|db2|maxdb|
 #no_key_options|no_tables_options|no_field_options
#多个值用逗号分隔，此处并不能保证全完兼容，是尽最大可能保证兼容
--flush-privileges #在导出mysql数据库之后，执行 FLUSH PRIVILEGES，
 #用于导出mysql数据库和依赖mysql数据库数据的任何时候
 
--character-sets-dir=dir #指定字符集文件目录 
--comments #导出备份文件中是否包含注释信息，默认项，可用--skip-comments 关闭
--compact #导出更少的信息，一般用于调试，比如说不导出注释信息等，配合下列选项一起
 #--skip-add-drop-table|--skip-add-locks|--skipcomments|--skip-disable-keys
 
--protocol=val #指定客户端连接协议tcp|socket|pipe|memory
--hex-blob #使用十六进制格式导出二进制字段,影响到的字段类型有BINARY, VARBINARY, BLOB
```

#### 4.4、mysqldump备份策略

```bash
#--single-transaction不锁表进行备份
#--flush-privileges会在生产的备份文件生成的末尾FLUSH PRIVILEGES，可以在导入后刷新权限
mysqldump -uroot -p -A -F -E -R --triggers --single-transaction --flush-logs --source-data=2 --flush-privileges --default-character-set=utf8 --hex-blob > ${BACKUP}/fullbak_${BACKUP_TIME}.sql

#如果版本比较久的话，--source-data要改为--master-data
```

### 5、热备份之xtrabackup物理备份

暂时略

### 6、增量备份

#### 6.1、mysqlbinlog

mysqlbinlog只是一个解析二进制日志的工具，因为可能乱码，它不是用来直接备份的工具。

解析二进制文件，使其不乱码，然后重定向到文件中。

```bash
#用法
mysqlbinlog [options] log-files

#常用选项
--start-position  #起始位置
--stop-position   #结束位置
--start-datetime  #起始时间点
--stop-datetime   #结束时间点

#样例
mysqlbinlog --start-position=157 /data/mysql/binlog.000001 > addbackup.sql
mysqlbinlog --start-position=157 --stop-position=687 /data/mysql/binlog.000001 | gzip > addbackup.sql.gz
mysqlbinlog --start-datetime="2023-08-14 10:00:00" /data/mysql/binlog.000001 > addbackup.sql
```

#### 6.2、增量备份

(1)第一步，进行全量备份

增量备份的前提是完全备份，在完全备份的基础上利用mysqlbinlog进行增量备份。

```bash
#先进行全量备份
mysqldump -uroot -p -A -F -E -R --triggers --single-transaction --flush-logs --source-data=2 --flush-privileges --default-character-set=utf8 --hex-blob > all.sql

#找到全量备份文件备份到了哪个二进制日志文件的哪个position
root@Ubuntu22:/data/mysql# head -n 30 backup.sql | grep -i change
-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000001', MASTER_LOG_POS=157;

```

(2)基于二进制日志进行增量备份

增量备份的过程其实就是备份二进制日志的过程，将全量备份后的二进制日志按照时间进行切割。

```bash
#mysqladmin -uroot -plqxLQX--%213 flush-logs
方案一：每天0点flush logs，然后将昨天的二进制日志备份到到远端
方案二：使用rsync实时同步二进制日志备份到远端
```

## 六、MySQL集群

### 1、MySQL主从

#### 1.1、主从复制原理

主从复制的架构主要由3个线程和2个日志组成。3个线程包括：主节点的dump线程，从节点的io线程以及sql线程。2个日志包括：主节点的二进制日志(binlog)和从节点的中继日志(relaylog)。

由从节点的io线程向主节点请求二进制日志，主节点收到请求后会生成一个dump线程来向从节点的io线程来提供本机的二进制日志。

从节点的io线程收到二进制日志后，会将其保存在从节点的中继日志中。从节点的sql线程会实时检测中继日志是否有更新，如果检测到更新，则将更新的日志解析为sql语句，还原到从节点的数据库中，以此来保证主从节点之间数据的同步。

![image-20230928175627631](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230928175627631.png)

#### 1.2、一主一从实现

| 环境                      | 主节点      | 从节点      |
| ------------------------- | ----------- | ----------- |
| IP                        | 10.0.0.165  | 10.0.0.164  |
| OS                        | Ubuntu22.04 | Ubuntu22.04 |
| DATABASE                  | MySQL8.0.34 | MySQL8.0.34 |
| firewalld                 | 关闭        | 关闭        |
| selinux(如果是reahat系列) | 关闭        | 关闭        |

<font color='#ff0000'> **主从节点的bind-address如有配置需要放行，特别是主节点，考虑到后期可能从会变为主，尽量放行合适的权限。**</font>

(1)主节点配置文件配置

```bash
#先(1)后(2)的原因是重新定义二进制日志文件的生成路径，保证mysqldump导出的数据包含了原目录所有的二进制日志文件内容，要不然源目录的日志文件有一部分，新的目录也有一部分，不太好做同步。当然如果数据库可以停很久的话那就没有啥问题了，而上面这种做法只需要在主节点配置文件写好后restart一下就行，不需要长时间的停着。
vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
#默认为1，主从节点的server-id必须不同
server-id=165
log_bin=/data/mysql/logbin/mysql-bin
#mysql8以上需要配置此项，否则默认插件是caching_sha2_password
default_authentication_plugin=mysql_native_password


mkdir -pv /data/mysql/logbin
chown -R mysql.mysql /data/mysql/
systemctl restart mysql.service
```

```bash
#注意：如果是apt安装的mysql，重启mysql会在日志文件中出现以下报错，这是因为apparmor对/data/mysql/logbin导致的，需要在apparmor配置文件中放行此文件。如果是二进制包安装的mysql，此步略过，因为二进制包安装的MySQL不会生成对应usr.sbin.mysqld文件
mysqld: File '/data/mysql/logbin/mysql-bin.index' not found (OS errno 13 - Permission denied)
#处理办法
vim /etc/apparmor.d/usr.sbin.mysqld

/data/mysql/logbin/ r,
/data/mysql/logbin/** rwk,

systemctl restart apparmor.service
#再次重启mysql，重启成功
systemctl restart mysql.service
```

(2)将主库的数据导出，再导入到从库

```bash
#将从库的二进制日志关闭，因为导入数据生成的二进制日志是没有意义的
mysql -uroot -p
set @@sql_log_bin=0;

#主节点数据导出
mysqldump -uroot -p -A -F -E -R --triggers --single-transaction --flush-logs --source-data=2 --flush-privileges --default-character-set=utf8 --hex-blob > all.sql

#将数据备份文件拷到从节点
scp all.sql 10.0.0.164:/root/

#在从节点登录数据库
mysql -uroot -p
#将数据导入从节点数据库
source all.sql;
```

(3)主节点创建具有复制权限的账号

```bash
create user slavenode@'10.0.0.%' identified by '123456';
grant replication slave on *.* to slavenode@'10.0.0.%';
```

(4)从节点配置文件配置

```bash
vim /etc/mysql/mysql.conf.d/mysqld.cnf

server-id=164
#将数据库设为只读
read_only=on
log_bin=/data/mysql/logbin/mysql-bin
relay_log=/data/mysql/relaylog/relaylog

mkdir -pv /data/mysql/logbin
chown -R mysql.mysql /data/mysql/logbin
mkdir -pv /data/mysql/relaylog
chown -R mysql.mysql /data/mysql/logbin

--------------------------------------------------------
#同样的，如果是apt安装的mysql，要改apparmor配置文件
vim /etc/apparmor.d/usr.sbin.mysqld

/data/mysql/logbin/ r,
/data/mysql/logbin/** rwk,
/data/mysql/relaylog/ r,
data/mysql/relaylog/** rwk,

systemctl restart apparmor.service
--------------------------------------------------------

systemctl restart mysql.service
```

(5)确定从库向主库的二进制日志的哪个position开始拉取数据

```bash
#主库导出的数据中可以查到导出的数据文件到了哪个日志文件的position了，可以以此position为起点拉取数据
root@Ubuntu22:~# head -n 30 all.sql | grep -i change
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=157;

#另外，如果以上述position数据，同步的时候是会包含创建的复制账号的。如果不想同步此账号，可以通过如下方式找到position
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      713 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

(6)从节点指定拉取数据的master节点、用户、密码、position

```bash
CHANGE MASTER TO MASTER_HOST='10.0.0.165', 
MASTER_USER='slavenode',                    
MASTER_PASSWORD='123456',                   
MASTER_LOG_FILE='mysql-bin.000002',         
MASTER_LOG_POS=157;



CHANGE MASTER TO MASTER_HOST='10.0.0.165', 
MASTER_USER='slavenode',                    
MASTER_PASSWORD='123456',                   
MASTER_LOG_FILE='mysql-bin.000007',         
MASTER_LOG_POS=157;

```

(7)查看从节点状态

```bash
#可以查看到change master配置的相关信息，并且io线程和sql线程并未开启
show slave status\G
```

![image-20230929212217540](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230929212217540.png)

(8)开启io线程和sql线程

```bash
start slave；
#再次查看从节点状态，看看是否开启
show slave status\G
```

![image-20230929212533177](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230929212533177.png)

(9)验证主从同步

在主节点修改插入数据，在从节点验证即可

(10)查看主节点dump线程

```bash
show processlist;
```

![image-20230929213050037](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230929213050037.png)

(11)如果从节点配置有问题，可以重置从节点的信息

```bash
stop slave;
reset slave all;
#后面就需要再重新写change master语句了
```

#### 1.3、一主多从实现

一主多从的实现只需要在一主一从的基础上，再给主节点配置一个从节点即可。

```bash
1、先mysqldump导出数据，在新的从节点导入
2、找到主节点导出数据的position，新的从节点需要从这个position读取数据
3、配置从节点mysql配置文件并重启mysql
4、关闭从节点二进制日志(根据实际情况关闭或打开)
5、从节点执行change master语句，账号密码信息用前一个从节点的即可，日志文件position改一下就行
6、show slave status\G  ----->  start slave  ----->  show slave status\G
7、查看io线程、sql线程是否开启，是否有其它报错
8、验证主从同步
```

#### 1.4、级联复制实现

| IP         | 角色     | MySQL版本 | os          |
| ---------- | -------- | --------- | ----------- |
| 10.0.0.165 | 主节点   | 8.0.34    | Ubuntu22.04 |
| 10.0.0.164 | 中间节点 | 8.0.34    | Ubuntu22.04 |
| 10.0.0.163 | 从节点   | 8.0.34    | Ubuntu22.04 |

级联复制，就是增加个中间节点，1个或多个从节点向中间节点获取数据，而不是直接通过主节点。

实际上中间节点也是主节点的一个从节点点，而上面的提到的从节点则是**主节点的从节点的从节点**。

需要注意的是，中间节点需要开启log_slave_updates。这个参数决定主节点同步过来的二进制日志，通过sql线程写入从节点的时候，是否写入从节点的二进制日志文件。

所以中间节点要保证sql_log_bin、log_bin、log_slave_updates都要开启。前俩个参数决定从节点是否开启二进制日志，最后个参数决定sql线程写入从库的时候是否写入二进制日志。

#### 1.5、主主复制实现

主主架构即俩个节点互为主备，两个节点都要开启二进制日志，都要有写权限。

在上面的一主一从架构中，已经实现了10.0.0.165为master，10.0.0.164为slave的架构。所以只需要将10.0.0.165配置为10.0.0.164的从节点即可实现主主架构。

(1)去掉从节点的read_only=on或设置read_only=0

(2)

#### 1.6、半同步复制实现



#### 1.7、GTID复制实现



#### 1.8、主从配置过程中的一些问题

**(1)MySQL身份认证插件问题**

MySQL默认的身份认证插件在MySQL8.0版本由mysql_native_password改为了caching_sha2_password，相比之下caching_sha2_password的安全性更高，但是兼容性也更差，有些比较旧的MySQL客户端不支持这种身份验证插件。

另外，MySQL 的身份验证插件是以用户为单位的。也就是说，不同的用户可以使用不同的MySQL身份验证插件。但是MySQL是有指定默认的身份认证插件的，如果在创建用户的时候没有手动指定身份认证插件，那就是使用默认的身份认证插件。

```bash
#创建用户时指定身份认证插件
CREATE USER 'user1'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
CREATE USER 'user1'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';
#修改用户使用的身份认证插件
ALTER USER 'user1'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
ALTER USER 'user1'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';
FLUSH PRIVILEGES;

---------------------------------------------------------
#修改MySQL默认采用的身份认证插件
vim /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
default_authentication_plugin=mysql_native_password|caching_sha2_password|sha256_password

systemcat restart mysql.service
```

所以在使用MySQL8.0以上版本搭建主从的时候会出现一个报错：

Last_IO_Error: Error connecting to source 'slavenode@10.0.0.161:3306'. This was attempt 1/86400, with a delay of 60 seconds between attempts. Message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.

```bash
#解决办法1
修改用于主从复制的账号的身份认证插件为mysql_native_password，建议在创建账号前就在配置文件修改
[mysqld]
default_authentication_plugin=mysql_native_password

#解决办法二
进入从节点数据库时加上--get-server-public-key，change master的时候也加上
mysql -uroot -p --get-server-public-key

CHANGE MASTER TO
MASTER_HOST='10.0.0.161',
MASTER_USER='slavebak',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=157,
get_master_public_key=1;
```



### 2、MyCat实现读写分离

mycat官网地址：http://www.mycat.org.cn/

#### 2.1、MyCat1.6安装

mycat官网地址：http://www.mycat.org.cn/

(1)mycat官网下载安装包

```bash
wget http://dl.mycat.org.cn/1.6.7.6/20220524101549/Mycat-server-1.6.7.6-release-20220524173810-linux.tar.gz
```

![image-20230930195507637](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230930195507637.png)

![image-20230930195544756](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230930195544756.png)

![image-20230930195644195](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230930195644195.png)

(2)安装java环境

```bash
#mycat是java语言编写的，需要先安装java环境
apt -y install openjdk-11-jdk
java -version
```

(3)解压mycat包到指定目录

```bash
#建这个目录是为了方便管理
mkdir /apps
tar xvf Mycat-server-1.6.7.6-release-20220524173810-linux.tar.gz -C /apps
```

(4)配置PATH变量

```bash
#/apps/mycat/bin下有一些可执行程序，加入到PATH中方便使用
echo "PATH=/apps/mycat/bin:$PATH" > /etc/profile.d/mycat.sh
. /etc/profile.d/mycat.sh
```

(5)启动mycat

```bash
mycat start
#可以看到打开了一些端口
ss -ntl
#查看mycat是否在running
mycat status
```

(6)找到mycat默认的密码

```bash
root@Ubuntu22:~# grep -C 3 password /apps/mycat/conf/server.xml 
	-->

	<user name="root" defaultAccount="true">
		<property name="password">123456</property>
		<property name="schemas">TESTDB</property>
		<property name="defaultSchema">TESTDB</property>
		<!--No MyCAT Database selected 错误前会尝试使用该schema作为schema，不设置则为null,报错 -->
--
	</user>

	<user name="user">
		<property name="password">user</property>
		<property name="schemas">TESTDB</property>
		<property name="readOnly">true</property>
		<property name="defaultSchema">TESTDB</property>
```

#### 2.2、MyCat2.0安装



#### 2.3、MyCat配置文件说明

**(1)mycat配置文件分类**

```bash
root@Ubuntu22:~# tree -L 1 /apps/mycat
/apps/mycat
├── bin          #存放二进制可执行程序
├── catlet       #扩展功能目录，默认为空
├── conf         #配置文件目录
├── lib          #引用的jar包
├── logs		 #日志目录
├── tmlogs		 #
└── version.txt  #版本说明文件
```

**(2)logs**

```bash
/apps/mycat/logs/mycat.log    #mycat启动日志
/apps/mycat/logs/wrapper.log  #mycat详细工作日志
```

<font color='#ff0000'> **(3)conf/server.xml**</font>

```bash
/apps/mycat/conf/server.xml    #存放Mycat软件本身相关的配置文件，比如：连接Mycat的用户，密码，数据库名称等

#相关配置项
user 		#用户配置节点
name 		#客户端登录MyCAT的用户名
password 	#客户端登录MyCAT的密码
schemas 	#数据库名，这里会和schema.xml中的配置关联，多个用逗号分开
privileges 	#配置用户针对表的增删改查的权限
readOnly 	#mycat逻辑库所具有的权限。true为只读，false为读写都有，默认为false
```

<font color='#ff0000'> **(4)conf/schema.xml**</font>

`<schema>`：定义了一个逻辑数据库的schema。`name`属性是逻辑数据库的名称，`checkSQLschema`属性决定是否检查SQL语句中的`schema`，`sqlMaxLimit`属性是SQL查询的最大返回行数，`dataNode`属性是这个schema对应的数据节点。

`<dataNode>`：定义了一个数据节点。`name`属性是数据节点的名称，`dataHost`属性是这个数据节点对应的数据主机，`database`属性是这个数据节点对应的数据库名称。

`<dataHost>`：定义了一个数据主机。`name`属性是数据主机的名称，`maxCon`和`minCon`属性是这个数据主机的最大和最小连接数，`balance`属性是负载均衡策略，`writeType`属性是写操作的类型，`dbType`和`dbDriver`属性是数据库的类型和驱动，`switchType`属性是切换类型，`slaveThreshold`属性是从服务器的阈值。`<heartbeat>`元素定义了心跳查询语句。

`<writeHost>`和`<readHost>`：定义了数据主机的写主机和读主机。`host`属性是主机的名称，`url`属性是主机的URL，`user`和`password`属性是连接主机的用户名和密码。

```bash
/apps/mycat/conf/schema.xml  #对应的物理数据库和数据库表的配置,读写分离、高可用、分布式策略定制、节点控制

#如果需要创建逻辑数据库，在schema.xml定义<schema></schema>就行，不用手动create database。另外server.xml给每个用户都定义了一个默认数据库，引用的是schema.xml中定义的逻辑数据库的名称，俩边名称要一致，否则mycat无法启动，会报错server.xml中填写的数据库名称do not exist！
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
        #name是逻辑数据库的名称，也就是mycat中show databases;查到的数据库名称，它与datanode中的数据库形成映射关系。比如这里的意思就是，mycat中的数据库TESTDB和数据节点dn1中的aaa进行映射，访问TESTDB其实就是访问的aaa。这里的dataNode="dn1"，是引用的后面定义的<dataNode />名称。
        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
        </schema>
        #name是定义的数据节点的名称，datahost是引用后面定义的数据主机的名称，database是数据主机中的一个库，用来和上面的定义的逻辑数据库做映射
        <dataNode name="dn1" dataHost="realserver" database="aaa" />
        #name是定义的数据主机的名称
        <dataHost name="realserver" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
            <heartbeat>select user();</heartbeat>
            <writeHost host="server1" url="10.0.0.165:3306" user="mycat1" password="123456">
            <readHost host="server2" url="10.0.0.164:3306" user="mycat1" password="123456" />
            </writeHost>
        </dataHost>
</mycat:schema>
```

另外需要注意的点：数据主机中的balance参数和writeType参数

```text
balance=0，不开启读写分离机制，所有读操作都发送到当前可用writehost；
balance=1，读操作会默认分发给readhost，当readhost挂掉读操作才会分发给writehost
balance=2，读操作会随机分发到所有的writehost和readhost上
balance=3，所有读请求随机的分发到wiriterHost对应的readhost执行，writerHost不负担读压力
```

```text
writeType=0，所有写操作发送到配置的第一个writeHost；
writeType=1，所有写操作都随机的发送到配置的writeHost；
writeType=2，不执行写操作。
```

**(5)conf/rule.xml**

```bash
/apps/mycat/conf/rule.xml   #Mycat分片（分库分表）规则配置文件,记录分片规则列表、使用方法等
```

#### 2.4、MyCat实现读写分离

| 主机IP     | 角色   | 版本                 |
| ---------- | ------ | -------------------- |
| 10.0.0.163 | mycat  | Mycat-server-1.6.7.6 |
| 10.0.0.165 | master | mysql-8.0.34         |
| 10.0.0.164 | slave  | mysql-8.0.34         |
| 10.0.0.160 | client | mysql-client         |

**(1)实现一主一从架构**

<font color='#ff0000'> **a、主从架构为前提**</font>

<font color='#ff0000'> **b、主从节点的bind-address记得放行，特别是从节点，别忘记了**</font>

<font color='#ff0000'> **c、重启mysql后，临时开启通用日志会失效，忘记了会干扰实验**</font>

**(2)修改mycat默认监听的端口号**

```bash
vim /apps/mycat/conf/server.xml
...
<property name="serverPort">3306</property>
...

mycat restart
#确认3306是否开启
ss -ntl
```

**(3)客户端连接mycat**

```bash
#客户端连接mycat使用的密码在mycat的server.xml文件中可进行配置，默认是123456
apt install -y mysql-client
mysql -uroot -p123456 -h10.0.0.163
```

**(4)在后端数据库中创建供 Mycat 连接的账号**

```bash
#在master节点上创建账号并授权，该帐号会被同步到slave节点
create user 'mycat1'@'10.0.0.%' identified by '123456';
grant all on *.* to 'mycat1'@'10.0.0.%';
flush privileges;
```

**(5)在schema.xml中定义逻辑数据库和后端数据库的映射关系**

```bash
vim /apps/mycat/conf/schema.xml
#配置文件解释见上面的配置文件说明
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
        </schema>
        <dataNode name="dn1" dataHost="realserver" database="aaa" />
        <dataHost name="realserver" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
            <heartbeat>select user();</heartbeat>
            <writeHost host="server1" url="10.0.0.165:3306" user="mycat1" password="123456">
            <readHost host="server2" url="10.0.0.164:3306" user="mycat1" password="123456" />
            </writeHost>
        </dataHost>
</mycat:schema>
```

**(6)重启mycat**

```bash
#配置文件更改后重启mycat
mycat restart

#如果日志出现MyCAT Server startup successfully即启动成功
tail -f /apps/mycat/logs/wrapper.log
```

**(7)开启主从节点的通用日志来观察mycat将sql分发到了哪个服务器**

```sql
#临时开启通用日志
SET GLOBAL general_log = 'ON';
#查看通用日志的路径,名字是主机名来命令的 主机名.log
select @@general_log_file;  

#主从节点可以看到10s一次的mycat发送的select user()心跳检测
tail -f /var/lib/mysql/主机名.log
```

**(8)用一台主机装上mysql客户端来连接mycat，做一些查询和修改操作，到主从的通用日志查看分发到了哪台主机，进行验证**

### 3、MHA实现MySQL高可用

#### 3.1、MHA的工作原理



3.2、MHA实现

(1)下载

```bash
#manager包只支持centos7系统，GitHub上还有Ubuntu的manager包，但是不知道支持到Ubuntu哪个版本
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

#node包centos8、rocky8都支持
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```



### 4、PXC实现MySQL高可用 

4.1、PXC原理与优缺点



4.2、centos7





4.3、Ubuntu

Ubuntu安装参考官网：https://docs.percona.com/percona-xtradb-cluster/8.0/apt.html

国内没有直接的安装包，deb仓库国内倒是有源

```
 https://mirrors.tuna.tsinghua.edu.cn/percona/apt/percona-release_1.0-26.generic_all.deb
```

另外，在percona-release setup pxc80改成57时候报错找不到包，不知道为啥

```bash
apt update && apt install -y wget gnupg2 lsb-release curl
wget https://mirrors.tuna.tsinghua.edu.cn/percona/apt/percona-release_1.0-26.generic_all.deb
dpkg -i percona-release_1.0-26.generic_all.deb
apt update
#设置Percona XtraDB Cluster 8.0的仓库，pxc57是设置5.7仓库，但是设置5.7仓库后安装percona-xtradb-cluster找不到包
percona-release setup pxc80
apt install -y percona-xtradb-cluster
```

### 5、MGR组复制实现MySQL高可用



### 6、主从延迟根因分析与解决方案

#### 6.1、主机从机性能差异

**根因分析：**

在某些生产环境下，存在着从机性能差于主机的情况。在这样的情况下，不管是读写都在主机上进行，还是写在主机读在从机上进行，主从性能差异都可能造成主从延迟的产生。从机如果性能差一点，那处理任务的速度肯定也是差一点的，如果从机上存在多个库都在抢占资源的情况。那就很可能造成主从延迟了。

**解决方案：**

在目前的生产环境下，主机从机一般都是用的同样规格的配置，做的对称部署。这样在主机宕机的时候，可以将从机切换为主机。

#### 6.2、从机压力大

**根因分析：**

在实际的生产环境中，从机往往是承担了一部分读请求的。如果这部分读请求在从机占用了大量的CPU资源，就会影响从机同步的速度，造成主从延迟。

**解决方案：**

(1)搭建一主多从架构，使用多个从机来分担读的压力

(2)通过 binlog 输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力

#### 6.3、大的事务

**根因分析：**

MySQL是在事务commit之后才会写入binlog，然后再进行同步并在从节点回放，同步日志和回放的过程就会造成延迟。

**解决方案：**

让开发将大的事务切割成小的事务

#### 6.4、主写binlog不及时

**根因分析：**

sync_binlog，这个参数用来控制binlog何时被写入磁盘。sync_binlog=0时表示每次事务提交不立即写入磁盘，靠操作系统判断什么时候写入；sync_binlog=1表示每次事务提交都立即刷新binlog到磁盘；sync_binlog=N表示收集N个binlog一起提交。

**解决方案：**

设置sync_binlog=1

#### 6.5、从节点sql单线程处理不及时

**根因分析：**

主库产生了比较多的日志，并同步到中继日志中累积，由于sql线程默认是单个线程来进行回放日志，单线程处理不过来造成延迟。

**解决方案：**

设置slave_parallel_workers参数来指定sql线程数量，设置slave_parallel_type参数为LOGICAL_CLOCK

#### 6.6、网络延迟

找网管吧。。。可能性较小

## 七、MySQL压力测试



有个问题请教下大家：

mysql是在事务commit之前还是在事务commit之后，将二进制日志持久化到磁盘的？

如果是commit之前就将二进制日志持久化到磁盘，那么在主从架构中，主执行事务里面的一句sql，就会写入日志，然后同步到从机执行，这样的话事务大小跟延迟无关吧？

如果是commit之后才将二进制日志持久化到磁盘的，那肯定大的事务造成的延迟就比较高了，但是如果是这样的话，同步复制又是怎么实现的呢？同步复制的原理难道不是主节点在commit之前就将二进制日志同步给从节点的中继日志吗，从节点返回写入成功后，主节点才提交事务吗？

我是观点是，在commit后才写入binlog的，但是不知道怎么解释同步复制这种情况。





# Redis篇

## 一、redis概述

### 1、什么是redis？

Redis 是一种基于内存的数据库，对数据的读写操作都是在内存中完成，因此**读写速度非常快**，常用于**缓存，消息队列、分布式锁等场景**。

Redis 提供了多种数据类型来支持不同的业务场景，比如 String(字符串)、Hash(哈希)、 List (列表)、Set(集合)、Zset(有序集合)、Bitmaps（位图）、HyperLogLog（基数统计）、GEO（地理信息）、Stream（流），并且对数据类型的操作都是**原子性**的，因为执行命令由单线程负责的，不存在并发竞争的问题。

除此之外，Redis 还支持**事务 、持久化、Lua 脚本、多种集群方案（主从复制模式、哨兵模式、切片机群模式）、发布/订阅模式，内存淘汰机制、过期删除机制**等等。

### 2、redis线程模型

我们知道redis的一大特性就是单线程，但redis真的是单线程吗？

实际上redis并不是单线程的，它的后台线程包括关闭文件、AOF 刷盘、释放内存的线程。

redis的单线程特性是指**接收客户端请求->解析请求 ->进行数据读写等操作->发送数据给客户端**这个过程是单线程的，而不是指整个redis只有一个线程

3、

4、



## 二、Redis的安装

### 1、包安装redis

#### 1.1、Ubuntu安装redis

```bash
apt -y install redis
```

#### 1.2、Centos安装redis

```bash
#需要装epel源
yum info redis
dnf -y install redis
systemctl enable --now redis
```

### 2、编译安装redis

官方源码包下载地址：

```text
http://download.redis.io/releases/
```

官方编译安装操作说明：

```text
https://redis.io/docs/getting-started/installation/install-redis-from-source/
```

(1)安装依赖

```bash
#centos安装依赖包
yum -y install gcc make jemalloc-devel systemd-devel
#Ubuntu安装依赖包
apt update && apt -y install make gcc libjemalloc-dev libsystemd-dev
```

(2)下载源码并解压

```bash
wget http://download.redis.io/releases/redis-6.2.9.tar.gz
tar xvf redis-6.2.9.tar.gz -C /usr/local/src/
```

(3)编译安装

```bash
cd /usr/local/src/redis-6.2.9
#如果不支持systemd管理则去掉USE_SYSTEMD=yes
make USE_SYSTEMD=yes PREFIX=/apps/redis install
```

(4)配置环境变量

```bash
echo 'PATH=/apps/redis/bin:$PATH' > /etc/profile.d/redis.sh
. /etc/profile.d/redis.sh
```

(5)准备相关目录和配置文件

```bash
mkdir /apps/redis/{etc,log,data,run}
cp /usr/local/src/redis-6.2.9/redis.conf /apps/redis/etc/
```

(6)前台启动redis

```bash
redis-server /apps/redis/etc/redis.conf

#可能出现以下warning

#warning1
WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
#解决办法
echo net.core.somaxconn=1024 >> /etc/sysctl.conf 
sysctl -p

#warning2
WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can can also cause failures without low memory condition, see https://github.com/jemalloc/jemalloc/issues/1328. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
#解决办法
echo vm.overcommit_memory=1 >> /etc/sysctl.conf 
sysctl -p

#warning3
WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
#解决办法
#ubuntu
vim /etc/rc.local
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled

chmod +x /etc/rc.local
#centos
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
```

(7创建 Redis 用户和设置数据目录权限

```bash

```

(8)
