---
layout: post
title: multiparty源码解析
tags:
- nodejs
- node modules
categories: nodejs
description: 上传文件是一个常用功能，用node处理上传文件，大家可能一般借助于现成的模块，这些模块也已经很成熟了，本文解析multiparty的源码，来讲述一种处理上传文件请求的方式。
---
## 前言
multiparty的代码不多，通读代码后，对于http的multipart类型，node的stream、buffer等会有一个比较清晰的应用层面的认识，能够搞懂在后端处理这种上传文件请求的方式并学到一些处理的技巧。

## 版本说明
本文基于的multiparty版本为4.1.3 。

## multipart/form-data
当我们上传文件的时候，大家看请求的content-type会看到为multipart/form-data; boundary=----WebKitFormBoundaryKHBNXr2W6kAKseh5，它基于post方法，同时我们还看到它有一个boundary这个是表示分割请求内容的分隔符，这个boundary是唯一的，不会与body内的其他内容冲突。让我们看一个请求内容的示例，这里提交了两个字段和两个文件。

```javascript
------WebKitFormBoundaryKHBNXr2W6kAKseh5
Content-Disposition: form-data; name="title"

1
------WebKitFormBoundaryKHBNXr2W6kAKseh5
Content-Disposition: form-data; name="upload"; filename="SysSavePK.s11"
Content-Type: application/octet-stream


------WebKitFormBoundaryKHBNXr2W6kAKseh5
Content-Disposition: form-data; name="name"

2
------WebKitFormBoundaryKHBNXr2W6kAKseh5
Content-Disposition: form-data; name="yaya"; filename="Save002.s11"
Content-Type: application/octet-stream


------WebKitFormBoundaryKHBNXr2W6kAKseh5--
```
我们能看到在两个boundary之间有内容，第一个为传的普通字符串（非文件），其实是k-v格式的数据，key为title，value为1，Content-Disposition是对该内容的描述信息。第二个为文件，key为upload，文件名为SysSavePK.s11，然后省略了文件内容，这里没有展示详细内容。后面的同理。

## 初步思路
其实到这里我们应该就能有一个大概的思路了，在服务器端，我们遍历这些分段内容，因为有boundary的分割，所以我们不会搞混，然后在处理每段内容的时候，根据Content-Disposition可以得到传递的一些特征，比如简单的k-v，或者文件的话的文件名这些。k-v直接拿出值就可以了，文件的话就读取文件了。读取文件后存到某个目录，这样这个请求的常规k-v和文件都被我们搞定了。

## 分析源码
虽然思路有了，但是真正处理的时候还是很讲究的，从源码我们也能看出来，处理起来很繁琐，但我们也能学到很多知识，特别是stream和buffer的知识。接下来，让我们开始分析源码。

```javascript
exports.Form = Form;

util.inherits(Form, stream.Writable); //Form继承可写流 事件那些也可以直接走起
function Form(options) {
  var self = this;
  stream.Writable.call(self); //构造函数继承

  options = options || {};

  self.error = null;

  self.autoFields = !!options.autoFields;
  self.autoFiles = !!options.autoFiles;

  self.maxFields = options.maxFields || 1000;
  self.maxFieldsSize = options.maxFieldsSize || 2 * 1024 * 1024;
  self.maxFilesSize = options.maxFilesSize || Infinity;
  self.uploadDir = options.uploadDir || os.tmpdir(); //没有指定上传目录，操作系统临时目录
  self.encoding = options.encoding || 'utf8';

  self.bytesReceived = 0;
  self.bytesExpected = null;

  self.openedFiles = [];
  self.totalFieldSize = 0;
  self.totalFieldCount = 0;
  self.totalFileSize = 0;
  self.flushing = 0;

  self.backpressure = false;  //流的背压（消费者速度小于生产者）
  self.writeCbs = [];

  self.emitQueue = [];  //事件emit队列

  self.on('newListener', function(eventName) {  //来个事件吧，设置auto
    if (eventName === 'file') {
      self.autoFiles = true;
    } else if (eventName === 'field') {
      self.autoFields = true;
    }
  });
}
```
这段代码的重点其实就是Form继承了可写流，除此之外就是构造函数的一堆配置，这些配置这里解释了一部分，比如说我们new的时候，传入的上传文件的路径，如果没传，会使用操作系统的临时目录，其他配置的现在不细说，后面讲到对应的部分时再说。

然后让我们直接看重点代码，这个方法是上传文件的主要方法。先把我加了详细注释的代码贴出来，后面进行解释。

