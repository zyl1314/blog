# 模块加载器（AMD规范）
本文的目标是实现一个遵循AMD规范的模块加载器，类似与requirejs

## requirejs
我们先看一下requirejs的用法

```
//index.html
<html>
    <head>
        <script src="require.js" data-main="index.js"></script>
    </head>
    <body>
    </body>
</html>


//index.js
define(['module.js'], function(module) {
    // do something
})

//module.js 
define(function() {
    // do something
})
```
requirejs遵循AMD规范，模块是由define定义，define的第一个参数是依赖项（数组），当然也可以没有依赖。主模块是在引入requirejs的时候通过data-main属性指明的。

## 代码

```
(function(g, doc){
    var fakeRequire = {
        config: {
            root: '/'
        },
        modules: {}
    }

    // 模块
    function Module(id) {
        this.id = id
        this.deps = []
        this.callbacks = {}
        this.status = 'PENDING'
        this.factory = null
        this.exports = null
        this.load()
        fakeRequire.modules[id] = this
    }

    Module.prototype = {
        // 插入模块
        load: function() {
            var src = fakeRequire.config.root + this.id
            var script = document.createElement('script')
            script.src = src 
            doc.head.appendChild(script)
            this.status = 'LOADING'
        },
        // 订阅
        on: function(ev, fn) {
            this.callbacks[ev] || (this.callbacks[ev] = [])
            this.callbacks[ev].push(fn)
        },
        // 触发
        fire: function(ev) {
            this.callbacks[ev].forEach(function(fn) {
                fn()
            })
        }
    }

    // 寻找主模块
    function main() {
        var id = doc.querySelector('script[data-main]').getAttribute('data-main')
        load(id)
    }
    setTimeout(function() {
        main()
    }, 0)

    // 载入模块
    function load(id) {
        var module
        // 模块已经存在
        if ((module = fakeRequire.modules[id]) && module.status == 'COMPLETED') {
            return new Promise(function(resolve, reject) {
                resolve(processModuleExports(module))
            })
        } else if ((module = fakeRequire.modules[id]) && module.status !== 'COMPLETED'){
            return new Promise(function(resolve, reject) {
                module.on('complete', function() {
                    resolve(processModuleExports(module))
                })
            })
        } else {
            module = new Module(id)
            return new Promise(function(resolve, reject) {
                module.on('complete', function() {
                    resolve(processModuleExports(module))
                })
            })            
        }        
    }

    function define(deps, factory) {
        var id = doc.currentScript.src.slice(fakeRequire.config.root.length)
        var module = fakeRequire.modules[id]
        if (Array.isArray(deps)) {
            module.deps = deps 
            module.factory = factory
            Promise.all(deps.map(function(id) {
                return load(id)
            })).then(function() {
                module.fire('complete')
                module.status = 'COMPLETED'
            })           
        } else {
            module.factory = deps
            module.fire('complete')
            module.status = 'COMPLETED'
        }
    }

    function processModuleExports(module) {
        var deps = module.deps
        var temp = deps.map(function(id) {
            return fakeRequire.modules[id].exports
        })
        fakeRequire.modules[module.id].exports = module.factory.apply(null, temp)
    }

    // 对外接口
    g.fakeRequire = fakeRequire
    g.define = define
}(window, document))
```

## 标识模块

如何标识模块且具有唯一性？很容易想到利用模块的路径，文中也是这么做的

## 解析模块依赖

解析模块依赖是模块加载器的核心所在。这里的解决方案是Promise + 发布订阅

首先每个模块需要订阅一个 complete 事件
```
module.on('complete', fn)

```
当模块加载完成是触发事件
```
module.fire('complete')
```
问题的关键是一个模块怎么知道它依赖的所有模块已经加载完成了？

```
// 首先我们分两种可能考虑
// 1、模块没有依赖

define(function(){}) 

这种情况很简单，直接触发完成即可

define(function(){
    module.fire('complete')
})

// 2、模块有依赖

define([deps], function(){})

利用Promise，模块的依赖项加载完成后模块触发完成函数

Promise.all([加载依赖]).then(function() {
    module.fire('complete')
})
```

## 如何避免重复加载

什么意思？指的是模块依赖a和b，b又依赖a，那么只需要加载一次a即可，然后缓存起来

给一个模块定义三个状态：

```
PENDING ：初始状态，模块已经穿建
LODING  : 已经将模块插入到文档
COMPLETE: 模块依赖已经解析完毕，模块已经存入模块池
```
当我们处理依赖的时候，也要分情况：

```
// 1、模块池不存在模块，那么就新建模块

var module = new Module

// 2、模块池已经存在模块，但是模块的状态没有完成(!==COMPLETE)，需要等待模块加载完成

new Promise(function(resolve, reject) {
    module.on('complete', function() {
        resolve()
    })
})

// 3、模块池已经存在模块，模块的状态已完成(===COMPLETE)，直接通知模块已完成

new Promise(function(resolve, reject) {
    resolve()
})

```