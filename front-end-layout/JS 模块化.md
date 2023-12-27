## 历史背景

JS 本身就是为了简单页面设计的：
页面动画、表单提交、并没有任何命名空间或者模块化相关的概念

> [!info]
> 随着业务的飞速扩张，针对 JS 的模块化涌现出了大量的解决方案

## 幼年期：无模块化

1. 开始需要在页面中增加不同类型的 JS 文件：动画.js、验证.js、格式化.js
2. 多种 JS 为了维护和可读性，被分在了不同的 JS 文件中
3. 不同的文件在同一模板中被引用

```html
	<script src="jquery.js"></script>
	<script src="main.js"></script>
	<script src="util.js"></script>
```

==总结==：认可（在当时），相比于使用一个 JS 文件包含所有的逻辑，这种多 JS 文件实现了最简单的模块化。


==问题== ：污染全局作用域，每个模块都是暴露在全局的，需要协调每个模块的变量、函数名都不可以相同，不利于大型项目的分工与维护

## 成长期：模块化雏形 — IIFE（立即调用函数表达式）

例：
```js
	let count = 0
	const increase = () => ++count
	const reset = () => {
		count = 0
	}

	increase()
	reset()

	// 包装，括号内箭头函数立即执行
	(() => {
		let count = 0
		++count
		console.log(count)
	})()
```

定义一个简单的模块
```js
	const iifeCounterM = (() => {
		let count = 0
		return {
			increase: () => ++count,
			reset: () => {
				count = 0
			}
		}
	})()

	iifeCounterM.increase()
	iifeCounterM.reset()
```

==总结==：完成模块的封装，实现了对外暴露功能，保留变量 + 不污染全局作用域

> 优化：封装的模块依赖其他模块

```js
	const iifeCounterM = ((dependencyM1, dependencyM2) => {
		let count = 0
		// 调用 dependencyM1, dependencyM2 做处理
		
		return {
			increase: () => ++count,
			reset: () => {
				count = 0
			}
		}
	})(dependencyM1, dependencyM2) // 传递参数
```

### 面试题

你了解早期 jQuery 的依赖处理以及模块加载方案吗？
答：IIEF 加上传参调配

实际书写上，jQuery 等框架实际应用会涉及到 revealing 的写法

揭示( reveal )：==将内部函数以接口的形式暴露出去，实现原理放在内部==

```js
	const revealingCounterM = (() => {
		let count = 0
		// 实现原理
		const increase = () => ++count
		const reset = () => {
			count = 0
		}
		
		// 暴露局部变量
		return { 
			increase,
			reset
		}
	})()
```
本质实现上和方案并无不同，只是在写法思想上，更强调所有 ==API== 以==局部变量==的形式定义在函数中，==对外暴露==可被调用的==接口==。

## 成熟期：

### CJS module：CommonJS

> [!info]+ 特征：
> - 通过 module + exports 来对外暴露接口
> >
> - require 来调用其他模块

模块组织方式：
```js
	// commonJsCounterModule.js
	const dependencyMo1 = require('./dependencyModule1')
	const dependencyMo2 = require('./dependencyModule2')

	let count = 0
	const increase = () => ++count
	const reset = () => {
		count = 0
	}

	// 直接对象
	exports.increase = increase
	exports.reset = reset

	// 另一种方式

	module.exports = {
		increase,
		reset
	}


	// main.js
	// 使用

	const { increase, reset } = require('./commonJsCounterModule')
	increase();

	const commonJsCounterModule = require('./commonJsCounterModule')
	commonJsCounterModule.increase();
```

实际执行处理（IIFE）：
```js
(function(exports, require, module, __filename, __dirname) {
	const dependency1 = require('./dependencyModule1')
	const dependency2 = require('./dependencyModule2')
	
	let count = 0
	const increase = () => ++count
	const reset = () => {
		count = 0
	}

	module.exports = {
		increase,
		reset
	}

	return module.exports
}).call(thisValue, exports, require, module, filename, dirname)

(function(exports, require, module, __filename, __dirname) {
	const commonJsCounterModule = require('./commonJsCounterModule')
	commonJsCounterModule.increase();
}).call(thisValue, exports, require, module, filename, dirname)

// 使用 call 调整 this 的指向，谁调用就指向谁
```

> [!example] 优缺点
> 优点：
> CommonJS 规范在服务器端率先完成成了 JavaScript 的模块化，解决了依赖、全局变量污染的问题，这也是 JS 在服务端运行的必要条件
> 
> 缺点：
> 由于服务端以及 CommonJS 是同步加载模块，无法做异步处理

### AMD 规范

> [!tip] 非同步加载模块，允许制定回调函数

经典的实现框架：`require.js`

新增了定义的方式：
```js
	// 通过 define 来定义一个模块，然后 require 加载
	define(id, [depends], callback)
	
	require([module], callback)
```

模块定义方式：
```js
	// 名称，依赖，依赖加载完成后，实际逻辑
	define('amdCounterModule', 
	['dependencyModule1', 'dependencyModule1'], 
	(dependencyModule1, dependencyModule1) => {
		let count = 0
		const increase = () => ++count
		const reset = () => {
			count = 0
		}

		return {
			increase,
			reset
		}
	})

	// 使用的模块，异步加在后使用的方式
	require(['amdCounterModule'], amdCounterModule => {
		amdCounterModule.increase()
		amdCounterModule.reset()
	})
```

#### 面试题 1

在 AMD 规范下使用 require 方式同步加载模块，可以吗？
答：可以，AMD 支持向前兼容，以提供回调的形式来做 require 方法动态加载模块

