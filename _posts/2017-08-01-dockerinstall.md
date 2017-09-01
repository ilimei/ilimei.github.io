---
layout: post
title: centos安装docker
keywords: centos docker docker-compose
desc: centos安装docker
photoUrl:
---


## 安装docker

1. 拥有root权限的用户。

2. 确保linux内核版本是3.10以上并且是64位的centos版本。如果不能满足这个前提，建议看官绕道走吧。
```
$ uname -r
```


1. 升级yum安装包，确保都是最新的版本
```
$ sudo yum update
```

* 在yum repository增加docker的repository
```
      $  sudo vim /etc/yum.repos.d/docker.repo

      ## 在vim编辑器中输入以下内容后保存

      [dockerrepo]
      name=Docker Repository
      baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
      enabled=1
      gpgcheck=1
      gpgkey=https://yum.dockerproject.org/gpg
```

* 安装docker-engine
```
      $  sudo yum install docker-engine
```

* 安装docker-engine完成后，启动docker服务
```	
      sudo systemctl start docker.service
```

*  测试docker服务是否成功
```
	   $ docker --version
```

## 安装docker-compose

1. 需要先安装企业版linux附加包（epel)
```
      $   yum -y install epel-release
```

* 安装pip
```
 $   yum -y install python-pip
```

* 更新pip
```
 $   pip install --upgrade pip
```

* 安装docker-compose
```
 $   pip install docker-compose
```

* 查看docker-compose版本信息
```
 $   docker-compose --version
```