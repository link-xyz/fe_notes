## 为什么需要中间件
接触过 Express 的同学对“中间件”应该并不陌生。在 Express 中，中间件就是定义了对特定请求处理过程的函数。作为中间件的函数是相互独立的，可以提供诸如记录日志、返回特定响应报头、压缩等操作。

一个典型的 Express 中间件如下所示：

```javascript
// a middleware function with no mount path. This code is executed for every request to the router
router.use(function (req, res, next) {
  console.log('Time:', Date.now());
  next();
});

// a middleware sub-stack shows request info for any type of HTTP request to the /user/:id path
router.use('/user/:id', function(req, res, next) {
  console.log('Request URL:', req.originalUrl);
  next();
}, function (req, res, next) {
  console.log('Request Type:', req.method);
  next();
});
```

中间件函数能够访问请求对象 (req)、响应对象 (res) 以及应用程序的请求/响应循环中的下一个中间件函数。下一个中间件函数通常由名为 `next` 的变量来表示。

同样的，在 Redux 中，action 对象对应于 Express 中的客户端请求，会被 Store 中的中间件依次处理。如下图所示：

![image](https://user-images.githubusercontent.com/58578193/70372865-488b7100-191f-11ea-8183-c9be44e029d0.png)

中间件可以实现通用逻辑的重用，通过组合不同中间件可以完成复杂功能。它具有下面特点：

- 中间件是独立的函数
- 中间件可以组合使用
- 中间件统一的接口

这里采用了 AOP （面向切面编程）思想。

对于面向对象，当需要对逻辑增加扩展功能时（如发送请求前的校验、打印日志等），我们只能在对应功能模块添加额外的扩展功能；或者选择共有类通过继承方式调用，但这将导致共有类的膨胀。否则，就只有将其散落在业务逻辑的各个角落，导致代码耦合。

而使用 AOP，可以解决代码冗余、耦合的问题。我们可以将扩展功能代码单独放入一个切面，待执行的时候才将其载入到需要扩展功能的位置，即切点。这样的好处是不用更改本身的业务逻辑代码，这种通过串联的方式传递调用扩展功能也是中间件的原理。

## 从零开发一个中间件

中间件需要有统一的接口，才能实现自由组合。每个中间件必须被定义成一个函数 `f1`，返回一个接收 `next` 参数的函数 `f2`，而 `f2` 又返回一个接收 `action` 参数的函数 `f3`。`next` 参数本身也是一个函数，中间件调用 `next` 函数通知 Redux 处理工作已经结束，可以将 action 对象传递给下一个中间件或 Reducer。

### 最简单的中间件

例如，可以编写一个什么都不做的中间件：

```javascript
function doNothingMiddleware ({ dispatch, getState }) {
  return function (next) {
    return function (action) {
      return next(action)
    }
  }
}
```

用箭头函数进行简化：

```javascript
const doNothingMiddleware = ({ dispatch, getState }) => (next) => (action) => next(action)
```

不管是中间件的实现，还是中间件的使用，都是让每个函数的功能尽量小，然后通过函数的嵌套组合来实现复杂功能，这是**函数式编程**中的重要思想。

### logger

接下来做一些扩展，实现一个可以记录日志的中间件：

```javascript
const logger = ({ dispatch, getState }) => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', getState())
  return result
}
```

经过这个中间件的处理，每次触发 action 时，会首先打印出当前 action 的信息，然后调用 `next(action)`，将 action 对象传递给下一个中间件或 Reducer，返回处理后的结果，然后获取最新 state 并打印。

### crashReporter

一个记录错误日志的中间件：

```javascript
const crashReporter = ({ dispatch, getState }) => next => action => {
  try {
    return next(action)
  } catch (err) {
    console.error('Caught an exception!', err)
    throw err
  }
}
```

用 `try ... catch ...` 将 `next(action)` 包裹，对于正常 action 不做任何处理，对于出错的 action 处理将会捕获异常，打印错误日志，并将异常抛出。

### 在 Redux 中组合使用中间件

在 Redux 中，通过 `applyMiddleware` 来应用中间件：

```javascript
import { createStore, combineReducers, applyMiddleware } from 'redux'

const todoApp = combineReducers(reducers)
const store = createStore(
  todoApp,
  applyMiddleware(logger, crashReporter)
)
```

通过 `store.dispatch(addTodo('use redux'))` 触发一个 action，将先后被 `logger` 和 `crashReporter` 两个中间件处理。

接下来我们看看 Redux 是怎么实现中间件的，这是 Redux 中的部分源码：

```typescript
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    ...args: any[]
  ) => {
    const store = createStore(reducer, ...args)
    let dispatch: Dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

假设有三个中间件 M1，M2，M3，应用 `applyMiddleware(M1, M2, M3)` 将返回一个闭包函数，该函数接收 `createStore` 函数作为参数，使得创建状态树 store 的步骤在这个闭包内执行；然后将 `store` 重新组装成 `middlewareAPI` 作为新的 `store`，即中间件最外层函数的参数，这样中间件就可以根据状态树进行各种操作了。

对中间件处理的关键逻辑在于

```javascript
  const chain = middlewares.map(middleware => middleware(middlewareAPI))
  dispatch = compose(...chain)(store.dispatch)
```

首先，将 `applyMiddleware` 函数中传入的中间件按顺序生成一个队列 `chain`，队列中每个元素都是中间件调用后的结果，它们都具有相同的结构 `next => action => {}`。

然后，通过 `compose` 方法，将这些中间件队列串联起来。`compose` 是一个从右向左的嵌套包裹函数，也是**函数式编程**中的常用范式，实现如下：

```typescript
export default function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    // infer the argument type so it is usable in inference down the line
    return <T>(arg: T) => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args: any) => a(b(...args)))
}
```

假设 `chain` 是包含 C1、C2、C3（对应 M1，M2，M3 第一层函数返回值） 三个函数的数组，那么 `compose(...chain)(store.dispatch)` 即为 `C1(C2(C3(store.dispatch)))`，而且：

- `applyMiddleware` 的最后一个中间件 M3 中的 `next` 就是原始的 `store.dispatch`
- M2 中的 `next` 为 `C3(store.dispatch)`
- M1 中的 `next` 为 `C2(C3(store.dispatch))`

最终将 `C1(C2(C3(store.dispatch)))` 作为新的 `dispatch` 挂载在 `store` 中返回给用户，作为用户实际调用的 `dispatch` 方法。由于已经层层调用了 C3，C2，C1，中间件的结构已经从 `next => action => {}` 被拆解为 `acion => {}`。此时，又回到了 `dispatch(action)` 的结构。

我们可以梳理一遍当用户触发一个 action 的完整流程：

1. 手动触发一个 action：`store.dispatch(action)`
2. 等价于调用 `C1(C2(C3(store.dispatch)))(action)`
3. 执行 C1 中的代码，直到遇到 `next(action)`，此时的 `next` 为 M1 中的 `next`，即：`C2(C3(store.dispatch))`
4. 执行 `C2(C3(store.dispatch))(action)`，直到遇到 `next(action)`，此时的 `next` 为 M2 中的 `next`，即：`C3(store.dispatch)`
5. 执行 `C3(store.dispatch)(action)`，直到遇到 `next(action)`，此时的 `next` 为 M3 中的 `next`，即：`store.dispatch`
6. 执行 `store.dispatch(action)`，`store.dispatch(action)` 内部调用 `root reducer` 更新当前 `state`
7. 执行 C3 中 `next(action)` 之后的代码
8. 执行 C2 中 `next(action)` 之后的代码
9. 执行 C1 中 `next(action)` 之后的代码

这就是所谓的洋葱模型，Koa 中的中间件执行机制也是如此。

<img src='https://user-images.githubusercontent.com/58578193/70387679-e6943f80-19e2-11ea-9d36-b72291767e24.png'  width='400' />

对于上面的 `applyMiddleware(logger, crashReporter)`，如果我们执行

```javascript
export const store = createStore(
  counter,
  applyMiddleware(logger, crashReporter)
);

