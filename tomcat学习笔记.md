### 一、JVM、JDK、JRE

#### 1、JVM

JVM是JAVA虚拟机，了解什么是java虚拟机，需要先了解java虚拟机的由来。

Java语言最初是为了开发嵌入式系统而设计的，但是由于当时不同的硬件平台有不同的指令集和操作系统，导致Java程序很难实现在不同的设备上运行。

比如不同的操作系统会选择不同的CPU体系架构，Windows主要支持X86和X64指令集，而Linux和Android可以支持多种指令集，如X86、X64、ARM、MIPS等。再比如不同的CPU体系有不同硬件，X86和ARM是两种不同的CPU体系架构，它们有着不同的寄存器、指令格式、寻址方式等。这样的话，在一个操作系统上编译出来的字节码文件，是无法适配其它操作系统的，那么就需要在每个操作系统都编译一次，这样就增加开发和维护的成本和复杂度。

为了解决这个问题，所以设计出了JAVA虚拟机，它是一个软件层面的虚拟机，能够将.class文件中的字节码指令转换为操作系统的API调用。。在javac将java编译为统一的字节码之后(不同的操作系统编译的内容也是相同的)，JVM会将编译好的字节码转换为本地的机器码，类似于一个字节码解释器，这样就解决java的跨平台问题。当然JVM的作用不止于此，它还提供了内存管理、即时编译、异常处理、类加载和运行时环境等功能。

#### 2、JRE

JRE（Java Runtime Environment）是Java运行时环境，它包含了JVM和Java程序所需的核心类库，比如java.lang、java.util、java.io等。它相对于 jvm 来说，多出来的是一部分的 Java 类库。

#### 3、JDK

JDK（Java Development Kit）是Java开发工具包，它包含了JRE和一些开发和调试工具，比如javac.exe（编译器）、java.exe（解释器）、jar.exe（打包工具）、javadoc.exe（文档生成工具）、jdb.exe（调试器）等。

最常用的JDK版本是JDK8，也叫JAVA SE 1.8.0。

### 二、JDK的安装

#### 1、ORACLE JDK二进制安装

(1)下载ORACLE JDK二进制文件

```b
https://www.oracle.com/java/technologies/downloads/
```

如果要选择具体的JDK版本进行下载，需要到JAVA的归档中去：

![image-20230810103141390](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230810103141390.png)

另外JDK8有俩个版本，它们的区别是Java SE 8 (8u211 and later)更适用于商用，Java SE 8 (8u202 and earlier)适用于个人。个人玩的话，随便哪个版本都行。

(2)将文件下载后上传到服务器上并解压

```bash
#为了方便管理上传到此目录下
cd /usr/local
tar xvf jdk-8u381-linux-x64.tar.gz
```

(3)初始化环境变量

```bash
vim /etc/profile.d/jdk.sh

export JAVA_HOME=/usr/local/jdk1.8.0_381
export PATH=$PATH:$JAVA_HOME/bin
```

(4)让当前的shell环境能够识别和使用这些环境变量

```bash
source /etc/profile.d/jdk.sh
```

(5)验证java版本

```bash
java -version
```

#### 2、Oracle JDK 的 rpm安装



3、openjdk的安装

### 三、TOMCAT的安装

#### 1、二进制安装tomcat

1、下载包并解压

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.91/bin/apache-tomcat-8.5.91.tar.gz
tar xf apache-tomcat-8.5.50.tar.gz -C /usr/local/
cd /usr/local/
```

2、创建/usr/local/apache-tomcat-8.5.50.tar.gz的软连接，便于管理

```bash
ln -s ./apache-tomcat-8.5.91 tomcat
```

3、修改tomcat文件权限

```bash
chown -R tomcat.tomcat /usr/local/tomcat/
```

4、配置PATH变量

```bash
vim /etc/profile.d/tomcat.sh

