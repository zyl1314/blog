# 模块加载器（CMD规范）
本文的目标是实现一个遵循CMD规范的模块加载器，类似与seajs

## seajs
我们先看一下seajs的用法

```
//index.html
<html>
    <head>
        <script src="sea.js"></script>
        <script>
            seajs.use(['a.js', 'b.js'], function(a, b) {
                // do something
            })
        </script>
    </head>
    <body>
    </body>
</html>


//a.js
define(function(require, exports, module) {
    var c = require('c.js')
    exports.key = c.key
})

//b.js 
define(function(require, exports, module) {
    exports.key = value
})

//c.js 
define(function(require, exports, module) {
    exports.key = value
})
```

## 代码

```
(function(g, doc){
    var fakeSea = {
        // 入口函数
        use: use,
        // 模块列表
        modules: {},
        // 配置项
        config: {
            root: '/'
        }
    }

    // 模块
    function Module(id) {
        this.id = id
        this.deps = []
        this.status = 'PENDING'
        this.callbacks = []
        this.factory = null
        this.exports = {}
        this.load()
        fakeSea.modules[id] = this
    }

    Module.prototype = {
        // 插入模块
        load: function() {
            var src = fakeSea.config.root + this.id
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

    // 处理依赖
    function define(factory) {
        var id = document.currentScript.src.slice(fakeSea.config.root.length), 
            module = fakeSea.modules[id],
            deps
        module.factory = factory
        deps = module.deps = getDependcencs(factory)
        // 没有依赖
        if (deps.length === 0) {
            module.status = 'COMPLETED'
            module.fire('complete')
        } else {
            Promise.all(deps.map(function(id) {
                return load(id)
            })).then(function() {
                module.status = 'COMPLETED'
                module.fire('complete')                
            })
        }
    }

    // 获取模块输出
    function require(id) {
        var module = fakeSea.modules[id]
        return getModuleExports(module)
    }

    function getModuleExports(module) {
        if (!Object.keys(module.exports).length) {
            module.factory(require, module.exports, module)
        }
        return module.exports
    }

    // 入口函数
    function use(deps, fn) {
        Array.isArray(deps) ? deps : (deps = [deps])
        Promise.all(deps.map(function(id) {
            return load(id)
        })).then(function(list) {
            fn.apply(null, list)
        })
    }

    // 加载模块
    function load(id) {
        var module
        // 模块已经存在
        if ((module = fakeSea.modules[id]) && module.status == 'COMPLETED') {
            return new Promise(function(resolve, reject) {
                resolve(getModuleExports(module))
            })
        } else if ((module = fakeSea.modules[id]) && module.status !== 'COMPLETED'){
            return new Prmise(function(resolve, reject) {
                module.on('complete', function() {
                    resolve(getModuleExports(module))
                })
            })
        } else {
            module = new Module(id)
            return new Promise(function(resolve, reject) {
                module.on('complete', function() {
                    resolve(getModuleExports(module))
                })
            })            
        }
    }

    // 解析依赖
    function getDependcencs(factory) {
        var deps
        factory = factory.toString()
        deps = factory.match(/require\(.+?\)/g) || []
        deps = deps.map(function (dep) {
            return dep.replace(/^require\(["']|["']\)$/g, "")
        })
        return deps
    }    

    // 对外接口
    g.fakeSea = fakeSea
    g.require = require
    g.define = define
}(window, document))
```

#### 原理和前文fakeRequire是类似的，只不过AMD规范事先并不给出依赖，需要先手动解析一遍。