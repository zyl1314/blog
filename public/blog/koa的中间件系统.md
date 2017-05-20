# koa的中间件系统
最近在看es6的一些东西，对genetator很感兴趣，顺便看一下koa

```
var koa = require('koa');
var app = koa();

// response-time中间件
app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  this.set('X-Response-Time', ms + 'ms');
});

// logger中间件
app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

// 响应中间件
app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
```
上面是koa的官方示例，相较于express（中间件内部主要依赖于回调），koa的中间件系统真正帮我们解决了回调。简要分析一下上述代码，use方法添加Generator函数，当有请求传进来，genetator对象会自动运行，遇到yeild next执行中间件的跳转，当前generator对象处理完毕会自动回滚至上一中间件执行未完成的逻辑。

我们来简单的实现以下这个中间件系统：

- 先简单一点，不考虑yield next中间件跳转，上一个中间件执行完毕自动执行下一个中间件

```
var gensFn = []
var gens = []
var flag = 0

function use(fn) {
  gensFn.push(fn)
}

function co(gen) {
  function next(data) {
    var result = gen.next(data)
    if (result.done) {
      i++
      gen[i] && co(gen[i])
    } else {
      result.value.then(function(data) {
          next(data)
      })
    }
  }

  next()
}

function triger() {
  // 获取genetator对象
  gensFn.forEach(function(fn) {
      gens.push(fn())
  })

  co(gens[0])
}

triger()
```
这个很简单，首先我们需要维护一个数组保存generator对象，然后执行第一个genetator对象，该genetator对象执行完毕就执行下一个generator独享，以此类推。

- 把yield next功能完成，执行yield next的时候就跳转中间件，大胆猜测一下这里next其实是下一个中间件的引用，那么如何将他们两者建立联系，参考koa的做法：

```
var prev = null
var i = gensFn.length
while (i--) {
    prev = gensFn[i].call(null, prev)
}
```
这样处理之后，每个generator对象生成时的next参数实际上就是其下一个generator对象。

相应的这时每个generator对象next之后的返回值有三种情况：

（1）value值是一个promise对象，则继续执行next

（2）value值是一个generator对象，则执行co，处理下一个genetator对象的逻辑

（3）done为true，表明此generator对象已经执行完毕，需要回滚执行上一个

这里面的难点在于如何实现回滚执行上一个generator对象的未完成逻辑，解决方案是利用Promise。co函数返回一个promise对象，当前generator对象的逻辑完成时resolve使上一个generator对象接到通知继续next操作。

```
// generator函数集合
var gens = []

function use(gen) {
    gens.push(gen)
}

function co(gen) {
    return new Promise(function(resolve) {
        function next(data) {
            var result = gen.next(data)
            var value = result.value
            // 对应情况（3）
            if (result.done) {
                resolve()
            // 对应情况（2）
            } else if (typeof value.next === 'function') {
                co(value).then(function() {
                    next()
                })
            // 对应情况（1）
            } else if (typeof value.then === 'function') {
                value.then(function(data) {
                    next(data)
                })
            }
        }
        next()
    })
}

function triger() {
    var prev = null
    var i = gens.length
    // 后一个generator对象挂在到next
    while (i--) {
        prev = gens[i].call(null, prev)
    }
    co(prev)
}
```

测试一下

```
use(function *aaa(next) {
    var aaa = yield new Promise(function(resolve) {
        setTimeout(function() {
            resolve(111)
        }, 1000)
    })

    yield next 

    console.log(aaa)
})

use(function *bbb(next) {
    var bbb = yield new Promise(function(resolve) {
        setTimeout(function() {
            resolve(222)
        }, 1000)
    })

    console.log(bbb)
})

triger()
```

两秒后输出

```
222
111
```

参考：http://www.cnblogs.com/Leo_wl/p/4684633.html






















