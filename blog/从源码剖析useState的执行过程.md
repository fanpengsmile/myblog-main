---
slug: 从源码剖析useState的执行过程
title: 从源码剖析useState的执行过程
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

**长文预警，如果觉得前戏太长可直接从第三章开始看~**

本文基于 [React 16.8.6](https://github.com/facebook/react/tree/v16.8.6 "https://github.com/facebook/react/tree/v16.8.6") 进行讲解

使用的示例代码：

```jsx
import React, { useState } from 'react'
import './App.css'

export default function App() {
  
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Star');
  
  // 调用三次setCount便于查看更新队列的情况
  const countPlusThree = () => {
    setCount(count+1);
    setCount(count+2);
    setCount(count+3);
  }
  return (
    <div className='App'>
      <p>{name} Has Clicked <strong>{count}</strong> Times</p>
      <button onClick={countPlusThree}>Click *3</button>
    </div>
  )
}
```

代码非常简单，点击button使count+3，count的值会显示在屏幕上。

![Jietu20190419-090633@2x.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5bf9094d7f0~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

## 一. 前置知识

### 1\. 函数组件和类组件

> 本节参考：[How Are Function Components Different from Classes?](https://overreacted.io/how-are-function-components-different-from-classes/ "https://overreacted.io/how-are-function-components-different-from-classes/")

**本节主要概念：**

+   函数组件和类组件的区别
+   React如何区分这两种组件

我们来看一个简单的Greeting组件，它支持定义成类和函数两种性质。在使用它时，不用关心他是如何定义的。

```jsx
// 是类还是函数 —— 无所谓
<Greeting />  // <p>Hello</p>
```

如果 `Greeting` 是一个函数，React 需要调用它。

```jsx
// Greeting.js
function Greeting() {
  return <p>Hello</p>;
}

// React 内部
const result = Greeting(props); // <p>Hello</p>
```

但如果 `Greeting` 是一个类，React 需要先将其实例化，再调用刚才生成实例的 `render` 方法：

```jsx
// Greeting.js
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// React 内部
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

**React通过以下方式来判断组件的类型：**

```jsx
// React 内部
class Component {}
Component.prototype.isReactComponent = {};

// 检查方式
class Greeting extends React.Component {}
console.log(Greeting.prototype.isReactComponent); // {}
```

### 2\. React Fiber

> 本节参考：[A cartoon intro to fiber](https://www.youtube.com/watch?v=ZCuYPiUIONs&list=PLb0IAmt7-GS3fZ46IGFirdqKTIxlws7e0&index=5 "https://www.youtube.com/watch?v=ZCuYPiUIONs&list=PLb0IAmt7-GS3fZ46IGFirdqKTIxlws7e0&index=5")

**本节主要概念（了解即可）：**

+   React现在的渲染都是由Fiber来调度
+   Fiber调度过程中的两个阶段(以Render为界)

Fiber（可译为丝）比线程还细的控制粒度，是React 16中的新特性，旨在对渲染过程做更精细的调整。

**产生原因：**

1.  Fiber之前的reconciler（被称为Stack reconciler）自顶向下的递归`mount/update`，无法中断（持续占用主线程），这样主线程上的布局、动画等周期性任务以及交互响应就无法立即得到处理，影响体验
2.  渲染过程中没有优先级可言

**React Fiber的方式：**

把一个耗时长的任务分成很多小片，每一个小片的运行时间很短，虽然总时间依然很长，但是在每个小片执行完之后，都给其他任务一个执行的机会，这样唯一的线程就不会被独占，其他任务依然有运行的机会。

React Fiber把更新过程碎片化，执行过程如下面的图所示，每执行完一段更新过程，就把控制权交还给React负责任务协调的模块，看看有没有其他紧急任务要做，如果没有就继续去更新，如果有紧急任务，那就去做紧急任务。

维护每一个分片的数据结构，就是Fiber。

有了分片之后，更新过程的调用栈如下图所示，中间每一个波谷代表深入某个分片的执行过程，每个波峰就是一个分片执行结束交还控制权的时机。让线程处理别的事情  

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5c351497594~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

**Fiber的调度过程分为以下两个阶段：**

**render/reconciliation阶段** — 里面的所有生命周期函数都可能被执行多次，所以尽量保证状态不变

+   componentWillMount
+   componentWillReceiveProps
+   shouldComponentUpdate
+   componentWillUpdate

**Commit阶段** — 不能被打断，只会执行一次

+   componentDidMount
+   componentDidUpdate
+   compoenntWillunmount

Fiber的增量更新需要更多的上下文信息，之前的vDOM tree显然难以满足，所以扩展出了fiber tree（即Fiber上下文的vDOM tree），更新过程就是根据输入数据以及现有的fiber tree构造出新的fiber tree（workInProgress tree）

与Fiber有关的所有代码位于[packages/react-reconciler](https://github.com/facebook/react/tree/v16.8.6/packages/react-reconciler "https://github.com/facebook/react/tree/v16.8.6/packages/react-reconciler")中，一个Fiber节点的详细定义如下：

```javascript
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // Instance
  this.tag = tag; this.key = key; this.elementType = null; 
  this.type = null; this.stateNode = null;

  // Fiber
  this.return = null; this.child = null; this.sibling = null; 
  this.index = 0; this.ref = null; this.pendingProps = pendingProps;
  this.memoizedProps = null; this.updateQueue = null;
  
  // 重点
  this.memoizedState = null;
  
  this.contextDependencies = null; this.mode = mode;

  // Effects
  /** 细节略 **/
}
```

我们只关注一下`this.memoizedState`

**这个`key`用来存储在上次渲染过程中最终获得的节点的`state`，每次`render`之前，React会计算出当前组件最新的`state`然后赋值给组件，再执行`render`。**— 类组件和使用useState的函数组件均适用。

记住上面这句话，后面还会经常提到memoizedState

> 有关Fiber每个key的具体含义可以参见[源码的注释](https://github.com/facebook/react/blob/487f4bf2ee7c86176637544c5473328f96ca0ba2/packages/react-reconciler/src/ReactFiber.js#L84-L218 "https://github.com/facebook/react/blob/487f4bf2ee7c86176637544c5473328f96ca0ba2/packages/react-reconciler/src/ReactFiber.js#L84-L218")

### 3\. React渲染器与setState

> 本节参考：[How Does setState Know What to Do?](https://overreacted.io/how-does-setstate-know-what-to-do/ "https://overreacted.io/how-does-setstate-know-what-to-do/")

**本节主要概念：**

+   React渲染器是什么
+   setState为什么能够触发更新

**由于React体系的复杂性以及目标平台的多样性。`react`包只暴露一些定义组件的API。绝大多数React的实现都存在于 渲染器（renderers）中。**

`react-dom`、`react-dom/server`、 `react-native`、 `react-test-renderer`、 `react-art`都是常见的渲染器

这就是为什么不管目标平台是什么，`react`包都是可用的。从`react`包中导出的一切，比如`React.Component`、`React.createElement`、 `React.Children` 和 `Hooks`都是独立于目标平台的。无论运行React DOM，还是 React DOM Server,或是 React Native，组件都可以使用同样的方式导入和使用。

所以当我们想使用新特性时，`react` 和 `react-dom`都需要被更新。

> 例如，当React 16.3添加了Context API，`React.createContext()`API会被React包暴露出来。 但是`React.createContext()` 其实并没有\_实现\_ context。因为在React DOM 和 React DOM Server 中同样一个 API 应当有不同的实现。所以`createContext()`只返回了一些普通对象： \*\*所以，如果你将react升级到了16.3+，但是不更新react-dom，那么你就使用了一个尚不知道Provider 和 Consumer类型的渲染器。\*\*这就是为什么老版本的`react-dom`会[报错说这些类型是无效的](https://stackoverflow.com/a/49677020/458193 "https://stackoverflow.com/a/49677020/458193")。

这就是`setState` 尽管定义在React包中，调用时却能够更新DOM的原因。它读取由React DOM设置的`this.updater`，让React DOM安排并处理更新。

```js
Component.setState = function(partialState, callback) {
  // setState所做的一切就是委托渲染器创建这个组件的实例
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

各个渲染器中的updater触发不同平台的更新渲染

```js
// React DOM 内部
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMUpdater;

// React DOM Server 内部
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMServerUpdater;

// React Native 内部
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactNativeUpdater;
```

至于updater的具体实现，就不是这里重点要讨论的内容了，下面让我们正式进入本文的主题：React Hooks

## 二. 了解useState

### 1\. useState的引入和触发更新

**本节主要概念：**

+   useState是如何被引入以及调用的
+   useState为什么能触发组件更新

所有的Hooks在`React.js`中被引入，挂载在React对象中

```javascript
// React.js
import {
  useCallback,
  useContext,
  useEffect,
  useImperativeHandle,
  useDebugValue,
  useLayoutEffect,
  useMemo,
  useReducer,
  useRef,
  useState,
} from './ReactHooks';
```

我们进入`ReactHooks.js`来看看，发现`useState`的实现竟然异常简单，只有短短两行

```javascript
// ReactHooks.js
export function useState<S>(initialState: (() => S) | S) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

看来重点都在这个`dispatcher`上，`dispatcher`通过`resolveDispatcher()`来获取，这个函数同样也很简单，只是将`ReactCurrentDispatcher.current`的值赋给了`dispatcher`

```javascript
// ReactHooks.js
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher;
}
```

所以`useState(xxx)` 等价于 `ReactCurrentDispatcher.current.useState(xxx)`

看到这里，我们回顾一下第一章第三小节所讲的React渲染器与setState，是不是发现有点似曾相识。

与updater是setState能够触发更新的核心类似，`ReactCurrentDispatcher.current.useState`是`useState`能够触发更新的关键原因，这个方法的实现并不在react包内。下面我们就来分析一个具体更新的例子。

### 2\. 示例分析

以全文开头给出的代码为例。

我们从Fiber调度的开始：`ReactFiberBeginwork`来谈起

之前已经说过，React有能力区分不同的组件，所以它会给不同的组件类型打上不同的tag， 详见[shared/ReactWorkTags.js](https://github.com/facebook/react/blob/v16.8.6/packages/shared/ReactWorkTags.js "https://github.com/facebook/react/blob/v16.8.6/packages/shared/ReactWorkTags.js")。

所以在beginWork的函数中，就可以根据workInProgess(就是个Fiber节点)上的tag值来走不同的方法来加载或者更新组件。

```javascript
// ReactFiberBeginWork.js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  /** 省略与本文无关的部分 **/

  // 根据不同的组件类型走不同的方法
  switch (workInProgress.tag) {
    // 不确定组件
    case IndeterminateComponent: {
      const elementType = workInProgress.elementType;
      // 加载初始组件
      return mountIndeterminateComponent(
        current,
        workInProgress,
        elementType,
        renderExpirationTime,
      );
    }
    // 函数组件
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      // 更新函数组件
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      );
    }
    // 类组件
    case ClassComponent {
      /** 细节略 **/
  	}
  }
