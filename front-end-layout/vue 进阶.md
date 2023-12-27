## 特征一：模板化 => `vue-template-compiler` (基于模板解析编译)

-  解析模板，调用转换出的 render 函数将内容渲染

存在缺点：静态，使用模板内嵌时将比较复杂

### vue 插槽 - 模板动态化

>[!tip] 插槽只存在 template 中

#### 默认插槽

>[!note] 组件外部维护参数以及结构，内部安排位置摆放

- 面试点：

1. 默认插槽实现方式，多元素聚合是如何处理的？
    答：元素全部放入插槽所在位置，顺序依照外部的结构

例：
```vue
	<!-- App.vue -->
	<HelloWorld>
		<p>{{ msg }}</p>
		<h3>Stardust</h3>
	</HelloWorld>

	<!-- components/HelloWorld.vue -->
	<template>
		<div class="hello">
			<slot></slot>
		</div>
	<template>
```

渲染结果：
```html
	<div class="hello">
		<p>hello vue</p>
		<h3>Stardust</h3>
	</div>
```

#### 具名插槽

弱相关 => 如何看待 HOC，高阶组件还是高阶函数？
注：高阶函数（HOC）

>[!note] 以 name 标识当前插槽的身份，从而在组件内部能够区分

注：具名插槽的结构被 template 包裹，对应的 name 在 template 上。

书写方式：
```vue
	<!-- App.vue -->
	<HelloWorld>
		...
		<template v-slot:header>
			<h3>header</h3>
			<span>III</span>
		</template>
	</HelloWorld>

	<!-- components/HelloWorld.vue -->
	<template>
		<div class="hello">
			...
			<slot name="header"></slot>
		</div>
	<template>
```

渲染结果：
```html
	<div class="hello">
		<h3>header</h3>
		<span>III</span>
		<p>hello vue</p>
		<h3>Stardust</h3>
	</div>
```

语法糖：==`v-slot:`== 可以简写成 ==`#`== 

上述书写方式可以这样写 ==`<template v-slot:header>`== -> ==`<template #header>`==

面试点：原理

name 仅起到索引作用，看渲染结果可知，节点独立渲染

问题：混合传参 -> 外部做结构描述，内部做参数混合

答：作用域插槽 ↓

#### 作用域插槽

>[!tip] 特点：父决定插槽整体结构，子决定插槽内部变量

### 模板二次加工

1. watch  |  computed
    => 配置 or hook 中书写，computed 一定有返回值，watch 则不然

2. 其余方案
	
	a. 过滤器 - filter：vue 3 已废弃
	追问：vue 2 中 filter 可以拿到内部 vue 的实例（this）么？
	答：不可以，其只能对传入的参数做改变。
		Filter -> 纯函数，对外界无污染，做入参修改然后返回修改后的内容。
	
	b. v-html
	追问：安全性，v-html 面对 `xss`，v-html 内容中若包含 vue 模板语法，其并不会被解析
	答：v-html 会将传入的内容作为普通的 HTML 插入，如果内容包含风险...
	解决办法：iframe sandbox => 使用 `<iframe>` 标签，并在上面增加属性 `sandbox`
		例：`<iframe src="demo_iframe_sandbox.htm" sandbox="" v-html="inner"></iframe> `
		
	上述例子中的内容加载后不会影响模板

### jsx 更自由的 all in js

面试点：

1. 指令实现
2. jsx 优势和劣势
    优势：更加自由、符合 js 的书写习惯和方式
    劣势：无法使用 vue 的指令，vue 的性能优化策略，比如 => vue 的编译过程：
    `template` => `render()` => `vm.render()`

双链 diff：

==1 2 3 4== 5 ==6 7==

==7 6== ==4 3 2 1== 0

先位移，再删除，再增加
有 key 更方便,

## 特征二：组件化

```js
	import Vue from 'vue'

	Vue.component('component', {
		template: '<h1></h1>'
	})

	new Vue({
		el: '#app'
	})
```

i：抽象复用
ii：精简 & 聚合

### 混入 mixin - 逻辑混入

>[!tip] 注入型逻辑块，每一块都是独立的，互相不耦合

应用场景：抽离公共逻辑（逻辑相同，模板不同，使用 mixin）

1. 面试点：在当前组件中放入多个 `mixin`，顺序为 `[dMixin, dMixinC, dMixinD ...]` ，他们的合并策略是？
   
    I：变量补充 => 补充额外变量（mixin 中定义的内容）, 不会覆盖组件原有的
    II：多 `mixin` 中相同的配置，比如 `dMixin` 的 `data` 中有属性 `msg：'hi'` ，`dMixinC` 的 `data` 中也有属性 `msg: 'hi vue'` 那么，后面的 `dMixinC` 的 `msg` 将替换 `dMixin` 的 `msg`
    III：生命周期 => `mixin` 在引用顺序的基础上加载生命周期，且都在组件（主体）之前。

### 继承拓展 extends - 逻辑上的共同拓展

>[!tip] 核心功能的继承和拓展

Vue 2 option：`extends` 接收 `js` 

```js
	import demoExtends from './fragment/demoExtends.js'

	export default {
		extends: demoExtends
	}
```

1. 应用 & 合并策略

	I：应用：核心逻辑的功能继承
	II：合并策略
		a. 变量补充 => 补充额外变量（extends 中定义的内容）, 不会覆盖组件原有的
		b. 生命周期 => 不论 mixin 还是业务代码全都在 extends 之后

### extend & plugin & Vue.prototype

`Vue.extend`
`Vue.use()` => 组件或是引用方的 apply，将其挂载在 vue 上
挂载在原型上，比如 `ajax：Vue.prototype.ajax`，调用：`this.ajax`

### vue 3 compositionAPI