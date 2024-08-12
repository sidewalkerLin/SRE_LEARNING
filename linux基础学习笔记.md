## 一、重定向

### 1、标准输出重定向

```bash
#覆盖操作符：
>		标准输出重定向  
1>		同上
2>		标准错误重定向
&>		标准输出和错误都重定向
>&		同上

#追加操作符
>>
1>>
2>>

#将标准输出和标准错误重定向到不同文件
ls /etc null > out.log 2>error.log

#将标准输出和标准错误重定向到相同文件
ls /etc null > out.log 2>out.log
ls /etc null &> out.log
ls /etc null > out.log 2>&1    (这种写法一定要要把&写在>后面，因为&是用来指定文件描述符的)
ls /etc null 2> out.log 1>&2

#错误的写法
ls /etc null 2> out.log 1&>2
原因：是1>&2而不是1&>2，这里的1、2都是文件描述符，&表示引用文件描述符。1&>2表示重定向到2这个文件。
ls /etc null 1>&2 2> out.log
原因：执行这个命令会将标准输出打印到屏幕上，而标准错误重定向到文件中。这是因为，不管是标准输出还是标准错误都是默认在屏幕上打印的，而命令的执行顺序是从左往右的，1>&2表示将标准输出重定向到标准错误输出，在这个节点标准错误输出的默认在屏幕打印的，再后面的节点才会打印到文件中。

关于重定向的坑，重点注意：
#这里cat 11虽然执行失败了，但是out.log依然被清空了
root@Ubuntu22:~# cat out.log 
ls: cannot access 'null': No such file or directory
root@Ubuntu22:~# cat 11 > out.log 
cat: 11: No such file or directory
root@Ubuntu22:~# cat out.log 
root@Ubuntu22:~# 
原因：在执行cat 11 > out.log的时候，系统会先去删除out.log中，再等待cat 11的标准输出。>是覆盖操作符，说是覆盖，其实也就是先删除再写入，所以只要用了>，不管三七二十一，系统会先清空out.log中的内容。
```

### 2、标准输入重定向

#### 2.1标准输入重定向

```bash
标准输入重定向是使用文件来代替键盘的输入
#注意：COMMAND必须是接受标准输入的命令，FILE是文件
#如何判断COMMAND是否接受标准输入？输入COMMAND直接回车，看命令是否等待标准输入。
#常见的接受标准输入的命令有cat、grep、sed、awk、bc、tr
格式：COMMAND < FILE
```

#### 2.2标准输入多行重定向

```bash
使用 "COMMAND<<终止词" 的形式可以在COMMAND读取到终止词的时候结束,一般终止词习惯采用EOF，使用其它一个或多个符号也行
root@Ubuntu22:~# cat << EOF
> 1
> 2
> 333
> EOF
1
2
333
```

#### 2.3高级重定向写法

