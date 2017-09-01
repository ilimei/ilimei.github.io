---
layout: post
title: docker-compose安装glusterfs
keywords: centos docker glusterfs
desc: glusterfs docker集群内高可用共享目录
photoUrl:
---

#  docker-compose安装glusterfs

## 介绍
glusterfs是一个分布式文件系统,支持 PB 级的数据量。GlusterFS 通过 RDMA 和 TCP/IP 方式将分布到不同服务器上的存储空间汇集成一个大的网络并行文件系统。docker可以将本地文件存储到GlusterFS中，保证文件备份。不会因为机器挂掉而丢失

## 编写docker-compose.yml
glusterfs的开发者制作了docker镜像 我们可以直接使用，到/home/下新建一个glusterfs目录
```
cd /home
mkdir glusterfs
cd glusterfs
```
编写docker-compose.yml
```yml
vim docker-compose.yml

version: '2'
services:
	gfs:
		image: gluster/gluster-centos
		network_mode: host
		privileged: true
		volumes:
			- /home/glusterfs/data:/var/lib/glusterd:rw
			- /home/glusterfs/volume:/data:rw
			- /home/glusterfs/logs:/var/log/glusterfs:rw
			- /home/glusterfs/conf:/etc/glusterfs:rw
			- /dev:/dev:rw
			- /etc/hosts:/etc/hosts:rw
```

## 多机器部署

假设有2台机器 10.2.4.201 10.2.4.202 部署的。

1. 每台机器编写一样的host

```shell
vim /etc/hosts
# hosts内容追加
10.2.4.201 gfs1
10.2.4.202 gfs2
```

2. 每台机器 创建docker-compose 并执行命令

```shell
cd /home/glusterfs

# 启动
docker-compose up -d

# 创建文件存储目录
docker-compose exec gfs mkdir /data/gfs

# 创建挂载目录
mkdir /home/share
```

3. 到201上执行

```shell
cd /home/glusterfs
# 进入容器
docker-compose exec gfs bash
### 容器内执行
##关联202的glusterfs
## 这个执行完后 2台glusterfs处于集群状态
gluster peer probe gfs2
#查看状态
gluster peer status
## 创建volume 命名为gfs 备份1份
glusterfs volume create gfs replicas 2 gfs1:/data/gfs gfs2:/data/gfs
## 启动gfs
glusterfs volume start gfs
## 退出容器
exit
```

4. 挂载到/home/share,两台机器都要执行

```shell
mount -t glusterfs gfs1:gfs /home/share
```

5. 测试

```shell
## 在201执行
cd /home/share
echo hello glusterfs > 1.txt
## 到202执行
cd /home/share
## 可以看到201创建的文件 证明集群成功
ls
```
