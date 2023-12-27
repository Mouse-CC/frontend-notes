### 前置内容

1. ==promise 有哪些状态？对应值有哪些？==
   - promise：`pending` | `fulfilled` | `rejected`
   - executor：new Promise 时立即执行，接收两个参数：`resolve`、`reject`

2. ==promise 的默认状态？状态如何流转？==
   - 默认状态：`pending`
   - 内部维护成功值：`undefined` | `thenable` | `promise`
   - 内部维护失败值：`reason`
   - promise 的状态流转：pending ⇒ rejected | pending ⇒ fulfilled

3. ==promise 的返回值？==
   - then：接收 `onFulfilled` 和 `onRejected`
   - 如果 then，promise 已经成功，执行 `onFulfilled`，参数 `value`
   - 如果 then，promise 已经失败，执行 `onRejected`，参数 `reason`
   - then 中有任何 error 异常 ⇒ onRejected
   
## 手写 promise

### 基础版：

```js
	const PENDING = "PENDING"
	const FULFILLED = "FULFILLED"
	const REJECTED = "REJECTED"

	class Promise {
		constructor(executor) {
			// 1. 默认状态 - PENDING
			this.status = PENDING
			// 2. 维护内部成功/失败 值
			this.value = undefined
			this.reason = undefined

			// 成功的回调
			let resolve = value => {
				// 单向流转
				if (this.status === PENDING) {
					this.status = FULFILLED
					this.value = value
				}
			}
	
			// 失败的回调
			let reject = reason => {
				// 单向流转
				if (this.status === PENDING) {
					this.status = REJECTED
					this.reason = reason
				}
			}

			// 主执行
			try {
				// 用户注册的函数, new 时立刻执行
				executor(resolve, reject)
			} catch(error) {
				reject(error)
			}
		}
		then(onFulfilled, onRejected) {
			if (this.status === FULFILLED) {
				onFulfilled(this.value)
			}

			if (this.status === REJECTED) {
				onRejected(this.reason)
			}
		}
	}
```

### 异步版：

```js
	const PENDING = "PENDING"
	const FULFILLED = "FULFILLED"
	const REJECTED = "REJECTED"

	class myPromise {
		constructor(executor) {
			this.status = PENDING
			this.value = undefined
			this.reason = undefined

			// 存放成功的回调
			this.onResolvedCallbacks = []
			// 存放失败的回调
			this.onRejectedCallbacks = []

			// 成功的回调
			let resolve = value => {
				if (this.status === PENDING) {
					this.status = FULFILLED
					this.value = value
					this.onResolvedCallbacks.forEach(fn => fn())
				}
			}

			// 失败的回调
			let reject = reason => {
				if (this.status === PENDING) {
					this.status = REJECTED
					this.reason = reason
					this.onRejectedCallbacks.forEach(fn => fn())
				}
			}

			// 主执行
			try {
				executor(resolve, reject)
			} catch(error) {
				reject(error)
			}
		}
		then(onFulfilled, onRejected) {
			if (this.status === FULFILLED) {
				onFulfilled(this.value)
			}

			if (this.status === REJECTED) {
				onRejected(this.reason)
			}

			// 当主体内部是异步的，状态还未流转
			if (this.status === PENDING) {
				// 存放执行队列
				this.onResolvedCallbacks.push(() => {
					onFulfilled(this.value)
				})
				this.onRejectedCallbacks.push(() => {
					onRejected(this.reason)
				})
			}
		}
	}
```

### 链式调用：

```js

const PENDING = "PENDING"
const FULFILLED = "FULFILLED"
const REJECTED = "REJECTED"

class myPromise {
	constructor(executor) {
		this.status = PENDING
		this.value = undefined
		this.reason = undefined

		this.onResolvedCallbacks = []
		this.onRejectedCallbacks = []

		// 成功的回调
		let resolve = value => {
			if (this.status === PENDING) {
				this.status = FULFILLED
				this.value = value
				this.onResolvedCallbacks.forEach(fn => fn())
			}
		}

		// 失败的回调
		let reject = reason => {
			if (this.status === PENDING) {
				this.status = REJECTED
				this.reason = reason
				this.onRejectedCallbacks.forEach(fn => fn())
			}
		}

		// 主执行
		try {
			executor(resolve, reject)
		} catch(error) {
			reject(error)
		}
	}
	then(onFulfilled, onRejected) {
		// 边缘检测，是否为 promise
		onFulfilled 
			= typeof onFulfilled === 'function' 
				? onFulfilled 
				: value => value

		onRejected 
			= typeof onRejected === 'function'
				? onRejected
				: error => { throw error }

		// 关注有哪些流转
		let promise2 = new myPromise((resolve, reject) => {
			if (this.status === 'FULFILLED') {
				setTimeout(() => {
					try {
						let x = onFulfilled(this.value)
						// 嵌套 promise
						this.resolvePromise(promise2, 
						x, resolve, reject)
					} catch (error) {
						reject(error)
					}
				}, 0)
			}

			if (this.status === 'REJECTED') {
				setTimeout(() => {
					try {
						let x = onRejected(this.reason)
						this.resolvePromise(promise2, 
						x, resolve, reject)
					} catch (error) {
						reject(error)
					}
				}, 0)
			}

			if (this.status === 'PENDING') {
				this.onResolvedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onFulfilled(this.value)
							this.resolvePromise(promise2, 
							x, resolve, reject)
						} catch (error) {
							reject(error)
						}
					}, 0)
				})
				
				this.onRejectedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onRejected(this.reason)
							this.resolvePromise(promise2, 
							x, resolve, reject)
						} catch (error) {
							reject(error)
						}
					}, 0)
				})
			}
		})

		// then 返回 promise
		return promise2
	} catch (onRejected) {
		return this.then(null, onRejected)
	}

	resolvePromise(promise2, x, resolve, reject) {
		if (promise2 === x) {
			// 已经是 promise，且链循环
			return reject(new TypeError(...))
		}

		let called = false
		if (x instanceof Promise) { // 是 promise 传入返回值
			x.then(y => {
				this.resolvePromise(promise2, y, resolve, reject)
			}, error => {
				reject(error)
			})
		} else if ( x !== null && 
			(typeof x === 'object' || typeof x === 'fcuntion')) { // 可执行
			
			try {
				let then = x.then
				if (typeof then === 'function') {
					// 传入的函数有 then，call 去继续函数 x 的 then。
					// 传入 onFulfilled 和 onRejected。
					then.call(x, y => {
						if (called) return
						called = true
						this.resolvePromise(promise2, y, resolve, reject)
					}, error => {
						if (called) return
						called = true
						reject(error)
					})
				} else {
					resolve(x)
				}
			} catch (error) {
				if (called) return
				called = true
				reject(error)
			}
		} else {
			// 确定为值，直接返回出去
			resolve(x)
		}
	}
}
```