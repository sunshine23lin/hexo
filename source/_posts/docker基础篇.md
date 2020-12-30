---
title: docker基础篇
date: 2020-12-22 09:13:29
categories: docker
tags: 运维与虚拟化技术
---

##  Docker简介

###  什么是Docker？

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的[镜像](https://baike.baidu.com/item/镜像/1574)中，然后发布到任何流行的 [Linux](https://baike.baidu.com/item/Linux)或[Windows](https://baike.baidu.com/item/Windows/165458) 机器上，也可以实现[虚拟化](https://baike.baidu.com/item/虚拟化/547949)。容器是完全使用[沙箱](https://baike.baidu.com/item/沙箱/393318)机制，相互之间不会有任何接口。

###  为什么用Docker?

作为一种新兴的虚拟化方法，Docker跟传统的虚拟化方式相比具有众多的优势。

- **更高效的利用系统资源**

  由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更效。因此，相比虚拟机技术，一个相同配置的主机，往往可以运行更多数量的应用。

- **更快速的启动时间**

  传统的虚拟机技术启动应用服务往往需要数分钟，而 Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。

- **一致的运行环境**

  开发过程中一个常见的问题是环境一致性问题。由于开发环境、测试环境、生产环境不一致，导致有些 bug 并未在开发过程中被发现。而 Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性，从而不会再出现 「这段代码在我机器上没问题啊」 这类问题。

- **持续交付和部署**

  对于开发和运维(DevOps)人员来说,最希望的就是一次创建或者配置，可以在任意地方正常运行。使用Docker可以通过定制应用镜像来实现持续继承、持续交付、部署。开发人员可以通过Dockerfile进行镜像构建，并结合持续集成系统进行集成测试。而运维人员可以直接在生产中快速部署该镜像，甚至结合持续部署系统进行自动部署。而使用`而使用Dockerfile`使镜像构建透明化，不仅仅开发团队可以理解应用运行环境，也方便运维团队理解应用运行所需条件，帮助更好的生产环境中部署该镜像。

- **更轻松的迁移**

  由于Docker确保了执行环境的一致性，使得应用的迁移更加容易。Docker可以在很多平台上运行，无论是物理机、虚拟机、公有云、私有云，甚至是笔记本，其运行结构是一致的。

- **更轻松的维护和扩展**

  Docker 使用的分层存储以及镜像的技术，使得应用重复部分的复用更为容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单。此外，Docker 团队同各个开源项目团队一起维护了一大批高质量的 官方镜像，既可以直接在生产环境使用，又可以作为基础进一步定制，大大的降低了应用服务的镜像制作成本。

  ![image-20201222093759205](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222093759205.png)

##   基本概念

docker包括三个基本概念

- **镜像（Image）**
- **容器（Container）**
- **仓库（Repository）**

###  Docker 镜像

我们都知道,操作系统分为内核和用户空间。对于Linux而言,内核启动后，会挂在`root`文件系统为其提供用户空间支持。而Docker镜像，就相当于一个`root`文件系统。比如官方镜像ubuntu:16.04就包含了完整的一套ubuntu16.04最小系统的`root`文件系统。Docker镜像是一个特殊文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外、还包含了一些为运行时准备的一些参数(如匿名卷、环境变量、用户等)。**镜像不包含任何动态数据。其内容在构建之后也不会被改变。**

**分层存储**

因为镜像包含操作系统完整的  root  文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 Union FS 的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是**由一组文件系统组成**，或者说，**由多层文件系统联合组成。镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层**。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，**在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉**。分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。

###  Docker 容器

镜像和容器的关系，就像是面向对象程序设计中类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、删除、暂停等。

容器实质是进程，但与直接在宿主执行的进程不同，容器进程运行属于自己的独立的命名空间。因此容器可以拥有自己`root`文件系统、自己的网络配置、自己进程空间，甚至自己的用户ID空间。容器内的进程是在运行一个隔离的环境中，使用起来好像是在一个独立于宿主的系统下的操作系统一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。

前面讲过镜像使用的是分层存储，容器也是如此。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。**容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。**

按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 **数据卷（Volume）**、**或者绑定宿主目录**，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

###  Docker 仓库

镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。一个 Docker Registry 中可以包含多个仓库（ Repository  ）；每个仓库可以包含多个标签（ Tag  ）；每个标签对应一个镜像。

通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过  <仓库名>:<标签>  的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以  latest  作为默认标签。

最常使用的 Registry 公开服务是官方的 **Docker Hub**，这也是默认的 Registry，并拥有大量的
高质量的官方镜像。国内也有一些云服务商提供类似于 Docker Hub 的公开服务。比如 时速云镜像仓库、网易云
镜像服务、DaoCloud 镜像市场、阿里云镜像库 等。

##  安装Docker

操作系统选择CentOS 7.0以上

安装之前如果需要卸载的话，执行以下指令

```bash
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```



1. 按照Docker所需要的依赖

   ```bash
   yum install -y yum-utils \ device-mapper-persistent-data \ lvm2
   ```

   

2. 设置阿里云镜像库

   ```bash
   yum-config-manager  --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

3. 安装Docker

   ~~~bash
   yum -y install docker-ce # ce 为社区版本免费，docker-ee 企业版本
   ~~~

4. 设置镜像库加速

   ![image-20201222105019871](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222105019871.png)

   ![image-20201222105056444](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222105056444.png)

   安装步骤设置即可

   ```bash
   sudo mkdir -p /etc/docker
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://qhyb8ixp.mirror.aliyuncs.com"]
   }
   EOF
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

   

