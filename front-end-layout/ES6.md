> ECMAScript

ES 6 (ES 2015) 以后根据年份来划分版本，如：ES 2020
- 从版本号 => 年份号
- ESNext 下一代 ES 版本（定案未发布）

==新的特性==
Stage（阶段）: 1 ~ 4
- 1. 提出
- 2. 使用形式规范语言精确描述语法和语义
- 3. 细化需要和手机用户反馈
- 4. 准备好包含在正式的 ECMAScript 标准中

What is ESNext ?

一句话：下一个版本，在 3 阶段的所有特性。即已经产生草案和候选人，但没有变成定案的全部内容。

## ES 新增的 API

### const 标识常量

```js
	const LIMIT = 10
	const OBJ_MAP = {
		a: 'A',
		b: 'B'
	}

	const QUEUE = [1, 2, 3, 4, 5]
```

#### 1 不允许重复声明赋值

```js
	// 变量

	var arg1 = 'es'
	arg1 = 'ESNext'

	// 变量
	// ES5

	Object.defineProperty(window, 'arg2', {
		value: 'yy',
		writable: false
	})

	console.log(arg2) // yy
	
	arg2 = 'yy1'
	console.log(arg2) // yy

	// ES6

	const arg3 = 'yy'
	arg3 = 'yy1' // 报错，提示无法给常量赋值

	// 声明，声明需要跟上初始化
	const arg4
	arg4 = 'yy' // 报错，声明阶段没有赋值

	const arg5 = 'yy'
	const arg5 = 'zz' // 报错，常量已经被声明
```

#### 2 块级作用域

```js
	if (true) {
		var arg1 = 'yy'
	}

	if (true) {
		const arg2 = 'yy1'
	}

	console.log(arg1)
	console.log(arg2)
```

#### 3 无变量提升

```js
	console.log(arg1)
	var arg1 = 'yy'

	// 相当于
	var arg1
	console.log(arg1)
	arg1 = 'yy'

	// 无变量提升
	console.log(arg2) // 报错，请声明 arg2
	const arg2 = 'yy'
```

#### 4 dead  zone — 死区

```js
	if (true) {
		console.log(arg3)
		const arg3 = 'yy3'
	}
```

#### 5 let  or  const — 能用 const 就用 const

```js
	const obj = { // 不能移动指向，即不能修改地址
		teacher: 'yy',
		student: 'ww'
	}

	obj.teacher = 'zz' // ok

	obj = {} // nok

	// 引用类型的原理 — 指向地址
	// Object.freeze()

	Object.freeze(obj) // 冻结属性，让属性无法修改，无法冻结引用类型(对象/数组)

	const obj2 = {
		teacher: 'yy',
		zhaowa: ['q1', 'q2']
	}

	obj2.zhaowa[0] = 'q' // ok
	
	// freeze 只能冻结 root 层

	function deepFreeze(obj) {
		Object.freeze(obj)
		(Object.keys(obj) || []).forEach(key => {
			if (typeof obj[key] === 'object') {
				deepFreeze(obj[key])
			}
		})
	}
```

### deconstruction 解构 — 解开对象的解构

```js
	const zhaowa = {
		teacher: 'yy',
		student: 'ww'
	}

	const { teacher, student } = zhaowa
```

#### key 解构技巧

```js
	const zhaowa = {
		teacher: {
			name: 'yy',
			age: 31
		},
		student: 'ww',
		name: 'es6'
	}

	// 别名
	const {
		teacher: {
			name,
			age
		},
		student,
		name: className
	} = zhaowa
```

使用场景

#### 形参结构

```js
	const sum = arr => {
		let res = 0

		arr.forEach(each => {
			res += earch
		})
	}

	// 确认就只有 3 个
	const sum = ([a, b ,c]) => {
		return a + b + c
	}
```

#### 结合初始值

```js
	const course = ({ teacher, student, course = 'zhaowa' }) => {
		console.log(teacher, student, course)
	}

	course({
		teacher: 'yy',
		student: 'ww'
	})
```

### arrow_function 箭头函数

> 只具备函数的基本能力

```js
	function test(a, b) {
		return a + b
	}

	const test2 = function (a, b) {
		return a + b
	}

	// 箭头函数
	const test3 = (a, b) => {
		return a + b
	}

	const test3 = (a, b) => a + b // 明显的 入参 和 出参
	const test4 = x => {
		// content
	}
```

#### 上下文

