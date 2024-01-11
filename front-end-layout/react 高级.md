## 生命周期

### 1.1 类组件

#### 1.1.1 初始化阶段

#### constructor 执行

- 初始化 state，截取路由参数
- 防抖、节流

```js
constructor(props) {
  super(props)
  console.log('constructor 执行')

  this.state = {
    number: 0
  }

  this.data = {
    nameL 'stardust'
  }
}
```

#### `getDerivedStateFromProps`

- 是一个静态函数，因为 react 官方，希望你把这个函数，当作一个纯函数来使用
- 传入 props 和 state，返回值将和之前的 state 进行合并，作为新的 state

```js
static getDerivedStateFromProps(props, state) {
  const { num } = props
  const number = num.length > 5 ? 'many...' : num.length
  return {
    number,
    nameList: props.name.split('')
  }
}
```

#### `componentWillMount`

在组件渲染之前，去执行的一个函数
- ==**若组件中，已经有了 `getDerivedStateFromProps` 生命周期函数，则是不会执行 `componetWillMount` 的**==
- 一般情况下，有时候对于一些需要对接口数据进行预加载的时候，会在这里提前进行数据请求。

#### `render` 

首次渲染的逻辑，已经完成。

#### `componentDidMount`

对标 vue 中的 mounted

#### 1.1.2 更新阶段

#### `componentWillReceiveProps`

- 有一些数据，props 发生改变的时候，props 的改变来决定 state 是否更新
- 弹窗 visible - state.
- ==**若组件中，已经有了 `getDerivedStateFromProps` , `componentWillReceiveProps` 就不执行**==

#### `shouldComponentUpdate`

相当于一个拦截器，返回值是一个 bool，**决定组件要不要更新**

`shouldComponentUpdate` 的 `nextState` 能够拿到 `getDerivedStateFromProps` 生命周期所更新的内容

#### `componentWillUpdate`

- 获取组件更新前的一些状态，DOM 位置

#### `render`

#### `getSnapShotBeforeUpdate`

更新前的快照

```js
getSnapShotBeforeUpdate(preProps, preState) {
  // 处理一些 DOM 相关的
  return null
}
```

#### `componentDidUpdate`

```js
componentDidUpdate(props, state, snapShot) {
  // 注意不要使用 setState，回造成死循环 ...
  // !!! 会造成死循环
  /* this.setState({
    number: this.state.number
  }) */

  // setState 请在 componentWillReceiveProps 中做
}
```

#### 1.1.3 销毁阶段

####  `componentWillUnmount`

闭包，事件管理，定时器，在这里销毁

### 1.2 函数组件（重点）

#### 1.2.1 Effect

#### `useEffect`

`useEffect(() => destory, deps)`

- `() => destory` ：第一个参数，callback 函数
     `destory` ：作为这个函数的返回值，会在下一次 callback 函数执行前调用，用于清除上一次 callback 的副作用

- `deps` ：第二个参数，是一个数组，数组中的值发生变化，会执行这个 callback 返回的 `destory` ，然后再次执行 callback 函数

问：Effect 是如何模拟生命周期的？

```js
  const [state, dispatch] = useState(() => {
   // getDerivedStateFromProps
    return true;
  });

 useEffect(() => {
    console.log("componentDidMount");
    return () => {
      console.log("componentWillUnmount");
    };
  }, []);

  useEffect(() => {
    // 绑定，当发生变化，执行
    console.log("componentWillReceiveProps");
  }, [props]);

  useEffect(() => {
    // 无绑定，怎么都会执行一次，必更新
    console.log("componentDidUpdate");
  });
```

## 一些高级的 API

### ref

#### 创建

类组件：`createRef`

```jsx
import React, { Component, createRef } from "react";

export default class ClassRef extends Component {
  constructor(props) {
    super(props);

    this.elRef = createRef();
    this.inputRef = createRef();
  }

  handleClick = () => {
    // 光标锁定 input
    this.inputRef.current.focus();
    // 通过 id 拿到 div 对象会破坏 react 的封装
  };

  render() {
    return (
      <div>
        <div id="is_your" ref={this.elRef}></div>
        <input ref={this.inputRef} />
        <button onClick={this.handleClick}>Focus</button>
      </div>
    );
  }
}
```

函数组件：useRef

```jsx
import React, { useRef } from "react";

export default function FunctionRef() {
  const inputRef = useRef(null);

  const handleClick = () => {
    inputRef.current.focus();
  };

  return (
    <div>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus</button>
    </div>
  );
}
```

#### 转发

