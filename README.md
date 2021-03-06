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

了解了上面的东西，你会发现其实redux的任务就是管理数据。redux的数据流可以用下面的图来标示：


![redux](https://github.com/azl397985856/learn-redux/blob/master/redux.png)

redux中核心就是一个单一的state。state通过闭包的形式存放在redux store中，保证其是只读的。如果你想要更改state，只能通过发送action进行，action本质上就是一个普通的对象。

你的应用可以通过redux暴露的subscribe方法，订阅state变化。如果你在react应用中使用redux，则表现为react订阅store变化，并re-render视图。

最后一个问题就是如何根据action来更新视图，这部分是业务相关的。 redux通过reducer来更新state，关于reducer的介绍，我会在后面详细介绍。

它精妙的设计我们在后面进行解读。
## 最小化实现REDUX
学习一个东西的最好方法就是自己写一个。好在redux并不复杂，重新实现一个redux并不困难。redux源码也就区区200行左右。
里面大量使用高阶函数，闭包，函数组合等知识。让代码看起来更加简短，结构更加清晰。

我们来写一个`"redux"`吧
### 实现
我们要实现的redux主要有如下几个功能：

- 获取应用state
- 发送action
- 监听state变化

让我们来看下redux store暴漏的api
```js
const store = {
  state: {}, // 全局唯一的state，内部变量，通过getState()获取
  listeners: [], // listeners，用来诸如视图更新的操作
  dispatch: () => {}, // 分发action
  subscribe: () => {}, // 用来订阅state变化
  getState: () => {}, // 获取state
}
```

我们来实现createStore，它返回store对象，
store的对象结构上面已经写好了。createStore是用来初始化redux store的，是redux最重要的api。
我们来实现一下：

`createStore`


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
可以看出我们已经完成了redux的最基本的功能了。 如果需要更新view，就根据我们暴漏的subscribe去更新就好了，这也就解释了 redux并不是专门用于react的，以及为什么要有react-redux这样的库存在。

为了方便各个阶段的人员能够看懂，我省略了applyMiddleware的实现，但是不要担心，我会在下面redux核心思想章节进行解读。

## REDUX核心思想
redux的核心思想出了刚才提到的那些之外。
个人认为还有两个东西需要特别注意。
一个是reducer, 另一个是middlewares
### reducer 和 reduce
reducer可以说是redux的精髓所在。我们先来看下它。reducer**被要求**是一个**纯函数**。 
- 被要求很关键，因为reducer并不是定义在redux中的一个东西。而是用户传进来的一个方法。
- 纯函数也很关键，reducer应该是一个纯函数，这样state才可预测(这里应证了我开头提到的Redux is a predictable state container for JavaScript apps.)。
- 


日常工作我们也会用到reduce函数，它是一个高阶函数。reduce一直是计算机领域中一个非常重要的概念。

reducer和reduce名字非常像，这是巧合吗？

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

如何理解reducer是累计`时间`上的变化？

我们每次通过调用dispatch(action)的时候，都会调用reducer，然后将reducer的返回值去更新store.state。

每次dispatch的过程，其实就是在空间上push(action)的过程，类似这样：

```js
[action1, action2, action3].reduce((state, action) => {
    const nextState = {};
    // xxx
    return nextState;
}, initialState)

```

因此说，reducer其实是时间上的累计，是基于时空的操作。
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

上面就是redux关于middleware的源码，非常简洁。但是想要完全读懂还是要花费点心思的。 首先redux通过createStore生成了一个原始的store(没有被enhance)，然后最后将原始store的dispatch改写了，在调用原生的reducer之间,插入中间件逻辑(中间件链会顺序依次执行). 代码如下：

```js
function applyMiddleware(...middlewares) {
  return createStore => 
    (...args) => { const store = createStore(...args);
    // let dispatch = xxxxx; return { ...store, dispatch } } 
} 

```

然后我们将用户传入的middlewares顺序执行，这里借助了compose，compose是函数式编程中非常重要的一个概念,他的作用就是将多个函数组合成一个函数，compose(f, g, h)()最终生成的大概是这样： 

```js
function(...args) { f(g(h(...args))) } 

```

因此chain大概长这个样子： 

```js
chain = [
  function middleware1(next) {
    // 内部可以通过闭包访问到getState和dispath },
  function middleware2(next) {
    // 内部可以通过闭包访问到getState和dispath },
... ]
```
有了上面`compose`的概念之后，我们会发现每一個middleware的input 都是一个参数next为的function，第一个中间件访问到的next其实就是原生store的dispatch。代码为证： `dispatch = compose(...chain)(store.dispatch)`。从第二个中间件开始，next其实就是上一个中间件返回的 action => retureValue 。 有没有发现这个函数签名就是dispatch的函数签名。 output是一個参数为action的function, 返回的function签名为 action => retureValue 用來作为下一个middleware的next。这样middleware就可以选择性地调用下一個 middleware(next)。 社区有非常多的redux middleware，最为经典的dan本人写的redux thunk，核心代码只有`两行`, 第一次看真的震惊了。从这里也可以看出redux 的厉害之处。 基于redux的优秀设计，社区中出现了很多非常优秀的第三方redux中间价，比如redux-dev-tool, redux-log, redux-promise 等等，有机会我会专门出一个redux thunk的解析。

## 总结
本篇文章主要讲解了redux是什么，它主要做了什么。然后通过不到20行代码实现了一个最小化的redux。最后深入讲解了redux的核心设计reducer和middlewares。 

redux还有一些非常经典的学习资源，这里推荐redux作者本人的[getting started with redux](https://egghead.io/courses/getting-started-with-redux)和[You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)。学习它对于你理解redux以及如何使用redux管理应用状态是非常有帮助的。
