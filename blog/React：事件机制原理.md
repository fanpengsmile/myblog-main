---
slug: React：事件机制原理
title: React：事件机制原理
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

## 1.序言

React 有一套自己的事件系统，其事件叫做合成事件。为什么 React 要自定义一套事件系统？React 事件是如何注册和触发的？React 事件与原生 DOM 事件有什么区别？带着这些问题，让我们一起来探究 React 事件机制的原理。为了便于理解，此篇分析将尽可能用图解代替贴 React 源代码进行解析。


## 2.DOM事件流

首先，在正式讲解 React 事件之前，有必要了解一下 DOM 事件流，其包含三个流程：事件捕获阶段、处于目标阶段和事件冒泡阶段。

W3C协会早在1988年就开始了DOM标准的制定，W3C DOM标准可以分为 DOM1、DOM2、DOM3 三个版本。

从 DOM2 开始，DOM 的事件传播分三个阶段进行：事件捕获阶段、处于目标阶段和事件冒泡阶段。


（1）事件捕获阶段、处于目标阶段和事件冒泡阶段

如果点击 p元素，那么 DOM 事件流如下图：

（1）事件捕获阶段：事件对象通过目标节点的祖先 Window 传播到目标的父节点。

（2）处于目标阶段：事件对象到达事件目标节点。如果阻止事件冒泡，那么该事件对象将在此阶段完成后停止传播。

（3）事件冒泡阶段：事件对象以相反的顺序从目标节点的父项开始传播，从目标节点的父项开始到 Window 结束。


（2）addEventListener 方法

DOM 的事件流中同时包含了事件捕获阶段和事件冒泡阶段，而作为开发者，我们可以选择事件处理函数在哪一个阶段被调用。


addEventListener() 方法用于为特定元素绑定一个事件处理函数。addEventListener 有三个参数：

element.addEventListener(event, function, useCapture)



另外，如果一个元素（element）针对同一个事件类型（event），多次绑定同一个事件处理函数（function），那么重复的实例会被抛弃。当然如果第三个参数capture值不一致，此时就算重复定义，也不会被抛弃掉。


## 3.React 事件概述

React 根据W3C 规范来定义自己的事件系统，其事件被称之为合成事件 (SyntheticEvent)。而其自定义事件系统的动机主要包含以下几个方面：

（1）抹平不同浏览器之间的兼容性差异。最主要的动机。

（2）事件"合成"，即事件自定义。事件合成既可以处理兼容性问题，也可以用来自定义事件（例如 React 的 onChange 事件）。

（3）提供一个抽象跨平台事件机制。类似 VirtualDOM 抽象了跨平台的渲染方式，合成事件（SyntheticEvent）提供一个抽象的跨平台事件机制。

（4）可以做更多优化。例如利用事件委托机制，几乎所有事件的触发都代理到了 document，而不是 DOM 节点本身，简化了 DOM 事件处理逻辑，减少了内存开销。（React 自身模拟了一套事件冒泡的机制）

（5）可以干预事件的分发。V16引入 Fiber 架构，React 可以通过干预事件的分发以优化用户的交互体验。


注：「几乎」所有事件都代理到了 document，说明有例外，比如audio、video标签的一些媒体事件（如 onplay、onpause 等），是 document 所不具有，这些事件只能够在这些标签上进行事件进行代理，但依旧用统一的入口分发函数（dispatchEvent）进行绑定。


## 4.事件注册

React 的事件注册过程主要做了两件事：document 上注册、存储事件回调。

（1）document 上注册

在 React 组件挂载阶段，根据组件内的声明的事件类型（onclick、onchange 等），在 document 上注册事件（使用addEventListener），并指定统一的回调函数 dispatchEvent。换句话说，document 上不管注册的是什么事件，都具有统一的回调函数 dispatchEvent。也正是因为这一事件委托机制，具有同样的回调函数 dispatchEvent，所以对于同一种事件类型，不论在 document 上注册了几次，最终也只会保留一个有效实例，这能减少内存开销。


由于 React 的事件委托机制，会指定统一的回调函数 dispatchEvent，所以最终只会在 document 上保留一个 click 事件，类似document.addEventListener('click', dispatchEvent)，从这里也可以看出 React 的事件是在 DOM 事件流的冒泡阶段被触发执行。


（2）存储事件回调

React 为了在触发事件时可以查找到对应的回调去执行，会把组件内的所有事件统一地存放到一个对象中（listenerBank）。而存储方式如上图，首先会根据事件类型分类存储，例如 click 事件相关的统一存储在一个对象中，回调函数的存储采用键值对（key/value）的方式存储在对象中，key 是组件的唯一标识 id，value 对应的就是事件的回调函数。

## 5.事件分发

事件分发也就是事件触发。React 的事件触发只会发生在 DOM 事件流的冒泡阶段，因为在 document 上注册时就默认是在冒泡阶段被触发执行。
其大致流程如下：
触发事件，开始 DOM 事件流，先后经过三个阶段：事件捕获阶段、处于目标阶段和事件冒泡阶段
当事件冒泡到 document 时，触发统一的事件分发函数 ReactEventListener.dispatchEvent
根据原生事件对象（nativeEvent）找到当前节点（即事件触发节点）对应的 ReactDOMComponent 对象
事件的合成
根据当前事件类型生成对应的合成对象
封装原生事件对象和冒泡机制
查找当前元素以及它所有父级
在 listenerBank 中查找事件回调函数并合成到 events 中
批量执行合成事件（events）内的回调函数
如果没有阻止冒泡，会将继续进行 DOM 事件流的冒泡（从 document 到 window），否则结束事件触发


注：上图中阻止冒泡是指调用stopImmediatePropagation 方法阻止冒泡，如果是调用stopPropagation阻止冒泡，document 上如果还注册了同类型其他的事件，也将会被触发执行，但会正常阻断 window 上事件触发。了解两者之间的详细区别

