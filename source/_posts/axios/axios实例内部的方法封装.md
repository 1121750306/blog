---
title: /lib/core/axios.js  axios实例上的各个方法
date: 2019-10-20
tags: [axios源码,源码]
---

### /lib/core/Axios.js

Axios.js中定义了Axios实例上的request，get，post，delete方法。get，post，delete等方法均基于Axios.prototype.request的封装。在Axios.prototype.request会依次执行请求拦截器，dispatchRequest，响应拦截器。

```javascript
var defaults = require('./../defaults');
var utils = require('./../utils');
var InterceptorManager = require('./InterceptorManager');
var dispatchRequest = require('./dispatchRequest');
var isAbsoluteURL = require('./../helpers/isAbsoluteURL');
var combineURLs = require('./../helpers/combineURLs');

// 创建一个新的Axios实例，参数为默认的实例配置
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(), // 请求拦截器
    response: new InterceptorManager() // 响应拦截器
  };
}

// 对于请求的特殊配置，和默认配置进行合并
Axios.prototype.request = function request(config) {
  // 如果config是一个字符串，则把config处理为请求的url
  if (typeof config === 'string') {
    config = utils.merge({
      url: arguments[0]
    }, arguments[1]);
  }

  // 默认指定get方法
  config = utils.merge(defaults, this.defaults, { method: 'get' }, config);

  // 有baseUrl并且配置的url不是绝对路径时，将url转为baseUrl开头的路径
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);
  }

  // Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);
  // 将请求拦截器，响应拦截器，以及实际请求dispatchRequest方法组合成数组，如下
  // [请求拦截器1success，请求拦截器1error，请求拦截器2success，请求拦截器2error，dispatchRequest，undefined，
  // 响应拦截器1success，响应拦截器1error，响应拦截器2success，响应拦截器2error]
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  // 开始执行整个请求流程 请求拦截器 => dispatchRequest => 响应拦截器
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};

// 提供其他的请求方法
utils.forEach(['delete', 'get', 'head'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});

module.exports = Axios;

```