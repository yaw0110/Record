# 搭建Hudi Docker Demo for Centos7

现在项目组那边要搞数据入湖，但只有一个正式环境，怕被我truncate数据了，不肯给服务器！因此搭建一个hudi用于自己研究学习。

## centos7虚拟机安装
   
在虚拟机搭建一台centos7虚拟机（自己研究一下吧，内存给8g，docker需要6g来运行hudi服务）

## 准备工作

演示通过主机名引用容器中运行的许多服务。将以下设置添加到`/etc/hosts`

```
127.0.0.1 adhoc-1
127.0.0.1 adhoc-2
127.0.0.1 namenode
127.0.0.1 datanode1
127.0.0.1 hiveserver
127.0.0.1 hivemetastore
127.0.0.1 kafkabroker
127.0.0.1 sparkmaster
127.0.0.1 zookeeper
```

## 安装所需程序

### java8

第一个是java8，其实centos7自带java8的，但只有一个jre，而部署hudi需要jdk，所以需要安装jdk。但使用yum安装jdk的话，在环境变量配置上面会有很大麻烦（使用`which java`找到java8在`bin\java`上，然后是软连接，就慢慢`cd`切换目录找），所以，还是少一点折腾，这里使用尚硅谷的java安装方式。

```shell
# 下载基本工具
yum install -y epel-release
yum install -y net-tools 
yum install -y vim

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld.service

# 删除自带的java
rpm -qa | grep -i java | xargs -n1 rpm -e --nodeps 
```

上传java安装包到虚拟机上面

```shell
tar -zxvf jdk-8u212-linux-x64.tar.gz -C /opt/software
```

打开`/etc/profile`文件，在最下面写入以下内容

```shell
vim /etc/profile
```

内容如下：

```shell
export MAVEN_HOME=/usr/local/apache-maven-3.8.1/
export PATH=${PATH}:${MAVEN_HOME}/bin
export JAVA_HOME=/opt/software/jdk1.8.0_212
export PATH=${PATH}:${JAVA_HOME}/bin
export SCALA_HOME=/opt/software/scala-2.11.12
export PATH=$PATH:$SCALA_HOME/bin
```

配置完成后，执行`source /etc/profile`命令，使配置生效。

运行`java -version`，终端打印如下信息：

```shell
[root@localhost ~]# java -version
java version "1.8.0_212"
Java(TM) SE Runtime Environment (build 1.8.0_212-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.212-b10, mixed mode)
```

### maven

第二个是maven，这里为什么要下载maven，因为hudi只能通过java源码编译的，所以需要maven进行源码构建，在这里可以直接命令行下载maven了，没有什么坑。

这里采用华为云的maven镜像，下载3.8.1版本，下载完成后，解压到`/usr/local/`目录下，并重命名为`apache-maven`。

```shell
wget https://repo.huaweicloud.com/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
tar -zxvf apache-maven-3.8.1-bin.tar.gz
mv apache-maven-3.8.1 /usr/local/
```

查看一下是否安装成功

```shell
mvn -verison
```

出现如下信息即可：

```shell
[root@localhost ~]# mvn -version
Apache Maven 3.8.1 (05c21c65bdfed0f71a2f2ada8b84da59348c4c5d)
Maven home: /usr/local/apache-maven-3.8.1
Java version: 1.8.0_212, vendor: Oracle Corporation, runtime: /opt/software/jdk1.8.0_212/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-1160.el7.x86_64", arch: "amd64", family: "unix"
```

配置一下镜像源和本地镜像仓库

```shell
vim /usr/local/apache-maven-3.8.1/conf/settings.xml 
```

添加如下内容：

```xml
<localRepository>/usr/local/apache-maven-3.8.1/repository</localRepository>
  <mirrors>
    <mirror>
      <id>aliyunCentralMaven</id>
      <name>aliyun central maven</name>
      <url>https://maven.aliyun.com/repository/central/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>centralMaven</id>
      <name>central maven</name>
      <url>http://mvnrepository.com/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>

    <mirror>
      <id>maven-default-http-blocker</id>
      <mirrorOf>external:http:*</mirrorOf>
      <name>Pseudo repository to mirror external repositories initially using HTTP.</name>
      <url>http://0.0.0.0/</url>
      <blocked>true</blocked>
    </mirror>
  </mirrors>
```

### scala

第三个是scala的安装，好奇？为什么要安装scala？因为hudi、spark的源码编译需要scala，所以需要安装scala。

这里下载2.11.12版本，下载完成后，解压到`/opt/software`目录下.

```shell
wget https://downloads.lightbend.com/scala/2.11.12/scala-2.11.12.tgz
tar -zxvf scala-2.11.12.tgz -C /opt/software
```

通过`scala -version`验证一下是否安装成功：

```shell
[root@localhost ~]# scala -version
Scala code runner version 2.11.12 -- Copyright 2002-2017, LAMP/EPFL
```

### git

git是因为hudi源码下载需要，这里直接使用yum安装即可。（如果网络不好，在Github下载zip文件，`unzip`解压也行）

```shell
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install gcc-c++ perl-ExtUtils-MakeMaker
yum install git
git --version
```

输出`git version 1.8.3.1`

配置信息：

```shell
# 创建一个github仓库
mkdir /opt/github
cd /opt/github/

# 初始化
git init

# 配置用户名和邮箱
git config user.name "用户名"
git config user.emial "邮箱"
```

## ssh远程连接

