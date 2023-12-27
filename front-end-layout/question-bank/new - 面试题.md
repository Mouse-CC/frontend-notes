
实现 new
```js
	function newFunc(Father) {
		if (typeof Father !== "function") {
			throw new Error("...")
		}

		// obj.__proto__ === Father.prototype
		var obj = Object.create(Father.prototype)
		var result = 
		Father.apply(obj, Array.prototype.slice.call(arguments, 1))
	
		return result && typeof result === "object" ? result : obj
	}
```

实现 Object.create
```js
	function inherit(o) {
		if (o === null) throw TypeError(...)
		if (Object.create) {
			return Object.create(o)
		}

		let t = typeof o
		if (t !== "object" && t !== "function") throw TypeError(...)

		// 核心
		function f() {}
		f.prototype = o
		return new f()
	}
```
