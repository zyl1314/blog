# vue@2.0源码学习---从hello world学习vue的内部做了什么
>源码版本为2.0.0

接[前文](https://github.com/zyl1314/blog/issues/1)。

前文讲到下面五个函数扩展了Vue的原型

```
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```
我画了一个图，是执行这几个mixin之后，Vue原型挂载的方法

![image](https://github.com/zyl1314/blog/raw/master/public/img/vue/step2/1.png)

## 一个简单的例子

```
window.app = new Vue({
    data: {
        msg: 'hello world',
    },
    render (h) { 
      return h('p', this.msg)
    }
}).$mount('#root')

setTimeout(() => {
    app.msg = 'hi world'
}, 2000)

```
毫无疑问屏幕上会先渲染hello world，隔两秒后变为hi world。  
本文将从这个简单的例子追本溯源，看看Vue究竟做了什么。

## init
我们沿着执行顺序一步一步的看，上文已经找到了Vue的构造函数如下：

```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```
所以执行`new Vue()`的时候，实例(vm)会首先执行初始化方法vm._init()，_init方法如下：

```
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++
    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      // console.log(resolveConstructorOptions(vm))
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    callHook(vm, 'beforeCreate')
    initState(vm)
    callHook(vm, 'created')
    initRender(vm)
  }
```
>由于本文是初步探索Vue，所以并没有涉及到`组件`这个概念，但是我拷贝过来的代码中会经常出现与组件逻辑相关的代码，直接略过即可。 

执行初始化操作首先给实例添加了几个私有属性，然后merge了options，vm.$options最终变为这样

```
vm.$options = {
	components: [..],
	directives: [],
	filters: [],
	vm: vm,
	data: {},
	render: function() {}
}
```
真正重要的操作是下面的几个init函数

```
initLifecycle(vm)    初始化生命周期
initEvents(vm)    初始化事件系统（这里面做的是父子组件通信的工作，所以这篇文章暂时略过）
callHook(vm, 'beforeCreate')    执行beforeCreate钩子
initState(vm)    初始化状态（包括data、computed、methods、watch）
callHook(vm, 'created')    执行created钩子
initRender(vm)    渲染页面	   
```
从上面可以看到，created钩子执行的时机是在数据被observe之后（此时数据还没有收集依赖）。看一下callHook函数：

```
export function callHook (vm: Component, hook: string) {
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      handlers[i].call(vm)
    }
  }
  vm.$emit('hook:' + hook)
}
```
handle中的this绑定了vm  

下面依次分析几个初始化函数做的工作
## initLifecycle

```
export function initLifecycle (vm: Component) {
  const options = vm.$options
  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
这里没什么好说的，vm._watcher和vm._isMounted后面会用到 
## initEvents 

这里做的是父子组件通信的相关工作，不在本篇的讨论范围内。

## initState

```
export function initState (vm: Component) {
  vm._watchers = []
  initProps(vm)
  initData(vm)
  initComputed(vm)
  initMethods(vm)
  initWatch(vm)
}
```
initProps也是组件相关，因此剩下四个是我们关心的。核心initData完成了数据的observe

#### 1） initData  
```
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? data.call(vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object.',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  let i = keys.length
  while (i--) {
	// data中的字段不能和props中的重复
    if (props && hasOwn(props, keys[i])) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${keys[i]}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else {
	  // 代理
      proxy(vm, keys[i])
    }
  }
  // observe data
  observe(data)
  data.__ob__ && data.__ob__.vmCount++
}
```

首先代理data里面的字段：  

在vue中通常这样访问一个值

```
this.msg

而不是

this._data.msg
```
正是因为`proxy(vm, keys[i])`已经对key值做了代理，如下：

```
function proxy (vm: Component, key: string) {
  if (!isReserved(key)) {
    Object.defineProperty(vm, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter () {
		// 访问vm[key]返回的事实上是vm._data[key]
        return vm._data[key]
      },
      set: function proxySetter (val) {
		// 设置vm[key]事实上给vm._data[key]赋值
        vm._data[key] = val
      }
    })
  }
}
```
接下来就是对数据observe（本文暂不考虑数组），数据的observe可以说是Vue的核心，网上很多文章已经介绍的十分详细，这里我把observe简化一下如下：

```
export function observe (value) {
  if (!isObject(value)) {
    return
  }
  let ob = new Observer(value)
  return ob
}

export class Observer {
  constructor (value) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)

    this.walk(value)
  }

  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }
}

