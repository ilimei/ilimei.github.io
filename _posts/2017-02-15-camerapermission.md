---
layout: post
title: 华为手机拦截相机权限的问题
keywords: 华为手机 相机 android
desc: 华为手机相机权限被拦截后 debug 日志输出 this app not allowed to StartActivity
photoUrl:
---

# 华为手机拦截相机权限的问题

## 华为的系统权限拦截

相机启动是通过intent，startActivityForResult 并没有任何异常信息，在Activity的onActivityRequestResult中也没有返回，整个app失去了对相机是否启动的知情权

## 猜测

华为系统是在intent发送出去后统一拦截的 整个消息链被切断了，想到的解决办法就是启动相机后当前activity会被onPause
通过延迟判断是否有onPause发生 判断是否启动了相机，如果用户在超时前使当前activity onPause 那么也符合我们判断相机是否启动的判断
所以要把超时时间设置的足够小
