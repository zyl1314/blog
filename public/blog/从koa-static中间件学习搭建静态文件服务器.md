# 从koa-static中间件学习搭建静态文件服务器  

## koa-send 
>Static file serving middleware  

koa-static中有说明它只是koa-send的一个包装

```
const send = require('koa-send');

app.use(async (ctx) => {
  await send(ctx, ctx.path, { root: __dirname + '/public' });
})
```

查看koa-send的源码可以发现，它做的工作是根据传入的path查找文件是否存在，如果存在就创建一个流，不存在就抛出错误。  

send函数可以传入第三个参数

- `maxage` Browser cache max-age in milliseconds. (defaults to `0`)
 
- `immutable` Tell the browser the resource is immutable and can be cached indefinitely. (defaults to `false`)
 
- `hidden` Allow transfer of hidden files. (defaults to `false`)
 
- [`root`](#root-path) Root directory to restrict file access.
 
- `index` Name of the index file to serve automatically when visiting the root location. (defaults to none)
 
- `gzip` Try to serve the gzipped version of a file automatically when `gzip` is supported by a client and if the requested file with `.gz` extension exists. (defaults to `true`).
 
- `brotli` Try to serve the brotli version of a file automatically when `brotli` is supported by a client and if the requested file with `.br` extension exists. (defaults to `true`).
 
- `format` If not `false` (defaults to `true`), format the path to serve static file servers and not require a trailing slash for directories, so that you can do both `/directory` and `/directory/`.
 
- [`setHeaders`](#setheaders) Function to set custom headers on response.
 
- `extensions` Try to match extensions from passed array to search for file when no extension is sufficed in URL. First found is served. (defaults to `false`)  


可以看一下index的作用，事实上当我们在地址栏输入

```
http://www.aaa.com/
或者
http://www.aaa.com/index.html

```
可以发现效果是一样的，原因就是配置了index选项，服务端首先检查你的path是否以 '/' 结尾，假如你配置了index选项且以 '/' 结尾，那么服务端会自动将你的path和index选项拼接，如下：

```
const trailingSlash = path[path.length - 1] === '/'

...

if (index && trailingSlash) path += index

```
再看一下format的作用，其实我们经常在地址栏输入的是

```
http://www.aaa.com
而不是
http://www.aaa.com/
```
但他们的效果也是一样的，原因就是配置了format，经过resolve之后的path返回的是一个绝对路径，它是其中一种状态（文件或者文件夹），如果是文件夹，且设置了format（默认为true）和index,那么就自动添加index

```
    stats = await fs.stat(path)

    // Format the path to serve static file servers
    // and not require a trailing slash for directories,
    // so that you can do both `/directory` and `/directory/`
    if (stats.isDirectory()) {
      if (format && index) {
        path += '/' + index
        stats = await fs.stat(path)
      } else {
        return
      }
    }
```

extensions的作用好像不多见，比如你的a文件夹

```
| - a
    | - demo.txt
    | - demo.json
	| - demo.html
```
假如你设置了extensions(假设为['json', 'txt'])，那么你在地址栏输入

```

http://www.aaa.com/a/demo

事实上等同于
http://www.aaa.com/a/demo.json
```

服务端会首先判断你是否设置了extensions且path不以 '.**' 结尾

```
  if (extensions && !/\..*$/.exec(path)) {
    const list = [].concat(extensions)
    for (let i = 0; i < list.length; i++) {
      let ext = list[i]
      if (typeof ext !== 'string') {
        throw new TypeError('option extensions must be array of strings or false')
      }
	  // ['.js'] 或者 ['js'] 均可以
      if (!/^\./.exec(ext)) ext = '.' + ext
      if (await fs.exists(path + ext)) {
        path = path + ext
        break
      }
    }
  }
```
然后按照extensions的顺序依次查找拼接的path是否存在，存在即停止查找

## koa-static  

koa-static的只是给koa-send包了一层，koa-send的第二个参数path是ctx.path  

koa-static有个defer选项

- `defer` If true, serves after return next(), allowing any downstream middleware to respond first. 

```
  if (!opts.defer) {
    return async function serve (ctx, next) {
      let done = false

      if (ctx.method === 'HEAD' || ctx.method === 'GET') {
        try {
		  // koa-send 输入的path不存在时抛错（404或者500）
          done = await send(ctx, ctx.path, opts)
        } catch (err) {
		  // 如果错误码是404说明请求的不是静态文件
          if (err.status !== 404) {
            throw err
          }
        }
      }

	  //  请求不是静态文件  继续执行下面的逻辑
      if (!done) {
        await next()
      }
    }
  }

  return async function serve (ctx, next) {
    await next()

	// 假如请求方法不是get  必然不是访问静态资源
    if (ctx.method !== 'HEAD' && ctx.method !== 'GET') return
    // 说明对请求已经做了响应
    if (ctx.body != null || ctx.status !== 404) return // eslint-disable-line

    try {
      await send(ctx, ctx.path, opts)
    } catch (err) {
      if (err.status !== 404) {
        throw err
      }
    }
  }
```













