---
title: redux之applyMiddleware
date: 2018-09-03 21:39:05
tags:
  - redux
  - react
  - 源码
categories: 前端
---

# redux之applyMiddleware

`redux` 函数式写法相当多，所以在 `applyMiddleware` 处理多个中间执行顺序的写法也是比较绕和难懂的。`redux` 的中间件用切面的思想，让第三方的中间件切入到 `reducer` 处理 `action` 之前，做一些处理比如 `log` 之类的。`applyMiddleware` 则是把众多的中间件汇聚在一起，一层一层地去增强了原本生成的`store` 上的 `dispath` 的功能。

## 用法

先看下怎么使用这个 `api`：

```js
import { createStore, applyMiddleware } from 'redux'
import todos from './reducers'

function logger({ getState }) {
  return (next) => (action) => {
    console.log('will dispatch', action)

    // 调用 middleware 链中下一个 middleware 的 dispatch。
    let returnValue = next(action)

    console.log('state after dispatch', getState())

    // 一般会是 action 本身，除非
    // 后面的 middleware 修改了它。
    return returnValue
  }
}

let store = createStore(
  todos,
  [ 'Use Redux' ],
  applyMiddleware(logger)
)

store.dispatch({
  type: 'ADD_TODO',
  text: 'Understand the middleware'
})
// (将打印如下信息:)
// will dispatch: { type: 'ADD_TODO', text: 'Understand the middleware' }
// state after dispatch: [ 'Use Redux', 'Understand the middleware' ]
```

官网的这个例子大概说明了 `applyMiddleware` 的用法了，这里用 `let store = createStore(todos, [ 'Use Redux' ],applyMiddleware(logger))` 生成了一个新的store，同样的也可以用 `let newStore = applyMiddleware(mid1, mid2, mid3, ...)(createStore)(reducer, initialState);` 的方式生成 `store`。`applyMiddleware` 返回一个 `store` 的 `enhancer` 然后生成增强后的 `createStore` ，部分细节可以参照 `createStore` 中的一小段代码：

`./createStore.js`

```js
if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
        // enhancer 不是函数就报错
      throw new Error('Expected the enhancer to be a function.')
    }
    return enhancer(createStore)(reducer, preloadedState)
  }
```

## 源码

```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    // const middle1 = next => action => next(action)
    // const middle2 = next => action => next(action)

    // dispatch = compose(middle1, middle2)(store.dispatch) = middle1(middle2(store.dispatch))
    // dispatch(actions) = middle1(middle2(store.dispatch))(actions)
    // middle2(store.dispatch) = action => store.dispath(action)
    return {
      ...store,
      dispatch
    }
  }
}
```

上来返回一个接收 `createStore`做为参数的函数，调用这个`createStore` 去生成对应的 `store`，之前也提到了`applyMiddleware` 主要目的是为了增强 `dispatch`的功能，所以最后是用拓展运算符返回了原本store上的属性以及增强过的`dispatch`。

### 细节

```js
(...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)
    return {
      ...store,
      dispatch
    }
```

中间件的签名是这样的 `({dispatch, getStore}) => next => action => {// 一些第三方中间件的处理}`，这里对整个`middlewares` 数组进行了一次 `map` 给每一个中间件传入 `middlewareAPI` 进行一次调用，可以看做是进行了一次剥皮，这样chain里面存的就都是 `next => action => {// 一些第三方中间件的处理}` 这样签名的函数了。最关键的一步来了，`dispatch = compose(...chain)(store.dispatch)` 用 `compose` 把`chain` 内部的函数全部组合起来，接受最原始的 `store.dispatch`，生成一个增强后的 `dispatch` 。讲清楚这一行代码，就能搞明白 `redux` 的中间件干了什么事情。

### 特别的一行代码

```js
dispatch = compose(...chain)(store.dispatch)
```

我们先假设传给了 `applyMiddleware` 这样的参数：

```js

const middleware1 = ({dispatch, getStore}) => next => action => {
    console.log(`middleware1 start`)
    next(action)
    console.log(`middleware1 end`)
}

const middleware2 = ({dispatch, getStore}) => next => action => {
    console.log(`middleware2 start`)
    next(action)
    console.log(`middleware2 end`)
}
const enhancer = applyMidleware([middleware1, middleware2])
```

到了上述这行代码的时候，`chain` 是这个样子了：

```js

const m1 = next => action => {
        console.log(`middleware1 start`)
        next(action)
        console.log(`middleware1 end`)
    }
const m2 = next => action => {
        console.log(`middleware2 start`)
        next(action)
        console.log(`middleware2 end`)
    }
chain = [m1, m2]
```

结合之前提到的 `compose` 组合函数，所以这行代码变形成这样：`dispatch = m1(m2(store.dispatch))`。这个时候假设需要`dispatch` 一个 `anAction` ，就会变成这样 `m1(m2(store.dispatch))(anAction)`。对于m1而言，`m2(store.dispatch)` 就是他的 `next` 参数，当 `m1` 执行到 `next(action)`的时候，相当于这时候调用了`m2(store.dispatch)(action)`, 然后进入到`m2`中间件的逻辑，此时`m2`中间件接受的 `next` 正是原本的 `store.dispatch`，运行到`m2` 内部的 `next(action)` 的时候，这时候才去真正的用`store.dispatch` 去处理这个 `action`。每一次调用 `next` 就把这个中间件的控制权交给下一个中间件。

这就有种俄罗斯套娃一样，一层一层地嵌套，`action` 经过多层的处理，最终到达最里层的才真正地被`dispatch` 。

## 总结

`applyMiddleware` 最为核心就是 `dispatch = compose(...chain)(store.dispatch)`，由此我想到了 `koa` 中间件的机制，有所相似，实现也不太一样。