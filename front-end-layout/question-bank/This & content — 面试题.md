
# 面试题 1

```javascript
	const foo = {
		bar: 10,
		fn: function() {
			console.log(this.bar)
			console.log(this)
		}
	}

	let f = foo.fn // 将函数赋值给全局变量
	f() // undefined window 对象

```
原因：此时 `f` 为全局变量，this 就指向全局，window 不存在属性 bar ，返回 undefined

# 面试题 2

```javascript

	// 改变属性指向
	const o1 = {
		text: "o1",
		fn: function() {
			console.log("o1_fn_this", this)
			return this.text
		}
	}
	
	const o2 = {
		text: "o2",
		fn: function() {
			return o1.fn();
		}
	}
	
	const o3 = {
		text: "o3",
		fn: function() {
			let fnc = o1.fn
			return fnc()
		}
	}

	console.log("o1fn", o1.fn()) // 'o1fn' 'o1_fn_this' o1 对象 'o1'
	
	console.log("o2fn", o2.fn()) // 'o2fn' 'o1_fn_this' o1 对象 'o1'
	
	console.log("o3fn", o3.fn()) 
	// 'o3fn' 'o1_fn_this' window 对象 undefined
```

分析：
1. `console.log('o1fn', o1.fn())` 使用自身属性，直接使用上下文（传统分活）
2. `console.log("o2fn", o2.fn())` 使用对象 `o1` 的方法，呼叫 `o1` 来处理（部门协作），活是 `o1` 做的
3. `console.log("o3fn", o3.fn())` 局部变量保存对象 `o1` 的方法，this 指向全局（面向全局）

## 追问：`console.log('o2fn', o2.fn())` this 如何指向 `o2`？

1. 修改 this 指向
```javascript
	...
	console.log('o2fn', o2.fn.call(o2))
```

2. 不改变 this
```javascript
	// fn 赋予 o1.fn => 原理：this 指向最后调用的对象
	// o2.fn()，this 指向 o2

	const o2 = {
		text: "o2",
		fn: o1.fn
	}
```

### 总结 ：

1. 在函数执行时，函数是被上一级调用，上下文就指向上一级
2. 公共函数，this 指向 window