5. 开启Docker

   ~~~bash
   service docker start
   ~~~

6. 测试

   ~~~bash
   docker run hello-world
   ~~~

   ![image-20201222105550892](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222105550892.png)

   

##  使用镜像

Docker运行容器前需要本地存在对应的镜像，如果本地不在该镜像,Docker会从镜像仓库下载该镜像。

- 从仓库获取镜像
- 管理本地主机上的镜像
- 镜像实现的基本原理

###  获取镜像

从 Docker 镜像仓库获取镜像的命令是  `docker pull`  。其命令格式为：

> docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]

具体的选项可以通过  docker pull --help  命令看到，这里我们说一下镜像名称的格式。

- Docker 镜像仓库地址：地址的格式一般是  <域名/IP>[:端口号]  。默认地址是 Docker
  Hub。
- 仓库名：如之前所说，这里的仓库名是两段式名称，即  <用户名>/<软件名>  。对于 Docker
  Hub，如果不给出用户名，则默认为  library  ，也就是官方镜像。

`docker pull centos`

![image-20201222111042648](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222111042648.png)

###  列出容器

`docker image ls`

> REPOSITORY            TAG                    IMAGE ID                CREATED              SIZE
> centos                      latest              300e315adb2f          2 weeks ago        209MB
> hello-world            latest               bf756fb1ae65           11 months ago    13.3kB

列表包含了  仓库名  、 标签  、 镜像 ID  、 创建时间  以及  所占用的空间  。

可以通过`docker system df`来便捷的查看镜像、容器、数据卷所占用的空间

> TYPE                   TOTAL     ACTIVE    SIZE      RECLAIMABLE
> Images                 2               1         209.4MB    209.3MB (99%)
> Containers          2               0               0B             0B
> Local Volumes   0                0               0B             0B
> Build Cache        0                0               0B             0B

###  删除镜像

如果要删除本地的镜像，可以使用  `docker image rm`  命令，其格式为：

> docker image rm [选项] <镜像1> [<镜像2> ...]

###  Commit命令

docker commit命令除了学习外，还有一些特殊的应用场景，比如被入侵后保存现场等，但不要使用该命令指定镜像，指定镜像应该使用**Dockerfile** 完成

> 使用  docker commit  意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。而且，即使是这个制作镜像的人，过一段时间后也无法记清具体在操作的。虽然  docker diff  或许可以告诉得到一些线索，但是远远不到可以确保生成一致镜像的地步。这种黑箱镜像的维护工作是非常痛苦的。而且，回顾之前提及的镜像所使用的分层存储的概念，除当前层外，之前的每一层都是不会发生改变的，换句话说，任何修改的结果仅仅是在当前层进行标记、添加、修改，而不会改动上一层。如果使用  docker commit  制作镜像，以及后期修改的话，每一次修改都会让镜像更加臃肿一次，所删除的上一层的东西并不会丢失，会一直如影随形的跟着这个镜像，即使根本无法访问到。这会让镜像更加臃肿。

###  DockerFile定制镜像

DcokerFile是一个文本文件，其中包含了一条条的指令，每一条指令构建一层，因此每一条指令的你内容，就是描述该层应当如何构建。

DockerFile构建指令

