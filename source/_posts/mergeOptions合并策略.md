---
title: mergeOptions合并策略
date: 2018-12-6
tags: [vue源码,基础]
---
### mergeOptions方法阅读
#### mergeOptions
```JavaScript
// 合并构造函数原本的options和构造时传入的options
export function mergeOptions (
  parent: Object, // 构造函数上的options
  child: Object, // 构造时传入的options
  vm?: Component // 实例化本身
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child) // 方法内容见下文
    // 检查组件名是否合法
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  // 这三个方法的作用类似，都是讲props，inject，directives属性转为对象的形式，例如有时候会传入数组的形似：
  // Vue.component('blog-post', {
  //    props: ['postTitle'],
  //    template: '<h3>{{ postTitle }}</h3>'
  // })

  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  // 上下部分代码：子options如果有mixins或者extends，则需要重新调用mergeOptions来合并这些属性
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  // defaultStrat 默认合并策略， 子没有就取父的，子有就取子的
  /*const defaultStrat = function (parentVal: any, childVal: any): any {
    return childVal === undefined
      ? parentVal
      : childVal
  }*/
```

#### checkComponents方法
检查构造时传入的options中，components中引入的组件名是否合法
```JavaScript
function checkComponents (options: Object) {
  for (const key in options.components) {
    // 遍历传入options中的components，并验证引入的组件的合法性
    validateComponentName(key)
  }
}

export function validateComponentName (name: string) {
  if (!/^[a-zA-Z][\w-]*$/.test(name)) { // 命名是否符合正则中的规则
    warn(
      'Invalid component name: "' + name + '". Component names ' +
      'can only contain alphanumeric characters and the hyphen, ' +
      'and must start with a letter.'
    )
  }
  if (isBuiltInTag(name) || config.isReservedTag(name)) {
    // isBuiltInTag方法是检验该名是否与svg标签或者html标签相同
    // isReservedTag方法，检验是否和保留字相同
    warn(
      'Do not use built-in or reserved HTML elements as component ' +
      'id: ' + name
    )
  }
}
```


#### 将props统一处理为对象形式的方法，inject和directive具体思路类似，区别在于各自特有的小部分
```JavaScript
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val) // 短横线转驼峰
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    // 验证是否是对象，isPlainObject内部调用的是Object.prototype.tostring.call()和[object Object]字符串进行对比
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      if (process.env.NODE_ENV !== 'production' && isPlainObject(val)) {
        validatePropObject(name, val, vm)

        /* const propOptionsRE = /^(type|default|required|validator)$/
        校验prop对象中，类似type或者required等是否正确合法
        function validatePropObject (
          propName: string,
          prop: Object,
          vm: ?Component
        ) {
          for (const key in prop) {
            if (!propOptionsRE.test(key)) {
              warn(
                `Invalid key "${key}" in validation rules object for prop "${propName}".`,
                vm
              )
            }
          }
        }*/

      }
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```
### 几种合并策略
#### 钩子函数的合并策略
```javascript
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}

```
流畅图如下

![钩子函数合并策略mergehook](https://github.com/1121750306/try/blob/master/imgLIst/%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0%E5%90%88%E5%B9%B6%E7%AD%96%E7%95%A5mergehook.png?raw=true)

#### props,methods,inject,computed 的合并
```javascript
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm) // 校验是否是对象的方法
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  // 合并并生成新的对象
  return ret
}
```
流程和钩子函数合并类似，只是处理父子合并的时候不同，此处是使用继承的方式，同名属性使用childVal中的值

#### components,directives,filters 的合并
```javascript
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}
```
逻辑同上

#### data,computed的合并
```javascript
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) { // 如果调用mergeOptions操作的不是vm实例，一般通过Vue.extend或者Vue.component调用
    // in a Vue.extend merge, both should be functions
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    // when parentVal & childVal are both present,
    // we need to return a function that returns the
    // merged result of both functions... no need to
    // check if parentVal is a function here because
    // it has to be a function to pass previous merges.
    // 下面的操作目的，都是想要封装返回为函数
    return function mergedDataFn () {
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else { // 调用mergeOptions操作的是vm实例，一般通过new Vue来创建调用
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```
上面代码的最后，都会调用mergeData，下面是mergeData内容：
```javascript
function mergeData (to: Object, from: ?Object): Object {
  if (!from) return to
  let key, toVal, fromVal
  const keys = Object.keys(from)
  for (let i = 0; i < keys.length; i++) {
    key = keys[i]
    toVal = to[key]
    fromVal = from[key]
    if (!hasOwn(to, key)) {
      set(to, key, fromVal)
    } else if (isPlainObject(toVal) && isPlainObject(fromVal)) {
      mergeData(toVal, fromVal)
    }
  }
  return to
}
```
to为childVal，from为parentVal
大体思路：
* 没有parent，直接返回child
* 有parent，遍历parent，如果child没有就直接set，
* 如果有child，如果都是对象，递归调用mergeData，不是对象以child为准