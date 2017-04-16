# throttle与debounce
#### throttle与debounce都是限制触发次数的函数，很多场景需要进行节流，最常见的就是监听键盘的输入事件并作出响应

## - debounce 
### 1 含义
#### debounce可以理解在给定时间后执行某操作，如果还没有到时间就再次触发操作则清除时间重新计时
### 2 实现

```
function debounce(time, action) {
    var timer = null
    return function() {
        var args = [].slice.call(arguments),
            This = this
        cleatTimeout(timer)
        timer = setTimeout(function() {
            action.apply(This, args)
        }, time)
    }
}
```
#### debounce的原理比较容易理解：如果重新触发那就清除上一个定时器并重新开一个定时器

## - throttle 
### 1 含义
#### throttle可以理解为在规定的时间间隔内最多只执行一次操作
### 2 实现

```
function throttle(time, action) {
    var lastTime = 0
    return function() {
        var timeNow = new Date.getTime(),
            args = [].slice.call(arguments)
        if (timeNow - lastTime > time) {
            action.apply(This, args)
            lastTime = timeNow
        }
    }
}
```
#### 如果已经触发了操作，在时间间隔内又再次触发这时什么也不做，如果再次触发的时间距离上次触发已经超了时间间隔，那么就执行此次操作并把当前时间修改为上次触发时间