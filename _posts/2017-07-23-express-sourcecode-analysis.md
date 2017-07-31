---
layout: post
title: express源码解析
tags:
- nodejs
- express
categories: nodejs
description: express是nodejs里非常流行的一个web框架，本文从头开始分析express源码，并沿着主线走通整个流程，包括初始化，路由中间件机制等。
---
## 前言
express是nodejs里非常流行的一个web框架，本文从头开始分析express源码，并沿着主线走通整个流程，包括初始化，路由中间件机制等。

## 版本说明
本文基于的express版本为4.15.3 。

## express源码目录结构说明
express的源代码位于/lib文件夹下，结构如下图所示：

![](http://otlcy9opw.bkt.clouddn.com/express-code-structure.png)

这里一句话概括一下各个文件的作用，具体的内容会在后面讲解，比如具体的router、route、layer的关系等会在后面内容进行讲解。

**express.js**  是主要入口，类似于index.js，这里集合了各个模块，暴露出对外的构造方法。

**application.js**  是为app进行扩充，添加了很多方法。

**request.js**  为req对象添加了各种方法，req指向了http.IncomingMessage的原型。

**response.js**  为res对象添加了各种方法，res指向了http.ServerResponse的原型。

**router/index.js**  是router的主要文件。

**router/layer.js** 是layer这个数据结构的主要文件，这个数据结构是路由部分的关键数据结构。

**router/route.js** 是route的主要文件。

**middleware/init.js** 是express自带的中间件初始化。

**middleware/query.js** 是express自带中间件的一些辅助方法。

**utils.js** 是工具类。

**view.js** 是视图（模板）的文件，这个是帮助定和处理义express视图的。

## 创建一个app

我们现在创建一个简单的express应用。

```javascript
var express = require('express');
	
var app = express();
	
app.get('/', function(req, res){
  res.send('Hello World');
});
	
app.listen(3000);
console.log('Express started on port 3000')
```
	
当我们require('express')，实际上是require了`/lib/express`，这是express的主要入口。让我们进入`express.js`，看它做了什么。

```javascript
var EventEmitter = require('events').EventEmitter;
var mixin = require('merge-descriptors');
var proto = require('./application');
var Route = require('./router/route');
var Router = require('./router');
var req = require('./request');
var res = require('./response');
```
	
它require了各个模块，这里对各个模块做说明：

1. EventEmitter 是一个对象，通过prototype可以得到events模块的所有方法，比如on、emit这些。
2. mixin 可以看作一个工具方法，目的把一个A对象的属性合并到B对象，会保留属性描述符。
3. proto、Route、Router、req、res是各个文件导出的模块，这些模块都会被express.js用到。

然后，我们来看express.js的核心部分，它是一个函数，它的内容如下。

```javascript
function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next); //这个实际上是请求来了之后的触发函数
  };

  mixin(app, EventEmitter.prototype, false); //为app添加EventEmitter的原型的各个方法
  mixin(app, proto, false); //为app添加application模块定义的属性和方法

  //创建一个对象，只有app属性，但是原型为request(原型为原始的IncomingMessage添加了几个方法的对象)
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })
//创建一个对象，只有app属性，但是原型为response(原型为原始的response添加了几个方法的对象)
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  app.init(); //初始化，这个方法是来自于application中定义的，主要执行一些默认设置
  return app;
}
```
	
这个函数是作为我们的入口点，所以每行代码我都进行了说明，这里我在补充几个重点。app = function(){} 这里可以看到它把app定义成了一个function，然后为这个function添加了很多属性方法。

我们知道，在js中函数其实也是一种引用类型，它是可以随意添加属性方法的，为一个函数添加属性和方法后有什么特殊用法呢，我这里举个例子。

```javascript
var app = function() {}
app.a = '11';
app.b = '22';

现在的app实际上变成了下面这样
	{
  funtion
  a: '11'
  b: '22'
}
```
	
上面例子的app可以作为一个函数直接执行app()，同时也可以用app.a这样得到它的属性。

我们再回到上面express中的createApplication函数，这里它为什么这样写呢？我们用最简单的那个express例子来体会一下。

通过var app = express() 我们得到了这个app对象(就是createApplication函数返回的那个app)，这个app我们上面说过了，可以作为一个函数直接执行，同时也有了各种属性方法可以调用，然后我们就可以app.get设置路由了，其实还有app.use这些，各种各样的api去为我们express应用设置各种东西。

重点来了，我们看app.listen这个函数的代码如下：

```javascript
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```
	
这里的this就是我们上面提到的那个app，刚才说了，它可以作为函数执行，所以当http.createServer时，它作为参数传了进去，它实际上是作为一个函数传进去的，就会被添加到这个服务器的request事件（可以查看nodejs api去查阅对应的http模块内容），这样的话，当每个请求来了，app函数就是处理函数了，它调用的是app.handle，这就是每个请求来了后会走的一个处理函数，你也可以把他看作一个请求看到我们服务器后的起跑线。

有同学会问，那后面那行return server.listen.apply(server, arguments)的作用呢，这里上面那行代码得到的server是个http.Server的实例，可以调用listen方法"启动服务器"，通过apply（server, arguments），以server作为上下文，传入了app.listen传入的参数，作为server.listen的配置。

这样服务器就启动起来了，并且app.handle作为处理函数来处理每次请求。

这里就迈出了epxress源码解析的第一步，后面我们会开始讲对app.handle这个每次请求来的时候调用的处理函数的分析，后面的角色比如Router和Route这些就开始粉墨登场了，let's go！

## 挂载中间件

### 理解中间件
上面提到每次请求来了，我们要进行处理，但是这个处理的逻辑是谁定的呢？中间价扮演了重要的角色，简单来说，我们这样想，一条路上有很多关卡，这每个关卡就是一个中间件，请求就是一辆车，它要开到终点路上必须经过这些关卡（中间价），如果每个关卡都有过路费，这辆车必须要条件匹配的关卡我才给你过路费（对应于是否执行中间件的逻辑）。

通过上面的那个简单举例，我们可以看出，中间件是串联的，一个接着一个，必须按顺序经过每一个中间价，但其实进不进去中间件执行，这个要看条件匹不匹配。这样子大家心里就有了个大概的流程了，下面我们开始细讲这个中间件。

中间件这里为了帮助理解我们进行一下划分，我们按应用角度把它划分成两类，一种是普通的中间件，这种也可以理解为通用的中间件，比如说session中间件。一种是路由中间件，这种是比如说我定义一个api路径'/api/users'，然后调用这个api时，才会进到这个中间件执行对应逻辑。这是这两种中间件的区分标准吗，其实不是，正在的从express数据结构的角度更直观的看到两种中间件的区分，这个我们后面再说。

接下来我们先介绍三个新角色router、layer、route，然后在讲这三个是什么关系，怎么联系起来的。

### router
router是express中的一个非常重要的数据结构，它是作为一个大容器存在的，里面顺序存放了各个layer，它的代码在`/lib/router/index.js`中，它是一个对象，拥有属性和方法，它的初始化代码如下。

```javascript
var proto = module.exports = function(options) {
  var opts = options || {};

  function router(req, res, next) {
    router.handle(req, res, next);
  }

  // mixin Router class functions
  setPrototypeOf(router, proto)

  router.params = {};
  router._params = [];
  router.caseSensitive = opts.caseSensitive;  //是否启用区分大小写路由
  router.mergeParams = opts.mergeParams;
  router.strict = opts.strict;  //是否启用严格路由
  router.stack = [];  //存放layer的数组

  return router;
};
```

这里的写法跟createApplication()一样的，就不说明了。router.handle我们后面在说明，其实是个分发请求的函数，跟app.handle类似。这里的stack是个重点，这是一个数组，里面将会存放layer。这里又出现了一个新的数据结构layer，下面我们先介绍layer。

### layer