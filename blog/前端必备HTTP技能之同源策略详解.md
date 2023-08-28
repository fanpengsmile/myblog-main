---
slug: 前端必备HTTP技能之同源策略详解
title: 前端必备HTTP技能之同源策略详解
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

## 前端必备HTTP技能之同源策略详解
​
**同源策略**在web应用的安全模型中是一个重要概念。在这个策略下，web浏览器允许第一个页面的脚本访问第二个页面里的数据，但是也只有在两个页面有相同的源时。源是由URI，主机名，端口号组合而成的。这个策略可以阻止一个页面上的恶意脚本通过页面的DOM对象获得访问另一个页面上敏感信息的权限。
​
对于普遍依赖于cookie维护授权用户session的现代浏览器来说，这种机制有特殊意义。客户端必须在不同站点提供的内容之间维持一个严格限制，以防丢失数据机密或者完整性。
​
## **历史**
​
同源策略的概念要追溯到1995年的[网景浏览器](https://link.jianshu.com/?t=https://en.wikipedia.org/wiki/Netscape_Navigator_2)。同源策略作为一个重要的安全基石，所有的现代浏览器都在一定程度上实现了同源策略。同源策略虽然不是一个明确规范，但是经常为某些web技术（例如[Microsoft Silverlight](https://link.jianshu.com/?t=https://en.wikipedia.org/wiki/Microsoft_Silverlight),[Adobe Flash](https://link.jianshu.com/?t=https://en.wikipedia.org/wiki/Adobe_Flash),[Adobe Acrobat](https://link.jianshu.com/?t=https://en.wikipedia.org/wiki/Adobe_Acrobat)）或者某些机制（例如XMLHttpRequest）扩展定义大致兼容的安全边界。
​
## **源决定规则**
​
[RFC6454](https://link.jianshu.com/?t=https://tools.ietf.org/html/rfc6454)中有定义URI源的算法定义。对于绝对的URIs，源就是{协议，主机，端口}定义的。只有这些值完全一样才认为两个资源是同源的。
​
为了举例，下面的表格给出了与URL`"http://www.example.com/dir/page.html"`的对比。
​
| 对比URL | 结果 | 结果 |
| --- | --- | --- |
| `http://www.example.com/dir/page2.html` | 同源 | 相同的协议，主机，端口 |
| `http://www.example.com/dir2/other.html` | 同源 | 相同的协议，主机，端口 |
| `http://username:password@www.example.com/dir2/other.html` | 同源 | 相同的协议，主机，端口 |
| `http://www.example.com:81/dir/other.html` | 不同源 | 相同的协议，主机，端口不同 |
| `https://www.example.com/dir/other.html` | 不同源 | 协议不同 |
| `http://en.example.com/dir/other.html` | 不同源 | 不同主机 |
| `http://example.com/dir/other.html` | 不同源 | 不同主机(需要精确匹配) |
| `http://v2.www.example.com/dir/other.html` | 不同源 | 不同主机(需要精确匹配) |
| `http://www.example.com:80/dir/other.html` | 看情况 | 端口明确，依赖浏览器实现 |
​
不像其他浏览器，IE在计算源的时候没有包括端口。
​
## **安全考量**
​
有这种限制的主要原因就是如果没有同源策略将导致安全风险。假设用户在访问银行网站，并且没有登出。然后他又去了任意的其他网站，刚好这个网站有恶意的js代码，在后台请求银行网站的信息。因为用户目前仍然是银行站点的登陆状态，那么恶意代码就可以在银行站点做任意事情。例如，获取你的最近交易记录，创建一个新的交易等等。因为浏览器可以发送接收银行站点的session cookies，在银行站点域上。访问恶意站点的用户希望他访问的站点没有权限访问银行站点的cookie。当然确实是这样的，js不能直接获取银行站点的session cookie，但是他仍然可以向银行站点发送接收附带银行站点session cookie的请求，本质上就像一个正常用户访问银行站点一样。关于发送的新交易，甚至银行站点的CSRF（跨站请求伪造）防护都无能无力，因为脚本可以轻易的实现正常用户一样的行为。所以如果你需要session或者需要登陆时，所有网站都面临这个问题。如果上例中的银行站点只提供公开数据，你就不能触发任意东西，这样的就不会有危险了，这些就是同源策略防护的。当然，如果两个站点是同一个组织的或者彼此互相信任，那么就没有这种危险了。
​
## **规避同源策略**
​
在某些情况下同源策略太严格了，给拥有多个子域的大型网站带来问题。下面就是解决这种问题的技术：
​
##### document.domain属性
​
如果两个window或者frames包含的脚本可以把domain设置成一样的值，那么就可以规避同源策略，每个window之间可以互相沟通。例如，`orders.example.com`下页面的脚本和`catalog.example.com`下页面的脚本可以设置他们的`document.domain`属性为`example.com`，从而让这两个站点下面的文档看起来像在同源下，然后就可以让每个文档读取另一个文档的属性。这种方式也不是一直都有用，因为端口号是在内部保存的，有可能被保存成null。换句话说，`example.com`的端口号80，在我们更新`document.domain`属性的时候可能会变成null。为null的端口可能不被认为是80，这主要依赖浏览器实现。
​
##### 跨域资源共享
前端必备HTTP技能之同源策略详解
同源策略在web应用的安全模型中是一个重要概念。在这个策略下，web浏览器允许第一个页面的脚本访问第二个页面里的数据，但是也只有在两个页面有相同的源时。源是由URI，主机名，端口号组合而成的。这个策略可以阻止一个页面上的恶意脚本通过页面的DOM对象获得访问另一个页面上敏感信息的权限。

对于普遍依赖于cookie维护授权用户session的现代浏览器来说，这种机制有特殊意义。客户端必须在不同站点提供的内容之间维持一个严格限制，以防丢失数据机密或者完整性。

历史
同源策略的概念要追溯到1995年的网景浏览器。同源策略作为一个重要的安全基石，所有的现代浏览器都在一定程度上实现了同源策略。同源策略虽然不是一个明确规范，但是经常为某些web技术（例如Microsoft Silverlight,Adobe Flash,Adobe Acrobat）或者某些机制（例如XMLHttpRequest）扩展定义大致兼容的安全边界。

源决定规则
RFC6454中有定义URI源的算法定义。对于绝对的URIs，源就是{协议，主机，端口}定义的。只有这些值完全一样才认为两个资源是同源的。

为了举例，下面的表格给出了与URL"http://www.example.com/dir/page.html"的对比。

对比URL	结果	结果
http://www.example.com/dir/page2.html	同源	相同的协议，主机，端口
http://www.example.com/dir2/other.html	同源	相同的协议，主机，端口
http://username:password@www.example.com/dir2/other.html	同源	相同的协议，主机，端口
http://www.example.com:81/dir/other.html	不同源	相同的协议，主机，端口不同
https://www.example.com/dir/other.html	不同源	协议不同
http://en.example.com/dir/other.html	不同源	不同主机
http://example.com/dir/other.html	不同源	不同主机(需要精确匹配)
http://v2.www.example.com/dir/other.html	不同源	不同主机(需要精确匹配)
http://www.example.com:80/dir/other.html	看情况	端口明确，依赖浏览器实现
不像其他浏览器，IE在计算源的时候没有包括端口。

安全考量
有这种限制的主要原因就是如果没有同源策略将导致安全风险。假设用户在访问银行网站，并且没有登出。然后他又去了任意的其他网站，刚好这个网站有恶意的js代码，在后台请求银行网站的信息。因为用户目前仍然是银行站点的登陆状态，那么恶意代码就可以在银行站点做任意事情。例如，获取你的最近交易记录，创建一个新的交易等等。因为浏览器可以发送接收银行站点的session cookies，在银行站点域上。访问恶意站点的用户希望他访问的站点没有权限访问银行站点的cookie。当然确实是这样的，js不能直接获取银行站点的session cookie，但是他仍然可以向银行站点发送接收附带银行站点session cookie的请求，本质上就像一个正常用户访问银行站点一样。关于发送的新交易，甚至银行站点的CSRF（跨站请求伪造）防护都无能无力，因为脚本可以轻易的实现正常用户一样的行为。所以如果你需要session或者需要登陆时，所有网站都面临这个问题。如果上例中的银行站点只提供公开数据，你就不能触发任意东西，这样的就不会有危险了，这些就是同源策略防护的。当然，如果两个站点是同一个组织的或者彼此互相信任，那么就没有这种危险了。

规避同源策略
在某些情况下同源策略太严格了，给拥有多个子域的大型网站带来问题。下面就是解决这种问题的技术：

document.domain属性
如果两个window或者frames包含的脚本可以把domain设置成一样的值，那么就可以规避同源策略，每个window之间可以互相沟通。例如，orders.example.com下页面的脚本和catalog.example.com下页面的脚本可以设置他们的document.domain属性为example.com，从而让这两个站点下面的文档看起来像在同源下，然后就可以让每个文档读取另一个文档的属性。这种方式也不是一直都有用，因为端口号是在内部保存的，有可能被保存成null。换句话说，example.com的端口号80，在我们更新document.domain属性的时候可能会变成null。为null的端口可能不被认为是80，这主要依赖浏览器实现。

跨域资源共享
这种方式使用了一个新的Origin请求头和一个新的Access-Control-Allow-Origin响应头扩展了HTTP。允许服务端设置Access-Control-Allow-Origin头标识哪些站点可以请求文件，或者设置Access-Control-Allow-Origin头为"*"，允许任意站点访问文件。浏览器，例如Firefox3.5，Safari4，IE10使用这个头允许跨域HTTP请求。

跨文档通信
这种方式允许一个页面的脚本发送文本信息到另一个页面的脚本中，不管脚本是否跨域。在一个window对象上调用postMessage()会异步的触发window上的onmessage事件，然后触发定义好的事件处理方法。一个页面上的脚本仍然不能直接访问另外一个页面上的方法或者变量，但是他们可以安全的通过消息传递技术交流。

JSONP
JOSNP允许页面接受另一个域的JSON数据，通过在页面增加一个可以从其它域加载带有回调的JSON响应的script标签。

WebSocket
现代浏览器允许脚本直连一个WebSocket地址而不管同源策略。然而，使用WebSocket URI的时候，在请求中插入Origin头就可以标识脚本请求的源。为了确保跨站安全，WebSocket服务器必须根据允许接受请求的白名单中的源列表比较头数据。

个例以及异常
在一些个例中，例如哪些没有明确定义主机名或者端口的协议(file:,data:,等)，同源检查以及相关机制如何运作没有很好的定义。这在历史上导致了很多安全问题，例如任意本地存储的HTML文件不能访问磁盘上的其他文件，也不能与任何网络上的站点通信。

另外，很多遗留的跨域操作，早期是不受同源策略限制的，例如script的跨域请求以及表单POST提交。

最后，某些类型的攻击，例如DNS重新绑定，服务端代理，可以破坏主机名检查，让流氓页面可以直接通过地址与站点通信，尽管地址不是同源的。这种攻击的影响仅限于某些特殊情况下，例如，如果浏览器仍然相信正在通信的攻击者的站点，然后没有公开第三方cookie或者其他敏感信息给攻击者。

变通方法
为了让开发者可以在可控情况下绕过同源策略，一些"hacks"方法可以被用来在不同域的文档之间传输数据，例如fragment identifier ，window.name属性。根据HTML5标准，一个postMessage接口可以实现这样的功能，但是只有最新的浏览器才支持。JSONP也可以用来保证通过类似Ajax的方式访问跨域资源。

做好前端开发必须对HTTP的相关知识有所了解，所以我创建了一个专题前端必备HTTP技能专门收集前端相关的HTTP知识，欢迎关注，投稿。




