---
title: axios的入口方法，实例化返回的axios
date: 2019-10-21
tags: [axios源码,源码]
---

## 实例化返回的axios

```javascript
var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var defaults = require('./defaults');

/**
 * Create an instance of Axios
 *
 * @param {Object} defaultConfig The default config for the instance
 * @return {Axios} A new instance of Axios
 */
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig); // 实例化axios
  // bind为自定义的方法，返回一个函数  () => Axios.prototype.request.apply(context.args)
  // 实际上将Axios.prototype.request的执行上下文绑定到context上
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance，  axios自带的工具类
  // 将Axios.prototype上的所有方法的执行上下文绑定到context，并且继承给instance
  utils.extend(instance, Axios.prototype, context);
  // Copy context to instance
  // context继承给instance
  utils.extend(instance, context);

  return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);

// Expose Axios class to allow class inheritance
axios.Axios = Axios;

// Factory for creating new instances
axios.create = function create(instanceConfig) {
  // 合并两个配置项
  return createInstance(utils.merge(defaults, instanceConfig));
};

// Expose Cancel & CancelToken
// 请求取消操作
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// Expose all/spread
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

module.exports = axios;

module.exports.default = axios;

```

util.js 中被用到的工具类：

```javascript
/**
 * Extends object a by mutably adding to it the properties of object b.
 *
 * @param {Object} a The object to be extended
 * @param {Object} b The object to copy properties from
 * @param {Object} thisArg The object to bind function to
 * @return {Object} The resulting value of object a
 */
function extend(a, b, thisArg) { // 将b的属性添加到a上面，thisArg转移属性为函数时绑定的执行上下文
  forEach(b, function assignValue(val, key) {
    if (thisArg && typeof val === 'function') {
      a[key] = bind(val, thisArg); // 替换执行上下文
    } else {
      a[key] = val;
    }
  });
  return a;
}
```
```javascript
/**
 * Example:
 * var result = merge({foo: 123}, {foo: 456});
 * console.log(result.foo); // outputs 456
 *
 * @param {Object} obj1 Object to merge
 * @returns {Object} Result of all merge properties
 * 合并多个对象为一个对象，同名属性取最后的值
 */
function merge(/* obj1, obj2, obj3, ... */) {
  var result = {};
  function assignValue(val, key) {
    if (typeof result[key] === 'object' && typeof val === 'object') {
      result[key] = merge(result[key], val);
    } else {
      result[key] = val;
    }
  }

  for (var i = 0, l = arguments.length; i < l; i++) {
    forEach(arguments[i], assignValue);
  }
  return result;
}
```

bind 方法：
```javascript
module.exports = function bind(fn, thisArg) {
  return function wrap() {
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```

## 所以`createInstance` 干了什么

instance有Axios.prototype.request和Axios.prototype的方法，并且这些方法的执行上下文都绑定到context中

instance 里面还有 context 上的方法。