```js
	const obj2 = {
		teacher: 'yy',
		student: 'ww',
		zhaowa: ['q1', 'q2'],
		getTeacher: function() {
			return this.teacher // 指向当前对象
		},
		getStudent: () => {
			return this.student // 箭头函数无内部上下文，this 指向全局
		}
	}
```

```js
	// 斐波那契数列：1，1，2，3，5，8 ...
	function fib(n) {
		if (n < 2) {
			return n
		}
		return fib(n - 1) + fib(n - 2)
	}

	// 尾递归
	function outer() {
		return inner()
	}
	outer()

	// 优化 - 不在栈内压太多东西
	function fib(n) {
		fibAdvantage(0, 1, n)
	}

	function fibAdvantage(a, b, n) {
		if (n === 0) {
			return a
		}

		return fibAdvantage(b, a + b, n - 1)
	}
```

#### 无 arguments

```js
	const test = function(tech) {
		console.log(arguments)
	}

	const test1 = tech => {
		console.log(arguments) // 执行，报错：arguments is not defined
	}
```

### Class 助力 JS 更面向对象 — 类

> 面向对象

```js
	// 传统对象 - function
	function Course(teacher, course) {
		this.teacher = teacher
		this.course = course
	}

	Course.prototype.getCourse = function() {
		return `teacher: ${this.teacher}, course: ${this.course}`
	}

	const course = new Course('yy', 'ES6')
	course.getCoures()

	
	// ES6
	class Course {
		constructor(teacher, course) {
			this.teacher = teacher
			this.course = course
		}
		static getCourse() { // static 挂载在类上
			return `teacher: ${this.teacher}, course: ${this.course}`
		}
	}

	const course = new Course('yy', 'ES6')
```

> 追问

#### Class 类型

```js
	console.log(typeof Course) // function
```

#### Class  &  函数对象的属性

```js
	course.hasOwnProperty('teacher') // ok, 有 object 的方法
```

#### 属性定义

```js
	class Course {
		constructor(teacher, course) {
			this._teacher = teacher
			this.course = course
		}
		static getCourse() { // static 挂载在类上
			return `teacher: ${this.teacher}, course: ${this.course}`
		}
		get teacher() {
			return this._teacher
		}
		set teacher(val) {
			this._teacher = val
		}
	}

	// 局部变量 — 闭包 => 实现私有属性
	class Course {
		constructor(teacher, course) {
			this._teacher = teacher
			let _course = 'es6'

			this.getCourse = () => {
				return _course
			}
		}
	}

	// 封装能力 — 适配器模式
	// 封装底层 core
	class utils {
		constructor(core) {
			this._main = core
			this._name = 'myName'
		}
		get name() {
			return {
				...this._main.name,
				name: `${this.name}`
			}
		}
		set name(val) {
			this._name = val
		}
	}

	// 静态方法
	class Course {
		constructor(teacher, course) {
			this.teacher = teacher
			this.course = course
		}
		static getCourse() { // static 挂载在类上
			return `teacher: ${this.teacher}, course: ${this.course}`
		}
		getCourse() {
			return `teacher: ${this.teacher}, course: ${this.course}`
		}
	}

	const course = new Course('yy', 'es6')
	course.getCourse()

	Course.getCourse() // 调用静态方法

	// 区别，属性是每个实例都有的，但是静态方法是唯一的，只在类上。
```

#### 继承

```js
	function Child() {
		Course.call(this, 'yy', 'es6')
		this.start = function() {
			// ...
		}
	}

	Child.prototype = Course.prototype

	// es6 class
	class Child extends Course {
		constructor() {
			super('yy', 'es6')
		}
		start() {
			// ...
		}
	}
```


### Other

#### 数组操作
- `forEach` - 遍历，每一项==（无返回值）==
- `map` - 映射，定向 (取/改)值 => ==返回新数组==
- `filter`  - 过滤 => ==返回数组==
- `find` - 查找，遍历每一项 => ==返回对象==
- `some / every` - 是否满足运算 => ==返回布尔==，some 一真即真 / every 一假即假
- `reduce` - 重构，将数组按照指定逻辑重新组装 => ==返回对象==

#### set & map

```js
	new Set()
	// set 和数组类比
	add() delete() clear() forEach() size

	let arr = [1, 2, 2, 3, 4, 5]
	let set = new Set(arr)

	let newArr = [...set]
	let newArr1 = Array.from(set)

	// map 和对象类似
	set() get() delete() clear() forEach() keys() values() size
	// map 的 key 可以是任意值
```

#### symbol — 用于保证值一定不相等

```js
	let _test = Symbol('_test')
	let _test1 = Symbol('_test')

	_test === _test1 // false
```