PATH=/usr/local/tomcat/bin:$PATH
```

5、在当前shell环境下将PATH生效

```bash
source /etc/profile.d/tomcat.sh
echo $PATH
```

6、创建tomcat专用账户

```bash
useradd -r -s /sbin/nologin tomcat
```

7、配置tomcat自启动service文件

```bash
vim /usr/local/tomcat/conf/tomcat.conf

JAVA_HOME=/usr/local/jdk1.8.0_381
```

```bash
vim /lib/systemd/system/tomcat.service

[Unit]
Description=Tomcat
#After=syslog.target network.target remote-fs.target nss-lookup.target
After=syslog.target network.target 
[Service]
Type=forking
EnvironmentFile=/usr/local/tomcat/conf/tomcat.conf
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
PrivateTmp=true
User=tomcat
Group=tomcat
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload

systemctl enable --now tomcat
```

8、查看tomcat状态

```bash
systemctl status tomcat
```

2、Ubuntu包安装tomcat



3、centos包安装tomcat

### 四、TOMCAT文件结构

#### 1、目录结构

bin：服务启动、停止等相关程序和文件

conf：配置文件

lib：库目录

logs：日志目录

webapps：应用程序，应用部署目录

work：jsp编译后的结果文件，建议提前预热访问，升级应用后，删除此目录数据才能更新

2、bin目录详解



### 五、JVM垃圾回收算法

#### 1、引用计数法

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值加１；当引用失效时，计数器值减１,引用数量为0的时候,则说明对象没有被任何引用指向,可以认定是”垃圾”对象。

JVM中并没有采用引用计数法来确定垃圾，在python语言中才采用这个方法。

引用计数法无法解决循环引用的问题，如果俩个对象互相引用，那么它们的计数器永远不会为0，那么它们的内存永远不会被释放，这就会造成内存泄露。如果出现大量的循环引用，大量的内存空间得不到释放，那么最后就会造成内存溢出。

另外还有1个问题：计数器为0的对象何时会被回收呢？有3种策略

(1)在每次引用变化时检查计数器的值，如果为0就立即回收该对象(python采用这种策略，内存能及时释放，但是会增加开销)

(2)定期扫描所有对象的计数器，如果为0就回收该对象

(2)在内存不足时才触发垃圾回收，释放计数为0的对象

#### 2、根搜索可达算法

(1)确定GC Roots的集合，这些是一些必须存活的对象，如虚拟机栈中的局部变量、方法区中的静态变量和常量等

(2)从GC Roots出发，遍历它们所引用的对象，将这些对象标记为可达。如果一个对象被多个GC Roots引用，只需要标记一次即可

(3)继续遍历被标记对象所引用的对象，将它们也标记为可达。这样就形成了一个递归的过程，直到没有更多的对象可以被标记为止。

(4)最后，对于没有被标记为可达的对象，就认为它们是不可达的，可以被垃圾收集器回收

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230811220545297.png" alt="image-20230811220545297" style="zoom: 67%;" />

举例：

```java
public class Test {
public static void main(String[] args) {
Object obj1 = new Object(); // obj1是一个GC Roots，因为它是一个局部变量
Object obj2 = new Object(); // obj2也是一个GC Roots，因为它也是一个局部变量
obj1 = obj2; // obj1指向obj2，原来obj1指向的对象就没有任何引用链相连，可以被回收
String str = "Hello"; // str是一个GC Roots，因为它是一个常量
System.out.println(str); // 打印str
}
}
```

在这个例子中，我们可以看到有三个对象被创建了：一个是obj1指向的对象，一个是obj2指向的对象，还有一个是str指向的字符串对象。其中，obj1和obj2都是虚拟机栈中的局部变量，所以它们都是GC Roots。str也是一个GC Roots，因为它是方法区中的常量。当执行到第四行代码时，obj1被重新赋值为obj2，这时候原来obj1指向的对象就没有任何引用链相连了，所以它就是不可达的，可以被垃圾回收器标记为垃圾对象。而obj2指向的对象和str指向的字符串对象都还有引用链相连，所以它们都是可达的，不能被回收。

#### 3、标记-清除算法

(1)标记阶段：通过根搜索可达算法，将递归扫描到的对象标记为可达，也就是可存活的对象

(2)清除阶段：GC遍历整个堆空间，将未被标记的对象清除掉

总结：

使用标记-清除算法会产生大量的内存碎片，但是不浪费空间，效率较高

#### 4、复制算法

(1)将一块内存分成2块内存空间，2块内存空间不一定要相等

(2)一块内存空间用来存放创建的对象，另一块则空闲

(3)当一块内存空间用完后，GC会通过可达性分析来识别出存活对象，将存活对象复制到另一空闲的内存空间中并清理干净剩余的垃圾

(4)继续重复(3)的过程

总结：

使用复制算法会比较浪费内存，因为总是需要保留一块空闲的内存空间。而且它比较适用于存活对象较少的场景，因为复制算法是需要将存活对象从一片空间移动到另一片空间的，如果存活对象多的话会大大增加复制的消耗，影响垃圾回收的性能。

#### 5、标记-压缩算法

(1)标记阶段：通过根搜索可达算法，将递归扫描到的对象标记为可达，也就是可存活的对象

(2)压缩阶段：将可存活的对象移动到内存的一端，按顺序排放，然后清理掉剩余的空间

总结：

标记-压缩算法整理后的空间是连续分配的，有大段的连续内存可以分配，没有内存碎片。它比较适用于存活对象比较多的场景。

**有人可能会问标记-压缩算法也是需要将存活移动到一端的，存活对象比较多的话，是不是移动的成本比较大呢？**

是，但是相对复制算法来说，它不需要牺牲掉一片内存空间，并且这种情况下复制算法同样面临大量的复制开销。其实存活对象比较多的话，也不需要进行频繁的GC，所以在存活对象比较多的场景下，标记-压缩算法更有优势。



#### 6、分代堆内存GC策略

在JAVA7以前，将heap内存空间分为三个不同类别: 年轻代、老年代、持久代；**在JAVA8以后**，由于方法区的内存不在分配在Java堆上，而是存储于本地内存元空间Metaspace中，所以**永久代就不存在了**。JAVA8以后堆内存分代如下图：

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230812112725132.png" alt="image-20230812112725132" style="zoom:80%;" />



heap堆内存分为年轻代(Young Generation)和老年代(Old Generation)。

其中年轻代又将空间分为eden区和survivor区。其中eden区用来保存程序运行时新创建的对象，而survivor区又被大小相等的俩个区，称作survivor0(s0)、survivor1(s1),也可以称作from区、to区。由于年轻代中垃圾的占比比较大，所以在年轻代中采用的时复制算法。

在老年代中由于存活对象占比比较大，所以采用的标准-压缩算法。

**下面来详细说说，在分代堆内存GC策略中，垃圾时如何回收的：**

(1)程序运行时，新创建的对象会保存在eden区中，如果eden区满了就会触发第一次Minor GC，得到的结果是：将eden区中存活的对象复制到S0中，并且这些存活对象的年龄都会加1，此时S1是空闲的，并清空eden中所有的垃圾。

PS：此时S0可以被看做from区，S1可以被看做to区。后面下一次Minor GC角色会互换，总之，哪个区是空闲的，那个区就是to区。

(2)如果某时刻eden区又满了，那么就会再一次触发Minor GC，这次Minor GC会将eden区以及S0中的存活对象都复制到S1中，并且清除eden区以及S0中的所有垃圾，此时S0就会变为空闲的空间。另外，所有复制到S0中的存活对象年龄都会加1。

(3)如此反复这个过程，eden中复制到survivor区中的存活对象，会在S0和S1反复移动，当然这个过程中会有一些对象被当作垃圾清除掉。如果有对象的年龄达到了15(默认)，那么说明这些对象的生命周期比较长，就把它们复制到老年代中。

(4)如果老年代满了，则会进行一次Full GC。由于老年代对象也可以引用新生代对象，所以会先执行一次Minor GC，然后再执行标记-压缩算法的Old GC。另外也可以通过手动调用system.gc()来进行Full GC。



1、如果eden中对象满了，S0有存活的对象，S1空闲，触发Minor GC，但是S1不足以保存eden、S0中的存活对象怎么办？

猜测a：复制之前先做一个计算，如果eden、S0中的存活对象占用空间大小大于S1空间大小，直接直接将所有的存活对象放进old中。



猜测b：将一部分的存活对象放入S1中，如果S1装不下了，剩余的存活对象放入old中。

如果是这种，eden中的对象和S0中的对象有没有优先级，比如年龄大的先放进去？S1满了，此时里面的存活对象怎么处理呢，再触发一次GC吗？

2、full gc



1、为什么eden中占比比较大？

ava中的对象大多数都是朝生夕灭的，是因为Java程序中很多对象都是用来执行一些临时的任务或者存储一些短暂的数据的，比如局部变量、字符串拼接、集合操作等。这些对象在完成它们的功能后，就会失去引用而成为垃圾，等待垃圾回收器回收它们占用的内存空间。根据一些统计数据，新生代中的对象98%是朝生夕死的



#### 7、STW

对于大多数垃圾回收算法而言，GC线程工作时，会停止所有工作的线程，称为Stop The World。GC 完成 时，恢复其他工作线程运行。这也是JVM运行中最头疼的问题。

Minor GC 可能会引起短暂的STW暂停。当进行 Minor GC 时，为了确保安全性，JVM 需要在某些特定 的点上暂停所有应用程序的线程，以便更新一些关键的数据结构。这些暂停通常是非常短暂的，通常在 毫秒级别，并且很少对应用程序的性能产生显著影响。 Major GC的暂停时间通常会比Minor GC的暂停时间更长，因为老年代的容量通常比年轻代大得多。这 意味着在收集和整理大量内存时，需要更多的时间来完成垃圾收集操作。 尽管Major GC会引起较长的STW暂停，但JVM通常会尽量优化垃圾收集器的性能，以减少这些暂停对应 用程序的影响。例如，通过使用并行或并发垃圾收集算法，可以减少STW时间，并允许一部分垃圾收集 工作与应用程序的线程并发执行。

#### 8、JVM内存常用参数

![image-20230812171327980](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230812171327980.png)

- -选项名称 此为标准选项,所有HotSpot都支持 

- -X选项名称 此为稳定的非标准选项 

- -XX:选项名称 非标准的不稳定选项，下一个版本可能会取消

  **PS：参数大小写敏感**

| 参数              | 说明                                                         | 举例                                     |
| :---------------- | ------------------------------------------------------------ | ---------------------------------------- |
| -Xms              | 设置应用程序初始使用的堆内存大小（年轻代 +老年代）           | -Xms2g                                   |
| -Xmx              | 设置应用程序能获得的最大堆内存 早期JVM不建议超过32G，内存管理效率下降 | -Xmx2g                                   |
| -XX:NewSize       | 设置初始新生代大小                                           | -XX:NewSize=128m                         |
| -XX:MaxNewSize    | 设置新生代最大内存空间                                       | -XX:MaxNewSize=256m                      |
| -Xmn              | 同时设置-XX:NewSize 和 -XX:MaxNewSize，代替两者              | -Xmn1g                                   |
| -XX:NewRatio      | 以比例方式设置新生代和老年代                                 | -XX:NewRatio=2 即：new:old=1:2           |
| -XX:SurvivorRatio | 以比例方式设置eden和survivor(S0或S1)                         | -XX:SurvivorRatio=6 即：Eden:S0:S1=6:1:1 |
| -Xss              | 设置每个线程私有的栈空间大小,依据具体线程大小和数量          | -Xss256k                                 |

(1)生产环境下-Xms的值和-Xmx的值如何设置？

一般来说，生产环境下-Xms和-Xmx应该配置相同的大小，这样就意味着堆内存的大小一直会是固定的，不会随着程序的运行而动态调整。如果-Xms和-Xmx配置的不同的大小，那就会进行堆内存的扩展和收缩。而当堆内存使用情况变化时，并不是单纯的扩大和缩小堆内存就完事了，在此之前还会执行GC（垃圾回收）操作。如果-Xms起初值设置的比较小，那么就频繁触发GC操作,当GC操作无法释放更多内存时，才会进行内存的扩充。而执行GC操作时会造成应用的停顿，增加延迟，为了避免这种情况，所以生产环境下会将-Xms和-Xmx设置为相同的值。

**总结一下：**

如果程序需要比较好的性能，为了避免由于heap内存扩大或缩小导致的应用停顿，以及达到降低延迟的效果，需要将-Xms和-Xmx设置为相同的值。这其实是一种用空间来换时间的做法，直接将堆内存设到最大值，这样GC次数就会减少，但这样可能会造成空间的浪费。

如果程序只需要较少的空间，可以考虑将-Xms和-Xmx设置为不同的值，即使初始的堆内存不够也可以靠JVM来自动调整堆内存的大小，但是这样会造成更多的GC，性能会较低。

而且生产环境往往一台服务器或一个容器只有一个服务，独占服务器意味着没有必要调整JVM大小，每次调整反而会加大开销。

(2)其它参数调整需要更多得考虑实际的情况，根据不同的数值来调试，尽量保证GC次数越少越好。

#### 9、吞吐量

吞吐量是指应用程序线程用时占程序总用时的比例。例如，吞吐量99/100意味着100秒的程序执行时间应用程序线程运行了99秒，而在这一时间段内GC线程只运行了1秒。堆内内存越大，吞吐量越大，是因为GC线程占用的时间相对减少了，应用程序线程可以更多地使用CPU资源来处理业务逻辑。

10、垃圾回收器



#### 11、tomcat的JVM参数设置

在bin/catalina.sh中增加一行

```bash
......
# OS specific support. $var _must_ be set to either true or false.
#添加下面一行
JAVA_OPTS="-server -Xms128m -Xmx512m -XX:NewSize=48m -XX:MaxNewSize=200m"
                                                            
