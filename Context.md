# 贯穿流程的上下文Context

ctx作为Koa请求处理中的核心变量，起到一个数据共享中枢的作用。任何一条新的请求，都会生成独立的ctx实例，同步更新绑定每一次请求，每一个中间件也通过ctx向下级传递数据，本文就来详细剖析Context的结构。

## 1. toJSON(),inspect()辅助函数

```js
const proto = module.exports = {
  inspect() {
    if (this === proto) return this;
    return this.toJSON();
  },
  toJSON() {
    return {
      // 分别调用三个实例的toJSON方法
      request: this.request.toJSON(), 
      response: this.response.toJSON(),
      app: this.app.toJSON(),
      originalUrl: this.originalUrl,
      req: '<original node req>', // req 指向http原生请求对象
      res: '<original node res>', // res 指向http原生回复对象
      socket: '<original node socket>' // socket指向原生sokcet对象
    };
  },

  get cookies() {
    if (!this[COOKIES]) {
      this[COOKIES] = new Cookies(this.req, this.res, {
        keys: this.app.keys,
        secure: this.request.secure
      });
    }
    return this[COOKIES];
  },

  set cookies(_cookies) {
    this[COOKIES] = _cookies;
  }
};

```

## 2. Cookies相关

```js
const Cookies = require('cookies'); // 引入的第三方cookies库
const COOKIES = Symbol('context#cookies');
// 获取Cookies
get cookies() {
  if (!this[COOKIES]) { // 默认初始化cookies
    this[COOKIES] = new Cookies(this.req, this.res, {
      keys: this.app.keys, // 配置混淆keys，app.keys是个数组，存放混淆加密的字符串
      secure: this.request.secure // 是否是https，从request.secure获取
    });
  }
  return this[COOKIES];
}
// 设置Cookies
set cookies(_cookies) {
  this[COOKIES] = _cookies;
}
```

## 3. 错误处理相关

```js
/**
   * Similar to .throw(), adds assertion.
   *
   *    this.assert(this.user, 401, 'Please login!');
   *
   * See: https://github.com/jshttp/http-assert
   *
   * @param {Mixed} test
   * @param {Number} status
   * @param {String} message
   */
// 实现HTTP断言,符合断言则抛出异常
assert: httpAssert
/**
   * Throw an error with `status` (default 500) and
   * `msg`. Note that these are user-level
   * errors, and the message may be exposed to the client.
   *
   *    this.throw(403)
   *    this.throw(400, 'name required')
   *    this.throw('something exploded')
   *    this.throw(new Error('invalid'))
   *    this.throw(400, new Error('invalid'))
   *
   * See: https://github.com/jshttp/http-errors
   *
   * Note: `status` should only be passed as the first parameter.
   */

// 用户级别异常，详细用法看上面注释
throw(...args) {
  throw createError(...args);
}

// 错误处理
onerror(err) {
  // don't do anything if there is no error.
  // this allows you to pass `this.onerror`
  // to node-style callbacks.
  if (null == err) return;

  if (!(err instanceof Error)) err = new Error(util.format('non-error thrown: %j', err));

  let headerSent = false;
  // http请求已经返回、或者ctx不可更改
  // this.writable -> response.writable 属性委托
  if (this.headerSent || !this.writable) {
    headerSent = err.headerSent = true;
  }

  // delegate
  // 出发app的error事件监听，在app上会执行 onerror(err, ctx)
  this.app.emit('error', err, this);

  // nothing we can do here other
  // than delegate to the app-level
  // handler and log.
  if (headerSent) { // 请求已经返回
    return;
  }

  const { res } = this;

  // first unset all headers
  if (typeof res.getHeaderNames === 'function') {
    res.getHeaderNames().forEach(name => res.removeHeader(name));
  } else {
    res._headers = {}; // Node < 7.7
  }

  // then set those specified
  // 设置错误的headers
  // this.set方法在后面提到是由委托注入的，也就是this.set实际调用的是respone.set
  this.set(err.headers); 

  // force text/plain 强制设置类型为 'text/plain'
  // this.type由委托注入，实际：this.type -> response.type
  this.type = 'text'; 

  // ENOENT support
  if ('ENOENT' == err.code) err.status = 404;

  // default to 500
  if ('number' != typeof err.status || !statuses[err.status]) err.status = 500;

  // respond
  const code = statuses[err.status];
  const msg = err.expose ? err.message : code;
  this.status = err.status;
  this.length = Buffer.byteLength(msg);
  res.end(msg);
}
```

## 4. ctx的扩展增强：委托注入response、request属性、方法

```js
// 将挂载在ctx中的response对象的属性，委托注入到ctx上，从而在ctx中可以直接访问
// ctx[target][name] -> ctx[name], 详细用法可看delegates库用法

// 挂在委托response属性、方法
delegate(proto, 'response')
	// 委托方法
  .method('attachment')
  .method('redirect')
  .method('remove')
  .method('vary')
  .method('set')
  .method('append')
  .method('flushHeaders')
	// 委托属性，setter、getter
  .access('status')
  .access('message')
  .access('body')
  .access('length')
  .access('type')
  .access('lastModified')
  .access('etag')
	// 只委托属性的getter
  .getter('headerSent')
  .getter('writable');

// 挂在委托request属性、方法
delegate(proto, 'request')
	// 委托方法
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  .method('is')
	// 委托getter、setter
  .access('querystring')
  .access('idempotent')
  .access('socket')
  .access('search')
  .access('method')
  .access('query')
  .access('path')
  .access('url')
  .access('accept')
	// 委托getter
  .getter('origin')
  .getter('href')
  .getter('subdomains')
  .getter('protocol')
  .getter('host')
  .getter('hostname')
  .getter('URL')
  .getter('header')
  .getter('headers')
  .getter('secure')
  .getter('stale')
  .getter('fresh')
  .getter('ips')
  .getter('ip');
```

通过委托机制、将reqeust、response的一系列属性方法注入到ctx，从而增强了ctx的可操作性，灵活方便，也提醒了ctx作为简单上下文的一个扩展增强。

