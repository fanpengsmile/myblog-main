---
slug: web前端安全攻击与防御
title: web前端安全攻击与防御
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---
## 前后端安全分类：

1、前端安全：发生在浏览器、单页面应用、web页面的安全问题，比如跨站脚本攻击xss就是前端安全问题

2、后端安全：发生在后端服务器、应用、服务当中的安全问题，比如：SQL注入漏洞发生在后端

## 前端安全攻击手段

1、XSS攻击

2、CSRF攻击

3、点击劫持

4、iframe带来的风险

5、不安全的第三方依赖包

## XSS攻击

XSS 即（Cross Site Scripting）中文名称为：跨站脚本攻击。恶意攻击者在web页面中会插入一些恶意的script代码。当目标网站、目标用户浏览器渲染HTML文档的过程中，出现了不被预期的脚本指令并执行时，XSS就发生了，因此达到恶意攻击用户的目的。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3bf398db3144~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

                                                           xss攻击示意图

根据攻击的来源，XSS 攻击可分为反射型、存储型和 DOM 型三种：

反射型XSS：发出请求时，XSS代码出现在URL中，作为输入提交到服务器端，服务器端解析后响应，XSS代码随响应内容一起传回给浏览器，最后浏览器解析执行XSS代码。这个过程像一次反射，故叫反射型XSS。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3bf91ef38b4c~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

反射型XSS示意图

1、用户A给用户B发送一个恶意构造了Web的URL。

2、用户B点击并查看了这个URL。

3、用户B获取到一个具有漏洞的HTML页面并显示在本地浏览器中。

4、漏洞HTML页面执行恶意JavaScript脚本，将用户B信息盗取发送给用户A，或者篡改用户B看到的数据等

#### 反射型XSS防御：

我们可以通过html转义来防范，最好是采用成熟的转义库处理

攻击者盗用了你的身份，以你的名义发送恶意请求，对服务器来说这个请求是完全合法的，但是却完成了攻击者所期望的一个操作，比如以你的名义发送邮件、发消息，盗取你的账号，添加系统管理员，甚至于购买商品、虚拟货币转账等。 如下：其中Web A为存在CSRF漏洞的网站，Web B为攻击者构建的恶意网站，User C为Web A网站的合法用户

存储型XSS：存储型XSS和反射型XSS的差别仅在于，提交的代码会存储在服务器端，下次请求目标页面时不用再提交XSS代码，这样，每一个访问特定网页的用户都会被攻击

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3c00c9a1faa1~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

存储型XSS示意图

1、用户A在网页上创建了某个账户，并且账户信息中包含XSS代码。

2、用户B访问该网站查看XSS代码账户详情页面。

3、服务端返回账户详情页面，和带XSS账户信息。

4、用户B浏览器执行XSS代码，将用户B信息盗取发送给用户A，或者篡改用户B看到的数据等。

#### 存储型XSS攻击防范：

1\. 后端需要对提交的数据进行过滤。

2\. 前端也可以做一下处理方式，比如对script标签，将特殊字符替换成HTML编码这些等。

DOM-based型XSS：DOM型XSS是基于DOM文档对象模型的一种漏洞,通过 HTML DOM，通过植入js代码，造成dom的更改，因此造成了XSS-DOM漏洞，所以DOM类型的XSS可能是反射型也可能是储存型

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3c0414b44ce4~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

DOM型XSS示意图

1、用户B访问网站url中带有XSS代码参数。

2、浏览器下载该网站JavaScript脚本。

3、JavaScript脚本有个方法获取URL中XSS代码参数，并且用innerHtml渲染在dom中。

4、触发XSS代码，造成XSS攻击，cookie数据失窃。

DOM 型 XSS 攻击，实际上就是网站前端 JavaScript 代码本身不够严谨，把不可信的数据当作代码执行了。

#### DOM 型 XSS防御：

1、在使用 .innerHTML、.outerHTML、document.write() 时要特别小心，不要把不可信的数据作为 HTML 插到页面上，而应尽量使用 .textContent、.setAttribute() 等。

2、在使用 .innerHTML、.outerHTML、document.write() 时要特别小心，不要把不可信的数据作为 HTML 插到页面上，而应尽量使用 .textContent、.setAttribute() 等。

3、在使用 .innerHTML、.outerHTML、document.write() 时要特别小心，不要把不可信的数据作为 HTML 插到页面上，而应尽量使用 .textContent、.setAttribute() 等。

4、Http Only cookie

5、对产品输入要求格式严谨检查过滤

## CSRF攻击

