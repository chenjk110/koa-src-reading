# Koa构造函数Application

[TOC]

## 1.从构造函数开始

```js
module.exports = class Application extends Emitter { // 继承了Node原生模块events
  /**
   * Initialize a new `Application`.
   *
   * @api public
   */

  constructor() {
    super();

    this.proxy = false; // 代理方式标识
    this.subdomainOffset = 2; // toJSON的方法调用
    this.env = process.env.NODE_ENV || 'development'; // 环境变量
    
    this.middleware = []; // 存放中间件，通过use方法推入
    
    // 注入三个主要对象， 上下文ctx，请求request，回复response
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
    
    // 若util中的inspect.custom自定义Symbol存在，则使用Symbol作为名称
    // 多一步注入，作为this.inspect的alias
    if (util.inspect.custom) {
      this[util.inspect.custom] = this.inspect; // 
    }
  }
  
  // ....
}
```

## 2.中间件的注册与处理，use()方法

```js
const convert = require('koa-convert');
/**
   * Use the given middleware `fn`.
   *
   * Old-style middleware will be converted.
   *
   * @param {Function} fn
   * @return {Application} self
   * @api public
   */

use(fn) {
  // 判断fn类型，非Function报错
  if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');

  // 判断是否为generator函数
  if (isGeneratorFunction(fn)) {
    // 提示不推荐使用generator函数，并且在v3版本去除，用async/awai函数
    deprecate('Support for generators will be removed in v3. ' +
              'See the documentation for examples of how to convert old middleware ' +
              'https://github.com/koajs/koa/blob/master/docs/migration.md');
    
    /* 核心步骤1: 利用koa-convert转换传入的中间价函数fn */
    fn = convert(fn);
    
  }
  
  debug('use %s', fn._name || fn.name || '-'); // debug提醒
  
  /* 核心步骤2: 将处理后的fn推入到this.middleware数组中 */
  this.middleware.push(fn);
  
  // 返回当前实例
  return this;
}
```

**总结：**

1. use方法将传入的中间件函数
2. 通过koa-convert(co库实现) 转换成promise风格的函数
3. 依次push到this.middleware数组中，完成中间件的初始注册。

### 2.1 koa-convert如何处理中间件函数

在`use(fn)`方法中，对于传入的回调函数fn，需要进一步处理，将generator函数转化成返回Promise的风格，函数结构的统一化处理，方便Koa的洋葱结构流程处理，接下来简单了解下koa-convert的关键源码:

```js
const co = require('co') // 使用co库做generator的函数处理
const compose = require('koa-compose')

function convert (mw) {
  // 普遍判断操作
  if (typeof mw !== 'function') {
    throw new TypeError('middleware must be a function')
  }
  
  // 利用function.constructor.name属性，可以具体获知函数的类型
  // 不为generator的函数则直接使用
  // 如：普通函数Function，异步函数AsyncFunction
  if (mw.constructor.name !== 'GeneratorFunction') {
    // assume it's Promise-based middleware
    return mw
  }
  
  // 关键步骤，转换generator函数成Promise风格
  // 返回的函数为Koa中间件标准格式: function(ctx, next) { ... }
  const converted = function (ctx, next) {
    // co的转换处理
    return co.call(ctx, mw.call(ctx, createGenerator(next)))
  }
  converted._name = mw._name || mw.name // 重新注入name属性到新构造的中间件
  return converted
}

// 将next创建成generator函数
function * createGenerator (next) {
  return yield next()
}
```


## 3. 通过callbak()了解请求流程中间件是如何运作的

```js
  /**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */
	// 创建一个用于httpServer的处理回调函数
  callback() {
    // 利用koa-compose将中间件组合中一个整体中间件
    // 这也是最终我们要的那个中间件`大洋葱`
    const fn = compose(this.middleware); 
		
    // 注册onerror回调
    // 由于Application是继承events,所以直接调用events的API
    if (!this.listenerCount('error')) this.on('error', this.onerror);
		
    // 定义请求处理函数，最终用于httpServer
    const handleRequest = (req, res) => {
      // 自定义创建上下文ctx，每次请求都需要创建一个与该request关联的上下文ctx实例
      const ctx = this.createContext(req, res);
      
      return this.handleRequest(ctx, fn); // 自定义请求处理
    };

    return handleRequest; // 执行 callback(), 返回请求处理回调
  }
```

### 3.1 中间件的组合方式koa-compose实现原理

koa-compose的作用，就是将我们的中间件函数按照洋葱模型进行组合，最终形成一个整体的中间件。根据中间件的注册流程，最先注册为最外层，反之则同理，就是将this.middleware的队列模型转成洋葱这种栈模型。

