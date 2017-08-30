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
express是nodejs里非常流行的一个web框架，本文从头开始分析express源码，并沿着主线走通整个流程，包括初始化，路由中间件机制等，对express的主要数据结构router、layer、route也会进行详细讲解。

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
上面提到每次请求来了，我们要进行处理，但是这个处理的逻辑是谁定的呢？中间价扮演了重要的角色。

简单来说，我们这样想，一条路上有很多关卡，这每个关卡就是一个中间件，请求就是一辆车，它要开到终点路上必须经过这些关卡（中间价），如果每个关卡都有过路费，这辆车必须要条件匹配的关卡我才给你过路费（对应于是否执行中间件的逻辑）。

通过上面的那个简单举例，我们可以看出，中间件是串联的，一个接着一个，必须按顺序经过每一个中间价，但其实进不进去中间件执行，这个要看条件匹不匹配。这样子大家心里就有了个大概的流程了，下面我们开始细讲这个中间件。

中间件这里为了帮助理解我们进行一下划分，我们按应用角度把它划分成两类。

一种是普通的中间件，这种也可以理解为通用的中间件，比如说session中间件。

一种是路由中间件，这种是比如说我定义一个api路径'/api/users'，然后调用这个api时，才会进到这个中间件执行对应逻辑。

这是这两种中间件的区分标准吗，其实不是，正在的从express数据结构的角度更直观的看到两种中间件的区分，这个我们后面再说。

接下来我们先介绍三个新角色router、layer、route，然后在讲这三个是什么关系，怎么联系起来的。

### router
router是express中的一个非常重要的数据结构，它是作为一个大容器存在的，一个应用只有一个实例，里面顺序存放了各个layer，它的代码在`/lib/router/index.js`中，它是一个对象，拥有属性和方法，它的初始化代码如下。

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

这里的写法跟createApplication()一样的，就不说明了。

router.handle我们后面在说明，其实是个分发请求的函数，跟app.handle类似。这里的stack是个重点，这是一个数组，里面将会存放layer。

这里又出现了一个新的数据结构layer，下面我们先介绍layer。

### layer
layer是存在router.stack数组中的一个数据结构，它的定义代码如下：
```javascript
function Layer(path, options, fn) {
  if (!(this instanceof Layer)) {
    return new Layer(path, options, fn);
  }

  debug('new %o', path)
  var opts = options || {};

  this.handle = fn;    //处理函数
  this.name = fn.name || '<anonymous>';    //处理函数的名称
  this.params = undefined;
  this.path = undefined;
  this.regexp = pathRegexp(path, this.keys = [], opts);    //正则表达式，用来匹配用户设置的对应的路径

  // set fast path flags
  this.regexp.fast_star = path === '*'
  this.regexp.fast_slash = path === '/' && opts.end === false
}
```
看到这里我们可以设想一下，如果你定义了一个路径api/users，然后对应的处理函数func1，其实在express内部就把对应的路径和处理函数传了进去，生成了一个layer实例。

我们现在就找到感觉了，肯定会想到时候根据请求来了后对应的路径会去匹配layer的这个regxp，如果ok，就会调用这个handle就是我们刚才传的那个func1？答案是的，但是这个layer有两种情况，这取决于你的定义方式。

如果你是用app.use这种去定义，那对应的layer就是router中的layer。

如果是app.get（或者router.route）这种的话，那对应的layer就是route.stack中的layer。接下来我们是时候介绍route。

### route
route它是一个路由中间件的重要组成部分，就是说比如用app.get（或者router.route）这种方式定义的中间件，那都会有一个route，它其实的结构是layer里面有route,route里面有个stack数组里有一个或多个layer……这个我们后面再解释，router、layer、route的关系和怎样串起来的，现在我们看route的构造函数的代码：
```javascript
function Route(path) {
  this.path = path;
  this.stack = [];   //这个数组用来存放layer

  debug('new %o', path)

  // route handlers for various http methods
  this.methods = {};
}
```
结构非常简单，其实核心就是这个stack，它其实是用来存layer的，所以我们现在明确了layer可以存在router.stack中，也可以存在route.stack中。

### app.use
router、layer、route这三兄弟关系是什么，这就得从app.use开始说起了，现在就介绍app.use做了什么。

