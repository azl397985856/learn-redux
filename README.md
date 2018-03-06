## 为什么写这篇文章
业余时间我也算看了不少优秀开源项目的源码比如react,redux,vuex,vue,babel，ant-design等，但是很少系统地进行总结，学到的知识非常有限，因此我一直想写一篇完善的源码解读方面的文章。


第二个原因是最近面试的过程中，发现很多候选人对redux的理解很浅，甚至有错误的理解。真正理解redux的思想的人非常好，更不要说理解它其中的精妙设计了。

因此就有了这篇文章的诞生。
## REDUX是什么
深入理解redux之前，首先来看下，redux是什么，解决了什么问题。

下面是redux官方给出的解释：

> Redux is a predictable state container for JavaScript apps.

上面的概念比较抽象，如果对redux不了解的人是很难理解的。

一个更容易被人理解的解释（同样是redux官方的解释）：

> redux是flux架构的实现，受Elm启发

首先科普两个名字，flux和Elm。
### flux
下面是facebook官方对flux的解释：

> Application Architecture for Building User Interfaces

更具体地说：

> An application architecture for React utilizing a unidirectional data flow.

flux是随着react一起推出的数据管理框架，它的核心思想是单项数据流。 

一图胜千言，让我们通过图来了解下flux

![flux](https://github.com/facebook/flux/blob/master/docs/img/flux-diagram-white-background.png)

通过react构建view，而react又是数据驱动的，那么解决数据问题就解决了view的问题，通过flux架构管理数据，使得数据可预测。这样
view也变得可预测。  非常棒～

### Elm
Elm是一门编译代码到javaScript的语言，它的特点是性能强和无运行时异常。Elm也有虚拟DOM的实现。

Elm的核心理念是使用Model构建应用，也就是说Model是应用的核心。
构建一个应用就是构建Model，构建更新Model的方式，以及如何构建Model到view的映射。

[更多关于elm的介绍](https://github.com/evancz/elm-architecture-tutorial/)

了解了上面的东西，你会发现其实redux的任务就是管理数据，它精妙的设计我们在后面进行解读。
## 最小化实现REDUX
其实写一个redux并不困难。redux源码也就区区200行左右。
里面大量使用高阶函数，闭包，函数组合等知识。让代码看起来更加简短，结构更加清晰。

我们来写一个`"redux"`吧
### 实现

开始之前我们先看下redux暴漏的api
```js
const store = {
  state: {}, // 全局唯一的state，内部变量，通过getState()获取
  listeners: [], // listeners，用来诸如视图更新的操作
  dispatch: () => {}, // 分发action
  subscribe: () => {}, // 用来订阅state变化
  getState: () => {}, // 获取state
}
```
上面是redux中最主要的api（其实还有一个applyMiddleware用来实现中间件的也非常重要，将会在redux的核心思想部分讲解，这里暂不讨论）


`createStore`

createStore是用来初始化redux store的，是最重要的api。
我们来实现一下：
```js
const createStore = (reducer, initialState) => {
  // internal variables
  const store = {};
  store.state = initialState;
  store.listeners = [];
  
  // api-subscribe
  store.subscribe = (listener) => {
    store.listeners.push(listener);
  };
  // api-dispatch
  store.dispatch = (action) => {
    store.state = reducer(store.state, action);
    store.listeners.forEach(listener => listener());
  };
  
  // api-getState
  store.getState = () => store.state;
  
  return store;
};

```

通过上面的20行左右的代码已经实现了redux的最基本功能了，是不是很惊讶？我们下面来试下。
### 使用
我们现在可以像使用redux一样使用了我们的`"redux"`了。

> 以下例子摘自官网

你可以把下面这段脚本加上我们上面实现的`"redux"`，拷贝到控制台执行，看下效果。是否和redux官方给的结果一致。
```js
// reducer
function counter(state = 0, action) {
  switch (action.type) {
  case 'INCREMENT':
    return state + 1
  case 'DECREMENT':
    return state - 1
  default:
    return state
  }
}

let store = createStore(counter)

store.subscribe(() =>
  console.log(store.getState())
)


store.dispatch({ type: 'INCREMENT' })
// 1
store.dispatch({ type: 'INCREMENT' })
// 2
store.dispatch({ type: 'DECREMENT' })
// 1
```

为了方便各个阶段的人员能够看懂，我省略了applyMiddleware的实现，但是不要担心，我会在下面redux核心思想章节进行解读。

## REDUX核心思想
redux的核心思想出了刚才提到的那些之外。
个人认为还有两个东西需要特别注意。
一个是reducer, 另一个是middlewares
### reducer 和 reduce
reducer可以说是redux的精髓所在。我们先来看下它。reducer和reduce名字非常像，这是巧合吗？

我们先来看下reducer的函数签名：

```js
fucntion reducer(state, action) {
    const nextState = {};
    // xxx
    return nextState;
}
```

再看下reduce的函数签名

```js
[].reduce((state, action) => {
    const nextState = {};
    // xxx
    return nextState;
}, initialState)
```

可以看出两个几乎完全一样。最主要区别在于reduce的需要一个数组，然后累计变化。
reducer则没有这样的一个数组。

更确切地说，reducer累计的`时间`上的变化，reduce是累计`空间`上的变化。
### middlewares
关于middleware的概念我们不多介绍，
感兴趣可以访问[这里](https://redux.js.org/advanced/middleware)查看更多信息。

如下可以实现一个redux的middlewares：

```js
store.dispatch = function dispatchAndLog(action) {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}


```

上述代码会在dispatch前后进行打印信息。类似的middleware我们可以写很多。
比如我们还定义了另外几个相似的中间件。

我们需要将多个中间件按照一定顺序执行：

```js
// 用reduce实现compose，很巧妙。
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}

// applyMiddleware 的源码
function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => null;
    let chain = [];

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 将middlewares组成一个函数
    // 也就是说就从前到后依次执行middlewares
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}

// 使用
let store = createStore(
  todoApp,
  // applyMiddleware() tells createStore() how to handle middleware
  applyMiddleware(logger, dispatchAndLog)
)
```
