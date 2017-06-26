# VUE源码学习一
## 初始化项目
首先新建一个目录，初始化我们的项目
```
npm init
```
目录设置如下
```
|- dist               // 打包后的文件
|- example            // 例子
|- node_modules       // 依赖的模块
|- src                // 这里面是我们的代码
 - .babelrc           // 转码规则
 - .gitignore         // 长传忽略文件
 - .package.json      // 项目说明文件
 - webpack.config.js  // webpack配置文件
 - README.md     
```
下面简要看一下webpack.config.js文件
```
var CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = {
    watch: true,
    // 定义了入口文件
    entry: {
        index: ['./src/index.js'],
        example: ['./example/index.js']
    },
    // 定义了打包后输出的文件夹 以及文件的名字
    output: {
        path: __dirname + '/dist',
        filename: "[name].js"
    },
    module: {
        loaders: [
            // .js文件需要经过babel转码
            // 此项目会用到es6的语法
            {
                test: /\.js$/,
                exclude: /node_modules/,
                loader: 'babel'
            }
        ]
    },
    plugins: [
        // 将example文件夹下的index.html文件拷贝到dist目录
        new CopyWebpackPlugin([
            {from: './example/index.html'}
        ], {})
    ]

};
```
看一下package.json
```
"scripts": {
    // 运行npm run build则自动执行webpack打包
    "build": "webpack",
    // dist文件下的文件发生改变浏览器会自动刷新，即我们打包后会自动刷新浏览器
    "watch": "browser-sync start --server --files 'dist/*.*'"
}
```
由于我们的关注点不在于以上，所以草草带过。

## 监听数据

数据监听是VUE的核心，了解过VUE的人大概都知道它是利用了Object.defineProperty做到数据监听的。  
首先明确一下我们需要解决的两个难点：
- 数据的深层监听
- 对数组对象的监听

首先看一下Object.defineProperty的用法：
```
let obj = { a: 1, b: 2 }
Object.keys(obj).forEach(key => {
    let val = obj[key]
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get() {
            console.log(`${key}的值是${val}`)
        },
        set(newVal) {
            console.log(`${key}的新值是${newVal}`)
            val = newVal
        }        
    })
})
```
![image](https://github.com/zyl1314/blog/raw/master/public/img/fake-vue-img/1.gif)
可以很显然的看出来，Object.defineProperty主要的作用就是数据劫持，相当于给我们提供了一个在获取数据之前改变数据的能力。其实上面我们已经完成了对简单数据的监听，但是通常情况下我们的逻辑不可能是像上例那么简单，不可能就是在获取或者重置数据时简单的打印出来。就比如VUE,在重置数据时所要做的工作很可能是重新渲染视图，这时候很自然可以想到发布订阅模式，首先订阅行为，在合适的时机触发行为（比如说set的时候）。
```
let obj = { a: 1, b: 2 }

let proxy = {
    _cbs: {},
    // 订阅事件
    $on(ev, fn) {
        this._cbs[ev] || (this._cbs[ev] = [])
        this._cbs[ev].push(fn)
    },
    // 触发事件
    $emit(ev) {
        this._cbs[ev].forEach(fn => fn())
    }
}

proxy.$on('change', () => {
    console.log('rerender...')
})


Object.keys(obj).forEach(key => {
    let val = obj[key]
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get() {
            console.log(`${key}的值是${val}`)
        },
        set(newVal) {
            val = newVal
            proxy.$emit('change')
        }        
    })
})
```
![image](https://github.com/zyl1314/blog/raw/master/public/img/fake-vue-img/2.gif)
我们已经完成了简单数据的监听以及自定义触发逻辑，下面考虑复杂数据：  
第一种复杂数据是深层次对象，这个问题我们很容易想到用递归解决。
```
let obj = {
    a: 1,
    b: {
        c: 3
    }
}
```
比如上例，我们先监听obj，然后再监听obj.b  

第二种复杂数据是数组：
```
let obj = {
    a: [1, 2, 3]
}
```
假如我们做如下的操作
```
obj.a.push(4)
```
毫无疑问数据是发生了改变的，但是按照我们上面的写法是检测不到变化的，所以需要我们针对数组做一些额外的工作。  

首先可以想到的解决方案是将全局数组的方法重写，但是这样是有问题的，我们的首要原则就是不能污染全局，否则我们在vue实例外使用数组岂不是麻烦大了。然后我们可能想到只重写vue实例内数组的方法，那么问题来了，怎么重写？是自己写push、pop这些方法吗？肯定不能，暂且不说性能问题，自己重写能否做到和原生的一致都不能确定。我们可以这么做
```
const aryMethods = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']
const arrayAugmentations = []
aryMethods.forEach((method) => {
    let original = Array.prototype[method]
    arrayAugmentations[method] = function() {
        console.log('changed...')
        return original.apply(this, arguments)
    }
})

let arr = [1, 2]
arr.__proto__ = arrayAugmentations
```
![image](https://github.com/zyl1314/blog/raw/master/public/img/fake-vue-img/3.gif)
我们做的就是将目标数组的原型改写，插入一些自定义的逻辑，但是核心工作还是利用原生的数组方法完成的。  

至此我们已经了解了完成数据监听的基础，详情看代码。
