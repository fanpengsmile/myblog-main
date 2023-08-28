---
slug: cookie、session、sessionStorage、localStorage区别
title: cookie、session、sessionStorage、localStorage区别
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---


无衬线
14px
经典蓝
github

## cookie、session、sessionStorage、localStorage区别
​
### cookie、session区别
​
+   cookie 存储于浏览器端，而 session 存储于服务端
+   cookie 的安全性相比于 session 较弱，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗  
    考虑到安全应当使用session。
+   session 会在一定时间内保存在服务器上。当访问增多时，会占用服务器的资源，所以考虑到服务器性能方面，可以使用cookie
+   cookie 存储容量有限制，单个cookie 保存数据不能超过4k，且很多浏览器限制一个站点最多保存20个cookie。而对于 session ，其默认大小一般是1024k
​
### cookie、sessionStorage、localStorage 异同点
​
**html5 中 webStorage 包含 sessionStorage 和 localStorage**  
共同点：
​
+   都保存在浏览器端，且是同源的
​
区别：
​
+   cookie 数据始终在同源的http请求中携带，而 webStorage 不会再请求中携带，仅仅在本地存储
+   存储大小区别，cookie 是4k，webStorage 可以达到5M甚至更大
+   数据有效时间区别： sessionStorage 仅仅是会话级别的存储，它只在当前浏览器关闭前有效，不能持久保持；localStorage 始终有效，即使窗口或浏览器关闭也一直有效，除非用户手动删除，其才会失效；cookie 只在设置的 cookie 过期时间之前一直有效。
+   作用域区别：sessionStorage **不在不同的浏览器窗口中共享**，即使是同一个页面； localStorage 和 cookie 在所有同源窗口是共享的
+   Web Storage 支持事件通知机制，可以将数据更新的通知发送给监听者。Web Storage 的 api 接口使用更方便。
​
#### web storage和cookie的区别
​
Web Storage的概念和cookie相似，区别是它是为了更大容量存储设计的。Cookie的大小是受限的，并且每次你请求一个新的页面的时候Cookie都会被发送过去，这样无形中浪费了带宽，另外cookie还需要指定作用域，不可以跨域调用。
​
除此之外，Web Storage拥有setItem,getItem,removeItem,clear等方法，不像cookie需要前端开发者自己封装setCookie，getCookie。
​
但是Cookie也是不可以或缺的：Cookie的作用是与服务器进行交互，作为HTTP规范的一部分而存在 ，而Web Storage仅仅是为了在本地“存储”数据而生。
​
Cookies:服务器和客户端都可以访问；大小只有4KB左右；有有效期，过期后将会删除；
​
本地存储：只有本地浏览器端可访问数据，服务器不能访问本地存储直到故意通过POST或者GET的通道发送到服务器；每个域5MB；没有过期数据，它将保留知道用户从浏览器清除或者使用Javascript代码移除
​
最后编辑于
​
：2017.12.10 06:43:14
​
+   序言：七十年代末，一起剥皮案震惊了整个滨河市，随后出现的几起案子，更是在滨河造成了极大的恐慌，老刑警刘岩，带你破解...
    
+   序言：滨河连续发生了三起死亡事件，死亡现场离奇诡异，居然都是意外死亡，警方通过查阅死者的电脑和手机，发现死者居然都...
    
+   文/潘晓璐 我一进店门，熙熙楼的掌柜王于贵愁眉苦脸地迎上来，“玉大人，你说我怎么就摊上这事。” “怎么了？”我有些...
    
+   文/不坏的土叔 我叫张陵，是天一观的道长。 经常有香客问我，道长，这世上最难降的妖魔是什么？ 我笑而不...
    
+   正文 为了忘掉前任，我火速办了婚礼，结果婚礼上，老公的妹妹穿的比我还像新娘。我一直安慰自己，他们只是感情好，可当我...
cookie、session、sessionStorage、localStorage区别
cookie、session区别
•cookie 存储于浏览器端，而 session 存储于服务端
•cookie 的安全性相比于 session 较弱，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗
考虑到安全应当使用session。
•session 会在一定时间内保存在服务器上。当访问增多时，会占用服务器的资源，所以考虑到服务器性能方面，可以使用cookie
•cookie 存储容量有限制，单个cookie 保存数据不能超过4k，且很多浏览器限制一个站点最多保存20个cookie。而对于 session ，其默认大小一般是1024k

cookie、sessionStorage、localStorage 异同点
html5 中 webStorage 包含 sessionStorage 和 localStorage
共同点：

•都保存在浏览器端，且是同源的

区别：

•cookie 数据始终在同源的http请求中携带，而 webStorage 不会再请求中携带，仅仅在本地存储
•存储大小区别，cookie 是4k，webStorage 可以达到5M甚至更大
•数据有效时间区别： sessionStorage 仅仅是会话级别的存储，它只在当前浏览器关闭前有效，不能持久保持；localStorage 始终有效，即使窗口或浏览器关闭也一直有效，除非用户手动删除，其才会失效；cookie 只在设置的 cookie 过期时间之前一直有效。
•作用域区别：sessionStorage 不在不同的浏览器窗口中共享，即使是同一个页面； localStorage 和 cookie 在所有同源窗口是共享的
•Web Storage 支持事件通知机制，可以将数据更新的通知发送给监听者。Web Storage 的 api 接口使用更方便。

web storage和cookie的区别
Web Storage的概念和cookie相似，区别是它是为了更大容量存储设计的。Cookie的大小是受限的，并且每次你请求一个新的页面的时候Cookie都会被发送过去，这样无形中浪费了带宽，另外cookie还需要指定作用域，不可以跨域调用。

除此之外，Web Storage拥有setItem,getItem,removeItem,clear等方法，不像cookie需要前端开发者自己封装setCookie，getCookie。

但是Cookie也是不可以或缺的：Cookie的作用是与服务器进行交互，作为HTTP规范的一部分而存在 ，而Web Storage仅仅是为了在本地“存储”数据而生。

Cookies:服务器和客户端都可以访问；大小只有4KB左右；有有效期，过期后将会删除；

本地存储：只有本地浏览器端可访问数据，服务器不能访问本地存储直到故意通过POST或者GET的通道发送到服务器；每个域5MB；没有过期数据，它将保留知道用户从浏览器清除或者使用Javascript代码移除





