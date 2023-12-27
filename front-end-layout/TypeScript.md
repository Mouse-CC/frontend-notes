
## TS 基础概念

### 什么是 TS

a. 原理
- 它是 JavaScript 的一个超集，在原有的语法基础上，添加了可选静态类型和基于类的面向对象编程

> **面向项目：**

TS - 解决大型复杂项目，架构以及代码维护复杂的场景
JS - 脚本化语言，用于面向简单的单一逻辑场景

> **自主检测：**

TS - 编译期间，主动发现并纠正问题
JS - 运行时报错

> **类型检测：**

TS - 支持动态和静态类型检测（==弱类型：隐式转换==）
JS - 弱类型

> **运行流程：**

TS - 依赖编译，依靠编译打包实现浏览器运行
JS - 可直接运行

> **复杂特性：**

模块化、泛型、接口

问题？TS 的好处，以及 TS 最佳实践...

b. 安装运行

```ts
npm install -g typescript

tsc -v

// 面试点：所有的类型检测和纠错阶段都在 —— 编译时
```

### TS 基础类型与语法

-  `boolean | string | number | array | undefined`

```ts
let isEnable = true
let classStr = 'zhaowa'
let classNum = 2
let u = undefined
let n = null
let classArr = ['basic', 'execute']

let isEnable: boolean = true
let classStr: string = 'zhaowa'
let classNum: number = 2
let u: undefined = undefined
let n: null = null
let classArr: Array<string> = ['basic', 'execute']
```

- tuple - 元组

```ts
	let tupleType: [string, boolean]

	tupleType = ['zhaowa', true]
```

- enum - 枚举

```ts
	// 数字类枚举 - 默认从零开始，依次递增

	enum Score {
		BAD,
		NG,
		GOOD,
		PERFECT,
	}

	let sco: Score = Score.BAD // 0

	// 往后依次递增

	// 字符串类型枚举
	
	enum Score {
		BAD = 'BAD',
		NG = 'NG',
		GOOD = 'GOOD',
		PERFECT = 'PERFECT',
	}

	// 反向映射

	enum Score {
		BAD,
		NG,
		GOOD,
		PERFECT,
	}

	let scoName = Score[0] // BAD
	let scoVal = Score['BAD'] // 0

	// 异构

	enum Enum {
		A,
		B,
		C = 'C',
		D = 'D',
		E = 6,
		F,
	} // 0, 1, 'C', 'D', 6, 7

	// 面试题：js手写枚举

	let Enum

	(function(Enum) {
		// 正向
		Enum['A'] = 0
		Enum['B'] = 1
		Enum['C'] = 'C'
		Enum['D'] = 'D'
		Enum['E'] = 6
		Enum['F'] = 7

		// 逆向
		Enum[0] = 'A'
		Enum[1] = 'B'
		Enum[6] = 'E'
		Enum[7] = 'F'
	})(Enum || Enum = {})
```

-  `any | unknown | void`

```ts
	// any - 绕过所有类型检查 => 所有的检测和编译筛查全部失效

	let anyValue: any = 123

	anyValue = 'anyValue'
	anyValue = false

	let valueBoolean: boolean = anyValue

	// unknown - 绕过赋值检查 => 禁止更改传递

	let unknownValue: unknown

	unknownValue = true
	unknownValue = 123
	unknownValue = 'unknownValue'

	let value1: unknown = unknownValue // ok
	let value2: any = unknownValue // ok
	let value3: boolean = unknownValue // nok

	// void 代表无返回，声明函数的返回值为空

	function voidFnc(): void {
		console.log('zhaowa')
	}

	// never - 永不返回 or 永远返回 error

	function llloop(): never {
		while(true) {
			// ...
		}
	}

	function error(msg: string): never {
		throw new Error(msg)
	}
```

- object  /  Object  -  对象

```ts
	// object - 非原始类型
	
	// TS 把 JavaScript object 分成两个接口来定义

	interface ObjectConstructor {
		create(o: object | null): any
	}

	const proto = {}

	Object.create(proto) // ok
	Object.create(null) // ok
	Object.create(undefined) // nok


	// Object
	// Object.prototype 上的属性
	interface Object {
		constructor: Function
		toString(): string
		toLocaleString(): string
		valueOf(): Object
	}

	// 定义 Object 类的属性
	interface ObjectConstructor {
		new(value: any): any
	}

	// {} - 定义空属性
	const obj = {}

	// object 可包括所有非基本类型
	// Object 是 TS 封装好的接口，包含对象属性的，且有原型
	// 一般被认为 JavaScript 的 Object

	obj.toString() // ok
```

###  接口 - interface

- 对行为的抽象，具体行为由类实现