export function defineReactive (obj, key, val) {
  const dep = new Dep()
  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
	// 取值时给数据添加依赖
    get: function reactiveGetter () {
      const value = val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
      }
      return value
    },
	// 赋值时通知数据依赖更新
    set: function reactiveSetter (newVal) {
      const value = val
      if (newVal === value) {
        return
      }
	  val = newVal
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
```
整个响应式系统的核心在于defineReactive这个函数，利用了一个闭包把数据的依赖收集起来，下文我们会看到Dep.target事实上是一个个watcher。

这里有个需要注意的地方：

```
if (childOb) {
   childOb.dep.depend()
}
```
为什么闭包里的dep已经收集过了依赖，这里还要加上这句代码？先看一个例子

```
data: {
	name: {
		first: 'zhang'
	}
}
```
假如数据是这样，我们这样改变数据

```
this.name.last = 'san'
```
想一下这样会出发依赖更新吗？事实上是不会的，因为last并没有被监听。Vue给我们指明了正确的姿势是：

```
this.$set('name', 'last', 'san')
```
来看一下set的源码(为方便，我已把数组相关的代码删掉) 

```
export function set (obj: Array<any> | Object, key: any, val: any) {
  if (hasOwn(obj, key)) {
    obj[key] = val
    return
  }
  const ob = obj.__ob__
  if (!ob) {
    obj[key] = val
    return
  }
  // 对新增的属性进行监听
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```
想一下，this.name变化时讲道理是应该通知闭包内name的依赖更新，但是由于新增属性并不会触发defineReactive，而this.name.__ob__的依赖和name属性的依赖是相同的，所以this.name.__ob__.notify()可达到相同的效果，这也是上面childOb.dep.depend()的原因。同理del也是如此：

```
export function del (obj: Object, key: string) {
  const ob = obj.__ob__
  if (!hasOwn(obj, key)) {
    return
  }
  delete obj[key]
  if (!ob) {
    return
  }
  ob.dep.notify()
}
```
#### 2）initWatch

```
function initWatch (vm: Component) {
  const watch = vm.$options.watch
  if (watch) {
    for (const key in watch) {
      const handler = watch[key]
      if (Array.isArray(handler)) {
        for (let i = 0; i < handler.length; i++) {
          createWatcher(vm, key, handler[i])
        }
      } else {
        createWatcher(vm, key, handler)
      }
    }
  }
}

function createWatcher (vm: Component, key: string, handler: any) {
  let options
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  vm.$watch(key, handler, options)
}
```
可以看出来initWatch最终调用的是$watch

```
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: Function,
  options?: Object
): Function {
  const vm: Component = this
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    cb.call(vm, watcher.value)
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```
最终实例化了一个Watcher，watcher可以分为两种，一种是用户定义的（我们在实例化Vue是传入的watch选项），一种是Vue内部自己实例化的，后文会看到。 

watcher的代码如下：

```
export default class Watcher {
  constructor (vm, expOrFn, cb, options) {
    this.vm = vm
    vm._watchers.push(this)
    // options
    this.deep = !!options.deep
    this.user = !!options.user
    this.lazy = !!options.lazy
    this.sync = !!options.sync
    this.expression = expOrFn.toString()
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = function () {}
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    const value = this.getter.call(this.vm, this.vm)
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            process.env.NODE_ENV !== 'production' && warn(
              `Error in watcher "${this.expression}"`,
              this.vm
            )
            /* istanbul ignore else */
            if (config.errorHandler) {
              config.errorHandler.call(null, e, this.vm)
            } else {
              throw e
            }
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subcriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed or is performing a v-for
      // re-render (the watcher list is then filtered by v-for).
      if (!this.vm._isBeingDestroyed && !this.vm._vForRemoving) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```
代码蛮长的，慢慢看  

watcher实例有一个getter方法，我们上文提到过watcher有两种，当watcher是用户创建时，此时的expOrFn就是一个expression，例如`name`或者`name.first`，此时它会被parsePath格式化为一个取值函数

```
const bailRE = /[^\w\.\$]/
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  } else {
    const segments = path.split('.')
	// obj为vue实例时   输出的便是
    return function (obj) {
      for (let i = 0; i < segments.length; i++) {
        if (!obj) return
        obj = obj[segments[i]]
      }
      return obj
    }
  }
}
```
格式化完getter函数之后紧接着执行get方法，数据的依赖正是在watcher的get方法执行时收集的，可以说get是连接observer和watcher的桥梁 

```
get () {
  pushTarget(this)
  const value = this.getter.call(this.vm, this.vm)
  // "touch" every property so they are all tracked as
  // dependencies for deep watching
  if (this.deep) {
    traverse(value)
  }
  popTarget()
  this.cleanupDeps()
  return value
}
```
get方法里面执行了getter，前面已经说过getter是一个取值函数，这不禁令我们联想到了数据的监听，当取值时假如Dep.target存在那么就可以收集依赖了，想想就激动。既然这样，`pushTarget`和`popTarget`必然是定义Dep.target

```
Dep.target = null
const targetStack = []

export function pushTarget (_target: Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```
如我们所想，pushTarget和popTarget定义了全局唯一的Dep.target（即调用get的watcher）。这里是需要思考的，源码的写法显然表明当getter函数调用时可能会触发其他watcher的get方法，事实上当我们watch一个计算属性或者渲染一个计算属性时便会出现这种情况，我们本篇暂不讨论。  

getter执行后，data相应闭包中的dep会执行`dep.depend()`，最终watcher会被添加到dep的订阅`subs`中，但data中的数据改变时，相应闭包中dep会`notify`它的subs(即watcher)依次`update`，最终调用watcher的run方法实现更新，看一下run方法：

```
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            process.env.NODE_ENV !== 'production' && warn(
              `Error in watcher "${this.expression}"`,
              this.vm
            )
            /* istanbul ignore else */
            if (config.errorHandler) {
              config.errorHandler.call(null, e, this.vm)
            } else {
              throw e
            }
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
```
run方法执行的时候会首先执行get方法，然后比较新的value的旧的value，如果不相同就执行`watcher.cb`。至此Vue的响应式雏形基本完成。

#### 3）initComputed  

先看代码（简化了）

```
function initComputed (vm) {
  const computed = vm.$options.computed
  if (computed) {
    for (const key in computed) {
      const userDef = computed[key]
      computedSharedDefinition.get = makeComputedGetter(userDef, vm)
      Object.defineProperty(vm, key, computedSharedDefinition)
    }
  }
}

function makeComputedGetter (getter, owner) {
  const watcher = new Watcher(owner, getter, noop, {
    lazy: true
  })
  return function computedGetter () {
    if (watcher.dirty) {
      watcher.evaluate()
    }
    if (Dep.target) {
      watcher.depend()
    }
    return watcher.value
  }
}
```
从代码可以看到，计算属性的值就是与之相关watcher的value。注意这里options的lazy为true，这表明创建watcher（称为a）的时候并不会执行get方法，也就是不会收集依赖。只有当我们取计算属性的值的时候才会收集依赖，那么什么时候会取计算属性的值呢？比如watch计算属性或者把计算属性写进render函数中。因为此get是惰性的，所以依赖于其他watcher（称为b）的唤醒，当执行完`watcher.evaluate()`之后，会把a添加到计算属性依赖数据dep的subs中,当执行完`watcher.depend()`之后，会把这个b添加到计算属性依赖数据dep的subs中。当依赖数据变化时，a和b（至少有这两个）watcher均会update，并且a的update是靠前的，因为其id在前面，所以当b进行update时获取到的计算属性为更新后的。   

这里比较绕，多想想吧。

#### initMethods 

```
function initMethods (vm: Component) {
  const methods = vm.$options.methods
  if (methods) {
    for (const key in methods) {
      if (methods[key] != null) {
        vm[key] = bind(methods[key], vm)
      } else if (process.env.NODE_ENV !== 'production') {
        warn(`Method "${key}" is undefined in options.`, vm)
      }
    }
  }
}
```
这个没什么好说的，就是将方法挂载到实例上。

## initRender 

initRender里面执行了首次渲染。  

在进行下面的内容之前我们先说明一下实例的_render方法，这个方法是根据render函数返回虚拟dom，什么是所谓的虚拟dom，看下Vue文档的解释：

>它所包含的信息会告诉 Vue 页面上需要渲染什么样的节点，及其子节点。我们把这样的节点描述为“虚拟节点 (Virtual Node)”，也常简写它为“VNode”。“虚拟 DOM”是我们对由 Vue 组件树建立起来的整个 VNode 树的称呼。  

至于vnode的生成原理不在本文的讨论范围。

进入正题，看下initRender的代码： 

```
export function initRender (vm: Component) {
  // 对于组件适用   其在父树的占位
  vm.$vnode = null // the placeholder node in parent tree
  // 虚拟dom
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null
  vm._renderContext = vm.$options._parentVnode && vm.$options._parentVnode.context
  vm.$slots = resolveSlots(vm.$options._renderChildren, vm._renderContext)
  // bind the public createElement fn to this instance
  // so that we get proper render context inside it.
  // 这就是render函数里面我们传递的那个参数   
  // 它的作用是生成vnode（虚拟dom）
  vm.$createElement = bind(createElement, vm)
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
``` 
initRender执行了实例的$mount，而$mount实际上是调用的内部方法`_mount`，现在来看_mount（简化了）

```
  Vue.prototype._mount = function (el, hydrating) {
    const vm = this
    vm.$el = el
    callHook(vm, 'beforeMount')
    vm._watcher = new Watcher(vm, () => {
      vm._update(vm._render(), hydrating)
    }, noop)
    hydrating = false
    // root instance, call mounted on self
    // mounted is called for child components in its inserted hook
	// 假如vm是根实例  那么其$root属性就是其自身
    if (vm.$root === vm) {
      vm._isMounted = true
      callHook(vm, 'mounted')
    }
    return vm
  }
```
_mount给我们提供了`beforeMount`和`mounted`两个钩子，可想而知实例化watcher的时候已经生成了虚拟dom，并且根据虚拟dom创建了真实dom并挂载到了页面上。  

上文我们已经讲过watcher的创建过程，所以可知vm._watcher的getter函数即为

```
 () => {
      vm._update(vm._render(), hydrating)
 }
```
并且此watcher的get并非为惰性get，所以watcher实例化之后便会立即执行get方法，事实上是执行`vm._render()`，并将获得的vnode作为参数传给`vm._update`执行。思考一下_render()函数执行时会发生什么，显然会获取data的值，此时便会触发get拦截器，从而将
vm._watcher添加至对应dep的subs中。

vm._update代码如下(简化了)： 

```
  Vue.prototype._update = function (vnode, hydrating) {
    const vm = this
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
    const prevVnode = vm._vnode
    vm._vnode = vnode
    if (!prevVnode) {
      // Vue.prototype.__patch__ is injected in entry points
      // based on the rendering backend used.
	  //  如果之前的虚拟dom不存在  说明是首次挂载
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating)
    } else {
	  // 之前的虚拟dom存在  需要先对新旧虚拟dom对比  然后差异化更新
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    if (vm._isMounted) {
      callHook(vm, 'updated')
    }
  }
```
可以看到_update的主要作用就是根据vnode形成真实dom节点。当data数据改变时，对应的dep会通知subs即vm._watcher进行update，update方法中会再次执行vm._watcher.get()，从而调用vm._update进行试图的更新。  

这里有个地方值得我们思考，更新后的视图可能不再依赖于上次的数据了，什么意思呢

```
更新前 <p>{{this.a}}</p>

更新后 <p>{{this.b}}</p>
```
也就是说需要清除掉a数据中watcher的依赖。看下Vue中的实现

`dep.depend`并没有我们想的那么简单，如下

```
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
```
相应的watcher的addDep如下，他会把本次更新依赖的dep的id存起来，如果更新前的id列表不存在新的dep的id，说明视图更新后依赖于这个dep，于是将vm._watcher添加到此dep的subs中

```
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
		// 更新后的视图依赖于此dep
        dep.addSub(this)
      }
    }
  }
```
假如之前dep的id列表存在存在某些id，这些id不存在与更新后dep的id列表，表明更新后的视图不在依赖于这些id对应的dep，那么需要将vm._watcher从这些dep中移除，这部分工作是在`cleanupDeps`中完成的，如下:

```
  cleanupDeps () {
    let i = this.deps.length
    // console.log(i)
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
```

## 结语

这篇文章只是对Vue内部实现机制的简单探索，很多地方没有涉及到，比如组件机制、模板的编译、虚拟dom树的创建等等，希望这些能在以后慢慢搞清楚。













































