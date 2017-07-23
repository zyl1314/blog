## 目标
实现类似与
```
<p>{{name}}</p>
```
写法的模板替换与自动更新 

## 全局渲染
我们在上一篇已经完成了对数据的监听并且实现一个很重要的功能，就是消息的传递，什么意思？
```
let obj = {
    a: {
        b: 1
    }
}

let observer = Observer.create(obj)
observer.on('set', function(ev, path, val) {
    ev     //  事件（可以死set get $delete $add等）
    path   //  属性路径  
    val    //  新值
})

obj.a.b = 2
```
那么ev => get，path => a.b， val => 2  

也就说根对象下面的子对象只要发生了改变那么跟对象便可以接受到消息（set），并且可以得到具体改变属性的路径。  

现在我们再想一想我们的目标是什么，其是就是利用正则把属性路径提取出来并替换为具体的属性值，那么怎么实现自动更新呢？可以给顶级observer的set事件添加一个render回调
```
observer.on('set', function(ev, path, val) {
    reRender(template)
})
```
只要数据发生了改变就重新解析一次模板并且重新渲染

## 局部渲染
上面可以工作，但是仔细想想好像不大对劲，我们只改变了一个数据为什么要重新解析模板？不是只应该重新渲染与改变数据相关的地方吗？
```
let obj = {
    a: 1,
    b: 2,
    c: {
        d: 3
    }
}

let template = `<p>{{a}}</p>
                <p>{{b}}</p>
                <p>{{c.d}}</p>`

obj.a = 4
```
理想的效果应该只将第一个p标签的内容换成4即可，而不是重新解析整个模板，那样效率太低  

上文说过我们的数据监听实现了消息的传递，我们可以在解析模板的时候给属性路径绑定更新函数（watcher） 
```
let _cbs = {
    'a': function(val) {
        node.nodeValue = val   // node代表文本节点
    },
    'b': function(val) {}，
    'c.d': function(val) {}
}
```
这样一来只要数据发生了改变我们就可以执行对应属性路径的更新函数，不用重新解析模板只更新发生改变的数据
