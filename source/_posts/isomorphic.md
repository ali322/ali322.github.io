---
layout: post
title: 基于 Koa@2+react 快速开发 isomorphic webapp
date: 2016-07-15 15:16:30
tags: NodeJS
---

### 项目地址
[https://github.com/ali322/isomorphic-boilerplate](https://github.com/ali322/isomorphic-boilerplate) 欢迎star,欢迎PR

<!-- more -->

快速开发 
===
- 运行`npm install`
- 运行`npm run develop-webpack` 注入js和css至模板文件
- 运行`npm run develop` 启动开发服务

部署至生产环境
===
- 运行`npm install --production`
- 运行`npm install pm2 -g`(更多文档请见[pm2 文档](https://github.com/Unitech/PM2))
- 运行`pm2 start app.js --name <项目名>` 部署至生产

目录结构
===

```sh
__test__/
    |-- client/ #前端单元测试
    |-- server/ #后端单元测试
client/
    |-- asset/      #图片,字体等等资源
    |-- bundle/
        |-- common/     #公共的css和js
        |-- component/  #组件的css
        |-- index/      #index 页面入口js和css
        |-- error/      #错误页面入口js和css
        |-- .../        #更多的页面入口js和css,类似index
    |-- vendor/     #第三方库文件
server/
    |-- controller/ #express 路由目录
    |-- lib/        #后端库(工具库等等)
    |-- router.js   #后端路由定义文件
    |-- bootstrap.js #初始化后端应用,加载中间件和设置应用
shared/
    |-- lib/        #共享库(后端和前端共用)
    |-- chunk/
        |-- common/     #通用组件(例如:错误组件)
        |-- index/      #index页面所有组件
        |-- .../        #更多的页面所有组件.类似index
task/
    |-- config/
        |-- module.json #定义页面配合,以及css和js路径
        |-- vendor.json #定义第三方库
    |-- environment.js  #定义模块的环境变量
    |-- hmr-server.js       #webpack dev server 入口文件
    |-- webpack-inject.js #注入编译好的js和css至模板
    |-- webpack.develop.js #为开发环境编译模块和第三方库
    |-- webpack.production.js #为生产环境编译模块和第三方库
    |-- webpack.hot-update.js #为热替换开发环境编译模块和第三方库
view/
    |-- layout.html #全部布局文件
    |-- index.html  #index页面模板
    |-- *.html      #更多的页面模板
app.js      #应用入口文件
gulpfile.js #任务入口文件
```



