---
layout: post
title: cef和node组合后Buffer内存回收
keywords: cef node
desc: 处理node和cef组合后内存回收崩溃问题
photoUrl: 
---
# cef和node组合后Buffer内存回收

## 目的
由于dll拥有单独的堆栈，所以在node.dll中的生成的ArrayBuffer只能在node.dll中回收，不能再libcef.dll中回收,但是node的buffer是通过v8的默认回收机制处理的,所以最终会在libcef中回收导致程序崩溃

## 原理
在v8中ArrayBuffer有两种模式
> 
  * kInternalized 表示参数`void* data`会被垃圾回收机制回收
在node的Buffer是用的这个选项 所以去掉这个选项并监听回收事件就可以了
  * kExternalized 默认选项

```c++
  /**
   * Create a new ArrayBuffer over an existing memory block.
   * The created array buffer is by default immediately in externalized state.
   * The memory block will not be reclaimed when a created ArrayBuffer
   * is garbage-collected.
   */
  static Local<ArrayBuffer> New(
      Isolate* isolate, void* data, size_t byte_length,
      ArrayBufferCreationMode mode = ArrayBufferCreationMode::kExternalized);
```

## 处理

修改node的Buffer生成方法,在node_buffer.cc中

```c++
//回收buffer
void FreeTheBuffer(char* buffer, void* hint) {
  delete buffer;
}

MaybeLocal<Object> New(Environment* env, char* data, size_t length) {
  EscapableHandleScope scope(env->isolate());

  if (length > 0) {
    CHECK_NE(data, nullptr);
    CHECK(length <= kMaxLength);
  }
  
  Local<ArrayBuffer> ab =
      ArrayBuffer::New(env->isolate(),
                       data,
                       length);
  Local<Uint8Array> ui = Uint8Array::New(ab, 0, length);
  //设置ArrayBuffer回收后的回掉方法
  CallbackInfo::New(env->isolate(), ui, FreeTheBuffer);
  Maybe<bool> mb =
      ui->SetPrototype(env->context(), env->buffer_prototype_object());
  if (mb.FromMaybe(false))
    return scope.Escape(ui);
  return Local<Object>();
}
```

## 重新编译node

这样在cef中调用http等包含Stream的方法就不会崩溃了