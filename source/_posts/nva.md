---
layout: post
title: nva 简洁高效的前端项目脚手架
date: 2018-07-13 11:13:30
tags: NodeJS
---

简洁高效的前端项目脚手架, [项目地址](https://github.com/ali322/nva)

## 介绍

nva是一个基于webpack,提供灵活配置的前端项目脚手架工具,既能支持纯前端项目(html+css+js)的开发需求,也能支持同构JS/SSR项目(node+react/node+vue)的开发

- 交互性方式初始化基于多种基础模板的项目
- 使用 js/json 定义满足项目需求的模拟数据
- 开箱即用的 Babel, Typescript, Sass, Less, Stylus 支持
- 单元测试和端到端测试的内置支持

<!-- more -->

## 快速开始

安装环境依赖: [Node.js](https://nodejs.org/en/) (>=4.x, 6.x preferred), npm 3+ and [Git](https://git-scm.com)

第一步: 安装nva命令行工具

```bash
npm install nva -g
```

第二步: 初始化项目

```bash
nva init my-project
```

根据命令行提示填写,包含项目模板,框架,是否单页应用,版本号,描述信息,仓库地址,发布协议等等

第三步: 开始开发

```bash
cd my-project
nva vendor
nva dev -p 3000
```

使用 `nva vendor` 将项目第三方依赖包预先打包,提升项目开发编译速度,然后使用 `nva dev` 启动开发服务器,打开浏览器输入 `http://localhost:3000`即可看到项目的初始化界面

第四步: 测试

```bash
npm test
```
根据不同的项目模板执行不同的单元测试

可选: 集成测试

```bash
npm run e2e
```
基于nightwatch的集成测试,测试浏览器为 chrome,可依照项目目录中的 `test/e2e` 对照修改为 firefox 或 IE

后续: 打包发布

```bash
nva build
```

完成源码的编译压缩,静态资源合并压缩,路径处理,html注入,构建版本号处理等等

## 模块化管理

- 增加模块

  添加一个空白模块
  
  ```bash
  nva mod my-module
  ```
  
  以 other-module 为模板添加一个模块
  
  ```bash
  nva mod my-module -t other-module
  ```
  
  支持批量添加,多个模块名使用英文逗号 `,` 分隔

- 删除模块

  删除一个已有的模块
  
  ```bash
  nva mod existed-module -d
  ```
  
  支持批量删除,多个模块名使用英文逗号 `,` 分隔

## 项目模板

- [纯前端模板](https://github.com/ali322/frontend-boilerplate)

  - react + redux 的多页面项目
  - react + redux + react-router 的单页面项目
  - vue + vuex 的多页面项目
  - vue + vuex + vue-router 的单页面项目
  
- [同构JS模板](https://github.com/ali322/isomorphic-boilerplate)

  - react + redux + koa@2 的多页面项目
  - react + redux + react-router + koa@2 的单页面项目
  - vue + vuex + koa@2 的多页面项目
  - vue + vuex + vue-router + koa@2 的单页面项目


## 配置参数

nva脚手架的工具的初衷是提供尽量简洁高效的方式进行前端项目开发,所以大部分时候使用默认配置即可,但是为了满足不同的业务场景,也提供了灵活的配置入口方便自定义,配置文件都位于项目的 .nva 目录下

.nva 目录结构如下

```bash
|-- .nva/
    |-- temp/   # 编译缓存目录
    |-- api/   # 模拟数据接口服务配置
        |-- user.json  # 模拟用户数据接口配置
        |-- ...
    |-- nva.json    # 全局配置
    |-- module.json # 项目模块设置
    |-- vendor.json # 项目第三方包依赖设置
```

- `nva.json` 全局配置

    ```js
    {
        "type":"isomorphic",    /* 项目类型: `frontend`,`isomorphic`,`react-native` */
        "spa":true            /* 是否单页面项目(SPA)? */
        "entryJSExt":".jsx",    /* 入口 js 文件扩展名 */
        "entryCSSExt":".styl",   /* 入口 css 文件扩展名 */
        "distFolder": "dist",   /* 源码编译目标目录名称 */
        "bundleFolder": "bundle",   /* 项目模块父目录名称 */
        "vendorFolder": "vendor",   /* 第三方依赖包编译目标目录名称 */
        "assetFolder": "asset",    /* 静态资源目录名称 */
        "spriteFolder": "sprites",    /* 雪碧图原始图目录名称 */
        "fontFolder": "font",   /* 字体目录名称 */
        "imageFolder": "image",    /* 图片目录名称 */
        "sourcePath": "src",    /* 源码目录名称(仅限纯前端项目) */
        "pagePath": "page",    /* html 文件目录名称 */
        "serverFolder": "server",   /* 服务端源码目录(仅限同构JS项目) */
        "serverEntryJS": "bootstrap.js",    /* 服务端入口文件(仅限同构JS项目) */
        "clientPath": "client"    /* 客户端源码目录(仅限同构JS项目) */
    }
    ```
- `module.json` 项目模块配置

    ```js
    {
        "index": {  /* 模块名称 */
            "html": ["index.html"],     /* 入口 html 文件 */
            "path": "index",            /* 相对目录名称(相对于 bundleFolder ) */
            "vendor": {"js": "react","css": "common"}   /* 模块依赖引用名称,引用自 `vendor.json` */
        }
    }
    ```

- `vendor.json` 第三方依赖包配置

    ```js
    {
        "js":{
            "react":["react","react-dom"]     /* 定义一个JS依赖引用 */
        },
        "css":{
            "common":["font-awesome/css/font-awesome.css"]     /* 定义一个css依赖引用 */
        }
    }
    ```
    
- `mock` 模拟数据接口服务配置

    简单的模拟接口配置

    ```js
    [{
        "url": "/mock/user",    /* 接口请求 url */
        "method": "get",        /* 接口请求方法名称 */
        "response": {           /* 接口响应 */
            "code": 200,
            "data": {
                "id": 6,
                "name": "Mr.smith"
            }
        }
    }]
    ```
    
    你也可以使用 [JSON Schema](http://json-schema.org) 一个更具语义化和持续化的模拟数据生成器来生成模拟数据

    ```json
    [{
        "url": "/mock/users",
        "method": "get",   
        "response": {        
            "type": "object",
            "properties": {
                "id": {
                    "$ref": "#/definitions/positiveInt"
                },
                "name": {
                    "type": "string",
                    "faker": "name.findName"
                },
            },
            "required": ["id", "name"],
            "definitions": {
                "positiveInt": {
                    "type": "integer",
                    "minimum": 0,
                    "exclusiveMinimum": true
                }
            }
        }
    },{
         "url": "/mock/user",
        "method": "post",w
        "response": {
            "code": 200,
            "data": {
                "status": "ok"
            }
        }
    }]
    ```
    
## 子包

packages 目录下的 `nva-core` `nva-task` `nva-server`的三个子包可以独立安装使用

- nva-core: 基础webpack编译配置,满足一般的构建需求

  ```javascript
  import config from 'nva-core'
  const buildConfig = config(constants)
  webpack({
    ...buildConfig,
    entry:'index.js',
    output:{
      ...
    }
  }).run((err,stats)=>{ ... })
  ``
  
- nva-task: nva构建任务集合,可以根据需求自定义组合

  ```javascript
  var tasks = require('nva-tasks')
  tasks.frontend.build() //前端项目构建
  task.isomorphic.build()  //同构JS项目构建
  ```
  
- nva-server: 基于connect的前端开发服务,带模拟数据接口功能

  ```javascript
  import App from 'nva-server'
  let app = App()
  app.listen(3000,()=>{
    console.log('==> server stared at %d',3000)
  })
  ```
  
  也可以通过命令行方式调用,具体参数说明请参见 [nva-task](https://github.com/ali322/nva/blob/master/packages/nva-server/README.md)
  
  ```bash
  nva-server -p 5000 -P src
  ```
    
## 常见问题

Q: 项目初始化缓慢?
A: 因为在项目初始化的使用 npm install 安装项目必须的依赖包,而 npm install 在国内网络环境下你懂的,因此如果长时间没有显示 success,推荐使用 ctrl+d 终止进行,然后进入项目目录执行 cnpm install 或者使用 yarn install

Q: nva安装缓慢?
A: 同上,建议使用 `cnpm install nva -g` 或者 `yarn global add nva`

Q: nva中的包是否可以独立使用?
A: 可以,nva的依赖包 nva-core 和 nva-server 都可以独立使用,使用 npm 安装即可

Q: 如何更新nva?
A: 使用 `npm install nva -g` 覆盖安装 


    
    



  











