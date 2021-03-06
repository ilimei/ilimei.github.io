---
layout: post
title: docker swarm集群搭建
keywords: centos docker swarm
desc: docker集群搭建
photoUrl:
---

# docker swarm集群搭建

## 假设机器有
10.2.4.201 10.2.4.202 10.2.4.203

## 设置iptables,每台机器都执行 保证可以相互访问

```shell
iptales -I INPUT -s 10.2.0.0/16 -j ACCPET
```

## 修改每台机器的hostname
```shell
#201执行
hostname docker201
echo docker201 > /etc/hostname

#202执行
hostname docker202
echo docker202 > /etc/hostname

#203执行
hostname docker203
echo docker203 > /etc/hostname
```

## 安装好docker后运行创建集群命令
进入201,执行命令
```shell
docker swarm init --advertise-addr 10.2.4.201
#
docker swarm join-token manager
# 复制命令到其他2台机器执行
```

## 每台机器分别运行
```shell
docker node ls
```
可以看见其他机器证明正确

集群搭建完成

## 配置下载镜像或者代理
docker pull国内加速镜像
```shell
vim /usr/lib/systemd/system/docker.service

## 如果机器通过代理上网的话
[Service]
Environment="HTTP_PROXY=http://172.20.109.77:3128/" 
ExecStart=/usr/bin/dockerd   -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H tcp://0.0.0.0:7654  --registry-mirror=https://l1uaai20.mirror.aliyuncs.com

## 保存退出
# 重启docker
systemctl daemon-reload
service docker restart
```

