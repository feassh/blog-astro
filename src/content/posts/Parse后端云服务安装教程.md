---
title: Parse后端云服务安装教程
date: 2020-06-08
category: Backend
tags: [Parse]
sticky: 0
draft: false
comments: false
---

Parse是一个基于云端的后端管理平台。对于开发者而言，Parse提供后端的一站式和一揽子服务：服务器配置、数据库管理、API、影音文件存储，实时消息推送、客户数据分析统计、等等。这样，开发者只需要处理好前端/客户端/手机端的开发，将后端放心的交给Parse即可。

脸书(Facebook)于2013年以8500万美金收购Parse。之后Parse的功能不断推陈出新。平台越来越强大的同时，越来越多的开发者也将App的后台工作完全交给Parse。但是由于脸书的云战略一直不是其主要方向，且Parse难以整合进脸书的其他产品，脸书决定于2017年1月28日彻底关闭Parse。

由于Parse关闭了，所以官方也开源了项目。

> [Parse的Github项目地址](https://github.com/parse-community)

本次我用的服务器是阿里云ECS，镜像为Ubuntu 18.04

## 第一步：安装nodejs和npm

```shell
执行 apt-get install nodejs 来安装nodejs
执行 apt-get install npm 来安装npm
```

上面两个命令执行完毕后，就完成了nodejs和npm的安装。此时，执行node -v和npm -v就可以看到对应的版本号了。我写这个博客的时候，apt安装的nodejs版本是8.10.0，npm的版本是3.5.2

由于npm版本过低，下面安装parse-dashboard的时候会报错，所以需要更新npm到最新版本。

```shell
执行 npm install -g npm@latest
```

执行完上面的命令后，关闭shell窗口重新打开，执行npm -v就可以看到版本更新到最新的了，我现在的版本是6.13.6

如果还想把nodejs的版本也同时更新了，可以执行下面的命令：

```shell
npm install -g n
n latest
```

执行完后在执行node -v就可以看到版本也变化了，我的版本为13.8.0

## 第二步：安装MongoDB

1.  通过公钥对资源包一致性和真实性进行验证：
    ```shell
    apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
    ```
2.  创建文件/etc/apt/sources.list.d/mongodb-org-4.0.list：
    ```shell
    echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list
    ```
3.  在安装之前，先更新系统资源包：
    ```shell
    apt-get update
    ```
4.  安装mongodb资源包：
    ```shell
    apt-get install mongodb-org
    ```

此时，MongoDB就安装完毕了。执行下面的命令查看版本号：

```shell
mongo --version
```

如果能正常看到版本号，就意味着安装完毕了，此时就可以启动数据库了：

```shell
systemctl start mongod #这条命令用来启动数据库服务
systemctl status mongod #查看当前服务运行状态
systemctl enable mongod #将mongod服务添加开机启动
```

## 第三步：部署parse-server

```shell
npm install -g parse-server
```

执行上面的命令来下载项目，下载完毕后执行：

```shell
parse-server --appId 这里写APPID --masterKey 这里写MASTERKEY --databaseURI mongodb://localhost:27017/dbname

--databaseURI mongodb://localhost:27017/dbname 这部分可以省略，省略的话，自动配置默认的：
mongodb://localhost/parse
```

此时会启动parse-server服务，如果用浏览器访问 http://内网或外网ip:1337/parse 返回{“error”:”unauthorized”}，说明运行成功，没有问题。

## 第四步：部署parse-dashboard

```shell
npm install -g parse-dashboard
```

执行完毕后，项目就下载下来了，开始启动dashboard：

```shell
parse-dashboard --appId 这里是APPID --masterKey 这里是MASTERKEY --serverURL "http://localhost:1337/parse" --appName 应用名字
```

启动之后，在内网环境下用浏览器访问 http://localhost:4040 可以正常进入，如果外网ip去访问，就会提示Parse Dashboard can only be remotely accessed via HTTPS，意思是外网环境只能用https来访问。

解决方案：在启动命令行后面加上–allowInsecureHTTP

```shell
parse-dashboard --appId 这里是APPID --masterKey 这里是MASTERKEY --serverURL "http://localhost:1337/parse" --appName 应用名字 --allowInsecureHTTP
```

parse允许配置多个app，有专门的配置文件。我用Ubuntu apt安装的npm，所以npm下载的项目默认存放在/usr/local/lib/node_modules这个目录下。cd进入这个目录会看到mongodb-runner parse-dashboard parse-server这些目录，此时继续进入parse-dashboard目录，ls一下，会有Parse-Dashboard这个目录，继续cd进去，ls就会看到parse-dashboard-config.json这个文件，这个就是配置app的json配置文件。

vim去编辑这个文件，按照下面的格式添加新的app和账号就ok了

```json
{
  "apps": [
    {
      "serverURL": "http://localhost:1337/parse",
      "appId": "123",
      "masterKey": "123",
      "appName": "My Parse Server App"
    },
    {
      "serverURL": "http://api.parse.com/1",
      "appId": "myAppId",
      "masterKey": "myMasterKey",
      "appName": "My Parse Server App"
    }
  ],
  "users": [
    {
      "user": "admin",
      "pass": "admin"
    },
    {
      "user": "myUserName",
      "pass": "myPassword"
    }
  ]
}
```

配置项里的users是网页进入dashboard的时候需要的登录账号。

一切就绪，启动带配置项的dashboard：

```shell
parse-dashboard --conifg parse-dashboard-config.json --allowInsecureHTTP
```

如果登录面板中app无法操作，提示：Server not reachable: unauthorized，这是因为在dashboard配置中需要把localhost改成你的公网ip，如下

```json
"serverURL": "http://你的外网ip:1337/parse"
```

## 完成部署

到此，估计也部署完成了。溜了溜了~