```jsx
import React, {
  forwardRef,
  useImperativeHandle,
  useRef,
  useState,
} from "react";

export default function FunctionRef() {
  const exposeRef = useRef(null);
  const modelRef = useRef(null);

  return (
    <div>
      <div>
        <FancInput ref={exposeRef} />
        <FancModel ref={modelRef} />
      </div>
      <button onClick={() => modelRef.current.setVisible(true)}>setting</button>
      <button onClick={() => exposeRef.current.focus()}>1</button>
      <button onClick={() => exposeRef.current.changeValue("hi stardust")}>
        2
      </button>
    </div>
  );
}

// 向外提供一些方法，让父组件直接调用，类似 vue 的 expose
// useImperativeHandle 函数签名：useImperativeHandle(ref, createHandle, [deps])
// ref 为需要赋值的 ref 对象，
// createHandle：函数，其返回值将作为 ref.current 的值，
// [deps]：依赖数组，依赖发生变化会重新执行 createHandle 函数
const MyInput = (props, ref) => {
  const inputRef = useRef();
  const focus = () => {
    inputRef.current.focus();
  };
  const changeValue = (val) => {
    inputRef.current.value = val;
  };
  useImperativeHandle(ref, () => ({
    focus,
    changeValue,
  }));
  return <input ref={inputRef} />;
};

// forwardRef：返回 react 组件，接收参数 => React.forwardRef(render)，render 函数
// render 函数签名：render(props, ref) 其中 ref 是组件定义的 ref，同时也将内部定义的属性转发给外部的组件
// 函数内部 useRef 创建的 ref 将和组件定义的 ref 合并。
const FancInput = forwardRef(MyInput);

const MyModel = (props, ref) => {
  const [vis, setVis] = useState(false);

  const setVisible = (val) => {
    setVis(val);
  };

  useImperativeHandle(ref, () => ({
    setVisible,
  }));

  return <div>state: {vis ? "显示" : "隐藏"}</div>;
};

const FancModel = forwardRef(MyModel);
```

### context

类似 `Vue` 中的 inject 和 provide
在 `React` 中是 consumer 和 provide

`ThemeContext.jsx`
```jsx
import React from "react";

export const ThemeContext = React.createContext("light");
```

#### 类用法 ：

`ClassContext.jsx`
```jsx
import React, { Component } from "react";
import { ThemeContext } from "./ThemeContext";

export default class ClassContext extends Component {
  state = {
    theme: "dark",
  };

  render() {
    return (
      <ThemeContext.Provider value={this.state.theme}>
        <Parent0 />
        <button onClick={() => this.setState({ theme: "light" })}>light</button>
        <button onClick={() => this.setState({ theme: "dark" })}>dark</button>
      </ThemeContext.Provider>
    );
  }
}

const Parent0 = () => (
  <div>
    <Child1 />
    <Child2 />
  </div>
);

class Child1 extends Component {
  static contextType = ThemeContext;
  render() {
    return <div>Child1 -- {this.context}</div>;
  }
}

class Child2 extends Component {
  render() {
    return (
      <div>
        <ThemeContext.Consumer>
          {(theme) => <div>Child2 -- {theme}</div>}
        </ThemeContext.Consumer>
      </div>
    );
  }
}
```

#### 函数用法 ：

`FuncContext.jsx`
```jsx
import React, { useContext } from "react";
import { ThemeContext } from "./ThemeContext";

const navigator = window.history;

export default function FuncContext() {
  return (
    <ThemeContext.Provider value={navigator}>
      <Parent />
    </ThemeContext.Provider>
  );
}

const withNavigator = (Component) => {
  return function () {
    const navigator = useContext(ThemeContext);
    return <Component navigator={navigator} />;
  };
};

const Parent = () => <Child />;

const Child = withNavigator((props) => {
  console.log(props.navigator);

  return <div>hello</div>;
});
```

上文代码 `withNavigator` 展示了高阶组件用法，React 高阶组件（HOC）：是灵活使用 `react` 组件的一种技巧，高阶组件本身不是组件，它是一个参数为组件，返回值也是组件的函数。
主要作用：**强化组件，复用逻辑，提升渲染性能 ...**

```jsx
const withNavigator = (Component) => { 
  // 参数 Component => Child_Old
  return function () {
    // 注入 navigator
	const navigator = useContext(ThemeContext);
    // 返回 <Component navigator={navigator} /> => Child
    return <Component navigator={navigator} />;
  };
};

const Child_Old = (props) => {
  // 层层注入，此处 navigator 等同于 props.navigator
  const navigator = useContext(ThemeContext);
  return (
    <div>
      <button onClick={() => navigator.pushState({}, undefined, "./hello")}>
        go hello
      </button>
    </div>
  );
};

const Child = withNavigator(Child_Old);
```