攻击者盗用了你的身份，以你的名义发送恶意请求，对服务器来说这个请求是完全合法的，但是却完成了攻击者所期望的一个操作，比如以你的名义发送邮件、发消息，盗取你的账号，添加系统管理员，甚至于购买商品、虚拟货币转账等。 如下：其中Web A为存在CSRF漏洞的网站，Web B为攻击者构建的恶意网站，User C为Web A网站的合法用户

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3c087aaf5013~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

CSRF攻击示意图

#### CSRF攻击防御：

1、验证 HTTP Referer 字段

2、使用 token验证

3、显示验证方式：添加验证码、密码等

4、涉及到数据修改操作严格使用 post 请求而不是 get 请求

## 点击劫持攻击：

点击劫持是一种视觉上的欺骗手段。攻击者使用一个透明的、不可见的iframe，覆盖在一个网页上，然后诱使用户在网页上进行操作，此时用户将在不知情的情况下点击透明的iframe页面。通过调整iframe页面的位置，可以诱使用户恰好点击在iframe页面的一些功能性按钮上。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/13/170d3c0d54171f64~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)  

点击劫持示意图

一个简单的点击劫持例子，就是当你点击了一个不明链接之后，自动关注了某一个人的博客或者订阅了视频

#### 点击劫持防御

**1****、****X-Frame-Options****浏览器机制：**X-Frame-Options HTTP响应头是用来给浏览器指示允许一个页面能否在frame、iframe、object中展现的标记，有三个可选的值：**DENY**：浏览器会拒绝当前页面加载任何frame页面（即使是相同域名的页面也不允许）**SAMEORIGIN**：允许加载frame页面，但是frame页面的地址只能为同源域名下的页面**ALLOW-FROM****：**可以加载指定来源的frame页面（可以定义frame页面的地址）但这个缺陷就是chrome、Safari是不支持ALLOW-FROM语法！

**2****、使用认证码认证用户：**点击劫持漏洞通过伪造网站界面进行攻击，网站开发人员可以通过认证码识别用户，确定是用户发出的点击命令才执行相应操作。识别用户的方法中最有效的方法是认证码认证。例如，在网站上广泛存在的发帖认证码，要求用户输入图形中的字符，输入某些图形的特征等

**3****、 使用** **FrameBusting** **代码“：**使用 JavaScript 脚本阻止恶意网站载入网页。如果检测到网页被非法网页载入，就执行自动跳转功能。如果用户浏览器禁用JavaScript脚本，那么FrameBusting代码也无法正常运行。所以，该类代码只能提供部分保障功能

## iframe 带来的风险

有些时候我们的前端页面需要用到第三方提供的页面组件，通常会以iframe的方式引入。典型的例子是使用iframe在页面上添加第三方提供的广告、天气预报、社交分享插件等等

iframe本身不受我们控制，那么如果iframe中的域名因为过期而被恶意攻击者抢注，或者第三方被黑客攻破，iframe中的内容被替换掉了，从而利用用户浏览器中的安全漏洞下载安装木马、恶意勒索软件等等，这问题可就大了

#### iframe防御：

iframe有了一个叫做sandbox的安全属性，通过它可以对iframe的行为进行各种限制，在 iframe 元素中添加上这个关键词就行，另外，sandbox也提供了丰富的配置参数，我们可以进行较为细粒度的控制。一些典型的参数如下：

**allow-forms****：**允许iframe中提交form表单

**allow-popups****：**允许iframe中弹出新的窗口或者标签页（例如，window.open()，showModalDialog()，target=”\_blank”等等）

**allow-scripts****：**允许iframe中执行JavaScript

**allow-same-origin****：**允许iframe中的网页开启同源策略

如果你只是添加上这个属性而保持属性值为空，那么浏览器将会对 iframe 实施史上最严厉的调控限制

## 不安全的第三方依赖

项目里面使用了很多第三方的依赖，不论应用自己的代码的安全性有多高，如果这些来自第三方的代码有安全漏洞，那么对应用整体的安全性依然会造成严峻的挑战。jQuery就存在多个已知安全漏洞，Node.js也有一些已知的安全漏洞

#### 第三方依赖包防御：

手动检查这些第三方代码有没有安全问题是个苦差事，主要是因为应用依赖的这些组件数量众多，手工检查太耗时，有自动化的工具可以使用，比如NSP(Node Security Platform)，Snyk、sonarQubej检测工具等等

## vue对前端安全的处理：Vue 的安全措施

当然还存在很多别的攻击手段，比如：

Https 也可能存在的风险（强制让HTTPS降级回HTTP，从而继续进行中间人攻击）

本地存储数据泄露（尽可能不在前端存储任何敏感、机密的数据）

cdn劫持（攻击者劫持了CDN，或者对CDN中的资源进行了污染）




