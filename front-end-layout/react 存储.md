# 什么是状态管理
状态
- 现在的前端 web 工程，我们不关心过程，只关心状态。本质上 “前端是一个状态机”

![[react 存储 2024-01-19 15.50.42.excalidraw]]


管理

- 在 web 端的声明周期变化时，数据的 model 是什么，scope (作用域范围)是什么。以及如何作用于所展示的 view。

```jsx
fucntion app() {
  const [useList, setUseList] = useState([])

  useEffect(() => {
    // 数据 model 如何控制
    fetch().then(({ data }) => setUseList(data))
  }, [])
  
  return (
    {/* 数据 model 和 view 之间的映射关系 */}
    <ul>
      { useList.map(item => <li key={item.id}> {item.value} </li>) }
    </ul>
  )
}
```

# 软件工程，在做什么？

>[!quote] 软件的本质，就是在管理数据
我们首先要考虑的是，一个数据的生命周期是什么？作用范围是什么？

- `DataBase`：用户名，账户信息，`juejin` 的观看历史
- `localStorage`：搜索引擎，搜索历史
- `Cookie` / `sessionStorage `：用户登录状态
- `Project runtime`：多个路由的统一筛选框
- `component`：`[state, props, data]`

## 我们了解的状态管理

`vuex`、`redux`、`mobx` 是属于 `Project runtime` 这个作用范围

- 一旦浏览器刷新，上述这些存储的数据都没了。 => js 重新执行了
- 没刷新，数据一直有。                                 => js 还在缓存着

## 上述这样的数据，存储在那里？

- Global / window 挂一个 data
- Closure 闭包

# 状态管理如何实现

## 1 . 组件之外，可以在全局共享数据 / 状态
   - 全局不一定是 window，Closure 闭包，可以解决这个事情
## 2 . 有修改这个数据的明确方法，并且，被修改了数据，相关方能感知到
   - 本质上，就是把监听这个数据的函数，放在一个地方，必要时取出来执行此函数。
   - - 发布订阅
   - - new Proxy 、Object.defineProperty
## 3 . 修改状态，会触发 UI 更新
   - setState
   - useState
   - forceState

# Redux 实践

`src/data/data.js`
```js
export const createData = function (initData, reducer) {
  let data = initData;

  let deps = [];

  function getData() {
    return data;
  }

  function subscribe(handler) {
    // handler 对数据的处理函数，响应变化
    // 我们希望，订阅了这个数据的 handler，在数据改变时，都能执行一下
    deps.push(handler);
  }

  function changeData(val) {
    // val 传入进来的修改内容
    // 我们提供一个，修改这个数据的方法
    data = val;
    deps.forEach((fn) => fn());
  }

  return {
    getData,
    subscribe,
    changeData
  };
};
```

`src/data/index.jsx`
```jsx
import React from 'react';

import { createData } from './data';
import { useState } from 'react';
import { useEffect } from 'react';

const initData = {
    count: 1,
};

const dataObj = createData(initData);

export default function DataIndex() {

    const [count, setCount] = useState(initData.count);

    useEffect(() => {
        dataObj.subscribe(() => {
            // 这里说明是，我在初始化的时候，订阅了这个值
            // 如果有人 changeData 了，我就到这里来
            let curData = dataObj.getData();
            console.log('data is ', curData)
            setCount(() => curData.count);
        })
    }, [])

    const handleAdd = () => {
        dataObj.changeData({ count: dataObj.getData()?.count + 1 })
    }

    const handleMinus = () => {
        dataObj.changeData({ count: dataObj.getData()?.count - 1 })
    }

    return (
        <div>
            <h5>count is: {count}</h5>
            <button onClick={handleAdd}>ADD</button>
            <button onClick={handleMinus}>MINUS</button>
        </div>
    )
}
```

结论：changeData 是 unsafe

## SAFE BY ACTION

Action 是什么？ Action 是一个动作，一般由 type (类型) 和 payload (负载) 组成。

```js
const action = {
  type: 'ADD_COUNT',
  payload: null
}
```

Reducer 是什么？Reducer 是一个计算函数，通过 action 和当前的 state，得出一个新的 state

![[react 存储 2024-01-22 10.35.26.excalidraw]]

`src/data/data.js`
```js
export const createData = function (initData, reducer) {
  // ...
  
  function UNSAFE_changeData(val) {
    // val 传入进来的修改内容
    // 我们提供一个，修改这个数据的方法
    data = val;
    deps.forEach((fn) => fn());
  }

  function setDataByAction(action) {
    data = reducer(data, action);
    deps.forEach((fn) => fn());
  }

  return {
    // ...
    UNSAFE_changeData,
    setDataByAction,
  };
};
```