store.subscribe(() => console.log("store change", store.getState()));

store.dispatch({ type: "INCREMENT" });
```

结果将是

![image](https://user-images.githubusercontent.com/58578193/70387959-b2bb1900-19e6-11ea-9633-3abaa4e8942b.png)

先触发 logger，输出 `dispatching`，执行 `next(action)`；然后在 crashReporter 中无异常，没有输出；执行 Reducer，得到新的 state，store 中监听到状态变化，输出 `store change`；最后执行 logger 中 `next` 之后的语句。

如果是 `store.dispatch()`，因为 action 必须是一个对象，所以在 crashReporter 中将会捕获异常，并抛出错误，结果为：

![image](https://user-images.githubusercontent.com/58578193/70388006-3e34aa00-19e7-11ea-8500-ac9636aa9d54.png)

[demo](https://codesandbox.io/s/redux-demo-nm51e) 
https://codesandbox.io/s/redux-demo-nm51e

## redux-thunk

这应该是最常用到的 Redux 中间件，也是我们在 Redux 中处理异步请求的常用方案。redux-thunk 的实现非常简单，只有[ 14 行代码（包括空行）](https://github.com/reduxjs/redux-thunk/blob/master/src/index.js)：

```javascript
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

其主要逻辑为，检查 `action` 的类型，如果是函数，就执行 `action` 函数，并把 `dispatch` 和 `getState` 作为参数传递进去；否则就调用 `next` 让下一个中间件继续处理 `action`。

所以，我们可以通过使用 redux-thunk 来在 action 生成器（action creator）中返回一个函数而不是简单的 action 对象。从而实现 action 的异步 dispatch，如：

```javascript
const INCREMENT_COUNTER = 'INCREMENT_COUNTER';

function increment() {
  return {
    type: INCREMENT_COUNTER,
  };
}

function incrementAsync() {
  return (dispatch) => {
    setTimeout(() => {
      // Yay! Can invoke sync or async actions with `dispatch`
      dispatch(increment());
    }, 1000);
  };
}
```

或在特定条件下才发送 action，如：

```javascript
function incrementIfOdd() {
  return (dispatch, getState) => {
    const { counter } = getState();

    if (counter % 2 === 0) {
      return;
    }

    dispatch(increment());
  };
}
```

## 参考文章
[《深入浅出 React 和 Redux》·程墨](https://book.douban.com/subject/27033213//)

[《redux 中间件入门到编写，到改进，到出门》](https://quanru.github.io/2017/03/18/%E7%BC%96%E5%86%99%20redux%20%E4%B8%AD%E9%97%B4%E4%BB%B6/)