app.use我们肯定不陌生，这是我们经常用来挂载中间件到app的方法，这里我们直接看app.use的代码。
```javascript
app.use = function use(fn) {   //app.use用来添加普通中间件
  var offset = 0;
  var path = '/';

  //这里是对app.use()传入的参数个数不同的处理
  if (typeof fn !== 'function') {
    var arg = fn;

    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }

    //如果第一个参数是path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }
  //如果传的是一个函数，则以'/'作为path，该函数作为handler   否则以第一个参数作为path，第二个函数（函数数组）作为handler
  var fns = flatten(slice.call(arguments, offset));//得到的数据结构是[handler, handler ...]

  if (fns.length === 0) {
    throw new TypeError('app.use() requires middleware functions');
  }

  // setup router
  this.lazyrouter();  //初始化router(如果router还未创建)，并且为app挂载一些默认中间件
  var router = this._router;

  fns.forEach(function (fn) {
    // non-express app
    if (!fn || !fn.handle || !fn.set) {   //如果不是express app则作为普通handler放上去
      return router.use(path, fn);
    }
    //express app有自己的挂载机制
    //后面的代码是作为express app挂载的代码，代码较多，这里省略
  }, this);

  return this;
};
```
上面的代码我添加的注释应该是非常详细了，这里概括一下三个重点。
1. app.use添加中间件的时候，如果传的是一个函数，则以'/'作为path，该函数作为handler；否则以第一个参数作为path，第二个函数（函数数组）作为handler。
2. this.lazyrouter()会去判断router是否已经存在，如果不存在的话新建一个router，并且调用router.use去给app挂载两个中间件。第一个中间件是用来解析请求参数的。第二个中间件是用来把req 和 res的原型指向了app.requsest和app.response的。
3. 实际上最后还是会调用router.use方法。

这里我们接下来要解析的目标也来了，那就是router.use方法。

### router.use
之前我们讲过router、layer，然后说router的stack中有layer，话光是这么说，肯定要拿出证据，这里证据就来了。我们先直接看代码：
```javascript
proto.use = function use(fn) { //默认以'/'作为path，挂载fn作为handler
  var offset = 0;
  var path = '/';

  //这里是对router.use()传入的参数个数不同的处理
  if (typeof fn !== 'function') {
    var arg = fn;

    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }

    // first arg is the path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }
  //如果传的是一个函数，则以'/'作为path，该函数作为handler   否则以第一个参数作为path，第二个函数（函数数组）作为handler
  var callbacks = flatten(slice.call(arguments, offset));//得到的数据结构是[handler, handler ...]

  if (callbacks.length === 0) {
    throw new TypeError('Router.use() requires middleware functions');
  }

  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];

    if (typeof fn !== 'function') {
      throw new TypeError('Router.use() requires middleware function but got a ' + gettype(fn));
    }

    debug('use %o %s', path, fn.name || '<anonymous>')

	//以我们传入的path和handler函数，新建一个layer
    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;  //route=undefined表示非路由中间件

    this.stack.push(layer);  //将layer加入router的stack中
  }

  return this;
};
```
这里的代码前面部分跟app.use基本是一样的，就不分析了。到后面就是重点了，用我们传入的path和handler函数新建了一个layer，然后添加进router的stack中，其实就是把layer一个个都放到了数组中。

这里要注意，layer.route赋值成了undefined，是表明这是个非路由中间件。所以我们一般创建express应用的时候，经常用app.use(function(){})这种，其实就是进了这里，用'/'作为路径，function作为handler，新建了个layer并放到了router的数组中。

如果第一个参数传入path的话就会以传入的参数作为路径，其实我们从代码里可以看出，handler还可以传多个，作为一个数组依次执行。
### app.get/app.post...
实际中我们还会用其他的方式挂载中间件，比如说app.get、app.post……这些方式其实我们常用来挂载路由中间件，就是说匹配路径、方法ok，那我们就执行对应的handler。

这种方式的原理是什么呢，我接着看下面的源代码。
```javascript
methods.forEach(function(method){
  app[method] = function(path){   //暴露出app.get这种函数接口，用来添加路由中间件
    if (method === 'get' && arguments.length === 1) { //如果app.get(path)只有一个参数，则调用set，相当于不是路由中间件了
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    var route = this._router.route(path);   //新建一个route并添加到router的stack中
    route[method].apply(route, slice.call(arguments, 1));   //为route添加stack
    return this;
  };
});
```
这里的methods其实是一个数组，里面存放了各种方法名字符串，比如说get、post、put…… 它这样一下子把各个方法名称作为app的方法名定义了，让我们可以调用。

还有当方法为get，但是参数只有一个的时候，调用的是set方法（为app的setings对象赋值），就是说没有挂载路由中间件，这里要注意，我们用app.get来挂载路由中间件的时候要注意参数个数。

this.lazyrouter和app.use里的一样，也是router实例没有的话就初始化一个。
这里的this._router.route(path)方法就是重点了。
### router.route
让我们看router.route的源代码：

