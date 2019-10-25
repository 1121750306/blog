---
title: 拦截器  /lib/core/interceptorManager.js
date: 2019-10-24
tags: [axios源码,源码]
---

### /lib/core/interceptorManager.js

interceptorManager.js定义了axios的拦截器，包含了拦截器的增加、删除、循环拦截器。响应拦截器和请求拦截器都是用数组存储的。

```javascript
var utils = require('./../utils');

// 拦截器对象
function InterceptorManager() {
  this.handlers = [];
}

// 添加一个新的拦截器到拦截器的数组中，两个参数分别为触发promise的then和reject的回调
// 返回当前拦截器在数组的下标作为id，id在移除拦截器时用到
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};

// 通过传入id，移除拦截器数组中的对应拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// 遍历所有注册的拦截器
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};

module.exports = InterceptorManager;

```