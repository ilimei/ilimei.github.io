---
layout: post
title: 阿里系网站全部变成您的连接不是私密连接
keywords: iptables
desc: 昨天访问阿里系的网站 如 淘宝钉钉 全部都变成了您的连接不是私密连接
photoUrl: 
---

# 阿里系网站全部变成您的连接不是私密连接

## 问题描述

昨天访问钉钉文档的时候,chrome 显示您的连接不是私密连接，手机访问却没有问题，别人电脑也没有问题。

* 重启浏览器 无效
* 重启路由器 无效
* 导出证书再导入 无效

## 解决

查看了证书路径

* GlobalSign
	+ GlobalSign Organization Validation CA - SHA256 - G2 
		* *.dingtalk.com


win+r 输入certmgr.msc 打开证书管理器

在受信任的根证书颁发机构 目录下找到了 GlobalSign 
把GlobalSign复制一份到受信任的发布者里面一份

重启浏览器->访问
不再显示 您的连接不是私密连接

## 原理 

我也不清楚，猜测可能是浏览器启动的时候,对系统证书列表和服务器证书进行判断分析