```

下面我们来找出useState发挥作用的地方。

#### 2.1 第一次加载

mount过程执行`mountIndeterminateComponent`时，会执行到`renderWithHooks`这个函数

```javascript
function mountIndeterminateComponent(
  _current,
  workInProgress,
  Component,
  renderExpirationTime,
) {
 
 /** 省略准备阶段代码 **/ 
  
  // value就是渲染出来的APP组件
  let value;

  value = renderWithHooks(
    null,
    workInProgress,
    Component,
    props,
    context,
    renderExpirationTime,
  );
  /** 省略无关代码 **/ 
  }
  workInProgress.tag = FunctionComponent;
  reconcileChildren(null, workInProgress, value, renderExpirationTime);
  return workInProgress.child;
}
```

**执行前：** nextChildren = value

![11.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5d14a94b351~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

**执行后：** value= 组件的虚拟DOM表示

![12.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5d14a94b351~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

**至于这个value是如何被渲染成真实的DOM节点，我们并不关心，state值我们已经通过renderWithHooks取到并渲染**

#### 2.2 更新

点击一下按钮：此时count从0变为3

更新过程执行的是updateFunctionComponent函数，同样会执行到renderWithHooks这个函数，我们来看一下这个函数执行前后发生的变化：

**执行前：** nextChildren = undefined

![13.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5d7c00bb593~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

\*\*执行后：\*\*nextChildren=更新后的组件的虚拟DOM表示

![14.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5d9fb21e17c~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

**同样的，至于这个nextChildren是如何被渲染成真实的DOM节点，我们并不关心，最新的state值我们已经通过renderWithHooks取到并渲染**

**所以，`renderWithHooks`函数就是处理各种hooks逻辑的核心部分**

## 三. 核心步骤分析

[ReactFiberHooks.js](https://github.com/facebook/react/blob/v16.8.6/packages/react-reconciler/src/ReactFiberHooks.js "https://github.com/facebook/react/blob/v16.8.6/packages/react-reconciler/src/ReactFiberHooks.js")包含着各种关于Hooks逻辑的处理，本章中的代码均来自该文件。

### 1\. Hook对象

在之前的章节有介绍过，Fiber中的`memorizedStated`用来存储state

在类组件中`state`是一整个对象，可以和`memoizedState`一一对应。但是在`Hooks`中，React并不知道我们调用了几次`useState`，**所以React通过将一个Hook对象挂载在`memorizedStated`上来保存函数组件的`state`**

Hook对象的结构如下：

```javascript
// ReactFiberHooks.js
export type Hook = {
  memoizedState: any, 

  baseState: any,    
  baseUpdate: Update<any, any> | null,  
  queue: UpdateQueue<any, any> | null,  

  next: Hook | null, 
};
```

重点关注`memoizedState`和`next`

+   `memoizedState`是用来记录当前`useState`应该返回的结果的
+   `queue`：缓存队列，存储多次更新行为
+   `next`：指向下一次`useState`对应的Hook对象。

结合示例代码来看：

```jsx
import React, { useState } from 'react'
import './App.css'