```javascript
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));  //等会路由中间件执行它的handle的时候，就会执行以这个以route为this的dispatch方法
//它会依次执行这个route的stack中的所有handler， handler1中调用next后，又去执行hanlder2
  layer.route = route; //路由中间件的route都有值的

  this.stack.push(layer);   //把layer添加到router    layer.route ->  route
  return route;
};
```

上面的代码添加了详细的注释，概括一下就是：
1. 用传进来的path新建一个route
2. 用path、options、和handler（这里不是简单的handler，后面会说明），新建一个layer
3. 将这个layer的route指向刚才新建的那个route
4. 将该layer添加进router的stack中
这里的layer是一个路由中间件的，它的route不是undefined，指向了一个route实例，这里就和之前将app.use时的layer（非路由中间件）区分开了。

这里还有个难点就是route.dispatch.bind(route)，是为了请求来了，如果需要执行这个路由中间件时，以该route作为上下文去执行dispatch方法，具体dispatch方法是干什么的呢，其实就是依次执行这个route的stack中的所有handler， handler1中调用next后，又去执行hanlder2。

回到上面methods.forEach的代码中：
```javascript
var route = this._router.route(path);   //新建一个route并添加到router的stack中
route[method].apply(route, slice.call(arguments, 1));   //为route添加stack
```

这里的this._router.route(path)我们已经讲了，现在得到了这个route。

接下来一行代码route[method].apply(route, slice.call(arguments, 1))实际上就是为刚才我们新建的那个route的stack数组中添加handler(slice.call(arguments, 1)其实拿到我们从第二个参数开始的handler处理函数)。

这里我们可以看下这部分的代码：
```javascript
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var handles = flatten(slice.call(arguments));//得到handler数组

    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];

      if (typeof handle !== 'function') {
        var type = toString.call(handle);
        var msg = 'Route.' + method + '() requires callback functions but got a ' + type;
        throw new Error(msg);
      }

      debug('%s %o', method, this.path)

      var layer = Layer('/', {}, handle); //新建一个layer实例
      layer.method = method;

      this.methods[method] = true;
      this.stack.push(layer); //将layer添加到route的stack数组中
    }

    return this;
  };
});
```

这里的代码也十分简单，跟上面很类似，也是遍历方法数组去为Route定义各种接口，比如get、post这种，做的事就是新建一个layer，这里的path是'/'，处理函数的话是我们传进去的处理函数。

之所以path是'/'其实因为这里也不需要用该layer的path去正则匹配了，因为这个route前面的那个layer已经path匹配过了，能进来route的stack里已经是路径ok的，不过method还是需要的，因为剩下的就是匹配这些在同一个route里的layer的方法了。
### 几种挂载中间件的方式区别
这里介绍了几种挂载中间件的方法app.use、router.use、app.get(post、put……)、router.route。app.use和router.use基本相同，这里讲一下app.get和router.route。

比如说我用app.get('api/users', fn)挂载路由中间件，最后的结果是Router的stack中的一个layer，这个layer有api/users的正则表达式（用来后面匹配），它的route指向的Route实例中，只有一个对应的layer（方法为get）。

其实我们一般还会用Router.route进行路由中间件挂载（我个人比较喜欢这种方式，主要感觉路由层次更加清晰），比如Router.route('api/list').get(fn)，这种的话实际上的结果和上面app.get一样，当我们继续.post(fn)、put(fn)的时候，这里的layer放到了对应的'api/list'的layer指向的Route的stack中，相当于stack中有get的layer、post的layer、put的layer。
### 总结
这一章终于讲完了我们常用那几个挂载中间件的方法，app.use、router.use、app.get(post、put……)、router.route。这几个方法都一一结合源代码作了解析，我们知道了这些方法怎么把router、layer、route的关系串起来的。

