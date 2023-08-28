---
slug: Redux原理解析与实现
title: Redux原理解析与实现
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

## Redux

对于 SPA 应用来说，前端所需要管理的状态渐渐增多，需要查询、更新、传递的状态也渐渐增多，如果让每个组件都存储自身相关的状态，理论上是不影响应用运行的，但是在开发以及后续升级维护阶段，我们将花费大量的精力去查询状态的变化过程，在多组合组件通信或客户端与服务端有较多交互过程中，我们往往需要去更新，维护并监听每一个组件的状态。

在这种情况下，如果有一种可以对状态做集中管理的地方是不是会更好呢？状态管理好比是一个集中在一起的总的配置箱，当需要更新状态的时候，我们仅对这个配置箱镜进行输入，而不用去关心开关状态是如何分发到每个组件内部的，这样开发者能把更多精力放在业务逻辑上。 今天我们来了详细了解下`redux`这个库，看看它能帮助我们干些什么....

## 什么是 Redux ？

“Redux is a pattern and library for managing and updating application state, using events called "actions". It serves as a centralized store for state that needs to be used across your entire application, with rules ensuring that the state can only be updated in a predictable fashion.”

简单意思就是；`Redux` 是一个有用的架构，用操作的事件来管理和更新应用的状态，在整个应用中，它用于状态集中存储，状态的更新必须是一种可预测的方式更新。

## 为什么要用 Redux ?

官方解释：" It helps to understand what this "Redux" thing is in the first place. What does it do? What problems does it help me solve? Why would I want to use it?

Redux is a pattern and library for managing and updating application state, using events called "actions". It serves as a centralized store for state that needs to be used across your entire application, with rules ensuring that the state can only be updated in a predictable fashion. "

其实，状态管理不是必须的，当你UI层比较简单或者没有较多的交互需要去改变状态的场景下，使用状态管理方式反而提高项目的复杂度，`Redux`作者 Daniel Abramov 就说过”只有遇到`React`实在解决不了的问题，你才需要`Redux`。

## 何时使用redux?

1.多交互，多数据源等场景：

+   用户的使用方式复杂
+   不同身份的用户有不同的使用方式（比如普通用户和管理员）
+   多个用户之间可以协作
+   与服务器大量交互，或者使用了`WebSocket`
+   View要从多个来源获取数据

2.组件角度，以下场景可考虑`Redux`：

+   某个组件的状态，需要共享
+   某个组件状态需要在任何地方都可以拿到
+   一个组件需要改变全局状态
+   一个组件需要改变另一个组件的状态

发生上面情况时，如果不使用`Redux`或者其他状态管理工具，不按照一定规律处理状态的读写，代码很快就会变成一团乱麻。你需要一种机制，可以在同一个地方查询状态、改变状态、传播状态的变化。 另外，本篇文章更关注业务模型层的数据流，业务模型是指所处领域的业务数据、规则、流程的集合。即使抛开所有展示层，这一层也可以不依赖于展示层而独立运行，这里要强调一点，`Redux`之类的状态管理库充当了一个应用的业务模型层，并不会受限于如`React`之类的`View`层。假如你已经明白了`Redux`的定位及应用场景的话，我们来对其原理一探究竟。

## 设计思想

`Redux`的设计思想很简单，用阮老师的两句话：

+   `Web`应用是一个状态机，视图与状态是一一对应的。
+   所有的状态，保存在一个对象里面。

## Redux的三大原则

+   单一数据源
    
    整个应用的`state` 被储存在一棵`object tree`中，并且这个`object tree`只存在于唯一一个`store`中。
    
    这让同构应用开发变得非常容易。来自服务端的`state`可以在无需编写更多代码的情况下被序列化并注入到客户端中。由于是单一的 `state tree`，调试也变得非常容易。在开发中，你可以把应用的`state`保存在本地，从而加快开发速度。此外，受益于单一的`state tree` ，以前难以实现的如“撤销/重做”这类功能也变得轻而易举。
    
+   State是只读的
    
    唯一改变`state`的方法就是触发`action`，`action` 是一个用于描述已发生事件的普通对象。 这样确保了视图和网络请求都不能直接修改`state`，相反它们只能表达想要修改的意图。因为所有的修改都被集中化处理，且严格按照一个接一个的顺序执行，因此不用担心竞争条件（race condition）的出现。 `action` 就是普通对象而已，因此它们可以被日志打印、序列化、储存、后期调试或测试时回放出来。
    
+   使用纯函数来执行修改
    
    为了描述`action` 如何改变`state tree` ，你需要编写`reducers`。 `Reducer`只是一些纯函数，它接收先前的`state`和`action`，并返回新的`state`。刚开始你可以只有一个`reducer`，随着应用变大，你可以把它拆成多个小的`reducers`，分别独立地操作`state tree`的不同部分，因为`reducer`只是函数，你可以控制它们被调用的顺序，传入附加数据，甚至编写可复用的`reducer`来处理一些通用任务，如分页器。
    

## 数据流向

