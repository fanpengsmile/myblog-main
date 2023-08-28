---
slug: react-redux之connect方法详解
title: react-redux之connect方法详解
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

Redux 是「React 全家桶」中极为重要的一员，它试图为 React 应用提供「可预测化的状态管理」机制。Redux 本身足够简单，除了 React，它还能够支持其他界面框架。所以如果要将 Redux 和 React 结合起来使用，就还需要一些额外的工具，其中最重要的莫过于 react-redux 了。

react-redux 提供了两个重要的对象，`Provider`和`connect`，前者使`React`组件可被连接（connectable），后者把 React 组件和 Redux 的 store 真正连接起来。react-redux 的文档中，对`connect`的描述是一段晦涩难懂的英文，在初学 redux 的时候，我对着这段文档阅读了很久，都没有全部弄明白其中的意思（大概就是，单词我都认识，连起来啥意思就不明白了的感觉吧）。

在使用了一段时间 redux 后，本文尝试再次回到这里，给[这段文档](https://link.segmentfault.com/?enc=J13Z2QXiks8vTWp12aptuw%3D%3D.7MdxRL92fQ3PDKndvytBjr%2Fxs5a3WJ9sB9sFLX7w31Cmh3Dq4QNHikXBkgkA3LOFq5y%2BXLAIuYR1quNfPrJiwS5dRyaOL9eU2UMIlA5ulXXhs8HeJc8wDVpB%2FC9CGIvIARjwqWLjvChJYQbpOWqHsjmpdENGGL67VF0VF8gsGg4%3D)一个靠谱的解读。

## 预备知识

首先回顾一下 redux 的基本用法。如果你还没有阅读过 redux 的文档，你一定要先去[阅读](https://link.segmentfault.com/?enc=0v0gsyuh%2BfiE%2FNd3XEnviQ%3D%3D.IGT4JSCGdhY1k5irJ101a2y28dkorOY7V0FUklWyO3xKYb5YlqPn8LvatTcJd18lgcqlHaBTViWXcXgLSPA6tQ%3D%3D)一下。

```javascript
const reducer = (state = {count: 0}, action) => {
  switch (action.type){
    case 'INCREASE': return {count: state.count + 1};
    case 'DECREASE': return {count: state.count - 1};
    default: return state;
  }
}

const actions = {
  increase: () => ({type: 'INCREASE'}),
  decrease: () => ({type: 'DECREASE'})
}

const store = createStore(reducer);

store.subscribe(() =>
  console.log(store.getState())
);

store.dispatch(actions.increase()) // {count: 1}
store.dispatch(actions.increase()) // {count: 2}
store.dispatch(actions.increase()) // {count: 3}
```

通过`reducer`创建一个`store`，每当我们在`store`上`dispatch`一个`action`，`store`内的数据就会相应地发生变化。

我们当然可以直接在 React 中使用 Redux：在最外层容器组件中初始化`store`，然后将`state`上的属性作为`props`层层传递下去。

```jsx
class App extends Component{

  componentWillMount(){
    store.subscribe((state)=>this.setState(state))
  }

  render(){
    return (
      <Comp
        state={this.state}
        onIncrease={()=>store.dispatch(actions.increase())}
        onDecrease={()=>store.dispatch(actions.decrease())}
      />
    )
  }
}
```

> 转载注：
> 
> +   另一种方法，在入口文件`index.js`中初始化`store`，并将其`export`出来，然后`import`到定义组件的文件中去。

但这并不是最佳的方式。最佳的方式是使用 react-redux 提供的`Provider`和`connect`方法。

## 使用 react-redux

首先在最外层容器中，把所有内容包裹在`Provider`组件中，将之前创建的`store`作为`prop`传给`Provider`。

```jsx
const App = () => {
  return (
    <Provider store={store}>
      <Comp/>
    </Provider>
  )
};
```

`Provider`内的任何一个组件（比如这里的`Comp`），如果需要使用`state`中的数据，就必须是「被 connect 过的」组件——使用`connect`方法对「你编写的组件（`MyComp`）」进行包装后的产物。

```javascript
class MyComp extends Component {
  // content...
}

const Comp = connect(...args)(MyComp);
```

可见，`connect`方法是重中之重。

## connect

究竟`connect`方法到底做了什么，我们来一探究竟。

首先看下函数的签名：

```javascript
connect([mapStateToProps], [mapDispatchToProps], [mergeProps], [options])
```

`connect()`接收四个参数，它们分别是`mapStateToProps`，`mapDispatchToProps`，`mergeProps`和`options`。

### mapStateToProps

```javascript
mapStateToProps(state, ownProps) : stateProps
```

这个函数允许我们将`store`中的数据作为`props`绑定到组件上。

```javascript
const mapStateToProps = (state) => {
  return {
    count: state.count
  }
}
```

这个函数的第一个参数就是 Redux 的`store`，我们从中摘取了`count`属性。因为返回了具有`count`属性的对象，所以`MyComp`会有名为`count`的`props`字段。

```jsx
class MyComp extends Component {
  render(){
    return <div>计数：{this.props.count}次</div>
  }
}

const Comp = connect(...args)(MyComp);
```

当然，你不必将`state`中的数据原封不动地传入组件，可以根据`state`中的数据，动态地输出组件需要的（最小）属性。

```javascript
const mapStateToProps = (state) => {
  return {
    greaterThanFive: state.count > 5
  }
}
```

函数的第二个参数`ownProps`，是`MyComp`自己的`props`。有的时候，`ownProps`也会对其产生影响。比如，当你在`store`中维护了一个用户列表，而你的组件`MyComp`只关心一个用户（通过`props`中的`userId`体现）。

```jsx
const mapStateToProps = (state, ownProps) => {
  // state 是 {userList: [{id: 0, name: '王二'}]}
  return {
    user: _.find(state.userList, {id: ownProps.userId})
  }
}

class MyComp extends Component {
  
  static PropTypes = {
    userId: PropTypes.string.isRequired,
    user: PropTypes.object
  };
  
  render(){
    return <div>用户名：{this.props.user.name}</div>
  }
}

const Comp = connect(mapStateToProps)(MyComp);
```

当`state`变化，或者`ownProps`变化的时候，`mapStateToProps`都会被调用，计算出一个新的`stateProps`，（在与`ownProps` merge 后）更新给`MyComp`。

这就是将 Redux `store`中的数据连接到组件的基本方式。

> 转载注：
> 
> +   **什么叫做“`MyComp`自己的`props`”？**假设在不使用 react-redux 的时候，`MyComp`的父组件是`ParentComp`，那么上文中的`ownProps`是`ParentComp`传递给`MyComp`的全部属性（对于下文中的方法`mapDispatchToProps`亦同）。**也就是说，`ownProps`与 Redux 的`store`与`state`完全无关**。
> +   **方法`mapStateToProps`为`MyComp`添加的属性，不可能被方法`mapDispatchToProps`访问到，反之亦然。**因为，这涉及到 render 的时机和顺序的问题，笔者在这上面踩过相当多的坑。至于笔者为什么有这种需求，因为笔者设计了一个按钮，功能是：在点击后，根据当前的 state 计算出下一个 state，并更新。在经历了无数`error`以后，笔者终于意识到：react-redux 根本就不是设计用来解决这类问题的。解决方案有两种：一是，在设计 reducer 的时候，就直接根据 state 更新；二是，导入全局`store`并使用 `store.getState()` 获得当前`state`，然后根据这个`state`进行更新。
> +   另外，如果使用[PropTypes](https://link.segmentfault.com/?enc=NAHThrozvnEJWihMg6iaaw%3D%3D.jIuZKGFC2HTJNcGahJqrj%2BOvLbtoSr4Y3cchdNlGZe3p9Tb58F8YL8sBX1vk3yqdA8b%2FvPDUnm7sdzFLKGpavg%3D%3D)对`MyComp`做属性类型检查，方法`mapStateToProps`和方法`mapDispatchToProps`为`MyComp`添加的属性是存在的。

### mapDispatchToProps

```javascript
mapDispatchToProps(dispatch, ownProps): dispatchProps
```

connect 的第二个参数是`mapDispatchToProps`，它的功能是，将`action`作为`props`绑定到`MyComp`上。

```jsx
const mapDispatchToProps = (dispatch, ownProps) => {
  return {
    increase: (...args) => dispatch(actions.increase(...args)),
    decrease: (...args) => dispatch(actions.decrease(...args))
  }
}

class MyComp extends Component {
  render(){
    const {count, increase, decrease} = this.props;
    return (<div>
      <div>计数：{this.props.count}次</div>
      <button onClick={increase}>增加</button>
      <button onClick={decrease}>减少</button>
    </div>)
  }
}

const Comp = connect(mapStateToProps， mapDispatchToProps)(MyComp);
```

由于`mapDispatchToProps`方法返回了具有`increase`属性和`decrease`属性的对象，这两个属性也会成为`MyComp`的`props`。

如上所示，调用`actions.increase()`只能得到一个`action`对象`{type:'INCREASE'}`，要触发这个`action`必须在`store`上调用`dispatch`方法。`dispatch`正是`mapDispatchToProps`的第一个参数。但是，为了不让 MyComp 组件感知到`dispatch`的存在，我们需要将`increase`和`decrease`两个函数包装一下，使之成为直接可被调用的函数（即，调用该方法就会触发`dispatch`）。

Redux 本身提供了`bindActionCreators`函数，来将`action`包装成直接可被调用的函数。

```javascript
import {bindActionCreators} from 'redux';

const mapDispatchToProps = (dispatch, ownProps) => {
  return bindActionCreators({
    increase: action.increase,
    decrease: action.decrease
  });
}
```

同样，当`ownProps`变化的时候，该函数也会被调用，生成一个新的`dispatchProps`，（在与`stateProps`和`ownProps` merge 后）更新给`MyComp`。注意，`action`的变化不会引起上述过程，默认`action`在组件的生命周期中是固定的。

> 转载注：
> 
> +   函数`connect`甚至`react-redux`的核心在于：将 Redux 中 store 的 state 绑定到组件的属性上，使得对 state 的修改能够直接体现为组件外观的更改。因此，参数`mapStateToProps`是非常重要的，但是参数`mapDispatchToProps`则比较多余——因为简单粗暴地导入全局 store 同样能达到相同的目的（事实上笔者就是这么做的）。

### mergeProps

```javascript
[mergeProps(stateProps, dispatchProps, ownProps): props]
```

之前说过，不管是`stateProps`还是`dispatchProps`，都需要和`ownProps` merge 之后才会被赋给`MyComp`。`connect`的第三个参数就是用来做这件事。通常情况下，你可以不传这个参数，`connect`就会使用`Object.assign`替代该方法。

## 其他

最后还有一个`options`选项，比较简单，基本上也不大会用到（尤其是你遵循了其他的一些 React 的「最佳实践」的时候），本文就略过了。希望了解的同学可以直接看文档。

（完）




