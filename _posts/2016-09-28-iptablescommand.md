---
layout: post
title: iptables的一些命令记录
keywords: iptables
desc: iptables的一些命令记录
photoUrl: 
---
# iptables的一些命令记录


## 查看规则

```
	iptables -L INPUT --line-numbers -n
```

## 禁止某端口的访问

```
	iptables -t filter -A INPUT -s 0.0.0.0/0 -p tcp --dport 3306 -j REJECT
```

## 开启某端口内网访问

```
	iptables -t filter -A INPUT -s 127.0.0.1 -p tcp --dport 3306 -j ACCEPT
```