export default function App() {
  
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Star');
  
  // 调用三次setCount便于查看更新队列的情况
  const countPlusThree = () => {
    setCount(count+1);
    setCount(count+2);
    setCount(count+3);
  }
  return (
    <div className='App'>
      <p>{name} Has Clicked <strong>{count}</strong> Times</p>
      <button onClick={countPlusThree}>Click *3</button>
    </div>
  )
}
```

第一次点击按钮触发更新时，memoizedState的结构如下

![Jietu20190419-100634@2x.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5de6dd38821~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

只是符合之前对Hook对象结构的分析，只是queue中的结构貌似有点奇怪，我们将在第三章第2节中进行分析。

### 2\. renderWithHooks

renderWithHooks的运行过程如下：

```javascript
// ReactFiberHooks.js
export function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  props: any,
  refOrContext: any,
  nextRenderExpirationTime: ExpirationTime,
): any {
  renderExpirationTime = nextRenderExpirationTime;
  currentlyRenderingFiber = workInProgress;
 
  // 如果current的值为空，说明还没有hook对象被挂载
  // 而根据hook对象结构可知，current.memoizedState指向下一个current
  nextCurrentHook = current !== null ? current.memoizedState : null;

  // 用nextCurrentHook的值来区分mount和update，设置不同的dispatcher
  ReactCurrentDispatcher.current =
      nextCurrentHook === null
      // 初始化时
        ? HooksDispatcherOnMount
  		// 更新时
        : HooksDispatcherOnUpdate;
  
  // 此时已经有了新的dispatcher,在调用Component时就可以拿到新的对象
  let children = Component(props, refOrContext);
  
  // 重置
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  const renderedWork: Fiber = (currentlyRenderingFiber: any);

  // 更新memoizedState和updateQueue
  renderedWork.memoizedState = firstWorkInProgressHook;
  renderedWork.updateQueue = (componentUpdateQueue: any);
  
   /** 省略与本文无关的部分代码，便于理解 **/
}
```

#### 2.1 初始化时

**核心：** 创建一个新的hook，初始化state， 并绑定触发器

初始化阶段`ReactCurrentDispatcher.current` 会指向`HooksDispatcherOnMount` 对象

```js
// ReactFiberHooks.js

