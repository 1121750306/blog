---
title: axios的请求取消操作  /lib/cancel/cancelToken.js
date: 2019-10-18
tags: [axios源码,源码]
---

## 具体使用

```javascript
// axios用于取消请求的类
const CancelToken = axios.CancelToken
// source方法会返回一个对象，对象包含
// {
  // token, 添加到请求的config，用于标识请求
  // cancel, 调用cancel方法取消请求
// }
const source = CancelToken.source()

axios.get('/info', {
  cancelToken: source.token
}).catch(function(error) {
  if (axios.isCancel(error)) {
    console.log('取消请求的错误')
  } else {
    // 其他错误
  }
})

// 调用source.cancel可以取消axios.get('/info')的请求
source.cancel('取消请求')

```

## 查看源码

```javascript
function CancelToken(executor) {  // @param {Function} executor The executor function.
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  // 在调用cancel方法之前一直处于pending状态
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }
    // 添加取消的理由
    token.reason = new Cancel(message);
    // 结束pending状态，设置为resolve
    resolvePromise(token.reason);
  });
}

// 判断该请求是否已经被取消的方法
CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
}

/**
 * Returns an object that contains a new `CancelToken` and a function that, when called,
 * cancels the `CancelToken`.
 * 返回一个对象，包含一个CancelToken对象，和让他请求取消的方法cancel
 */
CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};
```

```javascript
// xhr.js 相关代码

  if (config.cancelToken) {
    // Handle cancellation
    // 当调用了cancelToken.source返回对象中的cancel时，触发回调
    config.cancelToken.promise.then(function onCanceled(cancel) {
      if (!request) {
        return;
      }

      request.abort(); // 取消请求
      reject(cancel);
      // Clean up request
      request = null;
    });
  }
```

## 总结

  通过cancelToken，创建了一个额外的promiseA，这个promiseA对外暴露了它的resolve方法，这个promise被挂载在config下。
在调用send方法前，添加对promiseA的监听，当promiseA的状态发生变化，就会在promiseA的callback取消请求，并且将axios返回的的promiseB的状态设置为reject，从而取消请求。






