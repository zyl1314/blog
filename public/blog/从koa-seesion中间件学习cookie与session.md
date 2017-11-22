# 从koa-session中间件学习cookie与session

关于cookie和session是什么网上有很多介绍，但是具体的用法自己事实上一直不是很清楚，通过koa-session中间件的源码自己也算是对cookie和session大致搞明白了。

## 1、cookie和session的联系
在我了解cookie的时候，大多数教程讲的是这些：

```
function setCookie(name,value) 
{ 
    var Days = 30; 
    var exp = new Date(); 
    exp.setTime(exp.getTime() + Days*24*60*60*1000); 
    document.cookie = name + "="+ escape (value) + ";expires=" + exp.toGMTString(); 
} 
```
它给我一个错觉：cookie只能在客户端利用js设置读取删除等，但事实上很多的cookie是由服务端在header里面写进去的：

```
const Koa = require('koa');
const app = new Koa();

app.use((ctx) => {
  ctx.cookies.set('test', 'hello', {httpOnly: false});
  ctx.body = 'hello world';
})

app.listen(3000);
```
打开控制台可以看到：
![img](https://github.com/zyl1314/blog/raw/master/public/img/session/1.png)
![img](https://github.com/zyl1314/blog/raw/master/public/img/session/2.png)











