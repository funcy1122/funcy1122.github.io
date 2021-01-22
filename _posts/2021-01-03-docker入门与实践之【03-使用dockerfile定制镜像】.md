## 利用dockerfile定制镜像
Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

以定制nginx 镜像为例，使用 Dockerfile 来定制。

在一个空白目录中，建立一个文本文件，并命名为 Dockerfile：
```
$ mkdir mynginx
$ cd mynginx
$ touch Dockerfile
```
其内容为：
```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```
### 1 FROM 指定基础镜像
所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制.FROM 就是指定基础镜像，一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

### 2 RUN 执行命令
RUN 指令是用来执行命令行命令的。由于命令行的强大能力，RUN 指令在定制镜像时是最常用的指令之一。其格式有两种：

- shell 格式：```RUN <命令>```，就像直接在命令行中输入的命令一样。刚才写的 Dockerfile 中的 ```RUN``` 指令就是这种格式。
 ```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ```
- exec 格式：```RUN ["可执行文件", "参数1", "参数2"]```，这更像是函数调用中的格式。

**Dockerfile 中每一个指令都会建立一层**，RUN 也不例外。每一个 RUN 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。

Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。为了不生成非必要层，使用```RUN```指令时，可以使用如下方法：
```
FROM debian:jessie
RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    ···
```
这里仅仅使用一个 RUN 指令，并使用 && 将各个所需命令串联起来，将多层简化为了 1 层。**在撰写 Dockerfile 的时候，要经常提醒自己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建。**

Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注释等，会让维护、排障更为容易，这是一个比较好的习惯。

### 3.构建镜像
在 Dockerfile 文件所在目录执行：
```
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM nginx
 ---> e43d811ce2f4
Step 2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 9cdc27646c7b
 ---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c
```
可以使用```docker build``` 命令进行镜像构建。其格式为：
```
docker build [选项] <上下文路径/URL/->
如：docker build -t nginx:v3 . #指定最终镜像的名称： -t nginx:v3
```

### 4.镜像构建上下文（Context）
docker build 命令最后有一个 .。. 表示当前目录，而 Dockerfile 就在当前目录，因此不少初学者以为这个路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的，这是在指定上下文路径。

Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。

进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。

当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

**总结：** docker打包是在服务端，构建时会在上下文路径下的所有内容打包上传到服务端进行，后续的构建只会操作该路径下的内容。

默认情况下，如果不额外指定 Dockerfile 的话，会将上下文目录下的名为 Dockerfile 的文件作为 Dockerfile。
可以使用```-f  -f /PATH/MyDockerfile``` 参数指定某个文件作为 Dockerfile

### 5.其它 docker build 的用法
#### 5.1 直接用 Git repo 进行构建
```docker build``` 还支持从 URL 构建，比如可以直接从 Git repo 中构建：
```
$ docker build https://github.com/twang2218/gitlab-ce-zh.git#:8.14
```
#### 5.2 用给定的 tar 压缩包构建
如果所给出的 URL 不是个 Git repo，而是个 tar 压缩包，那么 Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。
```
docker build http://server/context.tar.gz
```
#### 5.3 从标准输入中读取 Dockerfile 进行构建
如果标准输入传入的是文本文件，则将其视为 Dockerfile，并开始构建。这种形式由于直接从标准输入中读取 Dockerfile 的内容，它没有上下文，因此不可以像其他方法那样可以将本地文件 COPY 进镜像之类的事情。
```
docker build - < Dockerfile
或 cat Dockerfile | docker build -
```

#### 5.4 从标准输入中读取上下文压缩包进行构建
如果发现标准输入的文件格式是 gzip、bzip2 以及 xz 的话，将会使其为上下文压缩包，直接将其展开，将里面视为上下文，并开始构建。
```
$ docker build - < context.tar.gz
```
