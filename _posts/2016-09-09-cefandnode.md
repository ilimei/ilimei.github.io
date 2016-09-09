---
layout: post
title: cef和node的事件组合
keywords: cef node
desc: 监听cef的事件循环，嵌入node的事件循环
photoUrl: 
---
# cef和node的事件组合

## 目的
cef的事件循环是独立，node的事件循环只是一个简单的do while,所以要将node的事件循环嵌入到cef的事件循环中

## 原理

cef的事件循环是MessageLoop类和一些delegate,UI线程用有单独的Delegate,
只要把delegate的DoIdleWork中插入自己的函数就可以监控到cef的事件循环,
node-webkit中是新建了一个MessageLoop::Delegate子类 替换了原有的delegate
那样改起来代码太多，所以我直接新建了一个函数类型,以及一个全局变量和函数
```c++
typedef void(*OnIdleMessageLoop)();
extern OnIdleMessageLoop userOnIdleMessageLoop;
extern "C" {
	__declspec(dllexport) void SetUserOnMessageIdleLoop(OnIdleMessageLoop proc);
}
```
在DoIdelWork中调用一下就可以

## 修改

修改browser_message_loop.h文件 让BrowserMessageLoop覆盖父类的
DoIdleWork方法
```c++
// Copyright (c) 2012 The Chromium Embedded Framework Authors. All rights
// reserved. Use of this source code is governed by a BSD-style license that can
// be found in the LICENSE file.

#ifndef CEF_LIBCEF_BROWSER_BROWSER_MESSAGE_LOOP_H_
#define CEF_LIBCEF_BROWSER_BROWSER_MESSAGE_LOOP_H_
#pragma once

#include "base/macros.h"
#include "base/message_loop/message_loop.h"

// Class used to process events on the current message loop.
class CefBrowserMessageLoop : public base::MessageLoopForUI {
  typedef base::MessageLoopForUI inherited;

 public:
  CefBrowserMessageLoop();
  ~CefBrowserMessageLoop() override;

  // Returns the MessageLoopForUI of the current thread.
  static CefBrowserMessageLoop* current();

  // Do a single interation of the UI message loop.
  void DoMessageLoopIteration();

  // Run the UI message loop.
  void RunMessageLoop();
  //继承覆盖父类的DoIdleWork方法
  bool DoIdleWork() override;
 private:
  DISALLOW_COPY_AND_ASSIGN(CefBrowserMessageLoop);
};

#endif  // CEF_LIBCEF_BROWSER_BROWSER_MESSAGE_LOOP_H_
```
browser_message_loop.cc文件后追加代码
```c++
OnIdleMessageLoop userOnIdleMessageLoop;
void SetUserOnMessageIdleLoop(OnIdleMessageLoop proc) {
	userOnIdleMessageLoop = proc;
}

bool CefBrowserMessageLoop::DoIdleWork() {
	bool re=MessageLoopForUI::DoIdleWork();
	if (userOnIdleMessageLoop != NULL) {
		userOnIdleMessageLoop();
	}
	return re;
}
```
## 重新编译

重新编译后 通过SetUserOnMessageIdleLoop

成功注入了监控方法