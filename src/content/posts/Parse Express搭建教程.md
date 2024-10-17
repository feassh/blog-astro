---
title: Parse Express搭建教程
date: 2020-06-09
category: Backend
tags: [Parse]
sticky: 0
draft: false
comments: false
---

之前说了如何搭建最原始的parse服务，但是生产环境并不方便那样做。所以就按照官方的例子自己搭建一个express。

搭建express之前只需要安装好node和npm还有mongodb这三个就行。

## 第一步：新建一个目录，创建index.js文件

index.js文件内容如下：

```javascript
var express = require('express')
var ParseServer = require('parse-server').ParseServer
var ParseDashboard = require('parse-dashboard')

const appMount = '/test'
const databaseUri = 'mongodb://localhost:27017' + appMount
const serverURL = 'http://localhost:1337' + appMount // 如果使用https不要忘了修改它
const port = 1337
const appID = 'test_id'
const clientKey = 'test_client'
const masterKey = 'test_master'
const appName = 'test app'

var api = new ParseServer({
  databaseURI: databaseUri,
  cloud: __dirname + '/cloud/main.js',
  appId: appID,
  javascriptKey: clientKey,
  masterKey: masterKey, //master key ，打死也不要告诉别人！
  serverURL: serverURL,
  liveQuery: {
    classNames: ['Test', 'Test2'], // 配置支持实时请求的class表
  },
  appName: appName,
})

var app = express()

//仪表盘可配置多应用以及访问权限，还可以作为单独的Node项目运行
var dashboard = new ParseDashboard({
  apps: [
    {
      serverURL: serverURL, //这里的serverURl不能是localhost了，要用域名或者外网ip，踩坑弄了好久。。。
      appId: appID,
      masterKey: masterKey,
      appName: appName,
    },
  ],
  users: [
    {
      user: '123456',
      pass: '123456',
    },
  ],
})

app.use('/dashboard', dashboard)

// 设置Parse服务地址
app.use(appMount, api)

// Parse Server plays nicely with the rest of your web routes
app.get('/', function (req, res) {
  res.status(200).send('嗨，很高兴见到你~')
})

var httpServer = require('http').createServer(app)
httpServer.listen(port, function () {
  console.log('parse-server running on port ' + port + '.')
})

// 启用实时请求
ParseServer.createLiveQueryServer(httpServer)
```

## 第二步：安装parse-server、parse-dashboard和express

在当前主目录执行：

```shell
npm install parse-server parse-dashboard express
```

等待安装完后，就可以执行node index.js来启动parse服务了！！

## 关于开启https

可以从网上申请免费或者付费ssl服务，下面演示自己创建ssl：

```shell
mkdir certs
cd certs
openssl genrsa -out privatekey.pem 1024
openssl req -new -key privatekey.pem -out certrequest.csr
openssl x509 -req -in certrequest.csr -signkey privatekey.pem -out certificate.pem
```

创建完毕，只需要修改index.js的部分代码：

```javascript
var fs = require('fs')

const httpsOption = {
  key: fs.readFileSync('./certs/privatekey.pem'),
  cert: fs.readFileSync('./certs/certificate.pem'),
}

// var httpServer = require('http').createServer(app);
var httpServer = require('https').createServer(httpsOption, app)

httpServer.listen(port, function () {
  console.log('parse-server running on port ' + port + '.')
})
```

这是后端修改完毕，前端也要把对应的网址改成https开头，就完成了！！

## 修改mongodb默认数据库配置

mongodb的默认配置文件在/etc/mongod.conf（Ubuntu）。
