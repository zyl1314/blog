# vue@2.0源码学习---从hello world学习vue的内部实现
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