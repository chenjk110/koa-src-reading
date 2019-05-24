# 回复对象Response

与Request对象道理相同，Response对象也是基于http原生res参数的进一步封装、提供了一系列额外的属性与方法、特别是对body、header的操作上、以及对常用回复类型如：404、500等提供了默认处理。Response对象与Request对象对应、属于Koa高层抽象数据结构，但你仍然可以利用注入的Response.res对象实现http底层的res对象的API调用，如常用的res.end()等方法。接下来就继续阅读挑选较为有特色的Response对象的一些属性、方法进行标注、阅读：

## 1. 头部信息获取 response.header

```js
// 获取头部信息，node < 7.7 不支持 res.getHeaders() API
get header() {
  const { res } = this;
  return typeof res.getHeaders === 'function'
    ? res.getHeaders()
  : res._headers || {};  // Node < 7.7
}
```

### 1.1 头部信息数据编辑 response.get、response.set

```js
// 获取对应field头部信息字符串
get(field) {
  return this.header[field.toLowerCase()] || '';
}

// 设置对应field到头部信息
set(field, val) {
  if (this.headerSent) return;
	
  if (2 == arguments.length) {
    // val是数组，并将内容转成string
    if (Array.isArray(val)) val = val.map(v => typeof v === 'string' ? v : String(v));
    // val非array、非string、转成string
    else if (typeof val !== 'string') val = String(val);
    // 设置
    this.res.setHeader(field, val);
  } else {
		// 这边如果参数是一个，默认约定filed传key-value到Object类型
    for (const key in field) {
      this.set(key, field[key]); // 递归调用this.set
    }
  }
}
```

### 1.2 头部信息条目编辑 response.append、response.remove

```js
// 直接添加一个Header条目
append(field, val) {
  const prev = this.get(field);

  if (prev) {
    val = Array.isArray(prev)
      ? prev.concat(val) // prev是数组，则concat(val)
    : [prev].concat(val); // prev非数组，说明单个值，则构造成数组，并concat(val)
  }

  return this.set(field, val); // 调用设置
}

// 删除对应的header条目
remove(field) {
  if (this.headerSent) return;

  this.res.removeHeader(field);
}
```

## 2. 状态码 response.status

```js
// 获取
get status() {
  return this.res.statusCode;
}
// 设置
set status(code) {
  if (this.headerSent) return; // http返回头部，则不继续执行
  
  // code类型断言
  assert(Number.isInteger(code), 'status code must be a number');
  // code范围断言
  assert(code >= 100 && code <= 999, `invalid status code: ${code}`);
  this._explicitStatus = true;
  // 设置原生res的statusCode
  this.res.statusCode = code;
  // http版本小于2，则多加statusMessage返回值、利用statuses[code]去生成对应的message
  // 如： statuses[404] => 'Not Found'
  if (this.req.httpVersionMajor < 2) this.res.statusMessage = statuses[code];
	// 对应状态码不需要body则清空body
  if (this.body && statuses.empty[code]) this.body = null;
},
```

## 3. 消息 response.message

```js
// 获取
get message() {
  // 优先从res.statusMessage查找，没有则从statuses生成
  return this.res.statusMessage || statuses[this.status];
}

// 设置
set message(msg) {
  this.res.statusMessage = msg; // 直接设置到res.statusMessage
}
```

## 4. 数据体 response.body

### 4.1 获取、设置body

```js
// 获取body
get body() {
  return this._body; // response._body的零时body数据
}

// 设置body
set body(val) {
  const original = this._body; // 原始body数据
  this._body = val; // 更新this._body数据

  // no content
  // 若本次设置操作是去除body内容，则需要删除对应的头部内容
  if (null == val) {
    // 若本次请求的类型非原生无内容
    // 则手动设置为'No Content'状态码204
    if (!statuses.empty[this.status]) this.status = 204; 
    
    // 删除于body相关的三个头部信息
    this.remove('Content-Type');
    this.remove('Content-Length');
    this.remove('Transfer-Encoding');
    return;
  }

  // set the status
  // 未主动调用this.status设置code，则需要设置code为200
  // _explicitStatus 标明是否已经主动设定了code值，逻辑在status的getter中标明
  if (!this._explicitStatus) this.status = 200;

  // set the content-type only if not yet set
  // 判断是否需要设定content-type头部字段
  const setType = !this.header['content-type'];
	
 	// 以下为不同类型的body值所需要的一些信息设置
  // content-lenth，content-type等
  
  // string
  if ('string' == typeof val) {
    // 按需设置content-type
    if (setType) this.type = /^\s*</.test(val) ? 'html' : 'text';
    // 利用Buffer.byteLength来获取获字符串信息的比特长度，而非直接使用lenth获取
    this.length = Buffer.byteLength(val);
    return;
  }

  // buffer // 二进制
  if (Buffer.isBuffer(val)) {
    if (setType) this.type = 'bin'; // 按需设置content-type
    this.length = val.length; // 获取比特长度
    return;
  }

  // stream // 流数据
  if ('function' == typeof val.pipe) {
    onFinish(this.res, destroy.bind(null, val)); // 清除strea流数据缓存、关闭操作句柄
    ensureErrorHandler(val, err => this.ctx.onerror(err)); // 发生错误、调用ctx.onerror

    // overwriting
    // 如果是覆盖，则需要清除上次body设置时候可能存在的content-length头部
    if (null != original && original != val) this.remove('Content-Length');

    if (setType) this.type = 'bin';
    return;
  }

  // json
  this.remove('Content-Length'); // 删除
  this.type = 'json';
}
```

### 4.2 获取，设置body的内容长度Content-Length

```js
// 设置
set length(n) {
  this.set('Content-Length', n); // 直接覆盖
}

// 获取
get length() {
  const len = this.header['content-length'];
  const body = this.body;

  if (null == len) {
    if (!body) return; // 无body内容
		// 不同类型的content-length获取
    // string
    if ('string' == typeof body) return Buffer.byteLength(body);
    // buffer
    if (Buffer.isBuffer(body)) return body.length;
    // json
    // 这边预先设置下json序列化后的比特长度
    if (isJSON(body)) return Buffer.byteLength(JSON.stringify(body));
    return;
  }
	
  // 保留整数位
  return Math.trunc(len) || 0;
}
```

