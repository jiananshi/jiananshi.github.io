---
title: Docker 101
date: 2017-01-05 23:27:34
tags:
---

记得两年前就接触过 Docker 了，当时觉得和自己毫不相干一直没看，后来到上海工作接触了代码部署、测试环境/PPE 环境等等，顿然醒悟当年如果学了 Docker 能减少多少折腾环境的工作量，这里写一下 Docker 的基本架构和一些常用操作。

![docker](http://oiw32lugp.qnssl.com/2017-01-05-%E4%B8%8B%E8%BD%BD.png)

## Docker 有哪些好处

- 将应用同环境一同打包分发
- 创建沙盒环境
- 相比 VM 性能开销小很多
...

至少「我这里没问题呀」这种情况可以被消灭了 😅

## Docker 基本架构

Docker 主要由以下几部分组成

- Images: 用于创建 Container 的模版（比如：Ubuntu、nginx）
- Container: 独立运行的应用
- Client: Docker 客户端，同 Docker 守护进程通信
- Host: 用于执行 Docker 守护进程和容器的机器
- Registry: 保存 Images 的仓库，可以理解为类似 github 的作用

## Docker 基本操作

### 运行交互式的容器

`docker run -i -t ubuntu /bin/bash`

- -t: 为容器指定一个终端
- -i: 允许同容器内的 stdin 进行交互

### 启动一个守护进程

`docker run -d -p 5000:5000 app python ./app.py`

- -d: 起一个 daemon 让容器在后台运行
- -p: 将容器的端口映射到主机端口

### 查看容器状态

- `docker ps`
- `docker logs`
- `docker top`
- `docker inspect`（查看底层信息）

## 国内 DockerHub 太慢辣？

注册 DaoCloud 的 [加速器](http://daocloud.io/mirror/) 配置到本机 Docker mirror 中即可



