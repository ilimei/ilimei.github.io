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
行号在前面的优先级比后面的高

## 删除规则
```
iptables -D INPUT 4 #上个命令取到的行号
```

## 开启某端口内网访问
要写在前面
```
iptables -t filter -A INPUT -s 127.0.0.1 -p tcp --dport 3306 -j ACCEPT
```

## 禁止某端口的访问

```
iptables -t filter -A INPUT -s 0.0.0.0/0 -p tcp --dport 3306 -j REJECT
```

