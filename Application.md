# Koa构造函数Application

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
		
    this.proxy = false; // 
    this.subdomainOffset = 2; // toJSON的
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

##2.中间件的注册与处理，use()方法

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

