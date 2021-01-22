### 1.COPY 复制文件
格式：
```
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```

COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。比如：
```
COPY package.json /usr/src/app/
```
<源路径> 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 [filepath.Match](https://golang.org/pkg/path/filepath/#Match) 规则

<目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 WORKDIR 指令来指定）。

### 2.ADD 更高级的复制文件
ADD使用格式与COPY一致，不同的是，如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。
比如官方镜像 ubuntu 中：
```
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
...
```
**COPY与ADD选用原则：** 所有的文件复制应使用 COPY 指令，仅在需要自动解压缩的场合使用 ADD。

### 3.CMD 容器启动命令
CMD 指令的格式和 RUN 相似，也是两种格式：
```
shell 格式：CMD <命令>
exec 格式：CMD ["可执行文件", "参数1", "参数2"...]
参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
```

如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：```CMD echo $HOME```在实际执行中，会将其变更为：
```CMD [ "sh", "-c", "echo $HOME" ]```

对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出。**因此，docker里```systemctl```、```service```等命令不能执行。**

### 4.ENTRYPOINT 入口点
```ENTRYPOINT``` 的目的和 ```CMD``` 一样，都是在指定容器启动程序及参数。

当指定了 ```ENTRYPOINT``` 后，```CMD``` 的含义就发生了改变，不再是直接的运行其命令，而是将 ```CMD``` 的内容作为参数传给 ```ENTRYPOINT``` 指令，换句话说实际执行时，将变为：
```
<ENTRYPOINT> "<CMD>"
```

示例：生成获取当前公网 IP 的镜像
```
FROM ubuntu:16.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]
```
执行：
```
#无参数
docker run myip
#有参数，ENTRYPOINT会把参数（-i）一起传入，然后执行。
#换句话说，ENTRYPOINT能获取命令执行的参数
docker run myip -i
```

### 5.ENV设置环境变量
格式有两种：
```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```
这个指令很简单，就是设置环境变量而已，无论是后面的其它指令，如 RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。
```
ENV VERSION=1.0 DEBUG=on \
    NAME="Happy Feet"
```
这个例子中演示了如何换行，以及对含有空格的值用双引号括起来的办法，这和 Shell 下的行为是一致的。

### 6.ARG 构建参数
格式：```ARG <参数名>[=<默认值>]```

Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 ```--build-arg <参数名>=<值>``` 来覆盖。

### 7.VOLUME 定义匿名卷
格式为：
```
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```
容器器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中.

为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。
```
VOLUME /data
```
这里的 /data 目录就会在运行时自动挂载为匿名卷，**任何向 /data 中写入的信息都不会记录进容器存储层**， 从而保证了容器存储层的无状态化。

当然，运行时可以覆盖这个挂载设置。比如：
```
docker run -d -v mydata:/data xxxx
```
在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代了 Dockerfile 中定义的匿名卷的挂载配置。

### 8.EXPOSE 声明端口
格式为 ```EXPOSE <端口1> [<端口2>...]```。

```EXPOSE``` 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。

要将 EXPOSE 和在运行时使用 ```-p <宿主端口>:<容器端口> ```区分开来:
- -p，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，
- EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。
-

### 9.WORKDIR 指定工作目录
格式为 ```WORKDIR <工作目录路径>```。

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录。

一个错误的例子：
```
RUN cd /app
RUN echo "hello" > world.txt
```

将这个 Dockerfile 进行构建镜像运行后，会发现找不到 /app/world.txt 文件，或者其内容不是 hello。原因其实很简单：**第一层 RUN cd /app 的执行仅仅是当前进程的工作目录变更。而到第二层的时候，启动的是一个全新的容器，跟第一层的容器更完全没关系**。

因此如果需要改变以后各层的工作目录的位置，那么应该使用 ```WORKDIR```指令。

### 10.USER 指定当前用户
格式：```USER <用户名>```
USER 指令用来改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。

示例：用redis用户启动redis
```
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

### 11.健康检查
格式：
```
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
支持下列选项：
    --interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
    --timeout=<时长>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
    --retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
```

为了帮助排障，健康检查命令的输出（包括 stdout 以及 stderr）都会被存储于健康状态里，可以用 ```docker inspect``` 来查看。

### 12.ONBUILD
格式：```ONBUILD <其它指令>```。

ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN, COPY 等，而这些指令，在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。

示例：
```
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```
假设以上镜像为my-node，后续基于此镜像构建时，使用
```
FROM my-node
```
假设此镜像为node

好处：

ONBUILD 后面的命令，在构建my-node时并不会执行，只有在构建node时才会执行。如果路径需要修改，或npm install需要添加参数时，只需要修改my-node的dockerfile，并重新构建node即可，而不用修改node的dockerfile。生产环境中，基于my-node构建的node可能不止不过，这样避免了多次修改。