严格的单向数据流是`Redux`架构的设计核心。

这意味着应用中所有的数据都遵循相同的生命周期，这样可以让应用变得更加可预测且容易理解。同时也鼓励做数据范式化，这样可以避免使用多个且独立的无法相互引用的重复数据。

看看`React`和`Redux`流程图： ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16fe5baac2224bdc8d27ed615cbaf05d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

举个最简单的例子：

```js
//创建一个最基本的store
const store =createStore(reducers);
// subscribe() 返回一个函数用来注销监听器
const unsubscribe = store.subscribe(()=>console.log(store.getState()))
// 发起一系列 action
store.dispatch(addTodo('Learn about actions'))
store.dispatch(addTodo('Learn about reducers'))
```

通过以上几句代码，我们已经实现了数据流从`dispatch(action)->reducer->subscribe->view`回调的整体流程(此处省略了`middleWare`的部分)，在这个例子中没有任何的`UI`层，`redux`也同样可以独立完成完整的数据流向。其中`subscribe`是对`state`变化更新的订阅功能，可以在回调函数中注册`view`渲染功能。

## Redux 实现

通过 SPA 项目里的状态管理我们了解到了`redux`和`redux`能解决哪些场景中遇到的问题。了解了`redux`解决了什么问题以及如何解决的，这样才能把握`redux`的设计思路。 `React`作为一个组件化开发框架，组件之间存在大量通信，而且组件之间通信可能跨域多层组件，或是多个组件之间共享同一数据，`React`有简单父子组件、非父子组件的通信不能满足我们的需求，所以我们需要有一个空间来存取和操作这些公用状态。而`redux`就为我们提供了一种管理公共状态的解决方案，接下来我们都围绕这个话题来展开。

## Redux 核心 API 实现

通过上面的数据流可以看出，`Redux`主要由三部分组成：`store`、`reducer`、`action`。`Redux`的核心就是`store`,它有`Redux`提供的`createStore`函数生成，该函数返回3个处理函数`getState`,`dispatch`,`subscribe`。

## Store

`Store`就是保存数据的地方，你可以把它看成一个容器。整个应用只能有一个`Store`。 接下来我们写`store`：

```js
export default function createStore() {
 let state = {} // 公共状态
 const getState = () => {} // 存储的数据，状态树；
 const dispatch = () => {} // 分发action，并返回一个action，这是唯一能改变store中数据的方式；
 const subscribe = () => {} //注册一个监听者，store发生变化的时候被调用。
 return {dispatch,subscribe,getState}
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35df32ed3d2e4527a9899987118d9cac~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) **getState()实现** 对象包含所有数据。如果想得到某个时点的数据，就要对`Store`生成快照。这种时点的数据集合，就叫做`State`。

```js
 const getState = () => {
   return state;
 }
```

**dispatch()的实现** 直接修改`state`,`state.num+'a'`,如修改`state`像这种情况，就会导致结果不是我们想要的，后果可能很严重，如果避免呢？ 如果可以随意修改`state`，会造成难以复现的`bug` ，我们需要实现有条件并且是具名修改的`store`数据，既然要分发`action`这里要传入一个`action`对象，另外这对象包括我们要修改的`state`和要操作的具名`actionType`,这里用`type`属性值的不同来对`state`做相应的修改，代码如下：

```js
 const dispatch = (action) => {
     switch (action.type) {
    case 'ADD':
      return {
        ...state,
        num: state.num + 1,
      }
    case 'MINUS':
      return {
        ...state,
        num: state.num - 1,
      }
    case 'CHANGE_NUM':
      return {
        ...state,
        num: state.num + action.val,
      }
    // 没有匹配到的方法 就返回默认的值
    default:
      return state
  }
 } 
```

从代码上看，这里并没有把`action`独立出来，接着往下看吧。 函数负责生成`State`。由于整个应用只有一个 `State`对象，包含所有数据，对于大型应用来说，这个`State`必然十分庞大，导致 `Reducer`函数也十分庞大。

## reducer

`reducer`是一个纯函数，它根据`action`和`initState`计算出新的`state`。

```js
reducer(action,initState)
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92681ebee93c409aaa44cdeb10942cf1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 强制使用`action`来描述所有变化带来的好处是可以清晰地知道应用中到底发生了什么。如果一些东西改变了，就可以知道为什么变。最后，为了把`action`和`state`串起来，就有了`reducer`。 `reducer.js`:

```js
export default function reducer(action, state) {
  //通过传进来的 action.type 让管理者去匹配要做的事情
  switch (action.type) {
    case 'ADD':
      return {
        ...state,
        num: state.num + 1,
      }
    case 'MINUS':
      return {
        ...state,
        num: state.num - 1,
      }
    case 'CHANGE_NUM':
      return {
        ...state,
        num: state.num + action.val,
      }
    // 没有匹配到的方法 就返回默认的值
    default:
      return state
  }
}
```

`test.js`测试结果：

```js
import createStore from './redux'
let initState = {
  num: 12,
}
const store = createStore(reducer, initState)
console.log(store.getState())
```

