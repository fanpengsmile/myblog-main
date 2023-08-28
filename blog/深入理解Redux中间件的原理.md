---
slug: 深入理解Redux中间件的原理
title: 深入理解Redux中间件的原理
author: 潜心专研前端的Peyton
author_title: 前端工程师
description: 请输入描述
tags: [前端, React]
# activityId: 相关动态 ID
# bvid: 相关视频 ID（与 activityId 2选一）
# oid: oid
---

1、redux中间件简介  
1.1、什么是redux中间件  
redux 提供了类似后端 Express 的中间件概念，本质的目的是提供第三方插件的模式，自定义拦截 action -> reducer 的过程。变为 action -> middlewares -> reducer 。这种机制可以让我们改变数据流，实现如异步 action ，action 过滤，日志输出，异常报告等功能。

通俗来说，redux中间件就是对dispatch的功能做了扩展。

先来看一下传统的redux执行流程：  
![image.png](https://bbs-img.huaweicloud.com/blogs/img/20220124/1643016257422092330.png)

代码示例：

```js
import { createStore } from 'redux';

/**
 * 这是一个 reducer，形式为 (state, action) => state 的纯函数。
 * 描述了 action 如何把 state 转变成下一个 state。
 */
function counter(state = 0, action) {
  switch (action.type) {
  case 'INCREMENT':
    return state + 1;
  case 'DECREMENT':
    return state - 1;
  default:
    return state;
  }
}

// 创建 Redux store 来存放应用的状态。
// API 是 { subscribe, dispatch, getState }。
let store = createStore(counter);

// 可以手动订阅更新，也可以事件绑定到视图层。
store.subscribe(() =>
  console.log(store.getState())
);

// 改变内部 state 惟一方法是 dispatch 一个 action。
// action 可以被序列化，用日记记录和储存下来，后期还可以以回放的方式执行
store.dispatch({ type: 'INCREMENT' });
// 1
store.dispatch({ type: 'INCREMENT' });
// 2
store.dispatch({ type: 'DECREMENT' });
// 1
```

Redux的核心概念其实很简单：将需要修改的state都存入到store里，发起一个action用来描述发生了什么，用reducers描述action如何改变state tree 。创建store的时候需要传入reducer，真正能改变store中数据的是store.dispatch API。

对dispatch改造后，效果如下：  
![image.png](https://bbs-img.huaweicloud.com/blogs/img/20220124/1643016304229047381.png)  
如上图所示，dispatch派发给 redux Store 的 action 对象，到达reducer之前，进行一些额外的操作，会被 Store 上的多个中间件依次处理。例如可以利用中间件来进行日志记录、创建崩溃报告、调用异步接口或者路由等等，那么其实所有的对 action 的处理都可以有中间件组成的。 简单来说，中间件就是对store.dispatch()的增强。  
1.2、使用redux中间件  
redux有很多中间件，我们这里以 redux-thunk 为例。

代码示例：

```js
import { applyMiddleware, createStore } from 'redux';
import thunk from 'redux-thunk';
 const store = createStore(
  reducers, 
  applyMiddleware(thunk)
);
```

直接将thunk中间件引入，放在applyMiddleware方法之中，传入createStore方法，就完成了store.dispatch()的功能增强。即可以在reducer中进行一些异步的操作。

Redux middleware 提供了一个分类处理 action 的机会。在 middleware 中，我们可以检阅每一个流过的 action,并挑选出特定类型的 action 进行相应操作，以此来改变 action。其实applyMiddleware就是Redux的一个原生方法，将所有中间件组成一个数组，依次执行。  
中间件多了可以当做参数依次传进去。

代码示例：

```js
import { applyMiddleware, createStore } from 'redux';
import thunk from 'redux-thunk';
import createLogger from 'redux-logger';

const logger = createLogger();

const store = createStore(
  reducers, 
  applyMiddleware(thunk, logger) //会按顺序执行
);
```

2、中间件的运行机制  
2.1、createStore源码分析  
源码：

```js
// 摘至createStore
export function createStore(reducer, rootState, enhance) {
    //...
    
    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
          throw new Error('Expected the enhancer to be a function.')
        }
       /*
        若使用中间件，这里 enhancer 即为 applyMiddleware()
        若有enhance，直接返回一个增强的createStore方法，可以类比成react的高阶函数
       */
       return enhancer(createStore)(reducer, preloadedState)
  }
    
  //...
}
```

对于createStore的源码我们只需要关注和applyMiddleware有关的地方， 通过源码得知在调用createStore时传入的参数进行一个判断，并对参数做矫正。 据此可以得出createStore有多种使用方法，根据第一段参数判断规则，我们可以得出createStore的两种使用方式：

```js
const store = createStore(reducer, {a: 1, b: 2}, applyMiddleware(...));
```

或：

```js
const store = createStore(reducer, applyMiddleware(...));

```

经过createStore中的第一个参数判断规则后，对参数进行了校正，得到了新的enhancer得值，如果新的enhancer的值不为undeifined，便将createStore传入enhancer(即applyMiddleware调用后返回的函数)内，让enhancer执行创建store的过程。也就时说这里的：

```js
enhancer(createStore)(reducer, preloadedState);
```

实际上等同于：

```js
applyMiddleware(mdw1, mdw2, mdw3)(createStore)(reducer, preloadedState);
```

applyMiddleware会有两层柯里化，同时表明它还有一种很函数式编程的用法，即

```js
const store = applyMiddleware(mdw1, mdw2, mdw3)(createStore);
```

这种方式将创建store的步骤完全放在了applyMiddleware内部，并在其内第二层柯里化的函数内执行创建store的过程即调用createStore，调用后程序将跳转至createStore走参数判断流程最后再创建store。

无论哪一种执行createStore的方式，我们都终将得到store，也就是在creaeStore内部最后返回的那个包含dispatch、subscribe、getState等方法的对象。

2.2、applyMiddleware源码分析  
源码：

```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    // 利用传入的createStore和reducer和创建一个store
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
      )
    }
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 让每个 middleware 带着 middlewareAPI 这个参数分别执行一遍
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 接着 compose 将 chain 中的所有匿名函数，组装成一个新的函数，即新的 dispatch
    dispatch = compose(...chain)(store.dispatch)
    return {
      ...store,
      dispatch
    }
  }
}
```

为方便阅读和理解，部分ES6箭头函数已修改为ES5的普通函数形式，如下：

```js
function applyMiddleware (...middlewares){
    return function (createStore){
        return function (reducer, preloadedState, enhancer){
            const store = createStore(reducer, preloadedState, enhancer);
            let dispatch = function (){
                throw new Error()
            };

            const middlewareAPI = {
                getState: store.getState,
                dispatch: (...args) => dispatch(...args)
            };
            //一下两行代码是所有中间件被串联起来的核心部分实现
			// 让每个 middleware 带着 middlewareAPI 这个参数分别执行一遍
            const chain = middlewares.map(middleware => middleware(middlewareAPI));
			// 接着 compose 将 chain 中的所有匿名函数，组装成一个新的函数，即新的 dispatch
            dispatch = compose(...chain)(store.dispatch);

            return {
                ...store,
                dispatch
            };
        }
    }
}

```

从上面的代码我们不难看出，applyMiddleware 这个函数的核心就在于在于组合 compose，通过将不同的 middlewares 一层一层包裹到原生的 dispatch 之上，然后对 middleware 的设计采用柯里化的方式，以便于compose ，从而可以动态产生 next 方法以及保持 store 的一致性。

在函数式编程(Functional Programming)相关的文章中，经常能看到 柯里化（Currying）这个名词。它是数学家柯里（Haskell Curry）提出的。

柯里化，用一句话解释就是，把一个多参数的函数转化为单参数函数的方法。

根据源码，我们可以将其主要功能按步骤划分如下：

1、依次执行middleware

将middleware执行后返回的函数合并到一个chain数组，这里我们有必要看看标准middleware的定义格式，如下：

```js
const chain = middlewares.map(middleware => middleware(middlewareAPI));

```

遍历所有的中间件，并调用它们，传入那个类似于store的对象middlewareAPI，这会导致中间件中第一层柯里化函数被调用，并返回一个接收next(即dispatch)方法作为参数的新函数。

```js
export default store => next => action => {}

// 即
function (store) {
    return function(next) {
        return function (action) {
            return {}
        }
    }
}

```

那么此时合并的chain结构如下：

```js
[    ...,
    function(next) {
        return function (action) {
            return {}
        }
    }
]
```

2、改变dispatch指向

```js
dispatch = compose(...chain)(store.dispatch);
```

我们展开了这个数组，并将其内部的元素(函数)传给了compose函数，compose函数又返回了我们一个新函数。然后我们再调用这个新函数并传入了原始的未经任何修改的dispatch方法，最后返回一个经过了修改的新的dispatch方法。

什么是compose？在函数式编程中，compose指接收多个函数作为参数，并返回一个新的函数的方式。调用新函数后传入一个初始的值作为参数，该参数经最后一个函数调用，将结果返回并作为倒数第二个函数的入参，倒数第二个函数调用完后，将其结果返回并作为倒数第三个函数的入参，依次调用，知道最后调用完传入compose的所有的函数后，返回一个最后的结果。

compose函数如下：  
\[…chain\].reduce((a, b) => (…args) => a(b(…args)))  
实际就是一个柯里化函数，即将所有的middleware合并成一个middleware，并在最后一个middleware中传入当前的dispatch。

```js
// 假设chain如下：
chain = [
    a: next => action => { console.log('第1层中间件') return next(action) }
    b: next => action => { console.log('第2层中间件') return next(action) }
    c: next => action => { console.log('根dispatch') return next(action) }
]
```

调用compose(…chain)(store.dispatch)后返回a(b(c(dispatch)))。  
可以发现已经将所有middleware串联起来了，并同时修改了dispatch的指向。  
最后看一下这时候compose执行返回，如下：

```js
dispatch = a(b(c(dispatch)))

// 调用dispatch(action)
// 执行循序
/*
   1. 调用 a(b(c(dispatch)))(action) __print__: 第1层中间件
   2. 返回 a: next(action) 即b(c(dispatch))(action)
   3. 调用 b(c(dispatch))(action) __print__: 第2层中间件
   4. 返回 b: next(action) 即c(dispatch)(action)
   5. 调用 c(dispatch)(action) __print__: 根dispatch
   6. 返回 c: next(action) 即dispatch(action)
   7. 调用 dispatch(action)
*/
```

总结来说就是：

在中间件串联的时候，middleware1-3的串联顺序是从右至左的，也就是middleware3被包裹在了最里面，它内部含有对原始的store.dispatch的调用，middleware1被包裹在了最外边。

当我们在业务代码中dispatch一个action时，也就是中间件执行的时候，middleware1-3的执行顺序是从左至右的，因为最后被包裹的中间件，将被最先执行。

3、常见的redux中间件  
3.1、logger日志中间件  
源码

```js
function createLogger(options = {}) {
  /**
   * 传入 applyMiddleWare 的函数
   * @param  {Function} { getState      }) [description]
   * @return {[type]}      [description]
   */
  return ({ getState }) => (next) => (action) => {
    let returnedValue;
    const logEntry = {};
    logEntry.prevState = stateTransformer(getState());
    logEntry.action = action;
    // .... 
    returnedValue = next(action);
    // ....
    logEntry.nextState = stateTransformer(getState());
    // ....
    return returnedValue;
  };
}

export default createLogger;
```

为了方便查看，将代码修改为ES5之后，如下：

```js
/**
 * getState 可以返回最新的应用 store 数据
 */
function ({getState}) {
    /**
     * next 表示执行后续的中间件，中间件有可能有多个
     */
    return function (next) {
        /**
         * 中间件处理函数，参数为当前执行的 action 
         */
        return function (action) {...}
    }
}

```

这样的结构本质上就是为了将 middleware 串联起来执行。

3.2、redux异步管理中间件  
在多种中间件中，处理 redux 异步事件的中间件，绝对占有举足轻重的地位。从简单的 react-thunk 到 redux-promise 再到 redux-saga等等，都代表这各自解决redux异步流管理问题的方案。

3.2.1、redux-thunk  
redux-thunk的使用：

```js
function getWeather(url, params) {
   return (dispatch, getState) => {
       fetch(url, params)
           .then(result => {
               dispatch({
                   type: 'GET_WEATHER_SUCCESS', payload: result,
               });
           })
           .catch(err => {
               dispatch({
                   type: 'GET_WEATHER_ERROR', error: err,
               });
           });
       };
}
```

在上述使用实例中，我们应用thunk中间到redux后，可以dispatch一个方法，在方法内部我们想要真正dispatch一个action对象的时候再执行dispatch即可，特别是异步操作时非常方便。

源码：

```js
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

为了方便阅读，源码中的箭头函数在这里换成了普通函数，如下：

```js
function createThunkMiddleware (extraArgument){
    return function ({dispatch, getState}){
        return function (next){
            return function (action){
                if (typeof action === 'function'){
                    return action(dispatch, getState, extraArgument);
                }
                return next(action);
            };
        }
    }
}

let thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;

```

thunk是一个很常用的redux中间件，应用它之后，我们可以dispatch一个方法，而不仅限于一个纯的action对象。它的源码也很简单，如上所示，除去语法固定格式也就区区几行。

下面我们就来看看源码(为了方便阅读，源码中的箭头函数在这里换成了普通函数)，首先是这三层柯里化：

```js
// 外层
function createThunkMiddleware (extraArgument){
    // 第一层
   return function ({dispatch, getState}){
      // 第二层
       return function (next){
           // 第三层
           return function (action){
               if (typeof action === 'function'){
                   return action(dispatch, getState, extraArgument);
               }
               return next(action);
           };
       }
   }
}
```

首先是外层，上面的源码可知，这一层存在的主要目的是支持在调用applyMiddleware并传入thunk的时候时候可以不直接传入thunk本身，而是先调用包裹了thunk的函数(第一层柯里化的父函数)并传入需要的额外参数，再将该函数调用的后返回的值(也就是真正的thunk)传给applyMiddleware，从而实现对额外参数传入的支持，使用方式如下：

```js
const store = createStore(reducer, applyMiddleware(thunk.withExtraArgument({api, whatever})))；

```

如果无需额外参数则用法如下：

```js
const store = createStore(reducer, applyMiddleware(thunk))；

```

接下来来看第一层，这一层是真正applyMiddleware能够调用的一层，从形参来看，这个函数接收了一个类似于store的对象，因为这个对象被结构以后获取了它的dispatch和getState这两个方法，巧的是store也有这两方法，但这个对象到底是不是store，还是只借用了store的这两方法合成的一个新对象？这个问题在我们后面分析applyMiddleware源码时，自会有分晓。

再来看第二层，在第二层这个函数中，我们接收的一个名为next的参数，并在第三层函数内的最后一行代码中用它去调用了一个action对象，感觉有点 dispatch({type: ‘XX\_ACTION’， data: {}}) 的意思，因为我们可以怀疑它就是一个dispatch方法，或者说是其他中间件处理过的dispatch方法，似乎能通过这行代码链接上所有的中间件，并在所有只能中间件自身逻辑处理完成后，最终调用真实的 store.dispath 去dispatch一个action对象，再走到下一步，也就是reducer内。

最后我们看看第三层，在这一层函数的内部源码中首先判断了action的类型，如果action是一个方法，我们就调用它，并传入dispatch、getState、extraArgument三个参数，因为在这个方法内部，我们可能需要调用到这些参数，至少dispatch是必须的。\*\*这三行源码才是真正的thunk核心所在。所有中间件的自身功能逻辑也是在这里实现的。\*\*如果action不是一个函数，就走之前解析第二层时提到的步骤。

3.2.2、redux-promise  
不同的中间件都有着自己的适用场景，react-thunk 比较适合于简单的API请求的场景，而 Promise 则更适合于输入输出操作，比较fetch函数返回的结果就是一个Promise对象，下面就让我们来看下最简单的 Promise 对象是怎么实现的：

```js
import { isFSA } from 'flux-standard-action';

function isPromise(val) {
 return val && typeof val.then === 'function';
}

export default function promiseMiddleware({ dispatch }) {
 return next => action => {
   if (!isFSA(action)) {
     return isPromise(action)
       ? action.then(dispatch)
       : next(action);
   }

   return isPromise(action.payload)
     ? action.payload.then(
         result => dispatch({ ...action, payload: result }),
         error => {
           dispatch({ ...action, payload: error, error: true });
           return Promise.reject(error);
         }
       )
     : next(action);
 };
}
```

它的逻辑也很简单主要是下面两部分：

先判断是不是标准的 flux action。如果不是，那么判断是否是 promise, 是的话就执行 action.then(dispatch)，否则执行 next(action)。  
如果是, 就先判断 payload 是否是 promise，如果是的话 payload.then 获取数据，然后把数据作为 payload 重新 dispatch({ …action, payload: result})；不是的话就执行 next(action)  
结合 redux-promise 我们就可以利用 es7 的 async 和 await 语法，来简化异步操作了，比如这样：

```js
const fetchData = (url, params) => fetch(url, params)
async function getWeather(url, params) {
    const result = await fetchData(url, params)
    if (result.error) {
        return {
            type: 'GET_WEATHER_ERROR', error: result.error,
        }
    }
        return {
            type: 'GET_WEATHER_SUCCESS', payload: result,
        }
    }
```

saga特点：

saga 的应用场景是复杂异步。  
可以使用 takeEvery 打印 logger（logger大法好），便于测试。  
提供 takeLatest/takeEvery/throttle 方法，可以便利的实现对事件的仅关注最近实践还是关注每一次实践的时间限频。  
提供 cancel/delay 方法，可以便利的取消或延迟异步请求。  
提供 race(effects)，\[…effects\] 方法来支持竞态和并行场景。  
提供 channel 机制支持外部事件。

```js
function *getCurrCity(ip) {
   const data = yield call('/api/getCurrCity.json', { ip })
   yield put({
       type: 'GET_CITY_SUCCESS', payload: data,
   })
}
function * getWeather(cityId) {
   const data = yield call('/api/getWeatherInfo.json', { cityId })
   yield put({
       type: 'GET_WEATHER_SUCCESS', payload: data,
   })
}
function loadInitData(ip) {
   yield getCurrCity(ip)
   yield getWeather(getCityIdWithState(state))
   yield put({
       type: 'GET_DATA_SUCCESS',
   })
}
```

总的来讲Redux Saga适用于对事件操作有细粒度需求的场景，同时它也提供了更好的可测试性，与可维护性，比较适合对异步处理要求高的大型项目，而小而简单的项目完全可以使用redux-thunk就足以满足自身需求了。毕竟react-thunk对于一个项目本身而言，毫无侵入，使用极其简单，只需引入这个中间件就行了。而react-saga则要求较高，难度较大。  