`src/data/index.jsx`
```jsx
import React from "react";

import { createData } from "./data";
import { useState } from "react";
import { useEffect } from "react";

const initData = {
  count: 1,
};

const myReducer = (data, action) => {
  switch (action.type) {
    case "ADD_COUNT":
      return { ...data, count: data.count + 1 };
    case "MINUS_COUNT":
      return { ...data, count: data.count - 1 };
    case "SET_COUNT":
      return { ...data, count: action.payload };
    default:
      return data;
  }
};

const dataObj = createData(initData, myReducer);

export default function DataIndex() {
  const [count, setCount] = useState(initData.count);

  useEffect(() => {
    console.log("effect");
    dataObj.subscribe(() => {
      let curData = dataObj.getData();
      console.log("data is ", curData);
      setCount(() => curData.count);
    });
  }, []);

  const handleAdd = () => {
    dataObj.setDataByAction({
      type: "ADD_COUNT",
    });
  };

  const handleMinus = () => {
    dataObj.setDataByAction({
      type: "MINUS_COUNT",
    });
  };

  const handleReset = () => {
    dataObj.setDataByAction({
      type: "SET_COUNT",
      payload: 0,
    });
  };

  return (
    <div>
      <h5>count is: {count}</h5>
      <button onClick={handleAdd}>ADD</button>
      <button onClick={handleMinus}>MINUS</button>
      <button onClick={handleReset}>RESET</button>
    </div>
  );
}
```

## Redux 的原理

![[react 存储 2024-01-22 11.27.31.excalidraw]]

蓝色框部分就是 Redux，触发 dispatch，传入 action ，reducers 调用 state 和 action 返回新的 state 由 store 发给对应组件。

Redux:
```js
export const createStore = function (reducer, initState) {
  let state = initState;

  let listeners = [];

  function getState() {
    return state;
  }

  function subscribe(handler) {
    // 我们希望，订阅了这个数据的 handler，在数据改变时，都能执行一下
    listeners.push(handler);
  }

  function dispatch(action) {
    // reducer => 是下方的 combineReducer
    const currentState = reducer(state, action);
    state = currentState;
    listeners.forEach((fn) => fn());
  }

  return {
    getState,
    subscribe,
    dispatch,
  };
};

export const combineReducer = function (reducers) {
  const keys = Object.keys(reducers); // 先拿到 ['counter', 'info']
  // 返回的是一个 reducer
  return function (state = {}, action) {
    const nextState = {};
    keys.forEach((key) => {
      const reducer = reducers[key]; // counterReducer \ infoReducer
      const prev = state[key]; // { count: 0 }, \ { name: 'luyi', age: 36 }
      // 假设 dispatch 的是一个 'ADD_COUNT' 的 action: return { count: state.count + 1 }
      // 当循环执行完了, 结果为 counter : { count: 1 }, \ info: { name: 'luyi', age: 36 }
      const next = reducer(prev, action);
      nextState[key] = next;
    });
    return nextState;
  };
};
```

Store:
```js
import { createStore, combineReducer } from "./redux";

let initState = {
  counter: {
    count: 0,
  },
  info: {
    name: "luyi",
    age: 36,
  },
};

const counterReducer = (state, action) => {
  switch (action.type) {
    case "ADD_COUNT":
      return { count: state.count + 1 };
    case "MINUS_COUNT":
      return { count: state.count - 1 };
    case "SET_COUNT":
      return { count: action.payload };
    default:
      return state;
  }
};

const infoReducer = (state, action) => {
  switch (action.type) {
    case "ADD_AGE":
      return { age: state.age + 1 };
    case "MINUS_AGE":
      return { age: state.age - 1 };
    case "SET_NAME":
      return { name: action.payload };
    default:
      return state;
  }
};

// 我要把多个 reducer 合并起来
const reducers = combineReducer({
  counter: counterReducer,
  info: infoReducer,
});

const store = createStore(reducers, initState);

export default store;
```

## React -Redux

提供一些 API，让我们更方便的进行 UI 的更新。

### Connect

```js
connect(mapStateToProps, mapDispatchToProps)(Component)
```

#### 实现

`src/index.js`
```js
// ...
import App from "./App";
import ReduxContext from "./store/context";
import store from "./store";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <ReduxContext.Provider value={store}>
    <App />
  </ReduxContext.Provider>
);
```

`src/store/connect.js`
```js
import { useContext } from "react";
import { useState } from "react";
import { useEffect } from "react";
import ReduxContext from "./context";

export const connect = (mapStateToProps, mapDispatchToProps) => (Component) => {
  return function ConnectComponent(props) {
    const _store = useContext(ReduxContext);
    const [, setBool] = useState(true);
    const forceUpdate = () => {
      setBool((val) => !val);
    };
    useEffect(() => {
      _store.subscribe(forceUpdate);
      // eslint-disable-next-line react-hooks/exhaustive-deps
    }, []);

    return (
      <ReduxContext.Consumer>
        {(store) => (
          <Component
            {...props}
            {...mapStateToProps(store.getState())}
            {...mapDispatchToProps(store.dispatch)}
          />
        )}
      </ReduxContext.Consumer>
    );
  };
};
```

