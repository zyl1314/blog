# axios之拦截器  

axios是一个适用于浏览器和nodejs的http库，支持Promise  

##　用法
```
// get
axios.get('/user/name').then(() => {
    ...
})
```
用于网络的原因，我们请求数据有时候会很耗时，一般会加载一个菊花图告诉用户正在加载。我们可能这么做  

首先写一个loading，注意并不是只要请求就会显示菊花图，假如数据在200毫秒内返回了，我们就不显示菊花图
```
// loading

let timer = null

loading.show = function() {
    timer = setTimeout(() => {
        showloading()  // 显示菊花图
    }, 300)
}

loading.hidden = function() {
    clearTimeout(timer)
}
```
在请求数据时可以这么写
```
loading.show() 
axios.get('/user/name').then((res) => {
    loading.hidden()
    ...   // do something with res
})
```
事实上，loading并不是我们关心的业务逻辑

## Interceptors（拦截器）  

> You can intercept requests or responses before they are handled by then or catch  

用法

```
// Add a request interceptor
axios.interceptors.request.use(function (config) {
    // Do something before request is sent
    return config;
  }, function (error) {
    // Do something with request error
    return Promise.reject(error);
  });

// Add a response interceptor
axios.interceptors.response.use(function (response) {
    // Do something with response data
    return response;
  }, function (error) {
    // Do something with response error
    return Promise.reject(error);
  });
```
上面的场景可以专门改写
```
// Add a request interceptor
axios.interceptors.request.use(function (config) {
    // Do something before request is sent
    loading.show()
    return config;
  }, function (error) {
    // Do something with request error
    return Promise.reject(error);
  });

// Add a response interceptor
axios.interceptors.response.use(function (response) {
    // Do something with response data
    loading.hidden()
    return response;
  }, function (error) {
    // Do something with response error
    loading.hidden()
    return Promise.reject(error);
  });
```