cygwin=false
darwin=false
........

重启tomcat
```

12、JVM性能监控工具jps、jstack、jstat、jmap、jinfo

在学习这些工具之前之前，需要知道一个点：

<font color='#ff6600'> 每个Java应用程序都会启动一个JVM进程，也可以说每一个java程序都会独占一个java虚拟机实例。每个JVM进程都会分配一个堆内存，不同的JVM进程之间是隔离的，它们不能直接访问彼此的堆内存。所以查看堆内存的分配策略是以一个java进程为单位的，使用一些java工具指定java进程的PID来查看。</font>

(1)jps

查看所有的JVM进程，常见的用法有：

```bash
#显示所有JVM进程号和短的类名称 
jps
#显示传递给Java虚拟机的参数
jps -v
```

![image-20230813200504180](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230813200504180.png)

jps实现的机制：

java程序启动后，会在目录 **`/tmp/hsperfdata_{userName}/`** 下生成几个文件，文件名就是java进程的 **`pid`** ，因此jps列出进程id就是把这个目录下的文件名列一下而已，至于系统参数，则是读取文件中的内容。

所以，如果在某个用户下执行jps查进程，需要这个用户对**`/tmp/hsperfdata_{userName}/`** 有查看权限才行。但是如果进程如果以普通用户身份运行，可能不管是root还是普通用户通过jps都查不到。如果查不到用其它命令，如ps等。

(2)jstack

jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码，所以它在JVM性能调优中使用得非常多。

https://cloud.tencent.com/developer/article/1116181

(3)jstat

jstat -gc pid



(4)

(5)