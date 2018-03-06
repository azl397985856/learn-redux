## 为什么写这篇文章
业余时间我也算看了不少优秀开源项目的源码比如react,redux,vuex,vue,babel，ant-design等，但是很少系统地进行总结，学到的知识非常有限，因此我一直想写一篇完善的源码解读方面的文章。


第二个原因是最近面试的过程中，发现很多候选人对redux的理解很浅，甚至有错误的理解。

因此才有了这篇文章的诞生。
## REDUX是什么
深入理解redux之前，首先来看下，redux是什么，解决了什么问题。

下面是redux官方给出的解释：

> Redux is a predictable state container for JavaScript apps.

上面的概念比较抽象，如果对redux不了解的人是很难理解的。

一个更容易被人理解的解释（同样是redux官方的解释）：

> redux是flux架构的实现，受Elm启发

### flux架构
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
## 最小化实现REDUX
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

### 使用
我们就可以像使用redux一样使用了。

> 以下例子摘自官网

你可以把下面这段脚本加上我们上面实现的`redux`，拷贝到控制台执行，看下效果。是否和redux官方给的结果一致。
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

## REDUX核心思想

### reducer 和 reduce

### middlewares
