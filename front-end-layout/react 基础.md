## React 浅析

### 1.1 传统方式，如何构建用户界面？

![[Drawing 2024-01-07 16.33.19.excalidraw]]

js - 浏览器脚本

本质上，js 是可以操作 dom 的

```html
<script>
  document.getElementById('app').addEventListener("click", () => {})
  const div = document.createElement('div')
</script>
```

### 1.2 基于 js 来创建 dom

```js
function createEl(type = '', params, children = []) {
  let el
  if (type.toLowerCase() === "div") {
    el = document.createElement('div)
  }
  if (type.toLowerCase() === "div") {
    ......
  }
  ......

  // 假设我们有个处理属性的方法
  Element.resolveAttributes.call(el, params)
  // 将 children 挂载到创建的元素上
  children.forEach(child => el.appendChild(child))
  return el
}

// 使用这个方法
const btn = createEl('button', {
  style: "background-color: 'red'"
})

const div = createEl('div', {
  className: "my_app"
}, [btn])
```

按照这个逻辑我们可以这样做

```html
<div id="app">
  <h1>hello</h1>
  <h2>this is</h2>
  <h3>bravado</h3>
</div>
```

```js
createEl('div', { id: "app" }, [
  createEl('h1', {}, ['hello'])
  createEl('h2', {}, ['this is'])
  createEl('h3', {}, ['bravado'])
])

// 如果还有别的内容，一直调用 createEl ，代码很复杂
// React 的方法是

// React 让 babel 团队 -> @babel/preset-react

function C() {
  return React.createElement("div", { id: "app" },
    React.createElement("div", null, "hello"),
    React.createElement("div", null, "this is"),
    React.createElement("div", null, "bravado")
  )
}

// 这里的 React.createElement 做了什么，为什么不再是用 document.createElement ？
```

### 1.3 React.createElement 做了什么，为什么不再是用 document. CreateElement ？

![[react 基础 2024-01-07 17.43.11.excalidraw]]

React 的 createElement 在生成 dom 前做了很多操作

### 1.4 前端在做什么

![[react 基础 2024-01-07 17.51.19.excalidraw]]

变化：
![[react 基础 2024-01-07 18.01.18.excalidraw]]

对比 vue 和 react 的循环书写

```vue
<div>
  <span v-for="item in list" :key="item.id">
    {{ item.name }}
  </span>
</div>
```

React 的写法非常灵活
```jsx
<div>
  {
    list.map(item => <span key={item.id}>{ item.name }</span>)
  }
<div>

<div>
  {
    (() => {
      const res = []
      for(item of list) {
	      // ...
      }
    })()
  }
<div>
```

## 创建 React 项目

 > [!quote] 语句
 > js: npx create-react-app program-name
 > 
 > ts: npx create-react-app program-name --template typescript
 
`npm run eject` 可以暴露执行脚本

## React 基本能力

### 3.1 父子组件
![[react-20240108-113745.png]]

组件：class 和 function 有什么区别？

16.8 之前：只有 class
- this 指向，组件状态管理，生命周期 ... 这些都比较复杂

16.8 之后
- function 函数式组件
- class 和 function 的 api 不同，但是对标下来都是有相同功能的

### 3.2 props & state

#### 3.2.1 state

更新界面的时候，需要修改 state ==（双向绑定）==
如果有一个变量，你希望它的改变，触发界面更新，
在 react 中需要把这个数据放到 state 里，并用特定的方法去更新它。

> [!note] 类组件：
> 类组件修改，必须使用 setState 方法
> 
> - state 的值，互相不影响
> - 第二个参数，cb，能拿到最新的值

