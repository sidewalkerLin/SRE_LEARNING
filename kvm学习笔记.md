### 一、

#### 1、查看当前环境

(1)内核配置文件

一般来说，内核配置文件的命名规则是：config-内核版本号-内核类型。所以config-5.15.0-76-generic和config-5.15.0-78-generic是内核俩个不通版本的配置文件，正常情况下只有1个配置文件，多个的原因可能是安装了多个不同版本或类型的内核。可以使用uname -r查看你操作系统目前使用的哪个版本的内核。

内核配置文件记录了内核编译时的选项和参数，我们可以通过查看此文件来了解操作系统支持哪些功能和模块。

在内核文件中记录了很多内核参数：

- 如果内核参数等于y，表示该参数对应的功能或模块是直接编译到内核中的，也就是说它是内核的一部分，不能被卸载或加载。
- 如果内核参数等于m，该参数对应的功能或模块是以模块的形式编译的，也就是说它是一个单独的文件，可以被动态地加载或卸载到内核中。
- 如果内核参数等于n，表示该参数对应的功能或模块是没有被编译到内核中的，也就是说它是被禁用的，您不能使用它。如果您想使用它，您需要重新编译内核，并将该参数设置为y或m。

```bash
ls /boot

#查看当前内核版本
uname -r
```

![image-20230821222803933](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230821222803933.png)

(2)查看关于kvm的功能是否开启

通过CONFIG_KVM_INTEL=m，CONFIG_KVM_AMD=m可以知道Intel和AMD的KVM功能是以模块的形式加载的，如果想用此功能需要将加载到内核中。

```bash
grep -i kvm /boot/config-5.15.0-78-generic
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230821222939568.png" alt="image-20230821222939568" style="zoom:80%;" />

(3)查看kvm相关的库文件

```bash
#modinfo kvm是用来显示kvm模块的信息，包括模块的名称，版本，作者，描述，参数，依赖等
modinfo kvm
```

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230821223019457.png" alt="image-20230821223019457" style="zoom:80%;" />

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230821223108474.png" alt="image-20230821223108474" style="zoom:80%;" />

(4)查看当前加载的kvm相关的模块

```bash
lsmod | grep -i kvm
```

![image-20230822112740933](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822112740933.png)

(5)验证是否开启虚拟化支持

```bash
#Intel CPU 对应 vmx
#AMD CPU 对应 svm 
grep -Em 1  "vmx|svm" /proc/cpuinfo
```

![image-20230822112935343](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822112935343.png)

(6)生产环境下

生产环境下一般默认支持开启了虚拟化支持，lsmod | grep -i kvm可以查看BIOS是否开启虚拟化。

参考：https://cloud.tencent.com/developer/article/1834814

vmware做实验需要勾选虚拟化引擎。

### 二、Ubuntu创建虚拟主机

#### 1、安装kvm相关包

(1)安装cpu-checker

```bash
#支持Ubuntu
apt -y install cpu-checker

#可以检查验证是否支持kvm
root@Ubuntu22:/boot# kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

(2)安装kvm工具

```bash
apt update && apt -y install qemu-kvm virt-manager libvirt-daemon-system
```

#### 2、virt-manager创建虚拟主机

(1)新建一个目录来存放操作系统镜像并启动virt-manager

```bash
#新建一个目录来存放操作系统镜像，将操作系统镜像放在此目录下
mkidr -pv /data/isos

virt-manager
```

(2)file===>New Virtual Machine===>选择Local install .......

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822172400312.png" alt="image-20230822172400312" style="zoom:80%;" />

(3)点击Browse选择iso镜像路径，默认只有default的/var/lib/libvirt/images的路径。点击左下角➕号，添加iso路径/data/isos并命名，我这里命名为了isos。选中路径选择镜像后，点击choose volume。

![image-20230822174222799](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822174222799.png)

(4)这里自动识别到了操作系统，点击forward即可。如果未识别到操作系统，需要取消勾选自动识别，并根据关键字搜索正确的操作系统填进去。

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822174454973.png" alt="image-20230822174454973" style="zoom:80%;" />

(5)分配cpu和内存

<img src="C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20230822174658990.png" alt="image-20230822174658990" style="zoom:80%;" />

(6)后面的不多说，就下一步下一步。。

3、使用已生成的