### 高阶组件（HOC）

#### 属性代理

```jsx
import React from "react";

function PropertyHOC() {
  return (
    <div>
      <span>ordinary component</span>
    </div>
  );
}

//  我们期望定义一个卡片装饰器，让这个组件，具有卡片的样式

const withCard = (color) => (Component) => {
  return (props) => {
    const cardStyle = {
      minWidth: "188px",
      background: `rgba(${color.r}, ${color.g}, ${color.b}, .33)`,
      fontSize: "18px",
      color: `rgb(${color.r}, ${color.g}, ${color.b}`,
      marginTop: "10px",
      padding: "120px 6px",
      borderRadius: "4px",
      display: "inline-block",
      boxShadow: `4px 4px 1px 0 rgba(${color.r}, ${color.g}, ${color.b}, .66)`,
    };

    return (
      <div id="card" style={cardStyle}>
        <Component {...props} />
      </div>
    );
  };
};

// 填入 rgb 数值
export default withCard({ r: 0, g: 0, b: 0 })(PropertyHOC);
```

#### 反向继承

```jsx
import React, { Component } from "react";

export default function ExtendingHOC() {
  return (
    <div>
      <LogIndex />
    </div>
  );
}

// 有一个 case，非常优雅的实现埋点曝光
function logProps(logMap) {
  return (wrappedComponent) => {
    const didMount = wrappedComponent.prototype.componentDidMount;
    // 通过继承去拿到传入组件的能力
    return class A extends wrappedComponent {
      componentDidMount() {
        // 劫持了 Index 组件的 componentDidMount, 在 Index 执行之后
        if (didMount) {
          didMount.apply(this);
        }
        // 在这之后执行埋点曝光
        Object.entries(logMap).forEach(([k, v]) => {
          if (document.getElementById(k)) {
            console.log("dig", v);
          }
        });
      }
    };
  };
}

class Index extends Component {
  render() {
    return (
      <div>
        <div id="text">simple component</div>
      </div>
    );
  }
}

const LogIndex = logProps({ text: "text_module_show" })(Index);
```

### 渲染优化

#### 类用法 ：

```jsx
import React, { Component } from "react";

export default class RenderControl extends Component {
  constructor(props) {
    super(props);
    this.state = {
      num: 0,
      count: 0,
    };

    // 子组件
    this.component = <Child num={this.state.num} />;
  }

  // 需求，当 num 发生改变再渲染子组件
  controlComponentRender = () => {
    const { props } = this.component;
    if (props.num !== this.state.num) {
      // 当前 num 未发生修改，完全克隆当前的子组件
      return (this.component = React.cloneElement(this.component, {
        num: this.state.num,
      }));
    }
    // 反之直接返回子组件就好
    return this.component;
  };

  // 因为被 render 函数包裹，所以当发生改变，子组件 Child 就会渲染，即使 num 没有改变
  render() {
    const { num, count } = this.state;

    return (
      <div>
        {this.controlComponentRender()}
        <button onClick={() => this.setState({ num: num + 1 })}>
          n + {num}
        </button>
        <button onClick={() => this.setState({ count: count + 1 })}>
          c + {count}
        </button>
      </div>
    );
  }
}

const Child = ({ num }) => {
  console.log("Child Render");
  return <div>th child num: {num}</div>;
};
```

#### 函数用法 ：

```jsx
import React, { useMemo, useState } from "react";

export default function RenderControlFunc() {
  const [num, setNum] = useState(0);
  const [count, setCount] = useState(0);

  return (
    <div>
      {useMemo(
        () => (
          <Child num={num} />
        ),
        [num]
      )}
      <button onClick={() => setNum(num + 1)}>n + {num}</button>
      <button onClick={() => setCount(count + 1)}>c + {count}</button>
    </div>
  );
}

const Child = ({ num }) => {
  console.log("Child Component");
  return <div>th Child num: {num}</div>;
};
```

### 缓存控制，在指定的内容发生变化才再次渲染

#### useMemo 

函数签名：`useMemo(function, deps)`
- `function`：对返回值进行缓存
- `deps`：依赖项，谁变化，再次执行 `function`

#### useCallback

函数签名：`useCallback(function, deps)`
- `function`：将函数进行缓存
- `deps`：依赖项，谁变化，再次执行 `function`