运行代码输出结果正常。

**subscribe()的实现** 尽管这里能存储公用`state`,但是`store`的变化并不能直接更新视图，所以这里需要监听`store`的变化，这里就用到了一个很常用的设计模式——观察者模式。 言归正传，我们来实现下`subscriber`：

```js
/**
 * store实现
 *
 * @param {Function} reducer  管理状态更新者 接收：action,state 两参数
 * @param {Object} initState 初始化状态，如果没有num,会导致num默认是NaN
 * @returns 返回 subscribe, dispatch, getState 
 * */
const createStore = (reducer, initState = { num: 10 }) => {
  let state = initState
  let subscribes = [] // 存放观察者
  // 增加观察者
  const subscribe = (fn) => {
    subscribes.push(fn)
  }
  // 通知所有观察者  这里不再是传状态了，而是改传改变状态的命令（通过固定指令告诉管理者需要做什么）
  const dispatch = (action) => {
    state = reducer(action, state) 
    // state发生变化，调用（通知）所有方法（观察者）
    subscribes.forEach(fn => fn())
  }
  // 这里需要添加这个获取state的方法
  const getState = () => {
    return state
  }
  return {
    subscribe,
    dispatch,
    getState,
  }
}
export default createStore
```

`test.js`测试代码

```javascript
import createStore from './redux'
let initState = {
  num: 12,
}
const store = createStore(reducer, initState)
store.subscribe(() => {
  let state = store.getState()
  console.log('收到通知：','state.num更新结果为'+state.num)
})
store.dispatch({type:'ADD'}) 
store.dispatch({type:'MINUS'}) 
store.dispatch({type:"CHANGENUM", val:20}) // 
```

执行结果正常：

## action

`Action`是把数据从应用（译者注：这里之所以不叫`view`是因为这些数据有可能是服务器响应，用户输入或其它非`view`的数据 ）传到 `store` 的有效载荷。它是 `store` 数据的唯一来源。一般来说你会通过 `store.dispatch()` 将 `action` 传到`store`。

`State` 的变化，会导致`View` 的变化。但是，用户接触不到 `State`，只能接触到 `View`。所以，`State` 的变化必须是 `View` 导致的。`Action` 就是 `View` 发出的通知，表示 `State` 应该要发生变化了。

`Action` 是一个对象。其中的`type`属性是必须的，表示 `Action` 的名称。其他属性可以自由设置，社区有一个规范可以参考。

可以这样理解，`Action` 描述当前发生的事情。改变`State` 的唯一办法，就是使用 `Action`。它会运送数据到 `Store`。

新建 `action.js`

```js
export const ADD = 'ADD'
export const MINUS = 'MINUS'
export const CHANGE_NUM = 'CHANGE_NUM'
/*
 * Action Creator 来生成action
 */
export function add(text) {
  return { type: ADD, text }
}
export function minus(index) {
  return { type: MINUS, index }
}
export function changeNum(filter) {
  return { type: CHANGE_NUM, filter }
}
相应的reducer得改造下：
import { ADD,MINUS,CHANGE_NUM } from './action'
export default function reducer(action, state) {
  //通过传进来的 action.type 让管理者去匹配要做的事情
  switch (action.type) {
    case ADD:
      return {
        ...state,
        text: action.text,
        num: state.num + 1,
      }
    case MINUS:
      return {
        ...state,
        index: action.index,
        num: state.num - 1,
      }
    case CHANGE_NUM:
      return {
        ...state,
        val: action.val,
        num: state.num + action.val,
      }
    // 没有匹配到的方法 就返回默认的值
    default:
      return state
  }
}
```

test.js测试应用

```js
import createStore from './redux'
import reducer from './reducer'
import { add,minus,changeNum } from './action'
let initState = {
  num: 12,
}
const store = createStore(reducer, initState)
store.subscribe(() => {
  let state = store.getState()
  console.log(state) 
  console.log('收到通知：','state.num更新结果为'+state.num)
})
store.dispatch(add('num+1'))
store.dispatch(minus(1))
store.dispatch(changeNum(20))
```

到这我们基本就是实现了`redux`,虽然相对粗糙，并不用影响我们对其思路的理解。

[在线源码](https://codesandbox.io/s/redux-and-react-redux-7l55p "https://codesandbox.io/s/redux-and-react-redux-7l55p")

## 总结

我们来梳理下`action`、`store`、`reducer`，`views`他们之间的交互流程，如下图： ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c88514d5440467bba61f9dc5fac2085~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`Redux` 本身和 `React` 没有关系，只是数据处理中心，那么他们是如何产生联系的呢，接下来就该 说到`react-redux`，下一篇就围绕`react-redux`来讲，以及它在`React`里的应用。

## 参考文档：

+   [Redux](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#why-should-i-use-redux "https://redux.js.org/tutorials/essentials/part-1-overview-concepts#why-should-i-use-redux")
+   [React-redux](http://cn.redux.js.org/docs/react-redux/ "http://cn.redux.js.org/docs/react-redux/")