```bash
（1）COMMAND <<< 字符串\命令
root@Ubuntu22:~# cat <<< "adc"
adc
root@Ubuntu22:~# cat <<< $(echo abc)
abc

（2）COMAND1 < <(COMMAND2)
#<(cmd2) 表示把COMMAND2的输出写入一个临时文件,这样就等价于COMAND1 < FILE
#错误的格式
root@Ubuntu22:~# bc << (echo 1+2+3) 
-bash: syntax error near unexpected token `('
root@Ubuntu22:~# bc <<(echo 1+2+3) 
-bash: syntax error near unexpected token `('
root@Ubuntu22:~# bc < < (echo 1+2+3) 
-bash: syntax error near unexpected token `<'
#正确的格式
root@Ubuntu22:~# bc < <(echo 1+2+3) 
6
root@Ubuntu22:~# bc <<< `echo 1+2+3`
6

```

## 二、管道与xargs

### 1、功能说明

- COMMAND1|COMMAND2|COMMAND3|......
- 将命令的标准输出发送到后面的子命令，如果需要让标准错误输出也传给后面的子命令，则格式为：

```bash
    COMMAND1 2>&1 | COMMAND2
    COMMAND1 |& COMMAND2
```

- 所有命令会在当前shell的子shell进程下执行

### 2、xargs

- 管道是接受前面的标准输入，而xargs是接收到标准输入之后进行处理然后转化为参数，传给后面的命令

xargs会将接收到的标准输入按照空格、换行符、tab进行分割，分割成一个一个的值按空格隔开，再将这个值当作参数传给后面的命令。

```bash
find . -name '*.txt' | xargs cat   ==》 cat 
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230729142945250.png" alt="image-20230729142945250" style="zoom:80%;" />

```
参考资料：
https://zhuanlan.zhihu.com/p/626809049
https://www.cnblogs.com/chenxiaomeng/p/16040498.html
https://zhuanlan.zhihu.com/p/608698918
```

## 三、用户和组

### 1、linux中用户构成

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230729145314203.png" alt="image-20230729145314203" style="zoom:80%;" />

其中，超级管理员root的UID是0，普通用户的UID是1-60000，系统用户的UID是1-999，登录用户的UID是1000-60000。

### 2、linux中组构成

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230729145651895.png" alt="image-20230729145651895" style="zoom:80%;" />

其中，超级管理员组root的GID是0，普通用户组的GID是1-60000，系统用户组的GID是1-999，登录用户组的GID是1000-60000。

### 3、用户和组的关系

- 一个用户至少有1个组，可以多个组
- 组可以0个用户，可以多个用户
- 用户的主要组：又称私有组，一个用户必须属于且只有一个主组，创建用户时， 默认会创建与其同名的组作为主组
- 用户的附加组：又称辅助组，一个用户可以属于0个或多个附加组； 对一个组授权，则该组下所有的用户能能继承这个组的权限

```bash
root@Ubuntu22:~# id linqixin
uid=1000(linqixin) gid=1000(linqixin) groups=1000(linqixin),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd)

root@Ubuntu22:~# groups linqixin
linqixin : linqixin adm cdrom sudo dip plugdev lxd

这里的俩个groups查到的第一个组是主组，后面都是附加组
```

### 4、配置文件

- /etc/passwd：用户及其属性信息(名称、UID、主组ID等） 
- /etc/shadow：用户密码及其相关属性 
- /etc/group：组及其属性信息 
- /etc/gshadow：组密码及其相关属性

### 5、passwd文件

```bash
login name #登录用户名
password #密码位，x只是表示一个占位符，可为空，具体是否有密码要看shadow文件
UID #用户ID，0 表示超级管理员
GID #所属组ID
GECOS #用户全名或注释，描述信息，可为空
directory #用户家目录，在创建用户时，默认会创建在/home 目录下
shell #用户默认shell，/sbin/nologin 表示不用登录的 shell，一般用 chsh 命令修改 chsh -s /bin/bash user

```

### 6、shadow文件

```bash
login name                    #登录用户名
encrypted password  #加密后的密文，一般用sha512加密，可为空，!、!!、*表示该用户无密码不能登录，但可以通过超级管理员切换用户
date of last password change  #上次修改密码的时间，自1970年开始，0表示下次登录之后就要改密码，为空表示密码时效功能无效
minimum password age  		  #最小时间间隔，当前密码最少能使用多少天，0表示随时可被变更
maximum password age          #最大时间间隔，当前密码最多能使用多少天，99999表示可以一直使用
password warning period       #警告时间，密码过期前几天开始提醒用户，默认为7
password inactivity period    #不活动时间，密码过期几天后帐号会被锁定，在此期间，用户仍然可以登录，为空表示不使用此规则
account expiration date       #失效时间，从1970年1月1日算起，多少天后帐号失效，为空表示永不过期
reserved field                #保留字段，无意义
```

可以通过在密码前加！的方法来禁用账号。注意，如果你通过这种方法禁用账号，root用户依然可以切换到这个用户。

### 7、group文件

```bash
group_name      #组名
password   		#组密码，当用户加组时，需要用此密码验证
GID       		#组ID
user_list 		#用户列表，多个用户用,分隔, 此处的用户将当前组作为附加组
```

### 8、gshadow文件

```bash
group name 				#组名
encrypted password 		#组密码，加密后的密文，!表示还没设密码
administrators 			#组管理员
members 				#用户列表，多个用户用,分隔, 此处的用户将当前组作为附加组
```

9、useradd

```bash
-u|--uid UID             	#指定UID
-g|--gid GID             	#指定用户组，-g groupname|--gid GID
-c|--comment COMMENT        #新账户的 GECOS 字段
-d|--home-dir HOME_DIR      #指定家目录，可以是不存在的，指定家目录，并不代表创建家目录
-s|--shell SHELL            #指定 shell，可用shell在/etc/shells 中可以查看
-r|--system                 #创建系统用户,CentOS 6之前 ID<500，CentOS7 以后ID<1000，不会创建登录用户相关信息
-m|--create-home            #创建家目录，一般用于登录用户
-M|--no-create-home         #不创建家目录，一般用于不用登录的用户
-p|--password PASSWORD      #设置密码，这里的密码是以明文的形式存在于/etc/shadow 文件中
-o|--non-unique             #允许使用重复的 UID 创建用户
-G|--groups GROUP1[,GROUP2,...]   #为用户指明附加组，组须事先存在
-N|--no-user-group          #不创建同名的组,使用users组做主组
-D|--defaults               #显示或更改默认的 useradd 配置，默认配置文件是/etc/default/useradd
-e|--expiredate EXPIRE_DATE #指定账户的过期日期 YYYY-MM-DD 格式
-f|--inactive INACTIVE      #密码过期之后，账户被彻底禁用之前的天数，0 表示密码过期立即禁用，-1表示不使用此功能
-k|--skel SKEL_DIR          #指定家目录模板，创建家目录，会生成一些默认文件，如果指定，就从该目录复制文件，默认		                                  是/etc/skel/，要配合-m
-K|--key KEY=VALUE          #不使用 /etc/login.defs 中的默认值，自己指定，比如-K UID_MIN=100 
-l|--no-log-init  #不将用户添加到最近登录和登录失败记录，前面讲到的3a认证审计，就在此处lastlog|lastb|cat/var/log/secure
```

10、

## 四、tr | sort | uniq | cut | wc | paste | head | tail 命令

### 1、tr命令

用于转换字符、删除字符和压缩重复的字符。它从标准输入读取数据并将结果输出到标准输出。<font color='#ff0000'> **注意：它不接受直接从文件读入**</font>。

```bash
tr [OPTION]... SET1 [SET2]

tr SET1 SET2            #将SET1中的字符都替换为SET2，如果SET2的范围小于SET1的范围，那么SET1超出的部分字符将会替换为SET2的最大临界值
root@Ubuntu22:~/test# echo 'abcddasdflzz' | tr 'a-z' 'A-D'
ABCDDADDDDDD


#常用选项 
-c|-C|--complement 		#用SET2替换SET1中没有包含的字符
-d|--delete         	#删除SET1中所有的字符，不转换
-s|--squeeze-repeats 	#压缩SET1中重复的字符，即删除重复的字符
-t|--truncate-set1   	#将SET1用SET2替换，SET2中不够的，就不处理

[:print:]     #包含了所有可打印的ASCII字符，也就是从32到126的ASCII码，包括空格、数字、字母、标点符号和一些特殊符号。
[:ascii:]     #包含了所有的ASCII字符，也就是从0到127的ASCII码，除了[:print:]字符集之外，还包括一些控制字符，比如换行、退格、制表符等。
```

范例1：<font color='#ff0000'> **tr SET1 SET2和tr -c SET1 SET2的区别**</font>

![image-20230819102407185](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819102407185.png)

范例2：<font color='#ff0000'> **tr -d SET1 **</font>

![image-20230819103227236](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819103227236.png)

范例3：<font color='#ff0000'> **tr -s SET1与tr -s SET1 SET2**</font>

tr -s SET1是用于将连续的重复字符压缩为单个字符

tr -s SET1 SET2是用于将连续的重复SET1字符替换为SET2

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819104709123.png" alt="image-20230819104709123" style="zoom:80%;" />

范例4：<font color='#ff0000'> **tr SET1  SET2与tr -t SET1 SET2**</font>

tr SET1 SET2会将SET1中的字符都替换为SET2，如果SET2的范围小于SET1的范围，那么SET1超出的部分字符将会替换为SET2的最大临界值。

tr -t SET1 SET2则是不会处理SET1超出的部分字符。

![image-20230819111229252](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819111229252.png)

### 2、sort命令

把整理过的文本显示在STDOUT，不改变原始文件。默认情况下，sort 按照每行的第一个字符进行排序，排序的规则默认是依赖ASCII 码进行排序。

```bash
sort [OPTION]... [FILE]...
#常用选项
-b|--ignore-leading-blanks 	#忽略文件中的空白
-f|--ignore-case           	#忽略大小写
-h|--human-numeric-sort 	#人类可读排序，按照单位大小
-M|--month-sort          	#以月份排序
-n|--numeric-sort         	#要求每一行的第一个字段必须是有效的数值，然后根据这个数值的大小进行排序。每一行的第一个字段是指以空格或制表符分隔的第一个字符串。
-R|--random-sort           	#随机排序
-r|-reverse               	#倒序
-t|--field-separator=SEP   	#指定列分割符
-k|--key=KEYDEF           	#指定排序列
-u|--unique               	#去重
-z|--zero-terminated       	#以 NUL 字符而非换行符作为行尾分隔符

```

sort排序规则：

```bash
sort      #以每一行的第一个字符排序，如果一行的第一个字符相同，则比较第二个字符，以此类推
sort -b   #如果行首存在空白字符，则忽略空白字符，取行首空白字符后面的第一个非空白字符来和其它行的第一个字符排序
sort -h	  #以单位进行排序，如K、M、G等
sort -n   #以每一行的第一个字段按数值大小排序，要求每一行的第一个字段必须是有效的数值，否则排序结果会出错。每一行的第一个字段是指以空格或制表符分隔的第一个字符串。
sort -r   #-r不会改变排序规则，它只是将排序的结果倒序显示。
sort -u   #清除重复行，并按sort默认排序
```

范例1：<font color='#ff0000'> **sort -b**</font>

sort -b用于忽略行首的空白字符。如果不加-b，那么行首是空白字符的行将会依靠来排序，并且默认情况下是按照ASCII 码排序。在下图的示例中，sort test.txt 和 sort -b test.txt 结果一样是因为受了本地locale的影响，正常按照ASCII 码排序，应该是LC_ALL=C sort test.txt得到的结果，空白字符的ASCII 码比字母会更靠前，所以会显示在最前面。

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819113659908.png" alt="image-20230819113659908" style="zoom:80%;" />

范例2：<font color='#ff0000'> **sort -n与sort**</font>

sort的排序规则：以每一行的第一个字符排序，如果一行的第一个字符相同，则比较第二个字符，以此类推

sort -n的排序规则：以每一行的第一个字段按数值大小排序，要求每一行的第一个字段必须是有效的数值。每一行的第一个字段是指以空格或制表符分隔的第一个字符串。

在下图的示例中，每行按空格或制表符分隔后的第一个字段分别是1、5、10、24、33，是有效的数值，所以sort -n会按他们的大小来排序。而对于sort来说，它会取每行的第一个字符排序，但是第一个字符存在相等的情况，那么就比较第一个字符存在相等的几行中第二个字符的大小，如果还存在相同，那就是继续比第三个字符....

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819145917273.png" alt="image-20230819145917273" style="zoom:80%;" />

范例3：<font color='#ff0000'> **sort -r**</font>

-r不会改变排序规则，它只是将排序的结果倒序显示

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819152410806.png" alt="image-20230819152410806" style="zoom:80%;" />

范例4：<font color='#ff0000'> **sort -k**</font>

指定了列以后，sort默认也是取得指定列的第一个字符进行排序的

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819153955932.png" alt="image-20230819153955932" style="zoom:80%;" />

范例5：<font color='#ff0000'> **sort -t**</font>

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819155307050.png" alt="image-20230819155307050" style="zoom:80%;" />

范例6：<font color='#ff0000'> **sort -u**</font>

清除重复行，并按sort默认排序

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230819160841867.png" alt="image-20230819160841867" style="zoom:80%;" />

### 3、uniq命令

uniq命令从输入中清除<font color='#ff0000'> **前后相邻的重复的行**</font>，常和 sort 配合使用，sort之后重复的行就相邻了

```bash
uniq [OPTION]... [INPUT [OUTPUT]]
#常见选项
-c|--count             #显示每行出现次数
-d|--repeated          #仅显示有重复行
-D                     #显示所有重复行具体内容
-u|--unique            #仅显示不重复的行
-z|--zero-terminated   #以 NUL 字符而非换行符作为行尾分隔符
```







### 4、cut命令





### 5、wc命令



6、paste命令





7、head命令





8、tail命令





## 五、mv、cp、scp、rsync

1、mv





2、cp







3、scp

scp 命令利用ssh协议在两台主机之间复制文件

格式：

```bash
scp [options] SRC... DEST/
#常用选项
-C 		#压缩数据流，指定加密算法进行压缩传输
-r 		#递归复制，复制目录文件时使用
-p 		#保持原文件的部分属性信息。保留全部属性用rsync
-q 		#静默模式
-P PORT #指定远程服务器的端口,默认22
```

范例1：将本机文件复制到远程主机的<font color='#ff0000'> **目录文件**</font>下

<font color='#ff0000'> **错误写法！！！！！！！！！！：**</font>

这种写法中，scp并不知道data文件是目录文件，它会在远程主机上创建一个普通文件data并将原本的目录文件data给覆盖掉

```
scp alpine-3.18.3.tar 10.0.0.150:/data
```

<font color='#ff0000'> **正确写法：**</font>

```bash
scp alpine-3.18.3.tar 10.0.0.150:/data/
```

范例2：将远程主机文件复制到本机

```bash
scp 10.0.0.163:/root/hosts .
```

范例3：将本机目录文件复制到远程主机或将远程主机目录文件复制到本机

```bash
#如果本机目录文件或远程主机目录文件为空，则无法复制。简单来说就是无法复制空的目录文件。

#假如10.0.0.163存在文件/root/test/alpine-3.18.3.tar
scp -r  10.0.0.163:/root/test/ .    #会将test目录复制过来
scp -r  10.0.0.163:/root/test  .    #会将test目录复制过来
scp -r  10.0.0.163:/root/test/* .   #会将test目录下的子文件复制过来，test目录文件本身不复制过来
```

4、rsync

scp 命令在复制文件时是全量复制，不管文件有没有改动，都是复制，速度慢，消耗资源。

rsync工具可以基于ssh和rsync协议实现高效率的远程系统之间复制文件，使用安全的shell连接做为传 输方式，比scp更快，基于增量数据同步，即只复制两方不同的文件，此工具来自于rsync包。

注意：通信两端主机都需要安装 rsync 软件。

```bash
rsync [OPTION]... SRC [SRC]... DEST
#常用选项
-n|--dry-run 	#只测试，不执行
-v|--verbose 	#显示详细过程
-r|--recursive 	#递归复制
-p|--perms 		#保留权限属性
-t|--times 		#保留修改时间戳
-g|--group 		#保留组信息
-o|--owner 		#保留所有者信息
-l|--links 		#将软链接文件本身进行复制（默认）
-L|--copy-links #将软链接文件指向的文件复制
-u|--update 	#如果接收者的文件比发送者的文件较新，将忽略同步
-z|--compress 	#压缩，节约网络带宽
-a|--archive 	#保留所有属性，但不包括acl和selinux
--delete 		#如果源数据中的文件被删除，则目标中的对应文件也要被删除，配合-r使用
--progress 		#显示进度
--bwlimit=5120 	#限速以KB为单位,5120表示5MB

```

六、ab、dd命令



七、ss、netstat

1、netstat

netstat 命令用于显示各种网络相关信息，如网络连接，路由表，接口状态 ，masquerade 连接，多播成员等等。

```bash
-a    #显示所有选项，默认不显示LISTEN相关
-t 	  #仅显示tcp相关选项
-u    #仅显示udp相关选项
-n    #拒绝显示别名，能显示数字的全部转化成数字
-l    #仅列出有在 Listen (监听) 的服務状态
-p    #显示建立相关链接的程序名
-r    #显示路由信息，路由表
-e    #显示扩展信息，例如uid等
-s    #按各个协议进行统计
-c    #每隔一个固定时间，执行该netstat命令。
```

2、ss



```bash
-n, --numeric   #不解析服务名称
-r, --resolve   #解析主机名
-a, --all       #显示所有套接字（sockets）
-l, --listening #显示监听状态的套接字（sockets）
-o, --options   #显示计时器信息
-e, --extended  #显示详细的套接字（sockets）信息
-m, --memory    #显示套接字（socket）的内存使用情况
-p, --processes #显示使用套接字（socket）的进程
-i, --info      #显示 TCP内部信息
-s, --summary   #显示套接字（socket）使用概况
-4, --ipv4      #仅显示IPv4的套接字（sockets）
-6, --ipv6      #仅显示IPv6的套接字（sockets）
-0, --packet    #显示 PACKET 套接字（socket）
-t, --tcp       #仅显示 TCP套接字（sockets）
-u, --udp       #仅显示 UCP套接字（sockets）
-d, --dccp      #仅显示 DCCP套接字（sockets）
-w, --raw       #仅显示 RAW套接字（sockets）
-x, --unix      #仅显示 Unix套接字（sockets）
```

