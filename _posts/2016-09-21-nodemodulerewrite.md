---
layout: post
title: cef和node合并之后 修改module.require 加载方式
keywords: cef node module.require
desc: 由于cef和node合并之后node中的require只能加载磁盘中的js,需要改写成加载资源文件包中的文件
photoUrl: 
---

# cef和node合并之后 修改module.require 加载方式

## 修改

阅读node中的module.js 有四个关键函数

> * `internalModuleStat` 判断是文件夹还是文件 以及是否存在
* `internalModuleReadFile` 读取文件内容
* `Module._extensions[".js"]` 读取js文件内容
* `Module._extensions['.json']` 读取json文件内容

这样扩展cef提供两个函数就可以了

> * getPakFile 获取文件内容
* GetPakSourceType 获取文件类型 文件返回0 文件夹返回1 不存在返回-1

改写一下就ok了

```javascript
	
//定义虚拟路径
const virualPath = process.cwd() + path.sep + "virualPath";

_internalModuleStat=internalModuleStat;

internalModuleStat = function (reqpath) {
	var re = _internalModuleStat(path._makeLong(reqpath));
	if (re < 0) {
		var basePath = reqpath.replace(virualPath, "").replace(/\\/g, "/").replace(/^\//, "");
		re = GetPakSourceType(basePath);
	}
	return re;
}

_internalModuleReadFile=internalModuleReadFile;

internalModuleReadFile = function(reqpath){
	if(reqpath.startsWith(virualPath)){
		return getPakFile(reqpath);
	}else{
		return _internalModuleReadFile(reqpath);
	}
}

// Native extension for .js
Module._extensions['.js'] = function (module, filename) {
    var content = "";
    if (filename.startsWith(virualPath)) {
        content = getPakFile(filename);
    } else {
        content = fs.readFileSync(filename, 'utf8');
    }
    module._compile(stripBOM(content), filename);
};


// Native extension for .json
Module._extensions['.json'] = function (module, filename) {
    var content = "";
    if (filename.startsWith(virualPath)) {
        content = getPakFile(filename);
    } else {
        content = fs.readFileSync(filename, 'utf8');
    }
    try {
        module.exports = JSON.parse(stripBOM(content));
    } catch (err) {
        err.message = filename + ': ' + err.message;
        throw err;
    }
};

//增加入口require方法
//如果bool为真 表示从虚拟路径查找
Module.require = function (request,bool) {
    request=bool?request.replace(/^\.\//g, "./virualPath/"):request;
    return Module._load(request);
}
```

## 修改完成

加载虚拟路径下的js文件
`NativeModule.require("module").require("./appjs/test",true);`

测试运行良好