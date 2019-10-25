---
title: adapter_xhr.js 默认的请求方法
date: 2019-10-19
tags: [axios源码,源码]
---
### /lib/adapter/xhr.js

xhr.js 暴露的xhrAdapter方法，是axios在浏览器环境下的默认请求方法，可以在配置中使用adapter配置项对默认的方法进行替换。

```javascript
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var requestHeaders = config.headers;

    // 判断是不是formData对象，如果是，删除header的Content-Type值，由浏览器自己决定
    if (utils.isFormData(requestData)) {
      delete requestHeaders['Content-Type']; // Let the browser set it
    }

    // 创建xml对象以及监听事件的名
    var request = new XMLHttpRequest();
    var loadEvent = 'onreadystatechange';
    var xDomain = false;

    // For IE 8/9 CORS support 针对ie8/9的cors支持
    // 只支持post和get的调用，并且不会返回相应的头
    // 对于test环境，或者没有XDomainRequest
    // 对以上情况进行特殊处理
    // Only supports POST and GET calls and doesn't returns the response headers.
    // DON'T do this for testing b/c XMLHttpRequest is mocked, not XDomainRequest.
    if (process.env.NODE_ENV !== 'test' &&
        typeof window !== 'undefined' &&
        window.XDomainRequest && !('withCredentials' in request) &&
        !isURLSameOrigin(config.url)) {
      request = new window.XDomainRequest();
      loadEvent = 'onload';
      xDomain = true;
      request.onprogress = function handleProgress() {};
      request.ontimeout = function handleTimeout() {};
    }

    // 设置http请求头中的Authorization字段
    // HTTP basic authentication
    if (config.auth) {
      var username = config.auth.username || '';
      var password = config.auth.password || '';
      requestHeaders.Authorization = 'Basic ' + btoa(username + ':' + password);
    }

    // 初始化请求方法
    // open(method, url, 是否支持异步)
    request.open(config.method.toUpperCase(), 
      buildURL(config.url, config.params, config.paramsSerializer), true);

    // 设置超时时间
    request.timeout = config.timeout;

    // Listen for ready state
    // 监听3readyState状态的变化，状态为4时才继续下一步
    request[loadEvent] = function handleLoad() {
      if (!request || (request.readyState !== 4 && !xDomain)) {
        return;
      }

      // 当这个请求出错，并且没有返回值时(request.status为0)，会触发onerror事件
      // 有一个情况除外，当请求使用了文件的协议时，即使发送成功，这个status也会为0
      if (request.status === 0 && !(request.responseURL && request.responseURL.indexOf('file:') === 0)) {
        return;
      }

      // 转化响应头部
      var responseHeaders = 'getAllResponseHeaders' in request ? 
        parseHeaders(request.getAllResponseHeaders()) : null;

      // 如果没有配置响应类型，或者响应类型为text，那么返回request.responseText， 否则返回request.response
      // responseType是一个枚举类型，手动设置返回数据的类型
      // responseText是全部后端的返回数据为纯文本的值
      // response为正文，response的类型取决于responseType
      var responseData = !config.responseType || config.responseType === 'text' ? 
        request.responseText : request.response;
      var response = {
        data: responseData,
        // IE sends 1223 instead of 204 (https://github.com/mzabriskie/axios/issues/201)
        status: request.status === 1223 ? 204 : request.status,
        statusText: request.status === 1223 ? 'No Content' : request.statusText,
        headers: responseHeaders,
        config: config,
        request: request
      };

      settle(resolve, reject, response);
      // 校验请求结果
      // if (!response.status || !validateStatus || validateStatus(response.status)) {
      //   resolve(response);
      // } else {
      //   reject(createError(
      //     'Request failed with status code ' + response.status,
      //     response.config,
      //     null,
      //     response
      //   ));
      // }

      // Clean up request
      request = null;
    };

    // ajax失败时触发
    request.onerror = function handleError() {
      reject(createError('Network Error', config));
      // 抛出network error的错误

      // Clean up request
      request = null;
    };

    // 请求超时的时候触发
    request.ontimeout = function handleTimeout() {
      reject(createError('timeout of ' + config.timeout + 'ms exceeded', config, 'ECONNABORTED'));

      // Clean up request
      request = null;
    };

    // 添加xsrf头部，只有在标准浏览器环境下才会执行
    // 尤其是web worker或者RN的环境下不会执行
    // xsrf header用来防御xsrf攻击，原理是服务端生成一个xsrf-token，浏览器在每次访问的时候都带上这个属性作为header，
    // 服务器会比较cookie中的这个值和header的这个值是否一致
    // 根据通源策略，非本源的网站无法读取修改本源的网站cookie，避免cookie被伪造
    if (utils.isStandardBrowserEnv()) {
      var cookies = require('./../helpers/cookies');

      // Add xsrf header
      var xsrfValue = (config.withCredentials || isURLSameOrigin(config.url)) && config.xsrfCookieName ?
          cookies.read(config.xsrfCookieName) :
          undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }

    // Add headers to the request
    if ('setRequestHeader' in request) {
      utils.forEach(requestHeaders, function setRequestHeader(val, key) {
        if (typeof requestData === 'undefined' && key.toLowerCase() === 'content-type') {
          // Remove Content-Type if data is undefined
          delete requestHeaders[key];
        } else {
          // Otherwise add header to the request
          request.setRequestHeader(key, val);
        }
      });
    }
    
    // 不同域下的XmlHttpRequest请求，无论Access-Cntrol-header设置成什么值，都无法改变自身站点下的cookie，
    // 除非将withCreden设置为true
    if (config.withCredentials) {
      request.withCredentials = true;
    }

    // Add responseType to request if needed
    if (config.responseType) {
      try {
        request.responseType = config.responseType;
      } catch (e) {
        if (request.responseType !== 'json') {
          throw e;
        }
      }
    }

    // 下载进度
    if (typeof config.onDownloadProgress === 'function') {
      request.addEventListener('progress', config.onDownloadProgress);
    }

    // 上传进度   request.upload返回一个XMLHttpRequestUpload对象，用来表示上传进度
    if (typeof config.onUploadProgress === 'function' && request.upload) {
      request.upload.addEventListener('progress', config.onUploadProgress);
    }

    if (config.cancelToken) {
      // 取消请求
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();
        reject(cancel);
        // Clean up request
        request = null;
      });
    }

    if (requestData === undefined) {
      requestData = null;
    }

    // Send the request
    request.send(requestData);
  });
};
```
