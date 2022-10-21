### 0 命令

1. `docker`会列出 Docker client 支持的所有命令
2. 查看版本: `docker --version`
3. `docker info`可以查看docker engine 的基本信息
4. `docker images`可以查看本地的所有镜像
5. `docker ps` 或者 `docker container ls`可以显示正在运行的容器
6. `docker pull 镜像名称`： 拉取一个镜像的最新版本到本地
7. `docker images 镜像名称`: 查看本地镜像的TAG，IMAGE_ID, SIZE等信息。
8. `docker run -it`： 以交互模式进入容器，并打开终端
9. `docker commit 正在运行的镜像名称 新名称`: 把正在运行的镜像及其修改保存为一个新的镜像
10. `docker history 镜像`: 会显示镜像的构建历史。

### 1 基础知识

Docker的核心组件包括:

* Docker客户端: Client
* Docker服务器: Docker daemon
* Docker镜像: Image
* Registry
* Docker容器: Container

![Docker是如何工作的](/Users/cxy/Documents/blog/tools/images/docker是如何工作的.png)

1. 容器只能使用Host的kernel，不能修改。比如CentOS 7 使用 3.xx的kernel，如果Docker Host是Ubuntu 16.04，那么CenterOS 容器中的使用的是Host4.x.x的 kernel。



#### 1.1 创建镜像

###### 1.1.1 docker commit 创建(不推荐)

`docker commit 正在运行的镜像名称 新名称`: 把正在运行的镜像及其修改保存为一个新的镜像。

#### 1.1.2 Dockerfile

首先有一个概念就是 build context，如名字所示。一个简单的Dockerfile如下:

```
FROM ubuntu
RUN apt update && apt install -y vim
```

执行如下命令: `docker build -t ubuntu-with-vi-dockerfile .`，注意`.`把当前目录设置为 build context，docker默认在 build context 目录下寻找`Dockerfile`，也可以通过 `-f`来制定 `Dockerfile`的位置。

在 build 时，Docker将 build context 中的所有文件发送给 Docker daemon。build context 为镜像构建提供所需要的文件或目录。Dockerfile 中的ADD、COPY等命令可以把 build context 中的文件添加到镜像中。

##### 镜像的缓存特性

修改上面的Dockerfile:

```
FROM ubuntu
RUN apt update && apt install -y vim
COPY HelloWorld.docker /
```

再次 build 时，输出的log如下:

```
=> [1/3] FROM docker.io/library/ubuntu
=> CACHED [2/3] RUN apt update && apt install -y vim // 注意使用了缓存的镜像。
=> [3/3] COPY HelloWorld.docker / 
=> exporting to image
=> => exporting layers
```

Docker会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在就直接使用，无需重新创建。在这里，由于之前已经运行过相同的RUN命令，这次直接使用缓存中的镜像层，然后在这个镜像层上叠加COPY(其过程是启动临时容器，复制 HelloWorld.docker，提交新的镜像层)。

可以通过 `--no-cache`来让这次 build 不使用缓存。

Dockerfile中的每一个指令都会创建一个镜像层，上层是依赖于下层的。无论什么时候，只要某一层发生变化，其上面所有层的缓存都会失效。

总结使用 Dockerfile 构建镜像的过程：

1. 从 base 镜像运行一个容器
2. 执行一条指令，对容器做修改
3. 执行类型 docker commit 的操作，生成一个新的镜像层
4. Docker再基于刚刚提交的镜像运行一个新的容器
5. 重复 2-4 步，直到Dockerfile中的所有指令执行完毕。

##### 调试Dockerfile（最新版待定）

基于前面的讨论，如果 Dockerfile 执行到某条指令失败了，我们也能够得到前一个指令成功构建出的镜像，基于这个镜像进行调试。

##### Dockerfile 常用指令

1. `FROM`： 指定 base 镜像
2. `MAINTAINER`: 设置镜像的作者，可以是任意字符串
3. `COPY`: 将文件从 build context 复制到镜像，支持两种格式：
   1. COPY src dest
   2. COPY ["src", "dest"]
4. `ADD`: 与 `COPY`类型，从 build context 复制文件到镜像。不同点：如果 src 是归档文件(tar, zip, tgz, xz等)，文件会被自动解压到 dest
5. `ENV`: 设置环境变量，环境变量可被后面的指令使用，比如: `EVN MY_VERSION 1.3 RUN apt install -y mypackage=$MY_VERSION`
6. `EXPOSE`: 指定容器中的进程会监听某个端口，Docker可以将该端口暴露出来。
7. `VOLUME`: 将文件或目录生命为 volume
8. `WORKDIR`: 为后面的 `RUN、CMD、ENTRYPOINT、ADD、COPY`指令设置镜像中的当前工作目录
9. `RUN`: 在容器中运行指定的命令
10. `CMD`: 容器启动时运行指定的命令。CMD只有最后一条生效，并且可以被 `docker run`中指定的参数替换。
11. `ENTRYPOINT`： 设置容器启动时运行的命令。`CMD` 或 `docker run` 之后的参数会被当做参数传递给 `ENTRYPOINT`