const HooksDispatcherOnMount: Dispatcher = {
/** 省略其它Hooks **/
  useState: mountState,
};

// 所以调用useState(0)返回的就是HooksDispatcherOnMount.useState(0)，也就是mountState(0)
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
    // 访问Hook链表的下一个节点，获取到新的Hook对象
  const hook = mountWorkInProgressHook();
//如果入参是function则会调用，但是不提供参数
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
// 进行state的初始化工作
  hook.memoizedState = hook.baseState = initialState;
// 进行queue的初始化工作
  const queue = (hook.queue = {
    last: null,
    dispatch: null,
    eagerReducer: basicStateReducer, // useState使用基础reducer
    eagerState: (initialState: any),
  });
	// 返回触发器
  const dispatch: Dispatch<BasicStateAction<S>,> 
    = (queue.dispatch = (dispatchAction.bind(
    	null,
    	//绑定当前fiber结点和queue
    	((currentlyRenderingFiber: any): Fiber),
    	queue,
  ));
  // 返回初始state和触发器
  return [hook.memoizedState, dispatch];
}

// 对于useState触发的update action来说（假设useState里面都传的变量），basicStateReducer就是直接返回action的值
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

重点讲一下返回的这个更新函数 `dispatchAction`

```js
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {

   /** 省略Fiber调度相关代码 **/
  
  // 创建新的新的update, action就是我们setCount里面的值(count+1, count+2, count+3…)
    const update: Update<S, A> = {
      expirationTime,
      action,
      eagerReducer: null,
      eagerState: null,
      next: null,
    };
	  
    // 重点：构建query
    // queue.last是最近的一次更新，然后last.next开始是每一次的action
    const last = queue.last;
    if (last === null) {
      // 只有一个update, 自己指自己-形成环
      update.next = update;
    } else {
      const first = last.next;
      if (first !== null) {
        
        update.next = first;
      }
      last.next = update;
    }
    queue.last = update;

    /** 省略特殊情况相关代码 **/
    
    // 创建一个更新任务
    scheduleWork(fiber, expirationTime);

}
```