> FROM                         # 基础镜像，一切从这里开始构建
>
> MAINTAINER             # 镜像是谁写的， 姓名+邮箱
>
> RUN                           # 镜像构建的时候需要运行的命令
>
> ADD                           # 步骤，tomcat镜像，这个tomcat压缩包！添加内容 添加同目录
>
> WORKDIR                  # 镜像的工作目录
>
> VOLUME                   # 挂载的目录
>
> EXPOSE                     # 保留端口配置
>
> CMD                          # 指定这个容器启动的时候要运行的命令，只有最后一个会生效，可被替代
>
> ENTRYPOINT            # 指定这个容器启动的时候要运行的命令，可以追加命令
>
> COPY                        # 类似ADD，将我们文件拷贝到镜像中
>
> ENV                          # 构建的时候设置环境变量！

####  定制centos镜像

1. 编写Dockerfile(文件名不是Dockerfile，构建是需要-f指定文件名)

   ```dockerfile
   FROM centos
   MAINTAINER MT<1172952007@qq.com>
   
   ENV MYPATH /usr/local
   WORKDIR $MYPATH
   
   RUN yum -y install vim
   
   EXPOSE 80
   
   CMD /bin/bash
   ```

2. 构建mycentos镜像

   ~~~bash
   docker build -f mycentos -t mycentosdemodo:1.0  .
   ~~~

3. 查看镜像历史

   ~~~bash
   docker history 镜像ID
   ~~~

#### 定制tomcat镜像

1. 编写Dockerfile(文件名不是Dockerfile，构建是需要-f指定文件名)

   ```dockerfile
   FROM centos
   MAINTAINER fortuneteller<1172952007@qq.com>
   
   COPY README.txt /usr/local/README.txt
   
   ADD jdk-8u271-linux-x64.tar.gz /usr/local
   ADD apache-tomcat-9.0.41.tar.gz /usr/local
   
   RUN yum -y install vim
   
   ENV MYPATH /usr/local
   WORKDIR $MYPATH
   
   ENV JAVA_HOME /usr/local/jdk1.8.0_271
   ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.41
   ENV CATALINA_BASH /usr/local/apache-toacat-9.0.41
   ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
   
   EXPOSE 8080
   
   CMD ["/usr/local/apache-tomcat-9.0.41/bin/catalina.sh", "run"]
   ```

2. 构建mytomcat镜像

   ~~~bash
   docker build  -f Dockerfile-tomcat -t mytomcat .
   ~~~

3. 启动tomcat

   ```bash
   docker run -d -p 3344:8080 --name mttomcat -v /data/docker-test/tomcat/test:/usr/local/apache-tomcat-9.0.41/webapps/test -v /data/docker-test/tomcat/test/logs:/usr/local/apache-tomcat-9.0.41/logs mytomcat
   ```

   ![image-20201222175017542](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201222175017542.png)

4. 进入容器

   ~~~bash
   #查看容器id
   docker container ls
   # 进入容器
   docker exec -it 容器ID /bin/bash
   ~~~



##  Docker 容器

###  启动容器

启动容器有两种方式，一种是基于镜像新建一个容器并启动,另外一个是将在终止状态（stopped）的容器重启启动

因为docker的容器实在太轻量级了，很多时候用户都是随时删除和新创建容器

- 新建容器并启动  `docker run`

  >  docker run -t -i mycentos:1.0 /bin/bash

- 存在容器启动 `docker container start`

**参数说明**

- -t让docker分配一个伪终端并绑定到容器的标准输入上
- -i 则让容器的标志输入保持打开

当利用`docker run` 来创建容器时，Docker在后台运行标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中
- 从地址池配置一个ip地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

###  终止容器

`docker container stop`

查看容器状态

`docker container ls -a`

###  进入容器

在使用  -d  参数时，容器启动后会进入后台。
某些时候需要进入容器进行操作，包括使用  `docker attach`  命令或  `docker exec`  命令，推荐大家使用  `docker exec`  命令

更多参数说明请使用  docker exec --help  查看。

###  删除容器

可以使用  `docker container rm`  来删除一个处于终止状态的容器。例如

`docker container prune`可以清理掉所有处于终止状态的容器。

##  Docker仓库

目前 Docker 官方维护了一个公共仓库 Docker Hub，其中已经包括了数量超过 15,000 的镜
像。大部分需求都可以通过在 Docker Hub 中直接下载镜像来实现。

###  注册

你可以在 https://cloud.docker.com 免费注册一个 Docker 账号。

###  登录

可以通过执行  `docker login`  命令交互式的输入用户名及密码来完成在命令行界面登录
Docker Hub。
你可以通过  `docker logout`  退出登录。

###  拉取镜像

你可以通过  `docker search`  命令来查找官方仓库中的镜像，并利用  **docker pull**  命令来将它
下载到本地。

