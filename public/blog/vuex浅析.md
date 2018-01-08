# vuex浅析

vuex解决的问题是多组件共享状态时的状态管理。

## 如何共享状态

多组件共享状态的复杂性在于：

- 多个视图依赖于同一状态
- 来自不同视图的行为需要变更同一状态  

这时候通过prop传值很复杂并且兄弟组件无法传值，使用`event bus`有时会变的难以维护。但是假如把所有组件共享的状态抽离为一个单例并做响应化处理，那么当这个共享的状态发生改变时所有依赖的组件都会变更。  

而vue本身就提供了数据`observe`功能，并且引用数据时可以自动收集依赖。我们完全可以把共享状态作为`data`生成一个vue实例，组件中用到共享状态时直接引用这个vue实例的状态即可，从而在组件`render`的时候触发依赖收集，当共享状态改变时组件自然会`update`。  

基于上述的分析，仿照`vuex`的api写一个简易版的[tiny-vuex](https://github.com/zyl1314/tiny-vuex)。

## vuex的实现

vuex的基本原理和上述分析的大致相似。下面看一下需要注意的几个点

#### Module

vuex提供了`module`功能，允许将store模块化。如下：

```
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```
对于state，应该解析成这样

```
const store = new Vuex.Store({
  state: {
    ...  //  rootstate
    a: {},
    b: {}
  }
})
```
我们知道在vue中，直接对data添加属性时不能被`observe`的，可以通过`set`响应式添加。如下：

```
// getNestedState 根据路径获取父级数据
const parentState = getNestedState(rootState, path.slice(0, -1))
const moduleName = path[path.length - 1]
store._withCommit(() => {
  Vue.set(parentState, moduleName, state || {})
})
```

对于mutation，通过闭包接受模块的局部state。

```
function registerMutation (store, type, handler, path = []) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    // 模块局部state注入
    handler(getNestedState(store.state, path), payload)
  })
}
```
action类似于mutation，也会通过闭包注入局部状态

```
function registerAction (store, type, handler, path = []) {
  const entry = store._actions[type] || (store._actions[type] = [])
  const { dispatch, commit } = store
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler({
      dispatch,
      commit,
      getters: store.getters,
      state: getNestedState(store.state, path),
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```
这里做了promise化处理以便我们可以组合使用actions，如下：

```
actions: {
  // ...
  actionB ({ dispatch, commit }) {
    return dispatch('actionA').then(() => {
      commit('someOtherMutation')
    })
  }
}
```

### _withCommit

阅读源码我们发现，改变状态时需要接mutation包装在`_withCommit`函数中执行

```
_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}
```
这是为了使变化可追踪，所有的状态改变必须通过commit提交，不能直接改变状态。怎么做到的呢？通过`深度`监测state，当状态发生改变时检查`_commiting`是否为true，如果不是则抛出警告，如下

```
function enableStrictMode (store) {
  store._vm.$watch('state', () => {
    assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
  }, { deep: true, sync: true })
}
```



















