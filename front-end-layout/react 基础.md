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

### 1.3 React. CreateElement 做了什么，为什么不再是用 document. CreateElement ？

![[react 基础 2024-01-07 17.43.11.excalidraw]]

React 的 createElement 在生成 dom 前做了很多操作

### 1.4 前端在做什么

![[react 基础 2024-01-07 17.51.19.excalidraw]]

变化：
![[react 基础 2024-01-07 18.01.18.excalidraw]]

对比 vue 和 react 的循环书写

```vue
<div>
  <span v-for="item in list" :key="item.id">{{ item.name }}</span>
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

