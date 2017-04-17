# promise
#### promise中文译为承诺，是一种异步编程的方式，在ES6中已经作为标准实现
#### promise的具体用法本文不在赘述，可参考http://es6.ruanyifeng.com/#docs/promise

## - 目标
#### 主要实现两个目标：1、链式调用 2、支持串行

```
//链式调用
new Promise(function(resolve, reject) {
    setTimeout(function() {
        resolve(1)
    }, 1000)
})
.then(function(val) {
    return val + 1
})
.then(function(val) {
    console.log(val)  //2
})
```

```
//串行
new Promise(function(resolve, reject) {
    setTimeout(function() {
        resolve(1)
    }, 1000)
})
.then(function(val) {
    console.log(val)  //1
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            resolve(val + 1)
        }, 1000)
    })
})
.then(function(val) {
    console.log(val)   //2 
})
```

## - 构建
### 直接看代码吧,很短只有五十行

```
function FakePromise(fn) {
    this.state = 'pending'
    this.doneList = []
    this.failList = []
    fn(this.resolve.bind(this), this.reject.bind(this))
}

FakePromise.prototype = {
    then: function(done, fail) {
        this.doneList.push(done)
        this.failList.push(fail || null)
        return this
    },

    resolve: function(val) {
        var This = this
        this.state = 'fulfilled'
        setTimeout(function() {
            while (fn = This.doneList.shift()) {
                This.failList.shift()
                val = fn(val)
                if (val instanceof FakePromise) {
                    val.doneList = This.doneList
                    val.failList = This.failList 
                    break
                }
            }
        }, 0)
    },

    reject: function(val) {
        var This = this 
        this.state = 'rejected'
        setTimeout(function() {
            val = This.failList.shift()(val) 
            This.doneList.shift()
            if (val instanceof FakePromise) {
                val.doneList = This.doneList
                val.failList = This.failList
            } else {
                while (fn = This.doneList.shift()) {
                    This.failList.shift()
                    val = fn(val)
                    if (val instanceof FakePromise) {
                        val.doneList = This.doneList
                        val.failList = This.failList 
                        break
                    }
                }
            }
        }, 0)
    }
}
```
### 说一下我的思路，很明显fakePromise是一个构造函数，他有三个实例方法then、resolve、reject。
#### 1. then方法

```
    then: function(done) {
        this.doneList.push(done)
        return this
    }
```
then的职责是把回调函数压入任务队列中，返回this是为了满足第一个目标：支持链式调用

#### 2. resolve(链式调用)

```
    resolve: function(val) {
        var This = this
        this.state = 'fulfilled'
        //setTimeout的作用是为了防止同步执行导致的回调为undefined
        setTimeout(function() {
            while (fn = This.doneList.shift()) {
                This.failList.shift()
                val = fn(val)
            }
        }, 0)
    }
```
链式调用实现起来比较简单，只需要将上个回调的输入作为下个回调的输入即可

#### 3. resolve(串行)

```
    resolve: function(val) {
        var This = this
        this.state = 'fulfilled'
        setTimeout(function() {
            while (fn = This.doneList.shift()) {
                val = fn(val)
                if (val instanceof FakePromise) {
                    val.doneList = This.doneList
                    break
                }
            }
        }, 0)
    }
```
前文中为了支持链式调用，then函数返回的都是this，但是一旦resolve的返回是fakePromise的实例时就不符合我们想要的了。我的解决思路是在运行回调时判断回调返回值的类型，
如果返回值是fakePromise的实例（newFP）那么就把doneList剩余未处理的任务添加至newFP的任务队列中，同时中断oldFP的迭代。

#### 4. reject
reject和resolve做的事情是类似的，首先判断reject的返回值，如果不是fakePromise的实例则继续执行doneList的任务，否则将未执行的doneList和failList
添加至newFP的任务队列。

#### 参考：http://www.jianshu.com/p/473cd754311f