例：
```js
	// require 方式：一整个回调函数，传入 require 动态加载, 并包含主逻辑

	define((require) => {
		const dependencyMo1 = require('./dependencyModule1')
		const dependencyMo2 = require('./dependencyModule2')

		let count = 0
		const increase = () => ++count
		const reset = () => {
			count = 0
		}

		return {
			increase,
			reset
		}

		// revealing
		
		// exports.increase = increase
		// exports.reset = reset
	})
```

#### 面试题 2

有什么方式可以统一兼容 AMD 和 CommonJS
答：UMD

实际处理（IIFE）：
```js
(define => define((require, exports, module) => {
	const dependencyMo1 = require('./dependencyModule1')
	const dependencyMo2 = require('./dependencyModule2')

	let count = 0
	const increase = () => ++count
	const reset = () => {
		count = 0
	}
	module.exports = {
		increase,
		reset
	}
}))(
	// 判断区分 AMD or CommonJS
	typeof module === 'object' 
		&& module.exports 
		&& typeof define !== 'function'
		? //CommonJS
			factory => module.exports = factory(require, exports, module)
		: // AMD
			define
)
```

> [!example]  优缺点
> 优点：
> 适合在浏览器环境中异步加载模块，同时又使用 CommonJS 能力
> 
> 缺点：
> 提高开发成本，并且无法按需加载，必须提前加载所有依赖

### CMD 规范

应用于可优化方案中，代表：`sea.js`，
特征：支持按需加载

例：
```js
	define(function(require, exports, module) {
		let $ = requier('jquery')
		let dependencyMo1 = require('./dependencyModule1')
		let dependencyMo2 = require('./dependencyModule2')

		let count = 0
		const increase = () => ++count
		const reset = () => {
			count = 0
		}

		// 两种输出都可
		exports.increase = increase
		exports.reset = reset

		module.exports = {
			increase,
			reset
		}
	})
```

#### 面试题：CMD 和 AMD 区别

例：
```js
	// AMD
	define([
		'./dependencyModule1',
		'./dependencyModule2'
	], function(dependencyModule1, dependencyModule2) {
		dependencyModule1.increase()
		dependencyModule2.reset()
	})

	// CMD - 依赖就近
	define(
		function(require, exports, module) {
			let dependencyMo1 = require('./dependencyModule1')
			dependencyMo1.increase()

			// 编译时
			if () { // 条件判断，假设 dependencyModule2 内容非常多这样做能节省很多资源，性能优化
				let dependencyMo2 = require('./dependencyModule2')
				dependencyMo2.reset()
			}
		}
	)
```

## ES6 模块化 - ESM

> [!quote] 走向新时代

==新增定义方式：==
- 引入：import
- 导出：export

模块引入、导出和模块主体：
```js
	import dependencyModule1 from './dependencyModule1'
	import dependencyModule2 from './dependencyModule2'

	let count = 0

	export const increase = () => ++count
	export const reset = () => {
		count = 0
	}

	// 或
	export default {
		increase,
		reset
	}
```

### 面试题：加载动态模块

```js
	import ('./esModule').then(({ increase, reset }) => {
		increase()
		reset()
	})

	import ('./esModule').then((dynamicESModule) => {
		dynamicESModule.increase()
		dynamicESModule.reset()
	})
```

## 解决模块化新思路

### 背景

根本问题：运行时分析依赖

> [!note] 上述都是基于代码做处理，现在根据文件做处理
> 前端的模块化处理方案依赖于运行时进行分析，并且同时进行依赖加载处理以及实际的逻辑执行

解决方案：线下执行
```html
	project
		| - lib
		|   | - xmd.js
		| - mods
			| - a.js
			| - b.js
			| - c.js
			| - d.js
			| - e.js
		| - index.html

	/** index.html 入口文件
	<!doctype html>
	<script src="lib/xmd.js"></script>
	<script>
		/* 等待构建工具生成数据替换 `__FRAMEWORK_CONFIG__` */
		require.config(__FRAMEWORK_CONFIG__)
	</script>

	<script>
		/* 业务代码 */
		require.async(['a', 'e'], function(a, e) {
			// ...
		})
	</script>
```
```js
	// mods/a.js
	define('a', function(require, exports, module) {
		let b = require('b')
		let c = require('c')

		exports.run = function() {
			// ...
		}
	})

	// 工程化模块构建
	1. 扫描生成依赖关系表

	   {
		   'a': ['b', 'c'],
		   'b': ['d']
	   }

	2. 生成构建模板
```
```html
	<!doctype html>
	<script src="lib/xmd.js"></script>
	<script>
		require.config({
			'deps': {
				'a': ['b', 'c'],
				'b': ['d']
			}
		})
	</script>

	<script>
		/* 业务代码 */
		require.async(['a', 'e'], function(a, e) {
			// ...
		})
	</script>
```
```js
	3. 转化配置为依赖加载代码

	define('a', ['b', 'c'], function(require, exports, module) {
		// 主逻辑，传入一个值，根据这个值加载对应的内容
		let name = _check ? 'b' : 'c'
		let mod = require(name)

		exports.run = function() {
			// ...
		}
	})

	4. 最终：
```
```html
	<!doctype html>
	<script src="lib/xmd.js"></script>
	<script>
		require.config({
			'deps': {
				// 最终依赖
				define('a', ['b', 'c'], function(require, exports, ...))
				'b': ['d']
			}
		})
	</script>

	<script>
		/* 业务代码 */
		require.async(['a', 'e'], function(a, e) {
			// ...
		})
	</script>
```

## 究极体：Webpack

知识体系：（上游）执行 & 作用域 & 原理 => 模块化 => 工程化