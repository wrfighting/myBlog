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