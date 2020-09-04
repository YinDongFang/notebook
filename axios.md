# axios源码解析
## 入口文件
`axios.js`
文件中通过`createInstance`创建实例，实际返回的是`request`函数，`bind`到`Axios`实例，并将`Axios.prototype`和实例的属性方法复制到返回的函数上，因此`axios`直接作为函数调用时相当于调用`Axios.prototype.request`，`axios.get`等函数调用相当于`Axios.prototype.get`等。
```javascript
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  utils.extend(instance, context);
  
  // 实际返回的是 bind(Axios.prototype.request, context);
  return instance;
}
var axios = createInstance(defaults);
```
## `Axios.js`内部实现
`get``post`等别名方法的实现很简单，就是对`request`方法的一层封装。
```javascript
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```
`Axios`构造器函数就是绑定默认配置和拦截器
```javascript
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```
重点内容在于`request`函数，内部逻辑执行顺序：
1. 标准化配置，合并配置。
2. 挂载拦截器，其实就是构造一个函数队列。
  2.1. 默认函数`dispatchRequest`是实际发送请求的处理函数。
  2.2. `request`拦截器通过`unshift`逆序插入`dispatchRequest`前。
  2.3. `response`拦截器通过`push`顺序放置在`dispatchRequest`后。
3. `Promise.resolve(config)`构造初始`promise`，通过`promise.then`顺序执行函数队列。
  3.1 `request`拦截器中接收参数为`config`，返回同样是`config`，顺序传递到`dispatchRequest`。
  3.2 `response`拦截器接收参数是`dispatchRequest`的返回结果`response`，同样返回`response`。
```
## 拦截器实现代码
```javascript
// Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
```
拦截器管理器内部维护一个数组，添加`push`，删除时对应下标置`null`。
## `dispatchRequest`内部实现
`dispatchRequest`负责对数据进行处理，并调用底层适配器发送请求。
1. 通过`transformData`转换请求data。
2. 调用适配器发送请求。
3. 通过`transformData`转换响应data。
