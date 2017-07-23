# 计算属性

## 回顾
在上一节我们完成了普通属性的响应式绑定，我们先来回顾一下整个流程是怎样的。

1. 第一步 监听数据
```
this.observer = this.observer.create(this.$data)
```
这里面完成了一个很重要的功能就是消息的传递，什么意思呢？
```
假如我们的数据是这样的
this.$data = {
    person: {
        sex: 'man',
        name: {
            first: 'zhang',
            last: 'yalong'
        }
    }
}

监测数据
this.observer = this.observer.create(this.$data)
this.observer.on('set', this._updateBindingAt.bind(this))

改变数据
this.$data.person.name.first = 'wang'
```
这时候实例的_updateBindingAt就会被触发，且会被传入path参数，path指的是改变属性的路径，在本例中即为
```
person.name.first
```
这样就实现了消息的传递，这是后面响应式绑定的基础 

那么_updateBindingAt函数是干什么的？他的作用是找到相应路径的watcher，然后执行watcher的更新函数，watcher我们下面看

2. 解析模板（根据directive绑定相应的watcher）
```
// 解析到指令就调用_bindDirective创建Directive的实例
// 我们前面实现了{{}}指令 下面以{{person.sex}}为例
// val即为person.sex  el为dom节点
this._bindDirective('text', value, el)
```
看一下Directive构造函数
```
function Directive(name, expression, el, vm) {
    ...
    this._bind()
}


Directive.prototype._bind = function() {
    this._watcher = new Watcher(
        this.vm,
        this.expression,
        // 这里的_update是根据指令的不同对应的更新函数
        this._update,
        this
    )
}
```
可以看到创建Directive实例时新建了Watcher实例
```
function Watcher(vm, expression, cb, ctx) {
    ...
    this.cb = cb
    this.initDeps(expression)
}

Watcher.prototype.initDeps = function(path) {
    let vm = this.vm 
    let binding = vm._createBindingAt(path)
    binding._addSub(this)
}
```
关键点在这个_createBindingAt函数，它的返回值是相应属性路径的订阅队列，然后将当前watcher添加到订阅队列中 

看这个例子
```
data: {
    person: {
        sex: 'man',
        name: {
            first: 'zhang',
            last: 'yalong'
        }
    }    
}


<div>
    <p>{{person.sex}}</p>
    <p>{{person.name.last}}</p>
</div>
```
我们得到的binding结构是这样的

可以看到person.sex 和 person.name.last 的_subs有一个watcher实例 

现在我们在联系前面的数据监测，当一个属性变化时，顶级observer实例可以监听到属性路径，然后我们再根据属性路径解析出binding实例，如果binding实例的_subs不为空，那么就依次调用subs（即watcher）的更新函数，从而实现数据的响应式绑定

## 计算属性
根据前面的分析我们可以知道，响应式数据绑定是先查找binding然后调用其_subs的更新函数做到的 

回到计算属性
```
<p>个人信息：{{info}}</p>

computed: {
    info() {
        return `性别：${this.person.sex}, 名字：${this.person.name.first} ${this.person.name.last}`
    }
}
```
联系上面的内容想一下，我们所需要做的是将计算属性的watcher添加到依赖属性binding的_subs下面，这样依赖属性变化了就会自动执行计算属性watcher的update函数，从而更新视图。 


我们依然利用消息传递来实现，具体的做法是首先监听顶级observer对象的get事件
```
this.observer.on('get', this._collectDep.bind(this))
```
_collectDep函数的作用是向path对应binding的_subs添加watcher 

然后对于计算属性，我们可以将他对应的取值函数设置为其get函数，并且在新建计算属性对应的watcher实例时执行一次其取值函数，由此便可以触发顶级observer对象的get事件，并且能够获得依赖属性的路径，从而可以将计算属性的watcher添加至依赖属性对应binding的_subs中，
那么依赖属性变化时就可以自动执行计算属性对应watcher的update函数。