这里自己找ssh远程连接的文档，不多写了。
简单来，`systemctl start sshd`打开ssh服务，`ifconfig`获取ens33网卡ip4信息，使用xshell连接虚拟机，输入ip4地址和root用户名密码即可。

相关链接:

https://blog.csdn.net/weixin_38924500/article/details/106783645 (Centos7安装、配置SSH服务远程登录)

https://blog.csdn.net/qq_45101279/article/details/112799091 (ens33网卡配置静态ip)

## hudi

参考了这两篇：

https://hudi.apache.org/docs/0.14.0/docker_demo/ （官方文档）

https://blog.csdn.net/yang_shibiao/article/details/123077266 （centos7使用docker进行Hudi的快速体验和使用）

### 将Hudi存储库克隆到本地机器

GitHub地址：https://github.com/apache/hudi

这里我选用了0.14.0版本

```shell
git clone https://github.com/apache/hudi.git
```

网络不行，下载zip包，解压到`/opt/github`目录下。

```shell
[root@localhost hudi-release-0.14.0]# pwd
/opt/github/hudi-release-0.14.0
```

进入目录里面进行编译，使用官网的mvn命令可能有编译错误，有一些包报错。这里改了一下。

```shell
mvn package -DskipTests
```

这里很慢，第一次编译要一个小时多吧。出现`BUILD SUCCESS`即可。

### 安装docker和docker-compose

这里安装最新版的，旧版在pull镜像时可能出现`missing signature key`错误。

```shell
# 准备工作，更新docker镜像源
yum install -y yum-utils device-mapper-persistent-data lvm2
yum update -y
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装docker
yum -y remove docker*
yum install docker-ce docker-ce-cli containerd.io

# 安装docker-compose
cd /usr/local/bin
wget https://github.com/docker/compose/releases/download/v2.24.0-birthday.10/docker-compose-linux-x86_64
mv docker-compose-linux-x86_64 docker-compose
chmod +x docker-compose 
```

查看安装情况：

```shell
docker version
docker-compose version
```

重点！配置docker镜像源，不然拉取镜像很慢，天天出问题，超时、拒绝连接、没有镜像等等。

先配置阿里云的镜像，参考文章如下：
https://blog.csdn.net/angellee1988/article/details/105468058

```shell
vim /etc/docker/daemon.json
```

然后加上我这里的镜像资源：

```json
{
  "registry-mirrors": [
    "https://mpoqfnbe.mirror.aliyuncs.com",
    "https://docker.888666222.xyz",
    "https://atomhub.openatom.cn/",
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://docker.nju.edu.cn",
    "https://reg-mirror.qiniu.com",
    "https://docker.rainbond.cc"
  ]
}
```

更新配置：

```shell
systemctl daemon-reload
systemctl restart docker
```

可以使用`docker info`验证一下。

进入hudi工作目录下的docker目录，运行setup_demo.sh脚本

```shell
[root@localhost docker]# pwd
/opt/github/hudi-release-0.14.0/docker
[root@localhost docker]# ll
total 40
-rwxr-xr-x. 1 root root  1316 Sep 27  2023 build_local_docker_images.sh
drwxr-xr-x. 2 root root   142 Sep 27  2023 compose
drwxr-xr-x. 4 root root  4096 Sep 27  2023 demo
-rwxr-xr-x. 1 root root 11986 Sep 27  2023 generate_test_suite.sh
drwxr-xr-x. 3 root root    20 Sep 27  2023 hoodie
drwxr-xr-x. 2 root root    36 Sep 27  2023 images
-rw-r--r--. 1 root root  9685 Sep 27  2023 README.md
-rwxr-xr-x. 1 root root  1640 Sep 27  2023 setup_demo.sh
-rwxr-xr-x. 1 root root  1301 Sep 27  2023 stop_demo.sh
[root@localhost docker]# sh setup_demo.sh 
```

出现20个容器启动即可，这里也是很慢，慢慢等待~

```shell
[+] Running 20/20
 ✔ Network hudi                                Created                                                  0.1s 
 ✔ Volume "compose_historyserver"              Created                                                  0.0s 
 ✔ Volume "compose_hive-metastore-postgresql"  Created                                                  0.0s 
 ✔ Container namenode                          Started                                                  0.2s 
 ✔ Container hive-metastore-postgresql         Started                                                  0.2s 
 ✔ Container kafkabroker                       Started                                                  0.2s 
 ✔ Container zookeeper                         Started                                                  0.2s 
 ✔ Container graphite                          Started                                                  0.3s 
 ✔ Container historyserver                     Started                                                  0.1s 
 ✔ Container hivemetastore                     Started                                                  0.1s 
 ✔ Container datanode1                         Started                                                  0.1s 
 ✔ Container trino-coordinator-1               Started                                                  0.1s 
 ✔ Container presto-coordinator-1              Started                                                  0.1s 
 ✔ Container hiveserver                        Started                                                  0.1s 
 ✔ Container sparkmaster                       Started                                                  0.1s 
 ✔ Container trino-worker-1                    Started                                                  0.1s 
 ✔ Container presto-worker-1                   Started                                                  0.1s 
 ✔ Container adhoc-1                           Started                                                  0.2s 
 ✔ Container spark-worker-1                    Started                                                  0.2s 
 ✔ Container adhoc-2                           Started                                                  0.2s 
Copying spark default config and setting up configs
Copying spark default config and setting up configs
```

到这里，hudi demo is ready!