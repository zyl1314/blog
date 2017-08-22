# vue实践之抽离api  

在公司实习，项目目录类似于
```
|—— src
    |—— components
    |—— store
    |—— libs
    |—— routes
    |—— api
     —— main.js
     —— app.vue
```
是把api单独的抽取出来，个人认为至少有两个好处  

1. 方便管理接口，有变动改一处就行了  

2. 一目了然，所有接口在一个地方方便查看

## 使用  

以前调用接口的形式可能是这样的

```
created() {
    axios.get('/user/name').then(() => {
        ...
    })
}
```
我们希望的是这么用
```
created() {
    this.$api.user.name().then(() => {
        ...
    })
}
```

## 实现  

api文件夹包括两个文件
```
|—— api
    |—— urls.js
    |—— index.js
```
url.js定义了接口的地址，请求的方式
```
// urls.js

export default {
    user: {
        name: {
            url: '/user/name',
            method: 'post'
        }
    }
}
```
index.js是具体的实现逻辑  

事实上，我们需要做的工作很简单
```
Vue.prototype.$api = {
    user: {
        name: function() {
            return axios.get('/user/name')
        }
    }
}
```

如下是简单的获取api对象的方法
```
import urls from 'urls.js'
import axios from 'axios'

const api = {}

function mix(api, urls) {
    let keys = Object.keys(urls)
    keys.forEach(( key) => {
        let value = urls[key]
        if (Object.keys(value).indexOf('url') !== -1) {
            api[key] = () => {
                return axios(value)                
            }
        } else {
            api[key] = {}
            mix(api[key], value)
        }
    })
}

mix(api, urls)

export default api
```
最后将api对象挂在到Vue原型上即可
```
import api from 'api/index.js'

Vue.prototype.$api = api
```