```jsx
import React, { Component } from 'react'

export default class ClassCom extends Component {
  constructor(props) {
    super(props)

    this.state = {
      number: 0,
      msg: 'hi'
    }
  }

  handleClick = (type) => {
    this.setState({
      number: this.state.number + (type === 'PLUS' ? 1 : -1)
    }, () => {
      // setState 的 第二个参数，为最新的 state
      console.log('cb state', this.state.number)
    })
    // state.number 还是当前值
    console.log('fn state', this.state.number)
  }

  handleChange = (event) => {
    this.setState({
      msg: event.target.value
    })
  }

  // 函数需要绑定 this
  handleClickFn = function(type) {
    this.setState({
      number: this.state.number + (type === 'ADD' ? 1 : -1)
    })
  }

  render() {
    const { number, msg } = this.state
    return (
      <div>
        {msg} ClassCom
        <p>{number}</p>
        {/* 这里形成了双向绑定的关系 */}
        <input value={msg} onChange={this.handleChange} />
        <button 
	      onClick={this.handleClick.bind(this, 'ADD')}>
	      +  
	    </button>
	    
        <button 
          onClick={() => this.handleClick('MINS')}>
          -
        </button>
        {/* 
          报错，为什么？
          因为这个地方的 this 没有指向 '类',
          解决办法：1. 要么 bind this，2. 要么使用箭头函数
        */}
        <button onClick={this.handleClickFn}>---</button>
      </div>
    )
  }
}
```

>[!note] 函数组件：
> `useState`
> 写法：
> `[state, dispatch] = useState(initState)`
> - `state`： 作为组件的状态，提供给 ui 渲染视图
> - `dispatch`： 修改 `state` 的方法，同时会触发组件更新
>   `dispatch` 值可以是函数，也可以不是。是函数：更新为函数的执行结果，不是：直接更新为值
> - `initState`： 初始值

```jsx
import React, { useState } from 'react'

export default function FunctionCom() {

  const [number, setNumber] = useState(0)
  const [msg, setMsg] = useState('hi')

  // 事件处理
  const handleChange = (event) => {
    setMsg(event.target.value)
  }

  return (
    <div>
      <h2>{msg} Function</h2>
      <div>the number is: {}</div>
      <p>{number}</p>
      <button onClick={() => setNumber(number + 1)}>+</button>
      <button onClick={() => setNumber(number - 1)}>-</button>
      <input value={msg} onChange={handleChange}/>
    </div>
  )
}
```

#### 3.2.2 props

Props 是父子组件传值的工具

父传子

类组件：`this.props.xxx`
函数组件：`props.xxx`

![[screenshot-20240108-162517.png]]

子传父

![[screenshot-20240108-163621.png]]

### 3.3 条件渲染与列表

#### 3.3.1 条件渲染

```jsx
import React, { useState } from "react";

// 我们可以在任意地方，使用三元的形式，显示我们想要的东西

const Hide = () => <div>No Title</div>;

const Title = ({ title }) => {
  return title.length ? <h2>{title}</h2> : <Hide />;
};

export default function FuncRender() {
  const [isShow, setIsShow] = useState(true);

  return (
    <div>
      <h2>条件渲染</h2>

      {/* React 组件处理 */}
      <Title title={isShow ? "render" : ""} />
      {/* 当成一个普通函数 */}
      {/* {Title({ title: isShow ? "render" : "" })} */}

      <button onClick={() => setIsShow(!isShow)}>
        {isShow ? "hide" : "show"}
      </button>
    </div>
  );
}
```

#### 3.3.2 列表

```jsx
import React, { useEffect, useState } from "react";

export default function FuncRender() {
  const [list, setList] = useState([]);

  useEffect(() => {
    setList([
      { name: "luyi" },
      { name: "yunyin" },
      { name: "xiaozao" },
      { name: "yufeng" },
    ]);
  }, []);

  return (
    <div>
      <h2>人员列表</h2>
      <ul>
        {list.map((item, idx) => (
          <li key={idx}>{item?.name}</li>
        ))}
      </ul>
      <ol>
        {(() => {
          let res = [];
          for (let item of list) {
            res.push(<li key={item?.name}>{item?.name}</li>);
          }
          return res;
        })()}
      </ol>
    </div>
  );
}
```
