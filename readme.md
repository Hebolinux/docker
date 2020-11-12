Docker基础入门
=============
虚拟机不等于容器，Docker也不等于容器，Docker只是使用容器的一种工具

虚拟化
------
提一提虚拟机与容器的区别

#### 一、主机级虚拟化
主机级虚拟化分为两种类型：
  - Ⅰ型虚拟化
  	Ⅰ型虚拟化又称为裸金属型虚拟化，它将传统的操作系统与硬件分离解耦，转而直接在硬件之上安装了一个虚拟化平台，也被称为hypervisor、VMM等，在这个虚拟化平台上再安装多个虚拟机以达到最大限度的发挥硬件的性能。esxi就是Ⅰ型虚拟化
  - Ⅱ型虚拟化
  	Ⅱ型虚拟化则是在宿主机上安装虚拟化的软件，实现安装虚拟机。比较常见的有virtualbox和vmware workstation
以上两种主机级虚拟化类型不管是哪一种，在想要做任何操作或部署服务之前都需要先进行一套完整的系统安装流程，而当需要部署的服务很少，甚至只有1个或2个服务时，虚拟机就显得无比臃肿

#### 二、容器级虚拟化
相比较虚拟机，容器则属于轻量级的，使用容器时不再需先安装操作系统，因为容器共享了底层的操作系统，使用容器时只需要操作系统划分出一个namespaces，服务则运行在此namespaces中。相比较虚拟机的安装和启动，一个容器的启动是非常快的，但容器的隔离性不如虚拟机
虚拟机和容器的对比：
```shell
虚拟机
--------------------------
虚拟硬件         namespaces
--------------------------
          VMM
--------------------------
       宿主操作系统
```
容器虚拟化的实现，首要的问题就是容器的隔离，为什么说容器的隔离性不如虚拟机，因为整个虚拟机从头到尾都是虚拟的，而容器却共享底层的操作系统，如果需要在两个容器内安装同一个服务，那该服务的所属用户、端口该如何处理，这里就用到了Linux下的namespaces <br/>
容器所需要隔离的六种资源(也是六种namespaces)：
	- UTS：主机名域名
	- Mount：挂载文件系统
	- IPC：进程间通信通道，信息量、消息队列和共享存储
	- PID：init 进程ID
	- User：root 用户ID
	- Net：网络设备、TCP/IP栈、套接字接口 Socket
此六种资源都已经通过Linux内核的namespaces技术原生支持，User的隔离在Linux内核版本3.8之后才被支持，只有namespaces还不够 <br />

为了防止namespaces中进程出现因为内存泄露而不断吃掉宿主机的内存，导致宿主机崩溃，Linux内核还需要一个功能来限制每一个namespaces可调用的资源总量，那就是cGroups

### Docker
早期使用容器时需要手写代码调用内核创建内核，门槛非常高，而为了使容器技术更加易用，产生了一组解决方案LXC，通过LXC工具可以快速创建一个namespaces，而后通过一个脚本模板文件，此模板文件会自动实现安装过程。此安装过程就是，对应namespaces中的Linux发行版，指向其发行版所对应的软件仓库下载镜像所对应的软件包进行安装 <br />
LXC虽然简化了容器的使用，但其复杂程度依旧非常高，并且对于数据迁移和批量创建上依然没有很好的解决，早期的Docker作为LXC的增强版，后来使用自建引擎runC，Docker的安装过程不再是依靠模板文件，而是提前将所需要的软件包打包成了一个镜像，将镜像放在一个共有仓库上以供下载 <br />
Docker与LXC的理念上也有所不同，LXC创建一个namespaces，基本等同于vmware创建了一个虚拟机，在这个namespaces中运行程序。而Docker的理念是每一个容器中仅运行一个进程

Docker的基本用法
===============
Docker是C/S架构，Client和Server可以共存在同一台主机上，Docker守护程序监听三个Socket，分别是IPv4、IPv6和unix socket，默认只监听unix socket

#### 仓库
仓库也被称为registry，本地存储是不便于存放过多的镜像的，为了提供一个集中的存储、分发镜像的服务，Docker registry就是这样的服务，registry允许用户免费上传、下载公开的镜像，最常用的registry公开服务是官方的DockerHub。

