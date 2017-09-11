
# docker 

[official document](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)

[docker 实践](https://yeasy.gitbooks.io/docker_practice/content/basic_concept/)

# 基本概念

理解了 镜像（Image）/ 容器（Container） / 仓库（Repository） 这三个概念 ，就理解了 Docker 的整个生命周期。

## 镜像（Image）

 - 操作系统分为内核和用户空间
 - 对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持
 - Docker 镜像（Image），就相当于是一个 root 文件系统
    - 比如官方镜像 ubuntu:14.04 就包含了完整的一套 Ubuntu 14.04 最小系统的 root 文件系统

### 分层存储

 - 镜像包含操作系统完整的 root 文件系统，其体积往往是庞大的
 - 因此在 Docker 设计时，就充分利用 [Union FS](https://en.wikipedia.org/wiki/Union_mount) 的技术，将其设计为分层存储的架构。
 - 所以严格来说，镜像只是一个虚拟的概念，其实际 由多层文件系统联合组成

---

 - 镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。
    - 比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。
    - 在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。
 - 因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。
 - 分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。


## 容器（Container）

 - 镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体.
 - 容器的实质是进程
    - 但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间
    - 因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。
    - 容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。
 - 镜像使用的是分层存储，容器也是如此
    - 每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，
    - 我们可以称这个为容器运行时读写而准备的存储层为**容器存储层**
 - **容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡**
    - 因此，任何保存于容器存储层的信息都会随容器删除而丢失
 - **容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化** 
    - 所有的文件写入操作，都应该使用 [数据卷（Volume）](https://docs.docker.com/engine/tutorials/dockervolumes/), 或者绑定宿主目录, 在这些位置的读写会跳过容器存储层，直接对宿主(或网络存储)发生读写，其性能和稳定性更高。
 - 数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。
    - 因此，使用数据卷后，容器可以随意删除、重新 run，数据却不会丢失。



## 仓库（Repository）

### Docker Registry

 - 镜像构建完成后，可以很容易的在当前宿主上运行
 - 但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务
 - [Docker Registry](https://docs.docker.com/registry/) 就是这样的服务。

---

 - 一个 Docker Registry 中可以包含多个仓库（Repository）；
 - 每个仓库可以包含多个标签（Tag）
 - 每个标签对应一个镜像。

---

 - 通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。
    - 我们可以通过 <仓库名>:<标签> 的格式来指定具体是这个软件哪个版本的镜像。
    - 如果不给出标签，将以 latest 作为默认标签。
 - 以 Ubuntu 镜像 为例，ubuntu 是仓库的名字，其内包含有不同的版本标签，如，14.04, 16.04。
    - 我们可以通过 ubuntu:14.04，或者 ubuntu:16.04 来具体指定所需哪个版本的镜像。
    - 如果忽略了标签，比如 ubuntu，那将视为 ubuntu:latest
 - 仓库名经常以 两段式路径 形式出现，比如 jwilder/nginx-proxy
    - 前者往往意味着 Docker Registry 多用户环境下的用户名，后者则往往是对应的软件名
    - 但这并非绝对，取决于所使用的具体 Docker Registry 的软件或服务。

### Docker Registry 公开服务

 - Docker Registry 公开服务是开放给用户使用、允许用户管理镜像的 Registry 服务。
 - 最常使用的 Registry 公开服务是官方的 [Docker Hub](https://hub.docker.com/) , 这也是默认的 Registry，并拥有大量的高质量的官方镜像。
    - 除此以外，还有 [CoreOS](https://coreos.com/) 的 [Quay.io](https://quay.io/repository/) , CoreOS 相关的镜像存储在这里
    - Google 的 [Google Container Registry](https://cloud.google.com/container-registry/) ，[Kubernetes](http://kubernetes.io/) 的镜像使用的就是这个服务。
 - 由于某些原因，在国内访问这些服务可能会比较慢。国内的一些云服务商提供了针对 Docker Hub 的镜像服务（Registry Mirror），这些镜像服务被称为**加速器**。
    - 常见的有 [阿里云加速器](https://cr.console.aliyun.com/#/accelerator) 、[DaoCloud 加速器](https://www.daocloud.io/mirror#accelerator-doc) 、[灵雀云加速器](http://docs.alauda.cn/feature/accelerator.html) 等

### 私有 Docker Registry

 - 除了使用公开服务外，用户还可以在本地搭建私有 Docker Registry。
 - Docker 官方提供了 [Docker Registry 镜像](https://hub.docker.com/_/registry/)   ，可以直接使用做为私有 Registry 服务。
 - 开源的 Docker Registry 镜像只提供了 [Docker Registry API](https://docs.docker.com/registry/spec/api/) 的服务端实现，足以支持 docker 命令，不影响使用
    - 但不包含图形界面，以及镜像维护、用户管理、访问控制等高级功能。
    - 在官方的商业化版本 [Docker Trusted Registry](https://docs.docker.com/datacenter/dtr/2.0/) 中，提供了这些高级功能。
 
---

# 安装
 
 [安装](https://yeasy.gitbooks.io/docker_practice/content/install/centos.html)

 [镜像加速](https://docs.docker.com/registry/recipes/mirror/#use-case-the-china-registry-mirror)

 [set docker proxy](https://docs.docker.com/engine/admin/systemd/#httphttps-proxy)

---

# 使用 Docker 镜像

Docker 运行容器前需要本地存在对应的镜像，如果镜像不存在本地，Docker 会从镜像仓库下载 ( 默认是 Docker Hub 公共注册服务器中的仓库 )

## 获取镜像

```
docker pull [选项] [Docker Registry地址]<仓库名>:<标签>
```

 - Docker Registry地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub。
 - 仓库名：如之前所说，这里的仓库名是两段式名称，既 `<用户名>/<软件名>`
    - 对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。

## 运行

以 ubuntu:14.04 为例，如果我们打算启动里面的 bash 并且进行交互式操作的话，可以执行下面的命令。

```
$ docker run -it --rm ubuntu:14.04 bash
```

 - -it 
    - 这是两个参数，一个是 -i：交互式操作，一个是 -t 终端
 - -rm 
    - 容器退出后随之将其删除
 - bash：放在镜像名后的是**命令**
    - 这里我们希望有个交互式 Shell，因此用的是 bash

进入容器后，我们可以在 Shell 下操作，在容器中 执行任何所需的命令

```
root@e7009c6ce357:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.5 LTS, Trusty Tahr"
ID=ubuntu
VERSION_ID="14.04"
...

root@e7009c6ce357:/# exit
exit
$
```

## 列出镜像

```
$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
redis                latest              5f515359c7f8        5 days ago          183 MB
<none>               <none>              00285df0df87        5 days ago          342 MB
ubuntu               16.04               f753707788c5        4 weeks ago         127 MB
ubuntu               latest              f753707788c5        4 weeks ago         127 MB
ubuntu               14.04               1e0c3dd64ccd        4 weeks ago         188 MB
```

 - `IMAGE ID` 是镜像的唯一标识
 - 一个镜像可以对应多个标签 TAG
    - ubuntu:16.04 和 ubuntu:latest 拥有相同的 ID，因为它们对应的是同一个镜像

### 虚悬镜像

 - 上面的镜像列表中，还可以看到一个特殊的镜像，这个镜像既没有仓库名，也没有标签，均为 <none>
 - 这个镜像原本是有镜像名和标签的，原来为 mongo:3.2，随着官方镜像维护，发布了新版本后，重新 docker pull mongo:3.2 时，mongo:3.2 这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 <none>。
    - 除了 docker pull 可能导致这种情况，docker build 也同样可以导致这种现象
 - 由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 <none> 的镜像。这类无标签镜像也被称为 **虚悬镜像(dangling image)**  
 - 可以用下面的命令专门显示这类镜像：
    - `$ docker images -f dangling=true`
 - 一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。
    - `$ docker rmi $(docker images -q -f dangling=true)`

### 中间层镜像

 - 为了加速镜像构建、重复利用资源，Docker 会利用 **中间层镜像**
 - 所以在使用一段时间后，可能会看到一些依赖的中间层镜像
 - 默认的 docker images 列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 -a 参数。
    - `$ docker images -a`
 - 这样会看到很多无标签的镜像，与之前的虚悬镜像不同，这些无标签的镜像很多都是中间层镜像，是其它镜像所依赖的镜像。这些无标签镜像不应该删除，否则会导致上层镜像因为依赖丢失而出错. 实际上，这些镜像也没必要删除.

### 列出部分镜像

根据仓库名列出镜像

```
$ docker images ubuntu
```

列出特定的某个镜像，也就是说指定仓库名和标签

```
$ docker images ubuntu:16.04
```

除此以外，docker images 还支持强大的过滤器参数 --filter，或者简写 -f

比如，我们希望看到在 mongo:3.2 之后建立的镜像，可以用下面的命令：

```
$ docker images -f since=mongo:3.2
```

此外，如果镜像构建时，定义了 LABEL，还可以通过 LABEL 来过滤。

```
$ docker images -f label=com.example.version=0.1
```

### 以特定格式显示

 - 默认情况下，docker images 会输出一个完整的表格，但是我们并非所有时候都会需要这些内容

```
$ docker images -q
5f515359c7f8
05a60462f8ba
fe9198c04d62
00285df0df87
```

 - `-q` 的 结果，可以直接 配合 `docker rmi ` 之类的命令进行批量处理

下面的命令会直接列出镜像结果，并且只包含镜像ID和仓库名：

```
$ docker images --format "{{.ID}}: {{.Repository}}"
5f515359c7f8: redis
05a60462f8ba: nginx
fe9198c04d62: mongo
00285df0df87: <none>
```

## 利用 commit 理解镜像构成

 - 在之前的例子中，我们所使用的都是来自于 Docker Hub 的镜像
 - 直接使用这些镜像是可以满足一定的需求，而当这些镜像无法直接满足需求时，我们就需要定制这些镜像。

现在让我们以定制一个 Web 服务器为例子，来讲解镜像是如何构建的。

```
docker run --name webserver -d -p 80:80 nginx
```

这条命令会用 nginx 镜像启动一个容器，命名为 webserver，并且映射了 80 端口，这样我们可以用浏览器去访问这个 nginx 服务器。

如果是在 Linux 本机运行的 Docker，或者如果使用的是 Docker for Mac、Docker for Windows，那么可以直接访问：http://localhost；如果使用的是 Docker Toolbox，或者是在虚拟机、云服务器上安装的 Docker，则需要将 localhost 换为虚拟机地址或者实际云服务器地址。

直接用浏览器访问的话，我们会看到默认的 Nginx 欢迎页面。

现在，假设我们非常不喜欢这个欢迎页面，我们希望改成欢迎 Docker 的文字，我们可以使用 docker exec 命令进入容器，修改其内容。

```
$ docker exec -it webserver bash
root@3729b97e8226:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
root@3729b97e8226:/# exit
exit
```

现在我们再刷新浏览器的话，会发现内容被改变了。

我们修改了容器的文件，也就是改动了容器的存储层。我们可以通过 docker diff 命令看到具体的改动。

```
$ docker diff webserver
C /root
A /root/.bash_history
C /run
A /run/nginx.pid
C /usr/share/nginx/html/index.html
C /var/cache/nginx
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp
```

现在我们定制好了变化，我们希望能将其保存下来形成镜像。

 - 当我们运行一个容器的时候（如果不使用卷的话），我们做的任何文件修改都会被记录于容器存储层里。
 - 而 Docker 提供了一个 docker commit 命令，可以将容器的存储层保存下来成为镜像
    - 换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。
    - 以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。

docker commit 的语法格式为：

```
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```

我们可以用下面的命令将容器保存为镜像：

```
$ docker commit \
    --author "Tao Wang <twang2218@gmail.com>" \
    --message "修改了默认网页" \
    webserver \
    nginx:v2
sha256:07e33465974800ce65751acc279adc6ed2dc5ed4e0838f8b86f0c87aa1795214
```

可以用 docker history 具体查看镜像内的历史记录，如果比较 nginx:latest 的历史记录，我们会发现新增了我们刚刚提交的这一层。

```
$ docker history nginx:v2
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
07e334659748        54 seconds ago      nginx -g daemon off;                            95 B                修改了默认网页
e43d811ce2f4        4 weeks ago         /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon    0 B
<missing>           4 weeks ago         /bin/sh -c #(nop)  EXPOSE 443/tcp 80/tcp
..l
```

新的镜像定制好后，我们可以来运行这个镜像。

```
docker run --name web2 -d -p 81:80 nginx:v2
```

这里我们命名为新的服务为 web2，并且映射到 81 端口。如果是 Docker for Mac/Windows 或 Linux 桌面的话，我们就可以直接访问 http://localhost:81 看到结果，

### 慎用 docker commit

使用 docker commit 命令虽然可以比较直观的帮助理解镜像分层存储的概念，但是实际环境中并不会这样使用。

 - 首先，如果仔细观察之前的 docker diff webserver 的结果，你会发现除了真正想要修改的 /usr/share/nginx/html/index.html 文件外，由于命令的执行，还有很多文件被改动或添加了。
    - 这还仅仅是最简单的操作，如果是安装软件包、编译构建，那会有大量的无关内容被添加进来，如果不小心清理，将会导致镜像极为臃肿。
 - 此外，使用 docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为**黑箱镜像**. 
    - 换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。
    - 这种黑箱镜像的维护工作是非常痛苦的
 - 如果使用 docker commit 制作镜像，以及后期修改的话，每一次修改都会让镜像更加臃肿一次，所删除的上一层的东西并不会丢失，会一直如影随形的跟着这个镜像，即使根本无法访问到™。这会让镜像更加臃肿。

docker commit 命令除了学习之外，还有一些特殊的应用场合，比如被入侵后保存现场等。但是，不要使用 docker commit 定制镜像，定制行为应该使用 Dockerfile 来完成。

---

## 使用 Dockerfile 定制镜像

镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

还以之前定制 nginx 镜像为例，这次我们使用 Dockerfile 来定制。

在一个空白目录中，建立一个文本文件，并命名为 Dockerfile：

```
$ mkdir mynginx
$ cd mynginx
$ vi Dockerfile
```

其内容为：

```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

这个 Dockerfile 很简单，一共就两行。涉及到了两条指令，FROM 和 RUN。

### FROM 指定基础镜像

 - FROM 是必备的指令，并且必须是第一条指令。
 - 在 [Docker Hub](https://hub.docker.com/explore/) 上有非常多的高质量的官方镜像， 有可以直接拿来使用的服务类的镜像
    - 如 [nginx](https://hub.docker.com/_/nginx/) 、[redis](https://hub.docker.com/_/redis/) 、[mongo](https://hub.docker.com/_/mongo/) 、[mysql](https://hub.docker.com/_/mysql/)  、[httpd](https://hub.docker.com/_/httpd/) 、[php](https://hub.docker.com/_/php/) 、[tomcat](https://hub.docker.com/_/tomcat/) 等
    - 也有一些方便开发、构建、运行各种语言应用的镜像
    - 如 [node](https://hub.docker.com/_/node/) 、[openjdk](https://hub.docker.com/_/openjdk/) 、[python](https://hub.docker.com/_/python/) 、[ruby](https://hub.docker.com/_/ruby/) 、[golang](https://hub.docker.com/_/golang/) 等
 - 如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像
    - 如: ubuntu、debian、[centos](https://hub.docker.com/_/centos/) 、fedora、alpine 等

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

```
FROM scratch
...
```

 - 如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。
 - 不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm、coreos/etcd
 - 对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。
 - 使用 Go 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。

### RUN 执行命令

RUN 指令是用来执行命令行命令的。由于命令行的强大能力，RUN 指令在定制镜像时是最常用的指令之一。其格式有两种：

 - shell 格式：RUN <命令>，就像直接在命令行中输入的命令一样。
    - 刚才写的 Dockrfile 中的 RUN 指令就是这种格式。
 - exec 格式：RUN ["可执行文件", "参数1", "参数2"]，这更像是函数调用中的格式。

既然 RUN 就像 Shell 脚本一样可以执行命令，那么我们是否就可以像 Shell 脚本一样把每个命令对应一个 RUN 呢？比如这样：

```
FROM debian:jessie

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

之前说过，Dockerfile 中每一个指令都会建立一层，RUN 也不例外。

每一个 RUN 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。

而上面的这种写法，创建了 7 层镜像。这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。 这是很多初学 Docker 的人常犯的一个错误。

Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。

上面的 Dockerfile 正确的写法应该是这样：

```
FROM debian:jessie

RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注释等，会让维护、排障更为容易，这是一个比较好的习惯。

此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 apt 缓存文件。这是很重要的一步，我们之前说过，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

很多人初学 Docker 制作出了很臃肿的镜像的原因之一，就是忘记了每一层构建的最后一定要清理掉无关文件。

### 构建镜像

好了，让我们再回到之前定制的 nginx 镜像的 Dockerfile 来。让我们来构建这个镜像吧。

在 Dockerfile 文件所在目录执行：

```
$ docker build -t nginx:v3 .
```

输出: 

```
Step 1/2 : FROM nginx
 ---> 66216d141be6
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in fc71a215b873
 ---> 3bee9a6a80f2
Removing intermediate container fc71a215b873
Successfully built 3bee9a6a80f2
Successfully tagged nginx:v3
```

docker build 命令

```
docker build [选项] <上下文路径/URL/->
```

在这里我们指定了最终镜像的名称 -t nginx:v3，构建成功后，我们可以像之前运行 nginx:v2 那样来运行这个镜像，其结果会和 nginx:v2 一样。

### 镜像构建上下文（Context）

我们的 docker build 命令最后有一个 `.` 

不少初学者以为这个路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定上下文路径。那么什么是上下文呢？

 - 首先我们要理解 docker build 的工作原理。Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。
 - Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。
 - 因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。
 - 当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。
 - 而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。

那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

 - 这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

如果在 Dockerfile 中这么写：

```
COPY ./package.json /app/
```

这并不是要复制执行 docker build 命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json，而是复制 上下文（context） 目录下的 package.json。

因此，COPY 这类指令中的源文件的路径都是相对路径。这也是初学者经常会问的为什么 COPY ../package.json /app 或者 COPY /opt/xxxx /app 无法工作的原因，因为这些路径已经超出了上下文的范围，Docker 引擎无法获得这些位置的文件。如果真的需要那些文件，应该将它们复制到上下文目录中去。

现在就可以理解刚才的命令 docker build -t nginx:v3 . 中的这个 .，实际上是在指定上下文的目录，docker build 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。

如果观察 docker build 输出，我们其实已经看到了这个发送上下文的过程：

```
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
...
```

一般来说，应该会将 Dockerfile 置于一个空目录下，或者**项目**根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 .gitignore 一样的语法写一个 .dockerignore，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。

### 其它 docker build 的用法

#### 直接用 Git repo 进行构建

docker build 还支持从 URL 构建，比如可以直接从 Git repo 中构建：

```
$ docker build https://github.com/twang2218/gitlab-ce-zh.git#:8.14
docker build https://github.com/twang2218/gitlab-ce-zh.git\#:8.14
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM gitlab/gitlab-ce:8.14.0-ce.0
8.14.0-ce.0: Pulling from gitlab/gitlab-ce
aed15891ba52: Already exists
773ae8583d14: Already exists
...
```

这行命令指定了构建所需的 Git repo，并且指定默认的 master 分支，构建目录为 /8.14/，然后 Docker 就会自己去 git clone 这个项目、切换到指定分支、并进入到指定目录后开始构建。

#### 用给定的 tar 压缩包构建

```
$ docker build http://server/context.tar.gz
```

如果所给出的 URL 不是个 Git repo，而是个 tar 压缩包，那么 Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。


#### 从标准输入中读取 Dockerfile 进行构建

```
docker build - < Dockerfile
```

或

```
cat Dockerfile | docker build -
```

如果标准输入传入的是文本文件，则将其视为 Dockerfile，并开始构建。这种形式由于直接从标准输入中读取 Dockerfile 的内容，它没有上下文，因此不可以像其他方法那样可以将本地文件 COPY 进镜像之类的事情。


#### 从标准输入中读取上下文压缩包进行构建

```
$ docker build - < context.tar.gz
```

如果发现标准输入的文件格式是 gzip、bzip2 以及 xz 的话，将会使其为上下文压缩包，直接将其展开，将里面视为上下文，并开始构建。

## Dockerfile 指令详解

### COPY 复制文件

```
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```

和 RUN 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。比如：

```
COPY package.json /usr/src/app/
```

<源路径> 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 filepath.Match 规则，如：

```
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

<目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 WORKDIR 指令来指定）。

目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

此外，还需要注意一点，使用 COPY 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。


### ADD 更高级的复制文件

ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。

 - 这个功能其实并不实用，而且不推荐使用
 - 如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。
 - 在某些情况下，这个自动解压缩的功能非常有用，比如官方镜像 ubuntu 中：

```
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
...
```

 - 但在某些情况下，如果我们真的是希望复制个压缩文件进去，而不解压缩，这时就不可以使用 ADD 命令了。
 - 在 Docker 官方的最佳实践文档中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。
 - 最适合使用 ADD 的场合，就是所提及的需要自动解压缩的场合。

### CMD 容器启动命令

CMD 指令的格式和 RUN 相似，也是两种格式：

```
shell 格式：CMD <命令>
exec 格式：CMD ["可执行文件", "参数1", "参数2"...]
```

Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。

在运行时可以指定新的命令来替代镜像设置中的这个默认命令。

比如，ubuntu 镜像默认的 CMD 是 /bin/bash，如果我们直接 docker run -it ubuntu 的话，会直接进入 bash。我们也可以在运行时指定运行别的命令，如 `docker run -it ubuntu cat /etc/os-release`。这就是用 cat /etc/os-release 命令替换了默认的 /bin/bash 命令了，输出了系统版本信息。

在指令格式上，一般推荐使用 exec 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 "，而不要使用单引号。

如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：

```
CMD echo $HOME
```

在实际执行中，会将其变更为：

```
CMD [ "sh", "-c", "echo $HOME" ]
```

这就是为什么我们可以使用环境变量的原因，因为这些环境变量会被 shell 进行解析处理。

---

提到 CMD 就不得不提容器中应用在前台执行和后台执行的问题。这是初学者常出现的一个混淆。

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 upstart/systemd 去启动后台服务，

**容器内没有后台服务的概念**。

一些初学者将 CMD 写为：

```
CMD service nginx start
```

然后发现容器执行后就立即退出了。甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。

对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

而使用 service nginx start 命令，则是希望 upstart 来以后台守护进程形式启动 nginx 服务。而刚才说了 CMD service nginx start 会被理解为 CMD [ "sh", "-c", "service nginx start"]，因此主进程实际上是 sh。那么当 service nginx start 命令结束后，sh 也就结束了，sh 作为主进程退出了，自然就会令容器退出。

正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：

```
CMD ["nginx", "-g", "daemon off;"]
```

### ENTRYPOINT 入口点

ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 格式和 shell 格式。

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：

```
<ENTRYPOINT> "<CMD>"
```

那么有了 CMD 后，为什么还要有 ENTRYPOINT 呢？这种 <ENTRYPOINT> "<CMD>" 有什么好处么？让我们来看几个场景。

**场景一：让镜像变成像命令一样使用**



---

# Docker 数据管理


在容器中管理数据主要有两种方式：

 - 数据卷（Data volumes）
 - 数据卷容器（Data volume containers）


## 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

 - 数据卷可以在容器之间共享和重用
 - 对数据卷的修改会立马生效
 - 对数据卷的更新，不会影响镜像
 - 数据卷默认会一直存在，即使容器被删除

*注意：数据卷的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的数据卷*

### 创建一个数据卷

 - 在用 docker run 命令的时候，使用 -v 标记来创建一个数据卷并挂载到容器里。
 - 在一次 run 中多次使用可以挂载多个数据卷。

下面创建一个名为 web 的容器，并加载一个数据卷到容器的 /webapp 目录。

```
docker run -d -P --name web -v /webapp training/webapp python app.py
```

*注意：也可以在 Dockerfile 中使用 VOLUME 来添加一个或者多个新的卷到由该镜像创建的任意容器。*

### 删除数据卷

 - 数据卷是被设计用来持久化数据的，它的生命周期独立于容器，Docker不会在容器被删除后自动删除数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的数据卷。
 - 如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 docker rm -v 这个命令。

### 挂载一个主机目录作为数据卷

使用 -v 标记也可以指定挂载一个本地主机的目录到容器中去。

```
docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py
```

上面的命令加载主机的 /src/webapp 目录到容器的 /opt/webapp 目录。

这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。本地目录的路径必须是绝对路径，如果目录不存在 Docker 会自动为你创建它。

*注意：Dockerfile 中不支持这种用法，这是因为 Dockerfile 是为了移植和分享用的。然而，不同操作系统的路径格式不一样，所以目前还不能支持*

Docker 挂载数据卷的默认权限是读写，用户也可以通过 :ro 指定为只读。

```
$ sudo docker run -d -P --name web -v /src/webapp:/opt/webapp:ro training/webapp python app.py
```

### 查看数据卷的具体信息

在主机里使用以下命令可以查看指定容器的信息

```
$ docker inspect web
```

### 挂载一个本地主机文件作为数据卷

-v 标记也可以从主机挂载单个文件到容器中

```
docker run --rm -it -v ~/.bash_history:/.bash_history ubuntu /bin/bash
```

这样就可以记录在容器输入过的命令了。


**注意：如果直接挂载一个文件，很多文件编辑工具，包括 vi 或者 sed --in-place，可能会造成文件 inode 的改变，从 Docker 1.1 .0起，这会导致报错误信息。所以最简单的办法就直接挂载文件的父目录**


## 数据卷容器

 - 如果你有一些持续更新的数据需要在容器之间共享，最好创建数据卷容器。
 - 数据卷容器，其实就是一个正常的容器，专门用来提供数据卷供其它容器挂载的。
 - 首先，创建一个名为 dbdata 的数据卷容器：

```
docker run -d -v /dbdata --name dbdata training/postgres echo Data-only container for postgres
```

然后，在其他容器中使用 --volumes-from 来挂载 dbdata 容器中的数据卷。

```
docker run -d --volumes-from dbdata --name db1 training/postgres
docker run -d --volumes-from dbdata --name db2 training/postgres
```

可以使用超过一个的 --volumes-from 参数来指定从多个容器挂载不同的数据卷。 也可以从其他已经挂载了数据卷的容器来级联挂载数据卷。

```
docker run -d --name db3 --volumes-from db1 training/postgres
```

**注意：使用 --volumes-from 参数所挂载数据卷的容器自己并不需要保持在运行状态**

如果删除了挂载的容器（包括 dbdata、db1 和 db2），数据卷并不会被自动删除。如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时使用 docker rm -v 命令来指定同时删除关联的容器。 这可以让用户在容器之间升级和移动数据卷。具体的操作将在下一节中进行讲解。

## 利用数据卷容器来备份、恢复、迁移数据卷

可以利用数据卷对其中的数据进行进行备份、恢复和迁移。

### 备份

首先使用 --volumes-from 标记来创建一个加载 dbdata 容器卷的容器，并从主机挂载当前目录到容器的 /backup 目录。命令如下：

```
docker run --volumes-from dbdata -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
```

容器启动后，使用了 tar 命令来将 dbdata 卷备份为容器中 /backup/backup.tar 文件，也就是主机当前目录下的名为 backup.tar 的文件。

### 恢复

如果要恢复数据到一个容器，首先创建一个带有空数据卷的容器 dbdata2。

```
docker run -v /dbdata --name dbdata2 ubuntu /bin/bash
```

然后创建另一个容器，挂载 dbdata2 容器卷中的数据卷，并使用 untar 解压备份文件到挂载的容器卷中。

```
docker run --volumes-from dbdata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar
```

为了查看/验证恢复的数据，可以再启动一个容器挂载同样的容器卷来查看

```
docker run --volumes-from dbdata2 busybox /bin/ls /dbdata
```




=====================================

---

# Misc

## stop / rm all container 

```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
```


## 继续一个刚刚结束的container 

```
docker start -a -i `docker ps -q -l`
```

Explanation: 

 - docker start start a container (requires name or ID)
    -a attach to container
    -i interactive mode
 - docker ps List containers
    -q list only container IDs
    -l list only last created container


## pass proxy to docker container 

 - set https_proxy and http_proxy in you host machine
 - `docker run -e https_proxy -e http_proxy ... ` 

