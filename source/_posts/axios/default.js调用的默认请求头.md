---
title: axios的默认请求头  /lib/defaults.js
date: 2019-10-22
tags: [axios源码,源码]
---

### /lib/defaults.js

defaults.js文件中配置了axios的默认请求头，不同环境下axios的默认请求方法，格式化请求正文的方法以及响应征文的方法等

```javascript
var utils = require('./utils');
var normalizeHeaderName = require('./helpers/normalizeHeaderName');

var PROTECTION_PREFIX = /^\)\]\}',?\n/;

// 默认content-type
var DEFAULT_CONTENT_TYPE = {
  'Content-Type': 'application/x-www-form-urlencoded'
};

// 在没有设置content-type的情况下，设置content-type
function setContentTypeIfUnset(headers, value) {
  if (!utils.isUndefined(headers) && utils.isUndefined(headers['Content-Type'])) {
    headers['Content-Type'] = value;
  }
}

// 根据当前环境，获取默认的请求方法
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined') {
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  }
  return adapter;
}

var defaults = {
  adapter: getDefaultAdapter(),

  // 格式化请求，在请求发送前使用
  transformRequest: [function transformRequest(data, headers) {
    normalizeHeaderName(headers, 'Content-Type');
    // 属性名为不规则大小的content-type格式化为正确的Content-Type
    // module.exports = function normalizeHeaderName(headers, normalizedName) {
    //   utils.forEach(headers, function processHeader(value, name) {
    //     if (name !== normalizedName && name.toUpperCase() === normalizedName.toUpperCase()) {
    //       headers[normalizedName] = value;
    //       delete headers[name];
    //     }
    //   });
    // };

    // formdata，二进制数组，流媒体，文件，二进制大对象
    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) {
      return data;
    }
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }

    // 如果是一个urlSearchParams对象，例如 new URLSearchParams("key1=value1&key2=value2")
    if (utils.isURLSearchParams(data)) {
      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');
      return data.toString();
    }
    if (utils.isObject(data)) {
      setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
      return JSON.stringify(data);
    }
    return data;
  }],

  transformResponse: [function transformResponse(data) {
    /*eslint no-param-reassign:0*/
    if (typeof data === 'string') {
      data = data.replace(PROTECTION_PREFIX, '');
      try {
        data = JSON.parse(data);
      } catch (e) { /* Ignore */ }
    }
    return data;
  }],

  timeout: 0,

  // scrf设置的cookie的key和header的key
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',

  maxContentLength: -1,

  // 验证请求状态，在处理请求的时会调用，之前文章也有提到
  validateStatus: function validateStatus(status) {
    return status >= 200 && status < 300;
  }
};

defaults.headers = {
  common: {
    'Accept': 'application/json, text/plain, */*'
  }
};

utils.forEach(['delete', 'get', 'head'], function forEachMehtodNoData(method) {
  defaults.headers[method] = {};
});

// 为post，put，patch请求设置默认的Content-Type
utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE);
});

module.exports = defaults;

```