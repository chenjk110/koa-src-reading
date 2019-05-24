# 请求对象Request

Request对象很多属性基本是基于http的原生请求对象req的getter、setter实现，以下就挑一些有自定义处理的属性getter、setter源码阅读

```js
// ...

// 获取origin站点信息，协议 + 主机地址
get origin() {
  return `${this.protocol}://${this.host}`;
}


// 获取完整链接
get href() {
  // support: `GET http://example.com/foo`
  // 测试originalUrl是否包含完整的协议头，不包含则添加协议信息
  if (/^https?:\/\//i.test(this.originalUrl)) return this.originalUrl;
  return this.origin + this.originalUrl;
}

// 获取请求的路径
const parse = require('parseurl'); // 解析url库
get path() {
  return parse(this.req).pathname; // 解析完后获取pathname
}

// 设置请求的路径
const stringify = require('url').format;
set path(path) {
  const url = parse(this.req);
  if (url.pathname === path) return; // 相等不操作
	
  url.pathname = path;
  url.path = null; // 去除path信息，只保留pathname
	// path、pathname 区别说明：
  // 'http://some.com/a/b/c?wd=1'
  // pathname -> '/a/b/c'
  // path -> '/a/b/c?wd=1'
  
  this.url = stringify(url); // 格式化后更新到this.url
}


// 获取query的key-value对象
const qs = require('querystring');
get query() {
  const str = this.querystring;
  const c = this._querycache = this._querycache || {}; // 初始化this._querycache并赋值c
  return c[str] || (c[str] = qs.parse(str)); // 利用qs解析query字符串 -> 存入cache -> 返回
}
// 设置query字符串 -> this.querystring
set query(obj) {
  this.querystring = qs.stringify(obj);
}

// querystring 的setter、getter，道理参照上面
get querystring() {
  if (!this.req) return '';
  return parse(this.req).query || '';
}
set querystring(str) {
  const url = parse(this.req);
  if (url.search === `?${str}`) return;

  url.search = str;
  url.path = null;

  this.url = stringify(url);
}


// 获取主机host
get host() {
  // 如果是代理模式、获取X-Forwarded-Host头部信息
  const proxy = this.app.proxy;
  let host = proxy && this.get('X-Forwarded-Host');
  
  if (!host) {
    if (this.req.httpVersionMajor >= 2) host = this.get(':authority'); // 获取认证主机信息
    if (!host) host = this.get('Host'); // 获取Host头部信息
  }
  if (!host) return '';
  return host.split(/\s*,\s*/, 1)[0]; // 用','切割Host头部信息，获取第一个值
  // '192.168.9.9:80,q=1.0;'
}

// 主机名 host = [hostname]:[port]
get hostname() {
  const host = this.host;
  if (!host) return '';
  if ('[' == host[0]) return this.URL.hostname || ''; // IPv6
  return host.split(':', 1)[0]; // '127.0.0.1:3000' -> '127.0.0.1'
}

// 获取通过new URL()解析后的URL实例
get URL() {
  // 初始化this.memoizedURL
  if (!this.memoizedURL) {
    const protocol = this.protocol;
    const host = this.host;
    const originalUrl = this.originalUrl || ''; // avoid undefined in template string
    try {
      this.memoizedURL = new URL(`${protocol}://${host}${originalUrl}`);
    } catch (err) {
      this.memoizedURL = Object.create(null); // {}
    }
  }
  return this.memoizedURL;
}

// 请求的数据未改变，对于GET、HEAD请求，Last-Modified、ETag匹配到
// 则返回2xx或者304
get fresh() {
  const method = this.method;
  const s = this.ctx.status;

  // GET or HEAD for weak freshness validation only
  if ('GET' != method && 'HEAD' != method) return false; // 非GET、HEAD直接返回false

  // 2xx or 304 as per rfc2616 14.26
  if ((s >= 200 && s < 300) || 304 == s) {
    return fresh(this.header, this.response.header);
  }

  return false;
}
// 请求到数据已改变、缓存过期等
get stale() {
  return !this.fresh
}

// 请求类型属于幂等
get idempotent() {
  // 幂等方法列表
  const methods = ['GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS', 'TRACE'];
  // 这边的位操作符"~"与indexOf的常规结合用法，即： ~0 => 0, ~[非零] => 1
  // !! 将值转换成boolean类型
  return !!~methods.indexOf(this.method);
}

// 获取内容长度content-length的值
get length() {
  const len = this.get('Content-Length');
  if (len == '') return;
  return ~~len; // 转成对应的number值
  // ~~'1' -> 1
}

// 获取子域名
get subdomains() {
  const offset = this.app.subdomainOffset; // app中的域名偏移在这里用到了, 默认2
  const hostname = this.hostname;
  if (net.isIP(hostname)) return []; // 是IP则返回空数组
  return hostname
    .split('.') // 字符串切分
    .reverse() // 反向
    .slice(offset); // 切割
  // 例如：'sub.main.com'
  // split('.') -> ['sub', 'main', 'com']
  // reverse() -> ['com', 'main', 'sub']
  // slice(2) -> ['sub']
}

// 获取header的具体值
get(field) {
  const req = this.req;
  switch (field = field.toLowerCase()) {
    case 'referer':
    case 'referrer':
      return req.headers.referrer || req.headers.referer || '';
      // referer有两种别名，则独立处理
    default:
      return req.headers[field] || ''; // 返回对应的头部信息
  }
}
```

基本上Request就是对请求的一些信息作进一步处理，比如headers的处理、url的解析等，进一步增强了http原生request对象的能力、这些处理方便开发时候使用。