#### 镜像
启动一个容器时首先会从本地搜索镜像，如果本地没有镜像，就回去registry下载后再启动容器，下载镜像时可使用两种协议，https和http，默认仅使用https协议，只有手动指定http时才会使用http协议下载镜像

#### 容器
容器的启动必须依赖**本地镜像**，容器的实质就是进程，运行在属于自己的独立的namespaces中，可以拥有自己的根文件系统、网络配置、进程空间，就像在使用虚拟机一样

#### 标签
除了registry，还有一个仓库repository，一个registry中可以存放多个repository，repository更多的表示软件的名字，比如nginx就是一个repository，但nginx也有不同的版本，这些不同的版本则用标签声明，比如nginx:1.15表示nginx仓库中的1.15版，下载镜像时如果不指定标签，默认下载最新版，也就是latest

#### 分层构建，联合挂载
Docker的镜像并不是传统的iso镜像，Docker的每一个镜像都分成了许多层，每一层都是只读层，使用镜像时就是将这些层挂载到了一起，从外面看起来就像是一个镜像，每个低一层都是高一层镜像的父镜像。镜像的所有层都是只读层，但容器必定会产生数据，所以在创建容器时，只读层全部挂载完毕后还会在最顶层额外挂载一个读写层，这个读写层随着容器的消亡而消亡 <br />
从dockerhub下载到本地的镜像占用的存储可能会比dockerhub上显示的要小一些，因为dockerhub上显示的存储大小是经过压缩过的，这样从dockerhub下载到本地时产生的流量会小一些。因为docker的镜像以分层的形式存在，那不同的镜像使用到相同的层时，再从dockerhub下载镜像，docker就不会再下载这些本地已有的层

### 安装docker
示例：此示例采用docker官方推荐的yum安装方式
```shell
1. 卸载旧版本docker
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

2. 安装yum工具，添加docker的yum源(阿里云)
$ sudo yum install -y yum-utils
$ sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

3. 安装docker
$ sudo yum install docker-ce docker-ce-cli containerd.io

4. 查看docker是否安装成功
$ sudo docker --version

5. 配置docker加速器
由于docker官方在国内的加速器不咋地，阿里云需要注册账号创建私人加速器，所以此步骤免了
```
[docker官网的安装手册入口](https://docs.docker.com/get-docker/)
[CentOS安装手册](https://docs.docker.com/engine/install/centos/)

#### docker镜像管理
docker镜像文件格式是：{registry name}:{port}/{namespaces}/{repository name}:{tag name}
默认不指定registry的情况下会使用dockerhub，也就是docker.io，{namespaces}一般指账户名，默认使用docker官方的library账户 <br />
示例：docker镜像管理命令
```shell
# docker search nginx    #查找nginx镜像
# docker pull busybox    #下载busybox镜像
# docker image ls    #查看已下载的镜像
```

#### dockers容器管理
容器的选项非常多，这里列出docker官网的命令手册：[docker命令手册](https://docs.docker.com/engine/reference/run/)
示例：常见的容器命令
```shell
# docker container ls    #查看当前正在运行的容器，-a选项查看所有容器
# docker container run --name box -it --rm busybox:latest    #使用busybox镜像运行一个容器，设置别名为box，-it进入交互模式，--rm退出容器时立马删除此容器

#在容器中启动httpd服务
/ # mkdir /var/www/html
/ # echo "This is a test" > /var/www/html/index.html
/ # httpd -f -h /var/www/html/

# docker inspect box    #查看容器的详细信息
# docker container start -a -i box    #启动容器，并进入容器
# docker container kill box    #强制终止容器
# docker container rm box    #删除容器，只有容器停止运行后才能删除
# docker container exec -it web /bin/sh    #绕过容器的边界进入已启动的容器内运行命令
# docker attach centos    #进入容器，与exec的区别是，exec是进入容器后开启一个新的终端，attach则是进入容器正在执行的终端
# docker logs web    #查看容器日志
```

#### docker容器网络
容器总共有三个网络：网桥、主机、null，默认使用网桥模式
```shell
# docker network ls    #查看docker网络
```