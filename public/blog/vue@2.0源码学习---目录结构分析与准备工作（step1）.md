## 前言
网上vue的源码分析也蛮多的，不过很多都是1.0版本的并且大多都是在讲数据的observe，索性自己看看源码，虽然很难但是希望能学到点东西。
>源码版本为2.0.0

## runtime和runtime-with-compiler
有必要了解这两个概念的区别。我们写vue程序的时候一般会给出template，但是仔细看过文档的话一定知道vue支持render函数的写法。runtime版本可直接执行render函数写法，假如是template写法，需要先利用compiler解析模板至render函数，再执行render函数渲染。  

为了学习起来简单，选择先从runtime版本入手，事实上两者相差的只是一个`将模板字符串编译成为 JavaScript 渲染函数`的过程。

## 入口文件  
上面已经说过从runtime版本入手，所以首要任务就是找到runtime版本的入口文件。  

![img](https://github.com/zyl1314/blog/raw/master/public/img/vue/step1/1.PNG)  

点开entries目录下的web-runtime.js，发现确实是导出了一个Vue构造函数。先写一个例子跑起来试试

```
import Vue from '../src/entries/web-runtime.js'

window.app = new Vue({
    data: {
        msg: 'hello',
    },
    render (h) {
      return h('p', this.msg)
    }
}).$mount('#root')

```
简单的写个webpack配置文件

```
'use strict'
const path = require('path')
const webpack = require('webpack')

module.exports = {
  watch: true,
  entry: {
    app: './index.js'
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, 'dist')
  },
  resolve: {
    extensions: ['.js'],
    alias: {
        vue: path.resolve(__dirname, '../src/entries/web-runtime-with-compiler'),
        compiler: path.resolve(__dirname, '../src/compiler'),
        core: path.resolve(__dirname, '../src/core'),
        shared: path.resolve(__dirname, '../src/shared'),
        web: path.resolve(__dirname, '../src/platforms/web'),
        server: path.resolve(__dirname, '../src/server'),
        entries: path.resolve(__dirname, '../src/entries'),
        sfc: path.resolve(__dirname, '../src/sfc')
    }
  },
  module: {
    rules: [
        { test: /\.js$/, loader: "babel-loader", include: [path.resolve(__dirname, '../src')] }
    ]
  }
}
```
> 注意配置一下extensions和alias，由于vue的源码编写时添加了类型检测（flow.js），所以需要在babel中添加插件`transform-flow-strip-types`

webpack打包一下打开浏览器，hello映入眼帘。  
至此，准备工作结束

## 扩展全局方法和实例方法

点开web-runtime.js可以发现Vue是从core/index.js导入的，代码如下：

```
import config from './config'
import { initGlobalAPI } from './global-api/index'
import Vue from './instance/index'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: () => config._isServer
})

Vue.version = '2.0.0'

export default Vue
```

又发现Vue是从instance/index.js导入的，代码如下：

```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

到此时，我们算是真正的看到了Vue的构造函数，构造函数做了两件事：

- 强制我们使用new关键字实例化
- 实例的初始化操作  

定义了构造函数之后，执行了五个函数，这五个函数扩展了Vue
的原型，也即定义了一些实例方法。

```
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this

...
```


再回过头看`initGlobalAPI(Vue)`，它给Vue构造函数扩展了一些方法

```
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      util.warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)
  Vue.util = util
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = util.nextTick

  Vue.options = Object.create(null)
  config._assetTypes.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  util.extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

```
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    Vue.options = mergeOptions(Vue.options, mixin)
  }
}
```

具体的逻辑后文看。