获得全局数据：`const _store = useContext(ReduxContext)` 通过 `useContext` 去获取全局 `store` 在 `useEffect` 中为 `store` 增加了 `subscribe` 。`ReduxContext.Consumer` 接收传递的 `store` 给返回的组件上加入 `mapStateToProps` 和 `mapDispatchToProps` , 给组件传递了 `state` 和 `dispatch` ...

# Mobx 浅析

## 响应式核心原理：
```js
let reactions = [];

let deps = null;

const handler = () => {
  return {
    get(target, key, descriptor) {
      if (deps) {
        // 执行过程中，触发了get, 说明 autorun 有调用我的响应式数据
        // deps 为打印，存入打印
        reactions.push(deps);
      }
      return Reflect.get(target, key, descriptor);
    },
    set(target, value, key, descriptor) {
      const res = Reflect.set(target, value, key, descriptor);
      // 修改响应式对象或其内部值，触发打印
      reactions.forEach((fn) => fn());
      return res;
    },
  };
};

function walk(data, handler) {
  if (typeof data !== "object" || data === null) return data;
  for (let key in data) {
    data[key] = walk(data[key], handler);
  }
  return new Proxy(data, handler());
}

function observable(data) {
  return walk(data, handler);
}

function autorun(fn) {
  deps = fn;
  fn();
  deps = null;
}

//  ******************************

const data = {
  count: 0,
};

const store = observable(data);

// console.log(store);

autorun(() => {
  console.log("autorun:", store.count);
});

store.count = 1;
store.count = 2;
```

解析：定义 `data` 对象包含 `count` 属性，值为数值 `0` ，传入函数 `observable` 执行 `walk(data, handler)` 

其中 `handler` 重写 `get` 和 `set` 使用 `Reflect.get(target, key, descriptor)` 获取 `target` (目标)对象上名为 `key` 属性的值 `targe[key]` 、`Reflect.set(target, value, key, descriptor)` 为 `target` (目标) 对象分配 `value` 给指定属性 `key`, 如果不存在属性则新建，否则覆盖 `target[key] = value` 。`get` 中设置了当前若存在依赖 `deps` 将其存入反应队伍 `reactions` 中，在 ` set ` 中执行...

函数 `walk` 遍历传入的 `data` 层层递进，给 `data` 中的每个属性加上 `Proxy` 代理其 `get` 和 `set` 。最终代理整个 `data` ：`return new Proxy(data, handler())` 使得整个 `data` 对象都是响应式的。

函数 `autorun` ：接收参数为函数 (此处函数为打印)，将函数赋值给依赖 `deps` 并执行函数，会发什么？

打印被记录到反应队伍中 `reactions` ，在函数执行时。语句 `console.log("autorun:", store.count)` 中 `store.count` 调用了代理对象 `get` ，存入打印 `reactions.push(deps)` ，返回 `0` ，打印执行 `autorun: 0`

`store.count = 1` ，触发代理对象 `set` 返回 true，执行存入 `reactions.forEach((fn) => fn())` ，后续和上述一样：语句 ...

## 常用 API

使用需要在 `package.json` 中配置对装饰器的解析
```json
"babel": {
  "presets": [
    "react-app"
  ],
  "plugins": [
    [
	  "@babel/plugin-proposal-decorators",
	  {
	    "legacy": true
	  }
    ]
  ]
},
"devDependencies": {
  "@babel/plugin-proposal-class-properties": "^7.18.6",
  "@babel/plugin-proposal-decorators": "^7.22.15"
}
```

`/src/mobx/store.js`
```js
import { computed, makeObservable, observable, action } from "mobx";

class Store {
  constructor() {
    makeObservable(this);
  }

  @observable count = 0;

  @observable info = { name: "luyi" };

  @action.bound
  add_count() {
    this.count = this.count + 1;
  }

  @action.bound
  setInfo(info) {
    this.info = info;
  }

  @computed get double() {
    return this.count * 2;
  }
}

/* 写法 2 */

class Doubler {
  value

  constructor(value) {
    makeObservable(this, {
      value: observable,
      double: computed,
      increment: action,
    })
    this.value = value
  }

  get doubule() {
    return this.value * 2
  }

  increment() {
    this.value++
  }
}

export default new Store();
```

```jsx
import React from "react";
import { Observer, useLocalObservable } from "mobx-react";
import store from "./store";

export default function MobxIndex() {
  // 根据 store 创建一个可观察对象
  const r_store = useLocalObservable(() => store);
  return (
    <div>
      <!-- <Observer>{() => rendering}</Observer> -->
      <!-- Observer 渲染括号内给定的函数，当观察值发生变化就会重新渲染  -->
      <Observer>
        {() => (
          <div>
            <h2>{r_store.count}</h2>
            <button onClick={() => r_store.add_count()}>+</button>
          </div>
        )}
      </Observer>
    </div>
  );
}
```