在`dispatchAction`中维护了一份query的数据结构。

query是一个有环链表，规则：

+   query.last指向最近一次更新
+   last.next指向第一次更新
+   后面就依次类推，最终倒数第二次更新指向last，形成一个环。

**所以每次插入新update时，就需要将原来的first指向query.last.next。再将update指向query.next，最后将query.last指向update.**

下面结合示例代码来画图说明一下：

前面给出了第一次点击按钮更新时，memorizedState中的query值  

![Jietu20190419-130310@2x.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5e40929acc0~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

其构建过程如下图所示：

![Jietu20190430-155220@2x.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5e851fc2ee3~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

**即保证query.last始终为最新的action, 而query.last.next始终为action: 1**

#### 2.2 更新时

**核心**：获取该Hook对象中的 queue，内部存有本次更新的一系列数据，进行更新

更新阶段 `ReactCurrentDispatcher.current` 会指向`HooksDispatcherOnUpdate`对象

```javascript
// ReactFiberHooks.js

// 所以调用useState(0)返回的就是HooksDispatcherOnUpdate.useState(0)，也就是updateReducer(basicStateReducer, 0)

const HooksDispatcherOnUpdate: Dispatcher = {
  /** 省略其它Hooks **/
   useState: updateState,
}

function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState);
}

// 可以看到updateReducer的过程与传的initalState已经无关了，所以初始值只在第一次被使用

// 为了方便阅读，删去了一些无关代码
// 查看完整代码：https://github.com/facebook/react/blob/487f4bf2ee7c86176637544c5473328f96ca0ba2/packages/react-reconciler/src/ReactFiberHooks.js#L606
function updateReducer(reducer, initialArg, init) {
// 获取初始化时的 hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  // 开始渲染更新
  if (numberOfReRenders > 0) {
    const dispatch = queue.dispatch;
    if (renderPhaseUpdates !== null) {
      // 获取Hook对象上的 queue，内部存有本次更新的一系列数据
      const firstRenderPhaseUpdate = renderPhaseUpdates.get(queue);
      if (firstRenderPhaseUpdate !== undefined) {
        renderPhaseUpdates.delete(queue);
        let newState = hook.memoizedState;
        let update = firstRenderPhaseUpdate;
        // 获取更新后的state
        do {
          const action = update.action;
          // 此时的reducer是basicStateReducer，直接返回action的值
          newState = reducer(newState, action);
          update = update.next;
        } while (update !== null);
        // 对 更新hook.memoized 
        hook.memoizedState = newState;
        // 返回新的 state，及更新 hook 的 dispatch 方法
        return [newState, dispatch];
      }
    }
  }
  
// 对于useState触发的update action来说（假设useState里面都传的变量），basicStateReducer就是直接返回action的值
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

#### 2.3 总结

单个hooks的更新行为全都挂在Hooks.queue下，所以能够管理好queue的核心就在于

+   初始化queue - mountState
+   维护queue - dispatchAction
+   更新queue - updateReducer

**结合示例代码：**

+   当我们第一次调用`[count, setCount] = useState(0)`时，创建一个queue
+   每一次调用`setCount(x)`，就dispach一个内容为x的action（action的表现为：将count设为x)，action存储在queue中，以前面讲述的有环链表规则来维护
+   这些action最终在`updateReducer`中被调用，更新到`memorizedState`上，使我们能够获取到最新的state值。

## 四. 总结

### 1\. 对官方文档中Rules of Hooks的理解

[官方文档](https://reactjs.org/docs/hooks-rules.html "https://reactjs.org/docs/hooks-rules.html")对于使用hooks有以下两点要求：

![Jietu20190418-094347@2x.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/4/30/16a6d5ec8ed212b5~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

#### 2.1 为什么不能在循环/条件语句中执行

**以useState为例：**

和类组件存储state不同，React并不知道我们调用了几次`useState`，对hooks的存储是按顺序的(参见Hook结构)，一个hook对象的next指向下一个hooks。所以当我们建立示例代码中的对应关系后，Hook的结构如下：

```javascript
// hook1: const [count, setCount] = useState(0) — 拿到state1
{
  memorizedState: 0
  next : {
    // hook2: const [name, setName] = useState('Star') - 拿到state2
    memorizedState: 'Star'
    next : {
      null
    }
  }
}

// hook1 => Fiber.memoizedState
// state1 === hook1.memoizedState
// hook1.next => hook2
// state2 === hook2.memoizedState
```

所以如果把hook1放到一个if语句中，当这个没有执行时，hook2拿到的state其实是上一次hook1执行后的state（而不是上一次hook2执行后的）。这样显然会发生错误。

> 关于这块内容如果想了解更多可以看一下[这篇文章](https://medium.com/the-guild/under-the-hood-of-reacts-hooks-system-eb59638c9dba "https://medium.com/the-guild/under-the-hood-of-reacts-hooks-system-eb59638c9dba")

#### 2.2 为什么只能在函数组件中使用hooks

只有函数组件的更新才会触发renderWithHooks函数，处理Hooks相关逻辑。

**还是以setState为例，类组件和函数组件重新渲染的逻辑不同 ：**

**类组件：** 用setState触发updater，重新执行组件中的render方法

**函数组件：** 用useState返回的setter函数来dispatch一个update action，触发更新(dispatchAction最后的scheduleWork)，用updateReducer处理更新逻辑，返回最新的state值(与Redux比较像)

### 2\. useState整体运作流程总结

说了这么多，最后再简要总结下useState的执行流程~

**初始化：** 构建dispatcher函数和初始值

**更新时：**

1.  调用dispatcher函数，按序插入update(其实就是一个action)
2.  收集update，调度一次React的更新
3.  在更新的过程中将`ReactCurrentDispatcher.current`指向负责更新的Dispatcher
4.  执行到函数组件App()时，`useState`会被重新执行，在resolve dispatcher的阶段拿到了负责更新的dispatcher。
5.  `useState`会拿到Hook对象，`Hook.query`中存储了更新队列，依次进行更新后，即可拿到最新的state
6.  函数组件App()执行后返回的nextChild中的count值已经是最新的了。FiberNode中的`memorizedState`也被设置为最新的state
7.  Fiber渲染出真实DOM。更新结束。




