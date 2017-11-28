# koa原理浅析
>选取的版本为koa2  

koa的源码由四个文件组成  

```
application.js    koa的骨架
context.js        ctx的原型
request.js        request的原型
response.js       response的原型
```

## 基本用法  

```
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```
## 初始服务器
利用http模块创建服务器

```
const app = http.createServer((req, res) => {
	...
})  
app.listen(3000)
```
事实上koa把这些包在了其listen方法中  

```
  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
```
显然this.callback()返回的是一个形如下面的函数  

```
(req, res) => {}
```
## 上下文ctx

callback方法如下  

```
  callback() {
    const fn = compose(this.middleware);

    if (!this.listeners('error').length) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
```
ctx在koa中事实上是一个包装了request和response的对象，从createContext中可以看到起继承自context  

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
可以看到ctx.request继承自request，ctx.response继承自response，查看response和request可以看到里面大都是set和get方法（获取query，设置header）等等。并且ctx代理了ctx.request和ctx.response的方法，在源码中可以看到  

```

delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  .method('remove')
  .method('vary')
  .method('set')
  .method('append')
  .method('flushHeaders')
  .access('status')
  .access('message')
  .access('body')
  .access('length')
  .access('type')
  .access('lastModified')
  .access('etag')
  .getter('headerSent')
  .getter('writable');

/**
 * Request delegation.
 */

delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  .method('is')
  .access('querystring')
  .access('idempotent')
  .access('socket')
  .access('search')
  .access('method')
  .access('query')
  .access('path')
  .access('url')
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
所以我们可以直接这么写

```
ctx.url

等价于

ctx.request.url
```

## 中间件

我们再看一下callback函数，观察发现compose模块十分的神奇，我暂且把它称为是一个迭代器，它实现了中间件的顺序执行  

```
const fn = compose(this.middleware);

打印fn如下

  function (context, next) {
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
```
最初接触koa的时候我疑惑为什么我写了  

```
ctx.body = 'hello world'
```
并没有ctx.response.end()之类的方法，事实上koa已经帮我们做了处理，在handleRequest方法中  

```
const handleResponse = () => respond(ctx);

// fnMiddleware即为上面compose之后的fn
fnMiddleware(ctx).then(handleResponse).catch(onerror)
```
fnMiddleware返回的是一个promise，在中间件逻辑完成后在respond函数中最终去处理ctx.body  

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

## 错误处理

- （非首部）中间件层处理（我瞎起的）  

对于每个中间件可能发生的错误，可以直接在该中间件捕获

```
app.use((ctx, next) => {

	try {
		...		
	} catch(err) {
		...
	}

})
```

- (首部)中间件层处理

事实上，我们只要在第一个中间件添加try... catch... ，整个中间件组的错误都是可以捕获的到的。

- (应用级别)顶层处理

```
app.on('error', (err) = {})
```
在上面中间件执行时看到，koa会自动帮我们捕获错误并处理，如下

```
      try {
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
		//  捕获错误
        return Promise.reject(err)
      }


//  在ctx.onerror中处理
const onerror = err => ctx.onerror(err);
fnMiddleware(ctx).then(handleResponse).catch(onerror)
```

我们看ctx.onerror发现它事实上是出发app监听的error事件  

```
  onerror(err) {


// delegate
    this.app.emit('error', err, this);

```

假如我们没有定义error回调怎么办呢，koa也为我们定义了默认的错误处理函数  


callback方法做了判断

```
  callback() {

    ...

    if (!this.listeners('error').length) this.on('error', this.onerror);

    ...
  }
```

全文完











