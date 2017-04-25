# BAE
#### 记录一下BAE的使用

## step1 : 在管理中心创建项目
#### 创建项目时会让你选择代码管理方式（我使用的git），还有注意会让你选择node的版本，注意勾选（这个坑死我了）

## step2 : 修改package.json

```
  "scripts": {
    "start": "node index.js"
  }
```
#### 需要在scripts添加start，node后面跟的是入口文件

## step3 : 数据库服务
#### 如果需要数据库服务可以在扩展服务里面选择，用户名和密码为Access Key ID和Secret Access Key

## step4 : push到远程即可