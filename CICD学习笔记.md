

## 一、Gitlab

### 1、Gitlab安装

Gitlab下载地址：

```bash
#官方下载地址
https://packages.gitlab.com/gitlab/gitlab-ce
#清华大学镜像下载地址
https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce
```

Gitlab配置要求：

```bash
#支撑500用户
4c4g
#支撑1000用户
8c8g
```

(1)下载gitlab

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu/pool/jammy/main/g/gitlab-ce/gitlab-ce_15.10.0-ce.0_amd64.deb
```

(2)安装gitlab

```
dpkg -i gitlab-ce_15.10.0-ce.0_amd64.deb
```

(3)修改配置文件

```bash
vim /etc/gitlab/gitlab.rb
.....
#替换为真实的域名或ip
external_url 'http://10.0.0.165'
.....
```

(4)重载配置文件

```bash
gitlab-ctl reconfigure
```

(5)查看初始密码

```bash
cat /etc/gitlab/initial_root_password
```

(6)修改密码

由于gitlab版本的不同，页面的布局可能不同，但是修改密码是在点击用户头像中出现的Edit profile中的。

![image-20231030222843974](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231030222843974.png)

![image-20231030223052691](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231030223052691.png)

(7)关闭匿名注册功能

![image-20231031105435922](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105435922.png)

![image-20231031105038971](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105038971.png)

![image-20231031105219981](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105219981.png)



![image-20231031105311858](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105311858.png)

(8)汉化

![image-20231031105617974](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105617974.png)

![image-20231031105724258](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031105724258.png)

### 2、Gitlab用户和组管理

#### 2.1、创建用户

![image-20231031164606409](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031164606409.png)

![image-20231031164635525](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031164635525.png)

#### 2.2、修改用户密码

![image-20231031165317641](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031165317641.png)

## 二、Git

```bash
git config --global user.name "linqixin"
git config --global user.email "..."

mkdir /project
git init
touch README.md
vim test.java
git add .
#git commit不用指定
git commit -m "提交消息"

#删除暂存区文件
git restore --staged <file>

#从暂存区恢复文件
git checkout -- <file>

#只删除了工作区文件
rm -f <file>

#同时删除了工作区、暂存区文件
git rm -f <file>

git reset --hard|soft|minxed HEAD~n 
git reset --hard|soft|minxed v1.0

#显示git提交历史
git log --pretty --oneline --graph --all

#查当前分支
git branch

#创建新分支不切换
git branch dev
#切换分支
git checkout dev
#创建并切换分支
git checkout -b dev


#要合并到哪个分支就要先切换到那个分支，然后使用git merge进行合并
git checkout master
git merge dev -m "为什么合并"

#给分支改名，先切换到对应分支再改
git branch dev
git branch -M devel

#给当前分支tag一个状态
git tag v1.0

#给远程仓库定义别名为“origin”，下次不需要再输仓库地址
git remote add origin https://gitee.com/.....
#查询远程仓库别名与仓库地址的映射
git remote -v 
```

## 三、Jenkins部署与基本配置

### 1、jenkins安装

jenkins二进制包下载地址：

```bash
#清华大学镜像源下载地址
https://mirrors.tuna.tsinghua.edu.cn/jenkins
#jenkins官网下载地址
https://get.jenkins.io/
```

jenkins系统要求：

```bash
#jenkins系统要求以及安装说明
https://www.jenkins.io/zh/doc/book/installing/
#java环境说明
https://www.jenkins.io/doc/book/platform-information/support-policy-java/
```

(1)安装JDK

```bash
apt update && apt -y install openjdk-11-jdk
```

(2)下载jenkins二进制安装包

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/jenkins/debian-stable/jenkins_2.361.3_all.deb
```

(3)安装jenkins

```bash
dpkg -i jenkins_2.361.3_all.deb
```

(4)登录jenkins并查找密码

![image-20231031193508905](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031193508905.png)

(5)图形化安装jenkins

![image-20231031201623315](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031201623315.png)

![image-20231031201645613](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031201645613.png)

![image-20231031201720344](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031201720344.png)

(6)修改管理员密码

![image-20231031202111806](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031202111806.png)

### 2、jenkins数据目录

jenkins的数据目录根据安装方式的不同，存放路径可能会有差异

![image-20231031225736716](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031225736716.png)

![image-20231031225758568](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031225758568.png)

### 3、jenkins插件管理

#### 3.1、jenkins汉化插件

安装好插件后要重启jenkins服务

![image-20231031225503109](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031225503109.png)

![image-20231031225545282](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031225545282.png)

#### 3.2、jenkins安装gitlab插件

![image-20231101143025597](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101143025597.png)

#### 3.3、jenkins安装ansible插件

如上搜索ansible安装即可

#### 3.4、jenkins安装DingTalk插件

配置钉钉通知用

### 4、jenkins优化配置

#### 4.1、ssh优化

jenkins在连接gitlab或后端服务器的时候会弹出交互式的确认连接，影响脚本的自动化

![image-20231101220803779](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101220803779.png)

方法一：将StrictHostKeyChecking ask注释掉并改为no

```bash
vi /etc/ssh/ssh_config
....
#StrictHostKeyChecking ask 
StrictHostKeyChecking no
...
```

方法二：将key验证策略改为No verification

注意：这个配置必须要安装gitlab插件才有，并且只对连接gitlab时生效，对于jenkins连接后端服务器没有用

![image-20231101221845107](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101221845107.png)

