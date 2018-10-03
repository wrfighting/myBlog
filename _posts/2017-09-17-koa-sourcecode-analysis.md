 ---
layout: post
title: koa源码解析
tags:
- nodejs
- koa
categories: nodejs
description: koa作为一个非常好用的http中间件框架，相比express具有更加鲜明的特点。个人从express一路用过来，后面遇到koa后，感觉koa确实好用，源代码很早就看过了，但是没有做总结，主要感觉确实比较简单，毕竟精简了很多东西，但其中的有些代码写法感觉还挺有意思，最近有空了，就写一下分析。
---
## 前言
koa源码主要可以分为两块，一部分是自身的那部分代码，另一部分是在koa-compose这个module里的。个人感觉自身koa部分的代码比较简单，精华其实是在koa-compose里，在这里先分析koa，然后分析koa-compose。

## 版本说明
本文基于的koa版本为2.3.0 。

## koa源码目录结构说明
源代码位于/lib文件夹下，就只有四个文件。

这里一句话概括一下各个文件的作用，具体的内容会在后面讲解。

**application.js**  主要文件，类似于我们的app.js或index.js，算是整个的入口，这个文件定义了koa的对象。

**request.js**  定义了一个prototype，包括了request的很多属性的getter和setter还有一些方法。

**response.js**  定义了一个prototype，包括了response的很多属性的getter和setter还有一些方法。

**context.js**  定义了context的prototype，除了context的方法外，还委托了request和response的属性。

## 创建一个app

我们现在创建一个简单的应用。

```javascript
const Koa = require('../lib/application');
const app = new Koa();

app.use(ctx => {
  ctx.body = 'Hello Koa';
});

app.listen(3000);
```
	
new了一个koa的实例，我们进到application.js先看构造函数。继承了emitter，这也是常规操作，就不说了。

```javascript

constructor() {
    super();

    this.proxy = false;
    this.middleware = []; //存放中间件的数组
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    this.context = Object.create(context);  //原型继承context
    this.request = Object.create(request);
    this.response = Object.create(response);
  }

```
	
这段代码比较简单，这里要说一下Object.create，因为context、request、response都是导出了一个prototype的对象，所以需要原型继承来使用这个原型，后面context的委托也是用的这个prototype。

```javascript

listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
}

```
	
listen函数，跟express的差不多，可以看出来，重点还是在callback这个函数上，是处理请求的函数。

讲callback函数之前，先说明一下use函数。

```javascript

//挂载中间件函数，生成器函数会被转换为async包装的函数，放进中间件数组
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }

```
	
use绑定的函数会被放进一个数组里，其实也跟express差不多。

```javascript

  callback() {
    const fn = compose(this.middleware);
    if (!this.listeners('error').length) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      res.statusCode = 404;
      const ctx = this.createContext(req, res); //挂载了很多东西app ctx request response等
      const onerror = err => ctx.onerror(err);
      const handleResponse = () => respond(ctx);  //返回函数，res设置一些信息，然后res.end
      onFinished(res, onerror);    //监听请求完成，如果请求完成或者出错，则执行onerrer,其实就是捕捉请求的错误
      return fn(ctx).then(handleResponse).catch(onerror);  //关键，compose会依次执行中间件的函数（洋葱模式），最后返回promise，执行返回的response helper
    };

    return handleRequest;
  }

```
	
compose函数先放着，后面重点讲解，这里createContext主要是创建了一个对象（原型继承context），然后在这个对象上挂载了request、response（原型继承request和response），还挂载了req（原始的req），res（原始的res）和app，还有一些其他属性比如cookies、originalUrl等。这个实际上就是我们写处理函数的时候的ctx。

onerror函数比较简单，主要就是打印错误信息。这里讲一下respond函数。

```javascript

 function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return; //res不被koa处理

  const res = ctx.res;
  if (!ctx.writable) return;

  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {  //未定义的code
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {  //body为空，取message或者code
    body = ctx.message || String(code);
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);  //是buffer直接写入
  if ('string' == typeof body) return res.end(body);  //string也是
  if (body instanceof Stream) return body.pipe(res);  //是流的话，pipe写入

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {  //header还没设置的话
    ctx.length = Buffer.byteLength(body);  //设置length
  }
  res.end(body);
}

```
我这里注释写的很详细了，可以看到koa对我们的res的一些处理，从代码里面看出，如果我不想要koa来处理，比如我想直接把一个流不经过koa直接pipe到原始的res里，可以先设置ctx.respond为false，这样就不会被koa处理了，算是对于返回我们的一些可以自定义操作吧。

## koa-compose

可以看出Koa本身的代码很少而且十分简单，其中起到关键作用的实际上还是koa-compose，下面讲解这个模块。

```javascript

function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) { //context就是我们的ctx
    // last called middleware #
    let index = -1
    return dispatch(0)  //开始
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]  //从middleware数组中取一个个中间件async函数
      if (i === middleware.length) fn = next  //退出递归条件，取完middleware数组最后一个后，i就和middleware.length相等了，这个时候fn=next实际上就等于undefined
      if (!fn) return Promise.resolve() //结束直接resolve出去触发后面的then
      try {
        return Promise.resolve(fn(context, function next () { //把ctx和next（触发递归进入下一个）传入到我们的async函数里，这时执行fn的代码，遇到await next就调用dispatch触发下一个
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)  //整个过程出现错误被这个reject
      }
    }
  }
}

```
代码其实是一个简单的递归并用到了闭包，由于递归其实类似于我们的数据结构栈，先入后出。所以简单来说可以理解为入的时候在这个函数的执行时间里（控制权在它手里），然后后面出的时候又到了它执行时间。这样其实就是我们说的Koa洋葱式调用顺序了。

还要注意的一点就是由于fn都是async函数（middleware数组里的），async函数返回一个promise。而且在最后compose是返回Promise.resolve（没有错误情况下），错误情况返回Promise.reject，所以在koa中的callback里，是直接
```javascript

   return fn(ctx).then(handleResponse).catch(onerror); 

```
这样会最后会执行handleResponse，当然有错的时候也能从catch里执行onerror。

## co去哪了
读到这里，有人会产生一个问题，之前听说co在koa中起到了很大的作用，但是现在咋没看到了，都没看到这篇文章提。其实以前co作用大是因为在koa1的时候，是生成器函数，需要co帮忙。现在koa2以async函数为主，就不咋需要co了。所以现在co主要用来对当use挂载的是生成器函数的时候进行转换为类似async函数（其实是转换为一个被co执行的函数，并且next改成了调用yield，co执行返回的是一个promise，这样可以看做类似于async函数了）的,这部分代码现在出现在koa-convert里。

具体co的代码就不分析了，代码不多，实现得有点巧妙，大家可以看一下，网上的分析文章也很多，推荐看es6入门里阮一峰讲解的co，结合他讲解的生成器函数，这样更容易理解。

## 总结
以上整个koa的源码分析，代码量比较少，写得十分精简，这种就是高质量代码的表现~，也在于koa定位就是一个精简的框架，相比express剥离了很多东西比如说路由。个人更喜欢koa，虽然最开始的时候使用的是express，而且用了比较长的时间……但后来接触了koa后，感觉让人眼前一亮，开发起来也爽了很多，所以我更推荐大家直接用koa吧，哈哈。