其实一个express app最后的中间件关系图可以像下面的图所示。
![](http://otlcy9opw.bkt.clouddn.com/express-router1.png)
这个图展示了一个express app常规的中间件结构，作为唯一实例Router，它的stack中有各个layer（中间件），有些路由中间件（layer.route不为undefined）有Route，对应的Route中的stack又有各个layer。

这里也能看出请求来了后的中间件执行顺序，先注册的先执行（前提是匹配路径成功（路由中间件还要匹配方法））。

## 执行流程
中间件我们已经都挂载好了，当请求来了之后执行流程是怎样的呢，这个我们之前就讲了，当请求来了后，执行的是app.handle的方法，接下来我们就来详解这个函数，然后来分析请求来了后的流程。

其实app.handle代码很简单，因为它调用了router.handle去执行来处理，在调用之前它会得到一个done函数，其实可以把它看做一个中断器（不走下面的遍历中间件流程了，会进行一些设置状态码等工作，然后调用res.end返回）。

接下来重点看router.handle的方法，这里是它的源代码核心，代码较多，我删去了不是很重要的部分，删了代码的地方用//...说明了，剩下的主要用来表现遍历路由的过程。
```javascript
proto.handle = function handle(req, res, out) {
  var self = this;

  debug('dispatching %s %s', req.method, req.url);

  var idx = 0;
  //...
  var stack = self.stack;   //这里拿出了路由的stack  [layer, layer, ....]
  //...
  var done = restore(out, req, 'baseUrl', 'next', 'params');

  req.next = next; //得到next遍历函数，就是遍历stack数组中的layer

  //...

  next();

  function next(err) {
    var layerError = err === 'route'
      ? null
      : err;

    //...


    // 出错退出路由遍历
    if (layerError === 'router') {
      setImmediate(done, null)
      return
    }

    // 如果stack已经遍历完了，退出路由
    if (idx >= stack.length) {
      setImmediate(done, layerError);
      return;
    }

    //得到pathname

    var path = getPathname(req);

    if (path == null) {
      return done(layerError);
    }

    //找到下一个要匹配的layer
    var layer;
    var match;
    var route;

    while (match !== true && idx < stack.length) {   //如果未匹配且还没遍历到stack数组末尾
      layer = stack[idx++];    //得到layer，idx加一（这样下次取就是后面一个layer了）
      match = matchLayer(layer, path);   //这个函数功能是看path和layer的正则路径是否匹配
      route = layer.route;   //拿到route，这里再提一下非路由中间件route是undefined

      if (typeof match !== 'boolean') {
        // 这里的match不等于布尔型就是报错了，这里用layerError来中断遍历下一次
        layerError = layerError || match;
      }

      if (match !== true) {
        continue;
      }

      if (!route) {
        // process non-route handlers normally
        continue;
      }

      if (layerError) {
        // routes do not match with a pending error
        match = false;
        continue;
      }

      var method = req.method;
      var has_method = route._handles_method(method);

      // build up automatic options response
      if (!has_method && method === 'OPTIONS') {
        appendMethods(options, route._options());
      }

      // don't even bother matching route
      if (!has_method && method !== 'HEAD') {
        match = false;
        continue;
      }
    }

    // no match
    if (match !== true) {
      return done(layerError);
    }

    // store route for dispatch on change
    if (route) {
      req.route = route;
    }

    // Capture one-time layer values
    req.params = self.mergeParams
      ? mergeParams(layer.params, parentParams)
      : layer.params;
    var layerPath = layer.path;

    // this should be done for the layer
    self.process_params(layer, paramcalled, req, res, function (err) {   //基本是立即执行function的东西
      if (err) {
        return next(layerError || err);
      }

      if (route) {    //这个时候是完全路径 方法都匹配的路由route
        return layer.handle_request(req, res, next);
        //会去执行layer的handle，由于这是路由中间件，所以会去执行route.dispatch.bind(route的this),遍历这个route的stack，依次执行里面的所有handler
        //这儿把next传给了该中间件的handler的next，然后执行handler的时候会调用next，去执行上面的next，即到下一个stack的layer
      }

      trim_prefix(layer, layerError, layerPath, path); //这里是执行匹配路径的普通中间件
    });
  }

  function trim_prefix(layer, layerError, layerPath, path) {
    if (layerPath.length !== 0) {
      // Validate path breaks on a path separator
      var c = path[layerPath.length]
      if (c && c !== '/' && c !== '.') return next(layerError)

      // Trim off the part of the url that matches the route
      // middleware (.use stuff) needs to have the path stripped
      debug('trim prefix (%s) from url %s', layerPath, req.url);
      removed = layerPath;
      req.url = protohost + req.url.substr(protohost.length + removed.length);

      // Ensure leading slash
      if (!protohost && req.url[0] !== '/') {
        req.url = '/' + req.url;
        slashAdded = true;
      }

      // Setup base URL (no trailing slash)
      req.baseUrl = parentUrl + (removed[removed.length - 1] === '/'
        ? removed.substring(0, removed.length - 1)
        : removed);
    }

    debug('%s %s : %s', layer.name, layerPath, req.originalUrl);

    if (layerError) {
      layer.handle_error(layerError, req, res, next);
    } else {
      layer.handle_request(req, res, next);  //这儿把next传给了该中间件的handler的next，然后执行handler的时候会调用next，去执行上面的next，即到下一个stack的layer
    }
  }
};
```