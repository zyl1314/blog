# vue@2.0源码学习--组件通信的几种方式

本文讨论的组件通信主要是指父子组件通信，父组件向子组件通信主要是通过prop完成，子组件向父组件通信主要是靠vue的自定义事件系统。我把插槽也视为父子组件通信的一种方式。

## Prop

首先prop会在组件对应的vnode生成时被提取出来 

```
export function createComponent (
  ...

  const propsData = extractProps(data, Ctor)
  
  ...
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children }
  )

  return vnode
```
对于提取到的数据，首先做判断是否符合prop的类型要求，然后再做响应化处理

```
if (process.env.NODE_ENV !== 'production') {
  defineReactive(vm, key, validateProp(key, props, propsData, vm), () => {
    if (vm.$parent && !observerState.isSettingProps) {
      warn(
        `Avoid mutating a prop directly since the value will be ` +
        `overwritten whenever the parent component re-renders. ` +
        `Instead, use a data or computed property based on the prop's ` +
        `value. Prop being mutated: "${key}"`,
        vm
      )
    }
  })
} else {
  defineReactive(vm, key, validateProp(key, props, propsData, vm))
}
```
注意最好避免直接操作prop属性，因为父组件只要重新渲染就会强制prop更新，从而导致子组件重新渲染，最好使用data或者computed属性。

## 自定义事件

首先listeners会在组件对应的vnode生成时被解析出来 

```
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children }
  )
```
然后在initEvents对组件实例绑定listeners

```
  vm._updateListeners = (listeners, oldListeners) => {
    updateListeners(listeners, oldListeners || {}, on, off)
  }
  if (listeners) {
    vm._updateListeners(listeners)
  }
```
组件实例内部触发(`emit`)自定义事件即可完成通信 

## 插槽

组件实例化时，插槽会作为`_renderChildren`传入options

```
const options: InternalComponentOptions = {
  _isComponent: true,
  parent,
  propsData: vnodeComponentOptions.propsData,
  _componentTag: vnodeComponentOptions.tag,
  _parentVnode: vnode,
  _parentListeners: vnodeComponentOptions.listeners,
  _renderChildren: vnodeComponentOptions.children
}
```
在initRender时会把插槽解析出来挂载到实例的`$slots`，实际上此时插槽是一组vnodes，在render函数中可直接引用。

---
很粗糙，以后完善。