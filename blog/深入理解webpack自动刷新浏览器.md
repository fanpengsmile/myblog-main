---
slug: 深入理解webpack自动刷新浏览器
title: 深入理解webpack自动刷新浏览器
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

我们在日常开发时，有一个需要在开发状态下的优化，就是**浏览器能自动显示修改后的代码，而无需我们手动刷新**

## 1\. 自动刷新浏览器

为了能实现浏览器自动刷新，需要做两件事情:

+   监听文件变化
+   自动刷新浏览器

### 1.1 监听文件变化

监听文件变化是在webpack模块进行。

#### 1.1.1 方式

需要在webpack中开启监听模式，有两种方式（**开启监听模式后，可以设置监听相关配置watchOptions**）：

**1\. 在webpack配置文件中添加watch:true**

```javascript
module.export = {
  watch: true,
  watchOptions: {
    // 不监听的文件或文件夹
    ignored: /node_modules/,
    // 监听到变化发生后会等300ms再去执行动作，防止文件更新太快导致重新编译频率太高  
    aggregateTimeout: 300,  
    // 判断文件是否发生变化是通过不停的去询问系统指定文件有没有变化实现的
    poll: 1000
  }
}
```

**2\. 在执行启动 Webpack 命令时，带上 --watch 参数**

#### 1.1.2 原理

+   在 Webpack 中监听一个文件发生变化的原理是**定时**（**可在watchOptions.poll中设置**）的去获取这个文件的最后编辑时间，每次都存下最新的最后编辑时间，**如果发现当前获取的和最后一次保存的最后编辑时间不一致，就认为该文件发生了变化**
+   当发现某个文件发生了变化时，并不会立刻告诉监听者，而是先缓存起来，收集**一段时间**(**可在watchOptions.aggregateTimeout中设置**)的变化后，再一次性告诉监听者。防止在编辑代码的过程中可能会高频的输入文字导致文件变化的事件高频的发生

### 1.2 自动刷新浏览器

监听到文件变化后需要去刷新浏览器，这部分在webpack-dev-server模块中进行。(**在使用 webpack-dev-server 模块去启动 webpack 模块时，webpack 模块的监听模式默认会被开启**)

#### 1.2.1 原理

自动刷新有三种方法：

+   借助浏览器扩展去通过浏览器提供的接口刷新
+   往要开发的网页中注入代理客户端代码，通过代理客户端去刷新整个页面。
+   把要开发的网页装进一个 iframe 中，通过刷新 iframe 去看到最新效果。

DevServer 支持第2、3种方法，第2种是 DevServer 默认采用的刷新方法。

## 2\. 模块热更新

以上自动刷新是会刷新整个页面，这种方式的缺点就是时间长，同时不能保存页面的状态。

而模块热更新即可在不刷新整个页面的情况下来实时预览。它只会在代码发生变化时，只编译发生变化的模块，并替换浏览器中的老模块。

### 2.1 两种方式

以下两种方式都能实现模块热替换，区别在于其内部使用的通信方式不同。前者使用webcoket通信，后者使用eventSource通信。

#### 2.1.1 webpack-dev-server

**1\. 开启方式**

+   开启DevServer的模块热替换模式： webpack-dev-server --hot
+   添加HotModuleReplacementPlugin插件

**2\. 原理**

[参考](https://link.segmentfault.com/?enc=ktxfoF77pOArOQikjflVcw%3D%3D.UujWfZWOVIzovPYco%2BYXpY97BoyFd0Mfp2gl9x9STc0lpCVWRMgpLUPgkU0wKUT7)  
**（1）webpack 对文件系统进行 watch 打包到内存中**

+   依赖**webpack-dev-middleware**，它会调用 webpack 的 api 对文件系统 watch
+   webpack-dev-middleware依赖**memory-fs库**，它将webpack 原本的 outputFileSystem 替换成了MemoryFileSystem 实例，这样代码就将输出到内存中

**（2）devServer 通知浏览器端文件发生改变**\~~~~

+   在启动 devServer 的时候，sockjs 在服务端和浏览器端建立了一个 **webSocket 长连接**。
+   webpack-dev-server 调用 webpack api监听 compile的 done 事件，将编译打包后的**新模块 hash 值发送到浏览器端**。

**（3）webpack-dev-server/client 接收到服务端消息做出响应**

+   webpack-dev-server默认**修改了webpack 配置中的 entry 属性**，在里面添加了 webpack-dev-client 的代码，这样在最后的 bundle.js 文件中就会有接收 websocket 消息的代码了。
+   webpack-dev-server/client 当接收到 type 为 hash 消息后会将 hash 值暂存起来，当接收到 type 为 ok 的消息后**对应用执行 reload 操作**(根据 hot 配置决定是刷新浏览器还是对代码进行热更新（HMR）).

**（4）webpack 接收到最新 hash 值验证并请求模块代码**

+   webpack/hot/dev-server监听webpack-dev-server/client发出的webpackHotUpdate 消息。
+   先调用hotDownloadManifest（ajax请求）来获取更新的文件列表。
+   再调用hotDownloadUpdateChunk获取更新的新模块代码

**（5）HotModuleReplacement.runtime 对模块进行热更新**

+   找出过期的模块和过期的依赖
+   从缓存中删除过期的模块和依赖
+   将新的模块添加到 modules 中

#### 2.1.2 webpack-hot-middleware

**1\. 开启方式**

+   Webpack配置添加HotModuleReplacementPlugin插件
+   Node Server引入webpack-dev-middlerware 和 webpack-hot-middleware插件（本人参与的项目中用到的是koa-webpack-middleware）
+   entry中注入热更新代码(必须要注入。为了保证服务端和客户端能够通信)  
    **webpack-hot-middleware/client?path=/\_\_webpack\_hmr&timeout=20000**

**2\. 原理**  
[原理参考](https://link.segmentfault.com/?enc=aHn8M0e%2Fth3Xmvbq%2BSWUOw%3D%3D.E8ZpVNG1LMrwp7J4O3sDuwaEWLYZCYqs4UmzrWkDM99JICzArtXzMwVsjV0fNOJy)