```ts
	// 描述对象内容

	interface YClass {
		name: string,
		time: number,
	}

	let zhaowa: YClass = {
		name: 'TypeScript',
		time: 10
	}

	// 只读 & 任意

	interface QClass {
		readonly name: string,
		time: number,
	}

	// 声明后不可变
	// 面试题 - 和 js 的引用做比较 const

	let arr: number[] = [1, 2, 3, 4, 5, 6]
	let ro: Readonly<number> = arr

	ro[0] = 12 // 赋值 - Error
	ro.push(5) // 增加 - Error
	ro.length = 100 // 修改长度 - Error
	ro = arr // 覆盖 - Error

	// 可添加任意属性

	interface DClass {
		readonly name: string,
		time: number,
		[propName: string]: any // 应用在 map 场景下
	}

	// map < = > obj
	// map 的 key 值可以是任意
```

### 交叉类型 - &

```ts
	interface A { x: D }
	interface B { x: E }
	interface C { x: F }

	interface D { d: boolean }
	interface E { e: string }
	interface F { f: number }

	type ABC = A & B & C

	let abc: ABC = {
		x: {
			d: false,
			e: 'class',
			f: 10
		}
	}

	// 合并冲突

	interface A { 
		a: string
		b: string
	}

	interface B {
		a: number
		c: string
	}

	type AB = A & B

	let ad: AB = {
		b: 'class1',
		c: 'class2'
	}

	// 合并的关系是且 a: never
	// 且：& -> 既要满足 string 又要满足 number，不存在这样的类型
```

### 断言 - 类型声明、转换（和编译器的告知交流）

- 编译时作用

```ts
	// 尖括号形式声明 阶段性声明

	let anyValue: any = 'hi'
	let anyLength: number = (<string>anyValue).length // 断言

	// as 声明

	let anyValue: any = 'hi'
	let anyLength: number = (anyValue as string).length

	// 非空判断

	type ClassTime = () => number
	const start = (classTime: ClassTime | undefined) => {
		let num = classTime!() // 具体类型待定，但确定非空
	}
```

### 类型守卫 - 保证在语法的规定范围之内，额外的确认

- 多态 - 多种状态（多种类型）

```ts
	interface Teacher {
		name: string
		courses: string[]
	}

	interface Student {
		name: string
		startTime: Date
	}

	type Class = Teacher | Student

	function startCoures(cls: Class) {
		if ('courses' in cls) {
			console.log('Teacher: ' + cls.courses)
		}
		if ('startTime' in cls) {
			console.log('Student: ' + cls.startTime)
		}
	}

	// typeof

	function class(name: string, score: string | number) {
		if (typeof score === number) {
			// ...
		}
		if (typeof score === string) {
			// ...
		}
	}

	// instanceof

	const getName = (cls: Class) => {
		if (cls instanceof Teacher) {
			// Teacher
		}
		if (cls instanceof Student) {
			// Student
		}
	}
```

### 补充

#### 1. 函数重载

```ts
	class Class {
		start(name: number, score: number): number;
		start(name: string, score: string): string;
		start(name: string, score: number): string;
		start(name: number, score: string): string;
		start(name: Combinable, score: Combinable) {
			if (typeof name === 'string' || typeof score === 'string') {
				// ...
			}
		}
	}
```

#### 2. 泛型 — 重用

- 让模块可以支持多种类型的数据 - 让类型和值一样，可以被赋值传递

```ts
	function startClass<T, U>(name: T, score: U): T {
		return name + score
	}

	console.log(startClass<string, number>('s', 10))
	// T, U, K 表示为键值类 | V 值 | E 节点

	function startClass<T, U>(name: T, score: U): string {
		return `${name}${score}`
	}

	function startClass<T, U>(name: T, score: U): T {
		return (name + String(score)) as any as T
	}

	// 返回断言，先转为 any，再转为函数的类型返回 T
```

#### 3. 装饰器 - decorator

```ts
	// tsc --target ES5 --experimentalDecorators
	// experimentalDecorators: true

	{
		'compilerOptions': {
			'target': 'ES5'
			'experimentalDecorators': true
		}
	}

	// 类装饰器
	function Zhaowa(target: Fcuntion): void {
		target.prototype.startClass = function(): void {
			// 装饰逻辑
		}
	}

	@Zhaowa
	class ClassA {
		constructor() {
			// 业务逻辑
		}
	}

	// 属性装饰器
	// target: Object - 被装饰的类; key — 被装饰类的属性
	function nameWrapper(target: any, key: string) {
		Object.definePrototype(target, key, {
		
		}) 
	}

	class ClassB {
		constructor() {
			// 业务逻辑
		}

		@nameWrapper
		public name: string
	}
```