方法三：打通基key验证，不仅不用连接确认也不用输入密码，jenkins可以和后端服务器打通key验证，gitlab则另有他法

```bash
#在~/.ssh/下生成ssh密钥对id_rsa、id_rsa.pub
ssh-keygen
#将公钥复制到远程机器上
ssh-copy-id target_ip
```



#### 4.2、性能优化

默认只能并行2个任务,需要根据CPU核心数,将执行者数量修改为CPU的核数

![image-20231031230404787](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031230404787.png)

![image-20231031230442994](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231031230442994.png)

#### 4.3、插件源优化

(1)修改default.json文件，更改jenkins的镜像源地址为国内镜像源

```bash
#注意：安装方式不同，可能数据目录的位置不一样
sed -i 's#www.google.com#www.baidu.com#g' /var/lib/jenkins/updates/default.json
sed -i.bak 's#updates.jenkins.io/download#mirror.tuna.tsinghua.edu.cn/jenkins#g' /var/lib/jenkins/updates/default.json
```

(2)修改升级站点URL

![image-20231101160216290](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101160216290.png)

![image-20231101160339250](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101160339250.png)

### 5、jenkins备份还原

jenkins进行备份，只需要对jenkins的数据目录定期进行备份即可，需要注意的是安装方式的不同jenkins的数据目录肯能存在差异，另外如果有相关脚本，也需要进行备份。

### 6、jenkins管理员密码找回

(1)修改配置文件并重启生效

```bash
systemctl stop jenkins.service
#将下图中卷起来的部分删除掉
vim /var/lib/jenkins/config.xml
systemctl start jenkins.service
```

![image-20231101201241171](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101201241171.png)

(2)登录jenkins页面

输入jenkins地址后发现无需输入用户密码，直接来到管理页面，且右上角无用户

![image-20231101202129516](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101202129516.png)

(3)将安全域修改为jenkins专有用户数据库

![image-20231101202521852](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101202521852.png)

(4)修改管理员密码

![image-20231101203632089](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101203632089.png)

![image-20231101203858012](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231101203858012.png)

## 四、Jenkins实现CICD

1、jenkins环境变量

jenkins环境变量有俩种：jenkins内置的环境变量、用户在jenkins配置中自定义的环境变量。在环境变量外，还有定义在脚本中的变量。当它们冲突的时候。生效的优先级为：**定义在脚本中的变量>用户在jenkins配置中自定义的环境变量>jenkins内置的环境变量**。

1.1、jenkins内置的环境变量



1.2、用户在jenkins配置中自定义的环境变量

![image-20231102175334033](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231102175334033.png)

2、jenkins凭据管理

![image-20231102215755761](C:\Users\26926\AppData\Roaming\Typora\typora-user-images\image-20231102215755761.png)





1、创建freestyle风格任务



开发人员将代码git push到gitlab上，然后jenkins通过git clone等命令将代码拉到jenkins上打个包，然后jenkins通过脚本推送到远程服务器上





测试

gtest





git clone

git banrh

bug  dev

bug分支最后都会master

远程分支合并分支





### 5、jenkins触发器

#### 5.1、触发远程构建(Build after other projects are built)

这种触发器允许通过HTTP请求来远程触发Jenkins构建。用户可以从一个脚本或其他服务中发送一个特定的URL请求，来启动构建过程。通常，这个URL会包含一个安全令牌以确保只有授权的服务或个人可以触发构建。

#### 5.2、其他工程构建后触发(Trigger builds remotely)

这个触发器允许用户配置一个项目，在其他项目构建完成后自动开始构建。这对于需要在其他项目更新后才能执行的项目非常有用，如部署项目通常在应用程序构建和测试完成后开始。

#### 5.3、定时构建(Build periodically)

与cron作业类似，这个触发器按照用户定义的时间表来触发构建。用户需要提供一个cron表达式来指定触发构建的时间。例如，可以配置Jenkins在每天深夜进行构建，或者在每周一的早晨进行构建。这种方式适用于需要定期执行的作业，比如夜间构建或定期的集成测试。

#### 5.4、SCM轮询(Poll SCM)

这个触发器会定期检查源代码管理（SCM）系统中的变化。如果发现代码有变更（例如，有新的提交或合并到指定的分支），Jenkins 会自动启动一个新的构建过程。用户可以配置轮询的时间间隔，比如每隔五分钟检查一次。这种方法的一个缺点是它不是实时的，并且会对SCM系统造成一定的负载。

#### 5.5、webhook触发器

webhook触发器不是jenkins内置的触发器，要创建此触发器需要先安装gitlab插件。

webhook以下使用场景：

替代SCM轮询：在gitlab上设置webhook，在代码变更后，立马通知jenkins进行构建，避免了SCM轮询不及时的情况；

定时的任务的即时触发：在设置定时任务后，如果不想等到固定的时间就触发构建，可以用过发送一个webhook请求到jenkins来构建；

链式构建：在复杂的流程中，一个项目的构建可能需要在另一个项目成功构建后立即开始。虽然可以使用"Build after other projects are built"触发器，但如果你想要更灵活的控制，或者要在不同的 Jenkins 实例或不同的CI/CD工具间协作，你可以在第一个项目构建成功后发送一个 Webhook 请求来触发下一个项目的构建；

集成第三方服务和工具：很多时候，除了源码变更，还可能需要基于其他第三方服务的事件来触发Jenkins构建。例如，部署工具、测试服务或项目管理工具也可以配置 Webhook，在特定事件发生时通知 Jenkins 进行构建或测试。









intel换到amd 













