---
title: Centos7安装Docker
cover: https://close2u.work/images/2.jpg
coverWidth: 1200
coverHeight: 750
date: 2023-07-10
top: true
tag:
  -	Centos7
  - Docker
  - Docker-registry
  - redis
  - Docker-Compose
categories: 
  -	Centos7
  - Docker
  - Docker-registry
  - redis
  - Docker-Compose
---



# Centos7安装Docker

# 0.安装Docker

Docker 分为 CE 和 EE 两大版本。CE 即社区版（免费，支持周期 7 个月），EE 即企业版，强调安全，付费使用，支持周期 24 个月。

Docker CE 分为 `stable` `test` 和 `nightly` 三个更新频道。

官方网站上有各种环境下的 [安装指南](https://docs.docker.com/install/)，这里主要介绍 Docker CE 在 CentOS上的安装。

# 1.CentOS安装Docker

Docker CE 支持 64 位版本 CentOS 7，并且要求内核版本不低于 3.10， CentOS 7 满足最低内核的要求，所以我们在CentOS 7安装Docker。

## 1.1.卸载（可选）

如果之前安装过旧版本的Docker，可以使用下面命令卸载：

```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine \
                  docker-ce
```

## 1.2.安装docker

首先需要大家虚拟机联网，安装yum工具

```
yum install -y yum-utils \           device-mapper-persistent-data \           lvm2 --skip-broken
```

然后更新本地镜像源：

```
# 设置docker镜像源
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo

yum makecache fast
```

然后输入命令：

```
yum install -y docker-ce
```

docker-ce为社区免费版本。稍等片刻，docker即可安装成功。

## 1.3.启动docker

Docker应用需要用到各种端口，逐一去修改防火墙设置。非常麻烦，因此建议大家直接关闭防火墙！

启动docker前，一定要关闭防火墙后！！

启动docker前，一定要关闭防火墙后！！

启动docker前，一定要关闭防火墙后！！

```
# 关闭systemctl stop firewalld# 禁止开机启动防火墙systemctl disable firewalld
```

通过命令启动docker：

```
systemctl start docker  # 启动docker服务systemctl stop docker  # 停止docker服务systemctl restart docker  # 重启docker服务
```

然后输入命令，可以查看docker版本：

```
docker -v
```

如图：

image-20210418154704436

![Untitled](images/Centos7安装Docker/Untitled.png)

## 1.4.配置镜像加速

docker官方镜像仓库网速较差，我们需要设置国内镜像服务：

参考阿里云的镜像加速文档：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

# 2.CentOS7安装DockerCompose

## 2.1.下载

Linux下需要通过命令下载：

```
# 安装curl -L https://github.com/docker/compose/releases/download/1.23.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

如果下载速度较慢，或者下载失败，可以使用课前资料提供的docker-compose文件：

![Untitled](images/Centos7安装Docker/Untitled%201.png)

image-20210417133020614

上传到`/usr/local/bin/`目录也可以。

## 2.2.修改文件权限

修改文件权限：

```
# 修改权限chmod +x /usr/local/bin/docker-compose
```

## 2.3.Base自动补全命令：

```
# 补全命令curl -L https://raw.githubusercontent.com/docker/compose/1.29.1/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose
```

如果这里出现错误，需要修改自己的hosts文件：

```
echo "199.232.68.133 raw.githubusercontent.com" >> /etc/hosts
```

# 3.Docker镜像仓库

搭建镜像仓库可以基于Docker官方提供的DockerRegistry来实现。

官网地址：https://hub.docker.com/_/registry

## 3.1.简化版镜像仓库

Docker官方的Docker Registry是一个基础版本的Docker镜像仓库，具备仓库管理的完整功能，但是没有图形化界面。

搭建方式比较简单，命令如下：

```docker
docker run -d \
    --restart=always \
    --name registry	\
    -p 5000:5000 \
    -v registry-data:/var/lib/registry \
    registry
```

命令中挂载了一个数据卷registry-data到容器内的/var/lib/registry 目录，这是私有镜像库存放数据的目录。

访问http://YourIp:5000/v2/_catalog 可以查看当前私有镜像服务中包含的镜像

## 3.2.带有图形化界面版本

使用DockerCompose部署带有图象界面的DockerRegistry，命令如下：

```yaml
version: '3.0'
services:
  registry:
    image: registry
    volumes:
      - ./registry-data:/var/lib/registry
  ui:
    image: joxit/docker-registry-ui:static
    ports:
      - 8000:80
    environment:
      - REGISTRY_TITLE=传智教育私有仓库
      - REGISTRY_URL=http://registry:5000
    depends_on:
      - registry
```

特别注意，如果说安装的版本是latest版本，将会出现

![Untitled](images/Centos7安装Docker/Untitled%202.png)

最好安装static版本

## 3.3.配置Docker信任地址

我们的私服采用的是http协议，默认不被Docker信任，所以需要做一个配置：

```
# 打开要修改的文件
vi /etc/docker/daemon.json
# 添加内容：
"insecure-registries":["http://43.136.131.133:8000"]
# 重加载
systemctl daemon-reload
# 重启docker
systemctl restart docker
```

## 4.Docker中安装redis

### 一、Docker搜索redis镜像

命令：docker search <镜像名称>

```xml
docker search redis
```

可以看到有很多redis的镜像，此处因没有指定版本，所以下载的就是默认的最新版本 。redis latest.

### 二、Docker拉取镜像

命令：：docker pull <镜像名称>:<版本号>

```xml
docker pull redis
```

### 三、Docker挂载配置文件

接下来就是要将redis 的配置文件进行挂载，以配置文件方式启动redis 容器。（挂载：即将宿主的文件和容器内部目录相关联，相互绑定，在宿主机内修改文件的话也随之修改容器内部文件）

1）、挂载 redis 的配置文件

2）、挂载 redis 的持久化文件（为了数据的持久化）。

配置文件是放在

liunx 下redis.conf文件位置： /home/redis/dockerredis/redis.conf

liunx 下redis的data文件位置 ： /home/dockerredis/myredis/data

### 四、启动redis 容器

```xml
docker run --log-opt max-size=100m --log-opt max-file=2 -p 6379:6379 --name myredis -v /home/redis/dockerredis/redis.conf:/etc/redis/redis.conf -v /home/redis/dockerredis/data:/data -d redis:7.0 redis-server /etc/redis/redis.conf  --appendonly yes  --requirepass lhj020826..
```

启动容器的时候redis的守护进程不能打开不然会和docker的 -d 出现冲突

redis.conf的

![Untitled](images/Centos7安装Docker/Untitled%203.png)

不能和

![Untitled](images/Centos7安装Docker/Untitled%204.png)

冲突

- `-restart=always` 总是开机启动
`--log`是日志方面的
`-p` 6379:6379 将6379端口挂载出去
`--name` 给这个容器取一个名字
`-v` 数据卷挂载
- `/home/redis/myredis/myredis.conf:/etc/redis/redis.conf` 这里是将 liunx 路径下的redis.conf 和redis下的redis.conf 挂载在一起。
- `/home/redis/myredis/data:/data` 这个同上
`-d redis:7.0` 表示后台启动redis的tag版本
`redis-server /etc/redis/redis.conf` 以配置文件启动redis，加载容器内的conf文件，最终找到的是挂载的目录 `/etc/redis/redis.conf` 也就是liunx下的`/home/redis/docerredis/myredis.conf`
 `–appendonly yes` 开启redis 持久化
`–requirepass lhj020826..`

### 五、测试
1、通过docker ps指令查看启动状态
`docker ps -a |grep myredis` # 通过docker ps指令查看启动状态，是否成功.

![Untitled](images/Centos7安装Docker/Untitled%205.png)

2、查看容器运行日志
命令：

```xml
docker logs --since 30m <容器名>
```

此处 --since 30m 是查看此容器30分钟之内的日志情况。

![Untitled](images/Centos7安装Docker/Untitled%206.png)

```xml
docker logs --since 30m myredis

```

3、容器内部连接进行测试
进入容器

命令：docker exec -it <容器名> /bin/bash

此处跟着的 redis-cli 是直接将命令输在上面了。

docker exec -it myredis redis-cli
进入之后，我直接输入查看命令：

error是没有权限验证。（因为设置了密码的。）

验证密码：

![Untitled](images/Centos7安装Docker/Untitled%207.png)

查看当前redis有没有设置密码：（得验证通过了才能输入的）

config get requirepass

![Untitled](images/Centos7安装Docker/Untitled%208.png)

### 六、配置文件
redis.conf

```xml
# bind 192.168.1.100 10.0.0.1

# bind 127.0.0.1 ::1

#bind 127.0.0.1

protected-mode no
port 6379
tcp-backlog 511
requirepass 000415
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""
databases 30
always-show-logo yes
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir ./
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly yes
appendfilename "appendonly.aof"
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-max-len 128
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

### 七、Docker删除Redis

6.1、删除Redis 容器
查看所有在运行的容器：
命令：

docker ps -a
1

停止运行的Redis。停止命令：docker stop <容器名>

docker stop myredis # myredis 是我启动redis 命名的别
1

删除redis 容器： 删除容器命令： docker rm <容器名>

docker rm myredis
1

6.2、删除Redis镜像
删除容器后，我们开始删除redis镜像。

查看全部镜像 命令：docker images

删除镜像 命令 docker rmi <容器 id>

docker rmi 739b59b96069 # 这是我镜像redis id