### 1.获取镜像
```
# 先查询镜像，找到自己想要的
docker search mysql
# 拉取镜像，可指定tag，如拉取mysql 5.7：docker pull mysql:5.7
# 如果未指定tag，则拉取最新版本
docker pull mysql
```

### 2.配置mysql目录及文件
由于docker容器的独立性，在容器删除时，mysql的文件也会删除。为了解决这个问题，我们在启动docker时，需要将mysql的数据文件目录映射到宿主机，这样只即使docker删除了，只要不删除宿主机的文件，数据文件会一直保留，下次要使用时，只须重新运行一个docker容器，将mysql的数据文件目录映射到宿主机的相应目录即可。

目录规划如下：
- 在~目录下新建mysql文件
- 进入mysql文件夹，再创建conf、data文件夹(data目录存放mysql数据，可酌情存放到其他盘)
- 进行conf文件夹，新建txt文件，内容如下：
```
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
lower_case_table_names=1
```
并将文件名保存为my.cnf。更多mysql配置，请参阅官方文档。

整个目录结构如下：
```
mysql
  ├─conf
  │  └─my.cnf
  └─data
```

### 3.启动docker
```
docker run -p 3306:3306 --name mysql -v ~/mysql/conf/my.cnf:/etc/mysql/my.cnf -v ~/mysql/logs:/data/docker_data/mysql_5.7/logs:/var/log/mysql/ -v ~/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123 -d mysql:latest
```
命令解释如下：
1. -p 端口映射，将docker上的3306端口映射到宿主机3306端口，访问时可以直接访问宿主机ip+端口即可。
2. -v 目录映射，将宿主机的目录映射到docker中，后续docker的操作都会反映宿主机的目录。
3. -e 设置环境变量。
4. -d 后台运行。

### 4.连接
可以在宿主上用命令行```mysql -uroot -p123 ```连接，也可以使用navicat或SQLyog等界面工具直接连接，这里就不具体演示。

**注意**：在宿主机上连接mysql时，ip使用localhost或127.0.0.1即可，原因是docker启动时已经做了端口映射。

### 5.错误解决
#### 5.1 plugin caching sha2 password could not be loaded
docker拉取的mysql版本是8.0，据说造成此问题的原因是，mysql8.0的密码验证方式有何改变，旧的客户端不支持，所以需要将密码验证方式修改成旧版方式。方法如下：
```
# 1.进行容器
docker exec -it mysql bash
# 2.进入mysql
mysql -u root -p123
# 3.修改加密规则 
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password' PASSWORD EXPIRE NEVER; 
# 4.更新一下用户的密码 
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'; 
# 5.刷新权限 
FLUSH PRIVILEGES; 
# 6.再重置下密码
alter user 'root'@'localhost' identified by '123';
```

附：docker安装mysql 5.7的shell脚本
```shell
docker pull mysql:5.7
# 创建目录,这里将相关的数据放在/data/docker_data/mysql_5.7目录下
mkdir -p /data/docker_data/mysql_5.7/conf
mkdir -p /data/docker_data/mysql_5.7/data
# mysql配置
cat>/data/docker_data/mysql_5.7/conf/my.cnf<<EOF
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
lower_case_table_names=1
EOF

# 启动命令
docker run -p 3306:3306 --name mysql \
-v /data/docker_data/mysql_5.7/conf/my.cnf:/etc/mysql/my.cnf \
-v /data/docker_data/mysql_5.7/logs:/var/log/mysql/ \
-v /data/docker_data/mysql_5.7/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123 -d mysql:5.7
```