```js
/* koa-compose 源码 */
function compose (middleware) {
  // middleware非数组报错
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  // fn非函数报错
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */
  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0) // 从第一个中件件开始一系列递归调用
    
    // 定义调用方法
    function dispatch (i) {
      // 在中间件中多次next(), 导致index >= i, 因为index=i在next(i + 1)之前
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      
      index = i // 中间件调用指针记录
      
      let fn = middleware[i] // 获取指定i
      
      if (i === middleware.length) fn = next // 到达最后一个中间件
      
      if (!fn) return Promise.resolve() // next非函数，则直接resolve
      
      try {
        // 递归调用下一个next，这边是实现洋葱模式运行流关键步骤
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

## 4.创建上下文ctx

每一次请求开始，都会创建一个基于this.context(构造函数那边注入到this)为原型的该请求的上下文ctx实例，并且这个ctx生命周期会贯穿整个请求流程，包括一系列中间件的执行，中间件通过ctx对象获取来自一切上游的数据共享。在任何中间件中，你可以修改、添加、获取任何ctx上的参数值，所以ctx实例是整个请求流的数据交通枢纽。但需要注意的一点是，中间件执行是有顺序，需要基于"U"形操作流对数据进行读写。

```js
/**
   * Initialize a new context.
   *
   * @api private
   */
createContext(req, res) {
  const context = Object.create(this.context); // 基于context创建 -> ctx
  const request = context.request = Object.create(this.request); // 注入: request -> ctx
  const response = context.response = Object.create(this.response); // 注入: respone -> ctx
  
  // 由于context是独立解耦的，所以每次请求，得进行其他属性的关联注入
  context.app = request.app = response.app = this;
  context.req = request.req = response.req = req;
  context.res = request.res = response.res = res;
  
  // request注入ctx
  request.ctx = response.ctx = context;
  
  // request、response相互引用
  request.response = response;
  response.request = request;
  
  context.originalUrl = request.originalUrl = req.url;
  context.state = {};
  
  // 最终返回关联后的ctx实例
  return context;
}
```

## 5.自定义的请求处理

从上文callback()函数里可以看出，callback作为创建的httpServer实例的处理回调函数，任何一次请求都会触发callback()返回的内部handleRequest函数，而handleRequest主要有两个内容:

1.创建上下文ctx

2.处理请求handleRequest(ctx, fn)

以下就是处理请求的handleRequest函数

```js
/**
   * Handle request in callback.
   *
   * @api private
   */
// 在callback()方法中使用的请求处理
// ctx 是在callback()中通过createContext()创建的实例ctx
// fnMiddleware 是在callback()中通过compose()合并后的`大洋葱`中间件函数
handleRequest(ctx, fnMiddleware) {
  const res = ctx.res;
  res.statusCode = 404; // 默认设置statusCode为404,当任何中间件未修改statusCode则是返回404
  
  const onerror = err => ctx.onerror(err); // 重新定义onerror函数
  const handleResponse = () => respond(ctx); // 处理回复，respond为外部定义的一个统一处理回复的函数
  
  // 结束请求处理，调用外部的'on-finished'库，
  // 官方解释：Execute a callback when a HTTP request closes, finishes, or errors.
  onFinished(res, onerror); 
  
  return fnMiddleware(ctx).then(handleResponse).catch(onerror); // 执行回复处理
}
```

## 8. 自定义请求回复处理

```js
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  // 使用statuses库，判断返回的code是否需要body字段
  // API说明：Returns true if a status code expects an empty body.
	// API用例：
  // status.empty[200] // => undefined
  // status.empty[204] // => true // 走缓存
  // status.empty[304] // => true // 重定向，走缓存
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end(); // 不需要body信息，则直接调用res.end()返回结果
  }
	
  // HEAD请求，不回复body，但需要获取body的数据长度
  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      // 返回body长度lenth
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body 返回状态形式的body信息，非实际数据
  // HTTP请求的通用返回格式，如果请求中的accept未指定接受的格式，
  // 则HTTP默认是content-type: '*/*';
  // 这边默认构造type -> 'text', body -> HTTP版本 >= 2 返回code, 否则返回message或者code
  if (null == body) { // body -> null, undefined
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body); // 返回二进制数据流
  if ('string' == typeof body) return res.end(body); // 返回文本数据
  
  // 流的形式返回结果，通过pipe管道移交给下一个请求处理
  if (body instanceof Stream) return body.pipe(res);

  // body: json // 这个就是不解释了
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```



## 7. 错误处理onerror

```js
  /**
   * Default error handler.
   *
   * @param {Error} err
   * @api private
   */

onerror(err) {
  if (!(err instanceof Error)) throw new TypeError(util.format('non-error thrown: %j', err));
	
  if (404 == err.status || err.expose) return; // 404或者错误已经向上抛了，则不处理
  if (this.silent) return; // this.slient 信息提示开关

  const msg = err.stack || err.toString(); // 获取错误栈信息
  console.error();
  console.error(msg.replace(/^/gm, '  '));
  console.error();
}
```