```javascript
Form.prototype.parse = function(req, cb) {
  var called = false;
  var self = this;
  var waitend = true; //等待是否结束标志

  if (cb) {
    // 如果提供了callback函数，则开启autoFields和autoFiles
    self.autoFields = true;
    self.autoFiles = true;

    // 等待请求结束调用done(回调函数)
    var end = function (done) {
      if (called) return;  //called表示是否已经调用过

      called = true;

      // 等待req事件触发结束 然后再调用done
      process.nextTick(function() {
        if (waitend && req.readable) {
          // 消耗请求剩余部分
          req.resume();   //继续流
          req.once('end', done); //结束时调用done
          return;
        }

        done();
      });
    };

    var fields = {};
    var files = {};
    self.on('error', function(err) { //发生错误直接转到结束
      end(function() {
        cb(err);
      });
    });
    self.on('field', function(name, value) { // 有field存到fieldsArray里去
      var fieldsArray = fields[name] || (fields[name] = []);
      fieldsArray.push(value);
    });
    self.on('file', function(name, file) { //文件也存到filesArray数组里
      var filesArray = files[name] || (files[name] = []);
      filesArray.push(file);
    });
    self.on('close', function() { //完成
      end(function() {
        cb(null, fields, files);
      });
    });
  }

  self.handleError = handleError; //错误处理 很多内容见下面这个方法的注释
  self.bytesExpected = getBytesExpected(req.headers); //从content-length里拿到字节数

  req.on('end', onReqEnd);
  req.on('error', function(err) { //出现错误不等结束，直接转到handleError
    waitend = false;
    handleError(err);
  });
  req.on('aborted', onReqAborted);

  var state = req._readableState; //_readableState是可读流的一个状态集合，following看状态 buffer看缓冲等
  if (req._decoder || (state && (state.encoding || state.decoder))) { //decoder一般挂一个string_decoder吧 解码对应的encoding
    // this is a binary protocol
    // if an encoding is set, input is likely corrupted
    validationError(new Error('request encoding must not be set'));
    return;
  }

  var contentType = req.headers['content-type'];
  if (!contentType) {
    validationError(createError(415, 'missing content-type header'));
    return;
  }
  // contentType比如为multipart/form-data; boundary=----WebKitFormBoundarygDr8leBuj6Rizus7
  var m = CONTENT_TYPE_RE.exec(contentType); //content type必须为multipart 毕竟是上传文件
  if (!m) {
    validationError(createError(415, 'unsupported content-type'));
    return;
  }
  //解析出来的m [ 'multipart/form-data;', index: 0, input: 'multipart/form-data; boundary=----WebKitFormBoundarygDr8leBuj6Rizus7' ]

  var boundary;
  CONTENT_TYPE_PARAM_RE.lastIndex = m.index + m[0].length - 1; //从哪儿开始才是boundary，就是不要那个multipart了

  while ((m = CONTENT_TYPE_PARAM_RE.exec(contentType))) { //拿到boundary
    if (m[1].toLowerCase() !== 'boundary') continue;
    boundary = m[2] || m[3];
    break;
  }

  if (!boundary) {
    validationError(createError(400, 'content-type missing boundary'));
    return;
  }

  setUpParser(self, boundary); //准备工作，这里保存了boundary，等会就拿这个去分隔每个部分
  req.pipe(self); //self是实现了_write方法的

  //终止req
  function onReqAborted() {
    waitend = false;
    self.emit('aborted');
    handleError(new Error("Request aborted"));
  }

  function onReqEnd() {
    waitend = false;
  }

  //错误处理
  function handleError(err) {
    var first = !self.error; //是不是首次出现error
    if (first) {
      self.error = err;
      req.removeListener('aborted', onReqAborted); //移除事件监听
      req.removeListener('end', onReqEnd);
      if (self.destStream) {  //???
        errorEventQueue(self, self.destStream, err);
      }
    }

    cleanupOpenFiles(self); //关闭文件可写流，清除临时文件

    if (first) { //首次error的话emit
      self.emit('error', err);
    }
  }

  function validationError(err) {
    // 保证事件监听触发了后才调用handle error
    // handle error on next tick for event listeners to attach
    process.nextTick(handleError.bind(null, err))
  }
};
```
这个方法内容较多，我们应该带着最开始指出的思路去理解的话，就很好理解了，毕竟再多的操作其实主线也是按照那个思路走的。