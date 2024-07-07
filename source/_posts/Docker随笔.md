---
title: Docker随笔
cover: http://ban.japaneast.cloudapp.azure.com/images/2.jpg
coverWidth: 1200
coverHeight: 750
date: 2023-07-10
tag:
  -	Docker
categories: 
  -	Docker
---



# Docker随笔

## ****如何从本地传文件进 docker 容器里面****

### ****一、查询容器ID****

```docker
docker ps -a
```

![Untitled](images/Docker随笔/Untitled.png)

### ****二、传文件进docker指定路径****

```docker
docker cp /路径/文件名 容器ID:/上传路径
```

### **三、从docker传文件到实体机**

同理，如果我们需要将docker中的文件传输到实体机上，我们只需要将之前的[cp命令](https://so.csdn.net/so/search?q=cp%E5%91%BD%E4%BB%A4&spm=1001.2101.3001.7020)方向反过来