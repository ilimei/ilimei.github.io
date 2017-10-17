---
layout: post
title: docker生产机概述以及重启方式
keywords: docker
desc: docker
photoUrl:
---

# 服务列表

* glusterfs服务
* manager服务
* docker集群

# glusterfs服务
glusterfs服务位于2个机器 11和12上 主要用来做目录共享和备份
11,12,13上的/home/share用的都是glusterfs提供的目录,所以他们的/home/share实际是同一个文件夹，相当于共享一个硬盘

## 查看是否挂载

11机器上执行 df -k
如果存在/home/share的挂载点 证明挂载正确
如果没有则挂载,则调用挂载
```shell
# 命令分解 -t glusterfs 指定的是驱动
# gfs1是个域名 配在hosts里 指向的机器是11
# gfs是个glusterfs提供的目录
# /home/share是挂载到本地的目录
mount -t glusterfs gfs1:gfs /home/share
```
12,13同理

## 启动glusterfs
在11和12上分别执行
```shell
cd /home/glusterfs
docker-compose restart
```

# manager服务

manager是管理界面的后台服务,必须保证docker service中的mysql_master是运行状态才能启动。

```shell
# 查看mysql_master状态
# 如果属于1/1表示正在运行
docker service ls
```

manager服务运行在12上，所以只操作12就可以

```shell
cd /home/manager
docker-compose restart
#查看日志
# 如果日志中出现 8808 表示运行成功
docker-compose logs --tail=100 -f
```

如果manager启动的时候数据库报错 可以先把mysql_master干掉

```shell
docker service rm mysql_master
cd /home/manager
docker-compose restart
#查看日志
# 如果日志中出现 8808 表示运行成功
docker-compose logs --tail=100 -f
```

# docker集群

由11,12,13组成的docker集群。

```shell
# 查看集群中的节点
docker node ls

# 查看运行的服务
docker service ls

# 重启集群中的服务
docker servie scale 服务名=0
docker servie scale 服务名=1

# 查看集群中服务运行在哪
docker service ps 服务名

# 删除集群中的服务
docker service rm 服务名
```