###  推送镜像

用户也可以在登录后通过  **docker push**  命令来将自己的镜像推送到 Docker Hub。

> docker push username/ubuntu:17.10



##  Docker数据管理

Docker 内部以及容器之间管理数据，在容器中管理数据主要有两种方式

- **数据卷(Volumes)**
- **挂载主机目录（Bind mounts）**

容器之间可以有一个数据共享的技术！Docker容器中产生的数据，同步到本地！

这就是卷技术！目录的挂载，将我们容器内的目录，挂载到Linux上面！

### 使用命令

> docker run -it -v 主机目录:容器内目录 /bin/bash

###  挂载方式

> -v 容器内路径                      # 匿名挂载
>
> -v 卷名:容器内路径             # 具名挂载 
>
> -v 宿主机路径:容器内路径  # 指定路径挂载

Docker容器内的卷、在没有指定目录情况下都在`/var/lib/docker/volumes/xxx/_data`下

###  扩展

>\# 通过 -v 容器内路径：ro rw 改变读写权限
>
>ro # readonly 只读
>
>rw # readwrite 可读可写
>
>docker run -d nginx01 -v nginxdemo:/etc/nginx:ro nginx



##  网络配置

###  实现原理

Docker使用linux桥接，在宿主机虚拟一个Docker容器网桥(docker0),Docker启动一个容器时会根据Docker网桥的网段分配给容器一个IP地址，称为Container-IP,同时**Docker网桥是每个容器默认网关**。因为在同一个宿主机的容器都接入同一个网桥，**这样容器之间就能够通过容器的Container直接通讯。**

Docker网桥是宿主机虚拟出来的，并不是真实存在的网络设备，外部网络是无法寻址到的，这意味着外部网络无法通过直接Container-IP访问到容器，可以通过映射容器到宿主机(端口映射)，即docker run创建容器时候通过 -p 或 -P 参数来启用，访问容器的时候就通过`[宿主机IP]:[容器端口]`访问容器。

###  网络模式

![image-20201223092909412](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201223092909412.png)

####  bridge模式(默认)

当Docker进程启动时，会在主机上创建一个名为docker0的虚拟网桥，此主机上启动的docker容器会连接到这个虚拟网桥上。从docker0子网络中分配一个IP给容器使用，并设置docker0的IP地址为容器的默认网关,在主机上创建一对虚拟网卡veth pair设备,docker将veth pair设备的一端放入新的创建的容器中，并命名eth0(容器的网卡)，另一端放在主机中。以vethxxx这样类似的名字命名,并将这个网络设备加入到doker0网桥中，可以通过brctl show命令查看

birdge模式是docker的默认网络模式，不写--net参数，就是bridge模式。使用docker run -p 时,docker实际就是做iptables做了DNAT规则，实现端口转发功能。可以使用iptables -t nat -vnl查看

![image-20201223093913511](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201223093913511.png)

####  host模式

如果启动容器 的时候使用host模式，那么这个容器将不会获得一个独立的Network Namespace,而是和宿主机共用一个Network Namespace 。容器将不会虚拟出自己的网卡，配置自己的ip等，而是使用宿主机ip和端口。但是容器的其他方面，如文件系统，进程列表等还是个宿主机隔离。

使用host模式的容器可以直接使用宿主机的ip地址与外界通信,容器内部的服务端口也可以使用宿主机的端口，不需要进行nat,host最大优势是网络性能比较好，但是docker host上已经使用的端口就不能再用了网络隔离性不好。

![image-20201223094523624](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201223094523624.png)

 ####  container 模式

这个模式指定新创建的容器和已经存在一个容器共享一个Network Namespace,而不是和宿主机共享。新创建的容器不会创建自己的网卡，配自己IP。而是和一个指定的容器共享IP、端口等。同样2个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。2个容器进程可以通过lo网卡设备通信。

![image-20201223095659128](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201223095659128.png)

####  none模式

使用none模式，docker容器拥有自己的Network Namespace,但是，并不为Docker容器进行任何网络配置，也就是说，这个容器没有网卡、IP、路由等信息。所以需要我们自己为docker容器添加网卡、配置IP等。

这种网络模式下容器只有lo回环网络，没有其他网卡。node模式可以在容器创建时通过--network=none来指定。这类型网络没有办法联网，封闭的网络能很好保证容器安全性。

![image-20201223103642080](https://jameslin23.gitee.io/2020/12/22/docker基础篇/image-20201223103642080.png)