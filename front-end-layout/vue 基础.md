
## 理论

### 面试题 1：简单聊聊对 MVVM 的了解

>[!info] web 应用的发展史以及旁支

**发展史**

1. 最初：语义化模板 => html 的语义化标签组成的结构

2. 发展：MVC => model view controller (属性，视图，监听)
    MVC 缺点：针对单元素具体属性修改，若多元素处理将变得麻烦。
  
3. 目前：MVVM => model view viewModel（数据，视图，视图数据组合层）
    i. 数据本身会绑定在 viewModel 层，并且自动将数据渲染到页面中（自动化渲染，数据到视图的正向关联）
    
    ii. 视图发生变化时，通知 viewModel 层传，递给数据

## 写法

### vue 是如何利用 MVVM 思想进行项目开发？

>[!info] 数据的双向绑定

a. 利用花括号：`{{}}`，构筑了 model 与 viewModel 的绑定关系 => buildTemplate 通过特定条件分类 template，区分静态 / 动态节点。静态保持内容，动态再做区分。变量 =>变量节点，在渲染前确定变量的值，组件 => 动态组件节点，根据路径寻找到组件并渲染在当前节点上，替换动态组件节点。

b. 通过视图绑定事件，去处理数据，例：`@click = "handleClick()"`
   语法糖 v-model === : value @input

例：
```vue
<basic-components v-model="arr" :value="arr" @input="handleInput">
</basic-components>

<script>
	export default {
		name: 'H',
		props: {
			msg: String
		},
		data() {
			return {
				arr: ['a', 'r', 'r']
			}
		},
		methods: {
			handleInput(val) {
				// ...
			}
		}
	}
</script>
```
```js
// basic-components

props: {
	value: {
		type: Array,
		default: () => []
	}
},
data() {
	return {
		data: []
	}
}
methods: {
	handleChangeData() {
		let c = this.value
		// ...
		this.data = c
		this.$emit('input', data)
	} // 组件向上传递数据
}
```

render 函数相当于 viewModel？

答：是

1. template 是 viewModel

2. BuildTemplate 将 template 组装为 `render()`，在要被渲染时调用
    ==注==：template => `render()`，template 转换为 render 函数

3. 此时 `render()` 就相当于 viewModel，函数内部包含静态和动态内容

  

渲染函数：h('', {}, onClick() | contents) => createElement
==注==：h 函数参数依次为：标签名称，属性，方法和内容

JSX 实例：

```js
render() {
	return {
		<div class="listForm">{{ this.list }}</div>
	}
}
```

等价于

```vue
<template>
	<div class="listForm">{{ list }}</div>
</template>
```

JSX 的缺点：JSX 无法使用 vue 的指令，且缺少 vue 对渲染的优化

vue 源码：优化部分，渲染前的处理
mergeOptions => 合并、处理组件配置，所有能被合并、处理的数据：model

### 生命周期

#### 面试题 2：生命周期

>[!info] 生命周期详述

创建阶段：beforeCreate => created => beforeMount => mounted

更新阶段：beforeUpdate => updated

销毁阶段：beforeDestroy => destroyed

1. beforeCreate 和 created 区别：
   
   - beforeCreate：`new Vue()` - 实例创建阶段，但实例未创建完成
   - created：data | props | method | computed - 内部属性的挂载与数据操作在此部分执行，完成实例化，不涉及 DOM 和 vDOM 操作

2. beforeMount 和 mounted 区别：
   
   - beforeMount：vDOM 更新完成，DOM 未更新
   - mounted：DOM 更新完成，能做任何操作

3. beforeUpdate 和 updated 区别：
   
   - beforeUpdate：vDOM 更新完成，DOM 未更新
   - updated：DOM 更新完成，谨慎更新数据操作，此处更新容易触发改动，陷入无限更新循环

4. beforeDestroy 和 destroyed 区别：
   
   - beforeDestroy：实例尚未被销毁，在清空 eventBus、rest store、 clear timer
   - destroyed：实例已经被销毁，收尾

### 监听  |  条件  |  循环

>[!info] 感知数据变化，劫持数据内容以及相关依赖，收集依赖转换为依赖链，变化后推送通知并渲染

#### 监听

1. 面试点：computed 和 watch
   
    相同点：
    
    i. 基于 vue 的依赖收集机制进行采集
    ii. 都是根据依赖变化所触发，进而改变，进行处理计算

	不同点：
	
	i. 入和出不同：
	computed：多入单出 - 多个值变化，组成一个最终输出值
	watch： 单入多出 - 观察单个值变化，进而影响一系列状态变更
	
	ii. 性能：
	computed：缓存读取计算值
	watch： 定向监听值的变化一定会执行回调
	
	iii. 写法：
	computed：必须有返回值 => return
	watch： 不一定 => 注重变化后要处理的逻辑
	
	iiii：时机：
	computed：从首次生成赋值时就开始计算（==生命周期：created==）
	watch： 首次不会运行，除非设置 => ==`immediate: true`==

#### 条件

>[!info] 内置指令 & 自定义指令

==`v-if`==  |  ==`v-show`==

相同点：都可以操纵一个 dom 元素的显示

不同点：当前的 dom 元素是否渲染，== `v-if="false"` == 不渲染此 dom 元素，==`v-show="false"`== dom 元素渲染，但其不可见，仍占据空间位置。

`v-else` | `v-else-if`

`v-for`

==`v-for`==  &  ==`v-if`==  的优先级

在 `vue 2.x` if 和 for 在同一个元素上使用，`v-for` 会优先作用

在 `vue 3.x` if 总是优先于 for

==`v-model`==  |  ==`v-once`==  |  ==`v-text`==  |  ==`v-html`==  |  ==`v-bind`==  |  ==`v-on`==

- 其中 `v-bind` 的语法糖是 `:`，`v-on` 的语法糖是 `@`

##### 自定义指令

例：
```js
import Vue from "vue";

// 参数：指令名称，指令方法
Vue.directive("", {
	update: function () {
		// ...
	},
});
// 调用指令：'v-' + 指令名称
```

#### 事件 - ==`v-on`==
 
常用的修饰符：`.stop` `.prevent` `.capture` `.self` `.once` `.passive`
详细见文档...

==注==：stopPropagation，preventDefault ...

##### 事件的设计 

为何 vue 将事件注册放在模板上而不是 js 中？

1. 模板定位视图 - 主观视觉上将模板上对应元素与事件绑定，更快速定位事件逻辑
    便于定位问题

2. js 与视图绑定解耦 - 便于测试隔离

3. destroyed 时，实例销毁，自动解绑事件 - 便于垃圾回收

## 组件化

### 一般组件  |  动态组件

例：
```vue
<template>
	<Mark></Mark>
</template>

  

<!-- 动态组件 -->

<template>
	<component :is="componentName"></component>
</template>

<script>
	data() {
		return {
			componentName: 'Mark'
		}
	}
</script>
```
