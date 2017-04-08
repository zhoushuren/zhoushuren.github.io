# koa2源码解读(1)-洋葱中间件实现

Koa 是一个非常优秀的Node.js框架、其特点是它的洋葱中间件。打开github 找到koa的源码，发现伟大的koa源码只有4个文件。

依次是：

```
application.js
context.js
request.js
response.js
```

根据模块规范，一个文件当属于一个模块。其中application为主文件入口。
其作用是整合其他三个模块。

```javascript
module.exports = class Application extends Emitter {
  /**
   * Initialize a new `Application`.
   *
   * @api public
   */

  constructor() {
    super();

    this.proxy = false;
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
  }
...
```

这里使用继承的方式加载三个模块


回顾一下我们在使用koa的时候。加载自己的中间件，这种写法大家应该不陌生吧。

```
app.use((ctx, next) => next().then(() => ctx.body = body));

```

好的，我们再来看看 app.use 这个方法的实现。

```
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
```

把传入的回调函数push到this.middleware这个数组里。返回this的目的是为了可以链式调用。

好的，以上是在node启动的时候执行的。那么我们怎么在访问的时候执行use里的回调函数呢？

看这里，还是application.js 启动是要listen一个端口吧。

```
  listen() {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen.apply(server, arguments);
  }
```
这里不懂的同学可以去看看原生的Node.js 的Http模块的http.createServer() 方法。
此方法里放入this.callback(）

那么this.callback(）的代码是什么样？

```
  callback() {
    const fn = compose(this.middleware);

    if (!this.listeners('error').length) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      res.statusCode = 404;
      const ctx = this.createContext(req, res);
      const onerror = err => ctx.onerror(err);
      const handleResponse = () => respond(ctx);
      onFinished(res, onerror);
      return fn(ctx).then(handleResponse).catch(onerror);
    };

    return handleRequest;
  }
```

到了这里是重点了。compose 是另一个模块页是koa官方开发的，这歌模块只有50行代码。。。npm上好多模块都是几十行代码，真不知道怎么想的，几十行代码页封装成一个独立的模块，也许npm就是靠这样刷数量成为世界上第一大开源包管理器吧。

为了文章精简，我去掉了注释，想看原版请移步到 [koa-compose](link:https://github.com/koajs/compose/blob/master/index.js)

```
'use strict'

const Promise = require('any-promise')

module.exports = compose

function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}

```

这里是一个闭包。dispatch 按照middleware 这个数组递归。。现在明白了吧。只有递归才是洋葱一样，怎么进去的就怎么出来。

再回到koa的application 里 callback函数的第一行 fn 是一个compose函数里的闭包。其中返回另一个闭包handleRequest
给listen() 方法里的  http.createServer(this.callback()).
所以当每次有请求的时候都会执行一次handleRequest闭包

下面再来看看handleRequest 的逻辑吧。

这里调用 createContext 创建执行上下文。

```
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.cookies = new Cookies(req, res, {
      keys: this.keys,
      secure: request.secure
    });
    request.ip = request.ips[0] || req.socket.remoteAddress || '';
    context.accept = request.accept = accepts(req);
    context.state = {};
    return context;
  }

```
它接受到的req, res 是 http.createServer() 方法接受的回调里的原生的req, res 

还记得上面说的那两个可怜的模块request，response吧。她们被卖给context.request，context.response
总之这里把所有的对象都挂在了context上。最后可怜的fn闭包仔最后一行被执行了。上面所的所有都是定义高阶函数，真正的请求执行了fn才会执行中间件的内容。

```
 return fn(ctx).then(handleResponse).catch(onerror);
```

当所有中间价执行完毕，再执行handleResponse 回调。也就是respond 函数， 在appcalition文件最后一个函数。
作用主要是处理返回值。body 等。

```
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  const res = ctx.res;
  if (!ctx.writable) return;

  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    body = ctx.message || String(code);
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}

```

至于request.js 和response.js 这两个模块这里就不叙述了，有兴趣自己看看源码。主要都对http请求和相应的一个属性的操作。

作者写了这么多，虽然你看不懂，但是有没有觉得很厉害的样子？那就点个赞吧。

说错了，不是我厉害。是伟大的Koa厉害。






