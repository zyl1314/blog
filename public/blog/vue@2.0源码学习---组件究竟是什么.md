# vue@2.0源码学习---组件究竟是什么  

本篇文章从最简单的情况入手，不考虑prop和组件间通信。  

## Vue.component  

vue文档告诉我们可以使用`Vue.component(tagName, options)`注册一个组件  

```
Vue.component('my-component', {
  // 选项
})
```
毫无疑问这是一个全局API，我们顺着代码最终可以找到`Vue.component`是这样的  

```
Vue.component = function(id, definition) {
	definition.name = definition.name || id
	definition = Vue.extend(definition)
    this.options[type + 's'][id] = definition
    return definition	
}
```
`Vue.component`实际上是`Vue.extend`的封装，`Vue.extend`如下：

```
Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {}
  const Super = this
  const isFirstExtend = Super.cid === 0
  if (isFirstExtend && extendOptions._Ctor) {
    return extendOptions._Ctor
  }
  let name = extendOptions.name || Super.options.name
  const Sub = function VueComponent (options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super
  // allow further extension
  Sub.extend = Super.extend
  // create asset registers, so extended classes
  // can have their private assets too.
  config._assetTypes.forEach(function (type) {
    Sub[type] = Super[type]
  })
  // enable recursive self-lookup
  if (name) {
    Sub.options.components[name] = Sub
  }
  // keep a reference to the super options at extension time.
  // later at instantiation we can check if Super's options have
  // been updated.
  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  // cache constructor
  if (isFirstExtend) {
    extendOptions._Ctor = Sub
  }
  return Sub
}
```
可以看到`Vue.extend`返回的实际上是一个构造函数`Sub`，并且此构造函数继承自Vue。里面有这么几行代码  

```
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
```
那么Super.options（即`Vue.options`）是什么呢？

```
  Vue.options = Object.create(null)
  //  包含components directives filters
  config._assetTypes.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  util.extend(Vue.options.components, builtInComponents)
```
Vue.options事实上存放了系统以及用户定义的component、directive、filter，`builtInComponents`为Vue内置的组件（如`keep-alive`），打印看下：

![image](https://github.com/zyl1314/blog/raw/master/public/img/vue/step3/1.png)

所以Sub构造函数的options不仅包含components、directives、filters，还包含预先定义的实例化时所需的选项。定义一个组件如下：

```
let MyComponent = Vue.extend({
    data() {
        return {
            msg: "this's compoennt"
        }
    },
    render(h) {
        return h('p', this.msg)
    }
})
```

打印MyComponent.options如下：

![image](https://github.com/zyl1314/blog/raw/master/public/img/vue/step3/2.png)

再回过头看`Vue.component`，可以发现他做的工作就是扩展一个Vue构造函数（`VueComponent`），并将这个构造函数（`VueComponent`）添加到Vue.options.components  


现在我们已经可以回答最开始的问题---vue的组件是什么？vue的组件其实就是扩展的Vue构造函数，并且在适当的时候实例化为Vue实例。 

## 组件对应的vnode

组件对应的vnode是什么样子？从一个简单的例子入手：

```
let MyComponent = Vue.component('my-component', {
    data() {
        return {
            msg: "this's component"
        }
    },
    render(h) {
        return h('p', this.msg)
    }
})

window.app = new Vue({
    render(h) {
        return h('my-component')
    }
}).$mount('#root')
```

上篇文章已经说道在initRender的时候会初始一个系统watcher，如下：

```
vm._watcher = new Watcher(vm, () => {
  vm._update(vm._render(), hydrating)
}, noop)
```
上篇文章提到`vm._render()`返回的是一个虚拟dom（`vnode`），具体到本篇，那么组件标签会被解析成什么样的虚拟节点呢？  

事实上render的时候会首先调用`createElement`，根据传入的tag（html标签或者组件标签）不同，vnode可以分为以下两种：

- platform built-in elements  

	这种就是普通的html标签（p、div、span等）对应的vnode

- component
	
	当tag是组件标签的时候，会调用`createComponent`，如下： 

```
    else if ((Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      
      return createComponent(Ctor, data, context, children, tag)
	}
```
这里的Ctor就是我们扩展的组件构造函数，`createComponent`最终返回的vnode如下：

```
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children }
  )
```  

![image](https://github.com/zyl1314/blog/raw/master/public/img/vue/step3/3.png)

需要注意的是data有一个操作：

```
  // merge component management hooks onto the placeholder node
  mergeHooks(data)
```
merge之后`data.hook`会添加四个方法：

 - init       实例化组件时调用
 - prepatch   patch之前调用
 - insert     真实节点插入时调用
 - destory    组件实例销毁时调用

## 实例化组件

前文看到组件构造函数实际上是存在组件对应vnode的componentOptions中，那么究竟是什么时候实例化组件呢？  

顺着vm._update(vm._render(), hydrating)往下看发现最终调用的是patch操作，而对于组件实例化而言并不存在与之对应的oldVnode（因为oldVnode是在组件更新后产生的），所以最终的逻辑归到`根据组件对应的vnode创建真实dom节点`，即

```
createElm(vnode, insertedVnodeQueue)
```
我们还记得组件的构造函数是`vnode.componentOptions.Ctor`，其实最终调用的也是这个构造函数。  

createElm函数中与组件初始化相关的关键代码如下： 

```
    const data = vnode.data
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.init)) i(vnode)

      if (isDef(i = vnode.child)) {
        initComponent(vnode, insertedVnodeQueue)
        return vnode.elm
      }
    }
```
init的代码如下： 

```
function init (vnode: VNodeWithData, hydrating: boolean) {
  if (!vnode.child || vnode.child._isDestroyed) {
    const child = vnode.child = createComponentInstanceForVnode(vnode, activeInstance)
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
  }
}


export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any // activeInstance in lifecycle state
): Component {
  const vnodeComponentOptions = vnode.componentOptions
  const options: InternalComponentOptions = {
    _isComponent: true,
    parent,
    propsData: vnodeComponentOptions.propsData,
    _componentTag: vnodeComponentOptions.tag,
    _parentVnode: vnode,
    _parentListeners: vnodeComponentOptions.listeners,
    _renderChildren: vnodeComponentOptions.children
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (inlineTemplate) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }

  return new vnodeComponentOptions.Ctor(options)
}
```
经过init之后可以看到组件`vnode.child`对应的就是组件的实例，且`child.$el`即为组件对应的真实dom，但是实际上createElm返回的是vnode.elm，怎么回事？事实上`initComponent`
中做了处理

```
vnode.elm = vnode.child.$el
```
综上，组件实例化是在由虚拟dom映射为真实dom时完成的。


---

写到这里已经对组件机制有了初步的认识，数据的传递、父子组件通信本文并没有涉及，留到以后再看。















