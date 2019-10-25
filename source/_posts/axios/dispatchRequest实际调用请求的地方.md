---
title: axios源码中实际调用请求的地方  /lib/core/dispatchRequest.js
date: 2019-10-23
tags: [axios源码,源码]
---

### /lib/core/dispatchRequest.js

dispatchRequest.js 文件是axios源码中实际调用请求的地方

```javascript

var utils = require('./../utils');
var transformData = require('./transformData');
var isCancel = require('../cancel/isCancel');
var defaults = require('../defaults');

// 如果请求被取消，则抛出已取消
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}

//  使用可配置的转换器想服务器发起请求
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  // Ensure headers exist
  config.headers = config.headers || {};

  // Transform request data
  // 使用config.transformRequest对data和headers进行格式化，具体见/lib/default.js
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );

  // 对不同配置的headers进行合并，有前到后优先级增高
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers || {}
  );

  // 删除headers中的method属性
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  // 如果配置了adapter，则用配置的adapter替换默认方法
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);

    // 格式化返回值的data呵headers
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    // 请求失败的回调处理
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
  });
};

```