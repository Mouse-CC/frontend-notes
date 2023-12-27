
>[!tip] 注：当前 vue 源码版本 2.7.16

## 入口

>[!note] 结构
> | - packages
> | - scripts
> | - src
> | - ...
> | - package.json

package.json:
```json
{
	// ...
	"scripts": {
		"dev": "rollup -w -c scripts/config.js --environment TARGET:full-dev",
		// ...
	}
}
```

执行脚本：命令 => `npm run dev` 即执行脚本，从代码可知调用了 rollup 执行 `scripts` 文件夹下的 ` config.js` 文件 `--environment` 后的参数标识执行环境，` TARGET ` 标识当前为开发环境。

`scripts/config.js`:
```js
// ...
const builds = {
	// ...
	// Runtime+compiler development build (Browser)
	'full-dev': {
		entry: resolve('web/entry-runtime-with-compiler.ts'),
		dest: resolve('dist/vue.js'),
		format: 'umd',
		env: 'development',
		alias: { he: './entity-decoder' },
		banner
	},
}

function genConfig(name) { // 生成配置
	const opts = builds[name]
	// ...
}

if (process.env.TARGET) { // 当前执行环境
	module.exports = genConfig(process.env.TARGET)
} else {
	exports.getBuild = genConfig
	exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```

>注：开发模式入口，`entry: resolve('web/entry-runtime-with-compiler.ts')`

>附：`process.env.TARGET` ，当前环境下的 `TARGET` 类型是在全局类型文件 `globals.d.ts` 的 `namespace NodeJS` 中扩展了 `interface Dict<T>` 返回类型为：`string | undefined`

`web/entry-runtime-with-compiler.ts`:
```ts
import Vue from './runtime-with-compiler' // 核心
import * as vca from 'v3'
import { extend } from 'shared/util'

extend(Vue, vca)

import { effect } from 'v3/reactivity/effect'
Vue.effect = effect

export default Vue
```

通过以上文件的分析，最终确定 `web/entry-runtime-with-compiler.ts` ，为打包的开始文件。
下一个目标是找到 Vue 的初始化是如何做的：`new Vue()` 这时候 `Vue` 是什么？

`web/runtime-with-compiler.ts`: 
```ts
import Vue from './runtime/index'
```

`web/runtime/index.ts`: ^ops1
```ts
import Vue from 'core/index'

// ...

import platformDirectives from './directives/index'
import platformComponents from './components/index'

// ...

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

```

`src/core/index.ts`: ^initGlobalAPI
```ts
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'
import { version } from 'v3'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get() {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = version

export default Vue
```

`src/core/instance/index.ts`: 
```ts
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'
import type { GlobalAPI } from 'types/global-api'

function Vue(options) { // 构造函数
	if (__DEV__ && !(this instanceof Vue)) {
	// 直接调用此函是错误的，Vue 是构造函数，应该用 new 关键字
	warn('Vue is a constructor and should be called with the `new` keyword')
	}
	this._init(options)
}

//@ts-expect-error Vue has function type
initMixin(Vue)
//@ts-expect-error Vue has function type
stateMixin(Vue)
//@ts-expect-error Vue has function type
eventsMixin(Vue)
//@ts-expect-error Vue has function type
lifecycleMixin(Vue)
//@ts-expect-error Vue has function type
renderMixin(Vue)

export default Vue as unknown as GlobalAPI
```

最终追踪到，`src/core/instance/index.ts` ，`Vue` 就是一个构造函数。其中可以看到构造函数中 `this._init(options)` ，这就是要找的初始化 ...

## Vue 初始化

```ts
	import { initMixin } from './init'

	// ...

	//@ts-expect-error Vue has function type
	initMixin(Vue)
```

在使用 Vue 的时候我们一般：
```js
	let v = new Vue({
		el: "#app",
		data: {
			a: 1,
			b: [1, 2, 3]
		}
	})
```

结合 `src/core/instance/index.ts` ^ctor
```ts
function Vue(options) { // 构造函数

	if (__DEV__ && !(this instanceof Vue)) {
	// 直接调用此函是错误的，Vue 是构造函数，应该用 new 关键字
	warn('Vue is a constructor and should be called with the `new` keyword')
	}
	this._init(options)
}
```

将 option 代入到 `initMixin` 函数 => `src/core/instance/init.ts`
```ts
export function initMixin(Vue: typeof Component) {
	// options==={ el: '#app, data: {a: 1, b: [1, 2, 3]} }
	Vue.prototype._init = function (options?: Record<string, any>) {
		const vm: Component = this
		// a uid
		vm._uid = uid++ 
		
		let startTag, endTag
		/* istanbul ignore if */
		
		if (__DEV__ && config.performance && mark) {
			startTag = `vue-perf-start:${vm._uid}`
			endTag = `vue-perf-end:${vm._uid}`
			mark(startTag)
		}
		
		// a flag to mark this as a Vue instance without having to do instanceof
		// check
		vm._isVue = true
		// avoid instances from being observed
		vm.__v_skip = true
		// effect scope
		vm._scope = new EffectScope(true /* detached */)
		vm._scope._vm = true
		// merge options
		if (options && options._isComponent) {
			// optimize internal component instantiation
			// since dynamic options merging is pretty slow, and none of the
			// internal component options needs special treatment.
			initInternalComponent(vm, options as any)
		} else {
			vm.$options = mergeOptions(
				resolveConstructorOptions(vm.constructor as any),
				options || {},
				vm
			)
		}
		
		/* istanbul ignore else */
		if (__DEV__) {
			initProxy(vm)
		} else {
			vm._renderProxy = vm
		}
		// expose real self
		vm._self = vm
		initLifecycle(vm)
		initEvents(vm)
		initRender(vm)
		// 生命周期
		callHook(vm, 'beforeCreate', undefined, false /* setContext */)
		// vm 状态初始化，prop/data/computed/method/watch 都在此做初始化
		initInjections(vm) // resolve injections before data/props
		initState(vm)
		initProvide(vm) // resolve provide after data/props
		// 代入生命周期 created
		callHook(vm, 'created')
		
		/* istanbul ignore if */
		if (__DEV__ && config.performance && mark) {
			vm._name = formatComponentName(vm, false)
			mark(endTag)
			measure(`vue ${vm._name} init`, startTag, endTag)
		}
		
		if (vm.$options.el) {
			vm.$mount(vm.$options.el) // 挂载
		}
	}
}
```

定义 `vm` 类型为 ` Component` 指向当前传入的 `Vue` 相当于当前的 `Vue` 实例。
过程中在 `vm` 上定义各种属性，重要的是对 `option` 的处理，给 `vm` 增加属性 `$options`

> 注：Vue 内部的框架变量，命名一般以 `$` 符开头。
> 注：注释 -> expose real self | 开始给实例初始化生命周期，事件，渲染 ....

最后是根据 `option` 的 `el` ，`Vue` 实例调用 `$mount` 将其挂载到对应的 ` el ` 上。

查看 `mergeOptions()` 中的第一个参数为函数 `resolveConstructorOptions(vm.constructor as any)` 的返回值

### 看看 resolveConstructorOptions 函数

`src/core/instance/init.ts` ^resolveCtor
```ts
export function resolveConstructorOptions(Ctor: typeof Component) {
//  在调用的时候是 resolveConstructorOptions(vm.constructor)
//  传入了 Vue 实例的构造函数
//
//  vm.constructor 就是 Vue 的方法
//  Vue.options = {
//    components: {},
//    directives: {},
//    filters: {},
//    _base: Vue
//  }
  let options = Ctor.options // Ctor: 构造函数
  if (Ctor.super) {
  // 有 super 属性，说明 Ctor 是 Vue.extend 构建的子类
    const superOptions = resolveConstructorOptions(Ctor.super)
    // Vue 构造函数上的options, 如 directives, filters,....
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
      // 如果 Ctor 是基础Vue构造器时，就如我们通过 new 关键字实例化 Vue 的时候，
      // options 就是 Vue 构造函数上的 options
  return options
}
```

函数 `resolveConstructorOptions` 的开头就让人游戏迷惑了吧，传入的参数是 `vm.constructor` 由 [[面向对象]] 这章我们可以知道的是 `constructor` 是构造函数本身，那么此处的类型判断是没问题的。疑惑的是 `let options = Ctor.options` 关于 `Ctor.options` ，构造函数的 `options` 在 [`src/core/instance/index.ts`](#^ctor) 并没有写
其实在 [`web/runtime/index.ts`](#^ops1) 中有关 `options` 的内容，是给 `options` 添加了属性，并没用真正的定义。往上一层 [`src/core/index.ts`](#^initGlobalAPI) 找，并在函数 `initGlobalAPI` 中看到如下的实现：

`src\core\global-api\index.ts`
```ts
export function initGlobalAPI(Vue: GlobalAPI) {
	...
	Vue.util = {
		warn,
		extend,
		mergeOptions,
		defineReactive
	}
	
	Vue.set = set
	Vue.delete = del
	Vue.nextTick = nextTick

	// 2.6 explicit observable API
	Vue.observable = <T>(obj: T): T => {
		observe(obj)
		return obj
	}
	
	Vue.options = Object.create(null)
	//  ASSET_TYPES = [ 
	//    'component', 
	//    'directive', 
	//    'filter' 
	//  ]
	ASSET_TYPES.forEach(type => {
		Vue.options[type + 's'] = Object.create(null)
	})
	
	// this is used to identify the "base" constructor to extend all plain-object
	// components with in Weex's multi-instance scenarios.
	Vue.options._base = Vue
	...
}
```

总结: Vue 构造函数上的 options 为

```ts
	Vue.options = {
		components: {},
		directive: {},
		filter: {},
		_base: Vue
	}
```

>图：

![[Pasted image 20231212160738.png]]

回到 [`resolveConstructorOptions`](#^resolveCtor) 当 `Ctor.super` 不存在，返回 `Vue` 构造函数的 `options` ( `Ctor.super` 是通过 ` Vue.extend` 构造子类的时候，`Vue.extend` 会为 `Ctor` 添加一个 `super` 属性，指向其父类构造器) 那么 `vm.$options = mergeOptions( resolveConstructorOptions( vm.constructor), options || {}, vm )` 中的参数应该是 `vm.$options = mergeOptions ( vm.options, options || {}, vm )` 。

### 看看 mergeOptions 函数

`src/core/util/options.ts`
```ts
const starts = config.optionMergeStrategies
...
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 * 用于实例和继承的核心代码
 */
export function mergeOptions(
    parent: Record<string, any>,
    child: Record<string, any>,
    vm?: Component | null
): ComponentOptions {
    //  传入的三个参数分别代表：
    //  1、Vue 实例构造函数上的 options
    //  2、实例化是我们自己传入的参数
    //  3、vm 实例本身
    //
    //  也就是这个方法就是合并 parent 和 child
    //
    //  parent = {
    //    components: {},
    //    directives: {},
    //    filters: {},
    //    _base: Vue
    //  }
    //
    //  child===={ el: '#app', data: { a:1, b: [1, 2, 3] } }
    if (__DEV__) {
        checkComponents(child); // 检查组件名称是否合法
    }

    if (isFunction(child)) {
	    // @ts-expect-error
        child = child.options; // 如果 child 是 function 类型的话，我们取options 属性作为 child
    }

    normalizeProps(child, vm); //把 props 属性转换成对象的形式
    normalizeInject(child, vm); //把 inject 属性转换成对象的形式
    normalizeDirectives(child); //把 directives 属性转换成对象的形式

	// Apply extends and mixins on the child options,
	// but only if it is a raw options object that isn't
	// the result of another mergeOptions call.
	// Only merged options has the _base property.
	if (!child._base) {
		// 当传入的 options 里有 extends 时，再次调用 mergeOptions 方法进行合并
		if (child.extends) {
			parent = mergeOptions(parent, child.extends, vm)
		}
		// 当传入的 options 里有 mixin 时，再次调用 mergeOptions 方法进行合并
		if (child.mixins) {
			for (let i = 0, l = child.mixins.length; i < l; i++) {
				parent = mergeOptions(parent, child.mixins[i], vm)
			}
		}
	}
    
    const options: ComponentOptions = {} as any;
    let key;
    for (key in parent) {
        mergeField(key);
    }
    for (key in child) {
        if (!hasOwn(parent, key)) {
            mergeField(key);
        }
    }
    function mergeField(key) {
        // const defaultStrat = function(parentVal: any, childVal: any): any {
        //   return childVal === undefined ? parentVal : childVal;
        // };
        // 
        // defaultStrat的逻辑就是，如果 child 上该属性值存在，就取 child 上的，如果不存在，就取 parent 上的

        //const strats = config.optionMergeStrategies;
        const strat = strats[key] || defaultStrat;
        options[key] = strat(parent[key], child[key], vm, key);
    }
    return options;
}
```

>图：==蓝色箭头画反==

![[Pasted image 20231212165443.png]]

### 参考链接

[人人都能懂的Vue源码系列(五)—mergeOptions-上\_慕课手记](https://www.imooc.com/article/29106)
[人人都能懂的Vue源码系列(六)—mergeOptions-下\_慕课手记](https://www.imooc.com/article/29108)

### 小结

Vue 初始化主要是把 `options` 进行合并，将生命周期、事件 ... 初始化，并把这些挂载在实例上。

## 数据观测

vue 是数据驱动的，当数据发生改变就会导致视图发生改变。所以本节的要点是 vue 如何知道数据发生变化，数据变化后如何更新对应的视图。其核心的方法是通过 `Object.defineProperty()` 来实现对==属性==的==访问劫持==和==变化劫持==。

>vue 的内部有很多个重要的部分：数据观测、模板编译、virtualDOM、整体运行流程 ...
>
>数据观测是 vue 框架中的一个重要部分，了解其原理有助于提高对 vue 的理解和使用

数据观测的核心机制是 ==观察者模式===

解释：

数据是被观察的一方，当数据发生变化时，通知所有的观察者，这样观察者可对变化做出响应。比如：重新渲染

通常将依赖数据的观察者称为 watcher。变化时，这种关系可表示为：`data -> watcher`

数据可以有多个观察者 (data 对象下的每一个属性)，如何记录这种依赖关系 =>

`Vue` 在 data 和 watcher 间创建一个 `dep` 对象：`data -> dep -> watcher`

`dep` 的结构简单，只有两个属性：id，subs

>注：id 为唯一表示，subs 为对所有观察者的记录

### 分析源码

入口 , `src/core/instance/init.ts` 的 `initMixin` 函数内的 `initState(vm)`

```ts
	// expose real self
	vm._self = vm
	initLifecycle(vm)
	initEvents(vm)
	initRender(vm)
	// 生命周期
	callHook(vm, 'beforeCreate', undefined, false /* setContext */)
	// vm 状态初始化，prop/data/computed/method/watch 都在此做初始化
	initInjections(vm) // resolve injections before data/props
	initState(vm)
	initProvide(vm) // resolve provide after data/props
	// 代入生命周期 created
	callHook(vm, 'created')
```

跟踪 `initState()` 查看代码
`src/core/instance/state.ts`:
```ts
export function initState(vm: Component) {
	const opts = vm.$options
	if (opts.props) initProps(vm, opts.props)
	
	// Composition API
	initSetup(vm)
	
	if (opts.methods) initMethods(vm, opts.methods)
	//开始处理option中的data
	if (opts.data) {
		initData(vm) // 重要
	} else {
		const ob = observe((vm._data = {})) //重要
		ob && ob.vmCount++
	}
	if (opts.computed) initComputed(vm, opts.computed)
	if (opts.watch && opts.watch !== nativeWatch) {
		initWatch(vm, opts.watch)
	}
}
```

#### 核心
有关的数据的两个方法：`initData()` | `observe()`

`src/core/instance/state.ts`:
```ts
function initData(vm: Component) {
	let data: any = vm.$options.data
	// 获取data这个对象
	data = vm._data = isFunction(data) ? getData(data, vm) : data || {}
	if (!isPlainObject(data)) {
		data = {}
		__DEV__ &&
		warn(
		'data functions should return an object:\n' +
		'https://v2.vuejs.org/v2/guide/components.html#data-Must-Be-a-Function', vm
		)
	}
	// proxy data on instance
	const keys = Object.keys(data)
	const props = vm.$options.props
	const methods = vm.$options.methods
	let i = keys.length
	while (i--) {
		const key = keys[i]
		if (__DEV__) {
			if (methods && hasOwn(methods, key)) {
				warn(`Method "${key}" has already been defined as a data property.`, vm)
			}
		}
		if (props && hasOwn(props, key)) {
		  __DEV__ &&
			warn(
			  `The data property "${key}" is already declared as a prop. ` + `Use prop default value instead.`, vm
			)
		} else if (!isReserved(key)) {
			// 通过this.xxx访问_data
			proxy(vm, `_data`, key)
		}
	}
  // observe data
  const ob = observe(data)
  ob && ob.vmCount++
}
```

`initData()` 主要做了几件事

- 从实例上获取 data 对象 
  `let data: any = vm.$options.data`
  `data = vm._data = isFunction(data) ? getData(data, vm) : data || {}`
- 判断 data 中 key 不能与 props 、methods 的 key 相同，优先级是：methods > props > 保留字段以 `$` 和 `_` 开头 (`isReserved`)
- `proxy` 代理  `vm` 下的 `_data ` ，就可以通过 `this.xxx` 访问 `vm._data` 下的属性 `xxx`
  // `this.xxx` => `vm._data.xxxx`
- 开始观测 data (`const ob = observe(data)`)

`src/core/observer/index.ts`:
```ts
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
export function observe(
  value: any,
  shallow?: boolean,
  ssrMockReactivity?: boolean
): Observer | void {
  if (value && hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    return value.__ob__
  }
  if (
    shouldObserve &&
    (ssrMockReactivity || !isServerRendering()) &&
    (isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value.__v_skip /* ReactiveFlags.SKIP */ &&
    !isRef(value) &&
    !(value instanceof VNode)
  ) {
    return new Observer(value, shallow, ssrMockReactivity)
  }
}
```

`observe` 主要的参数为第一个 `value` ，传入的 `data` 对象
`new Observe` ，初始化传入的 ` data ` 对象，` data ` 对象可能是数组，若 ` data ` 不为对象或数组，则无法继续执行，其最终返回 ` Observer ` 实例。

如果 data 有 `__ob__` 属性且 `__ob__` 为 `Observer` 构造函数的实例引用，表示这个对象被观测过直接返回, `value.__ob__` 就是现有的观察者。

构造函数 Observer
`src/core/observer/index.ts`:
```ts
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
export class Observer {
  dep: Dep
  vmCount: number // number of vms that have this object as root $data

  constructor(
	public value: any, 
	public shallow = false, 
	public mock = false) {
	
	// value 为传入的 data
	
    // this.value = value
    this.dep = mock ? mockDep : new Dep()
    this.vmCount = 0
    
	// define a property 给对象上定义一个新属性
	// data.__ob__ {
	//   value: Observer
	//   enumerable: !!enumerable,
	//   writable: true,
	//   configurable: true
	// }
    def(value, '__ob__', this)
	
    if (isArray(value)) {
      if (!mock) {
        if (hasProto) {
          /* eslint-disable no-proto */
          ;(value as any).__proto__ = arrayMethods
          /* eslint-enable no-proto */
        } else {
          for (let i = 0, l = arrayKeys.length; i < l; i++) {
            const key = arrayKeys[i]
            def(value, key, arrayMethods[key])
          }
        }
      }
      if (!shallow) {
        this.observeArray(value)
      }
    } else {
    // 老版本中的 walk(value) 在新版本中被拍平
      /**
       * Walk through all properties and convert them into
       * getter/setters. This method should only be called when
       * value type is Object.
       */
      const keys = Object.keys(value)
      for (let i = 0; i < keys.length; i++) {
        const key = keys[i]
        defineReactive(value, key, NO_INITIAL_VALUE, undefined, shallow, mock)
      }
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray(value: any[]) {
    for (let i = 0, l = value.length; i < l; i++) {
      observe(value[i], false, this.mock)
    }
  }
}
```

`new Observer()` 做了什么，看 `constructor` 做了以下事：

- 把 data 赋值到 Observer 实例的 value 属性
- 生成 dep 对象，赋值到 observe 实例的 dep 属性
- value 新增属性 `__ob__` ，引用 observe 实例
- 如果 value 是对象，遍历属性，给每一个属性加上依赖收集
- 如果 value 是数组，调用 `observeArray()`，观测数组的每个元素
- `observeArray()` 会遍历数组中的每一项，并对每一项使用 `observe(value[i])` 观测。对象则是对每一项的属性调用 `defineReactive(value, key)` ，设置其 getter 和 setter 方法。

分析 defineReactive 函数
`src/core/observer/index.ts`:
```ts
/**

* Define a reactive property on an Object.

*/

export function defineReactive(
obj: object,
key: string,
val?: any,
customSetter?: Function | null,
shallow?: boolean,
mock?: boolean,
observeEvenIfShallow = false
) {
  const dep = new Dep() 

  const property = Object.getOwnPropertyDescriptor(obj, key)
  // property: {
  //   value,
  //   writable,
  //   get,
  //   set,
  //   configurable, (可配置)
  //   enumerable
  // }
  if (property && property.configurable === false) {
	return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  if (
	(!getter || setter) &&
	(val === NO_INITIAL_VALUE || arguments.length === 2)
  ) {
	val = obj[key]
  }
}

// ...
```

- `const dep = new Dep()` ，每个 `key` 都在 `getter` 和 `setter` 中引用了一个 `dep` ，用来收集 key 的依赖。
- 通过传入的对象和属性，返回该对象属性的描述对象 (包含以下属性：value | writable | get | set | configurable | enumerable) ，[具体🔗](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor) 。若对象的 `configurable` 为 `false` 直接返回，并将对象的 `get` 和  `set` 赋值给常量 `const getter = property && property.get` 、`const setter = property && property.set` ，获取传入 `key` 的具体 `value` ，`val = obj[key]`。
- 观测属性值 `observe(val)` ，并将返回值赋值给变量 `childOb`
- 用 `Object.defineProperty(obj, key, ...)` 重写属性的 `get` 和 `set`

- 重写 get
  
```ts

// 子对象
let childOb = shallow ? val && val.__ob__ : observe(val, false, mock)

get: function reactiveGetter() {
  const value = getter ? getter.call(obj) : val
    
  if (Dep.target) {
    if (__DEV__) {
      dep.depend({
        target: obj,
        type: TrackOpTypes.GET,
        key
      })
    } else {
      dep.depend()
    }
    if (childOb) {
      childOb.dep.depend()
      if (isArray(value)) {
        dependArray(value)
      }
    }
  }
  return isRef(value) && !shallow ? value.value : value
}
```

`dep.ts` 中的 `pushTarget()` 函数被调用才会进入这步，当页面首次渲染会去获取值，这时候就会调用 `pushTarget(this)` 见 [[#^watcherGet]]，这里的 this 为 beforeCreate 初始化的 Watcher，读取 `data` 中的变量的时候触发 `getter`，此时无 `__DEV__`，执行 `dep.Depend()`

返回时做了判断如果是 Ref，外面有一层包装，返回 `value.value` 否则返回 value

- 重写 set
  
```ts
set: function reactiveSetter(newVal) {
  // ...
  if (setter) {
    setter.call(obj, newVal)
  } else if (getter) {
    // #7981: for accessor properties without setter
    return
  } else if (!shallow && isRef(value) && !isRef(newVal)) {
    value.value = newVal
    return
  } else {
    val = newVal
  }
  // ...
}
```

如果原属性对象有 `set` 则用原有的，无 `set` 有 ` get ` => 返回，如果旧值是 Ref，新值不是，`value.value = newVal` ，以上都无，说明原属性既不是对象也不是 Ref，原始类型？，直接赋新值 `val = newVal`

```ts
childOb = shallow ? newVal && newVal.__ob__ : observe(newVal, false, mock)
if (__DEV__) {
  dep.notify({
    type: TriggerOpTypes.SET,
    target: obj,
    key,
    newValue: newVal,
    oldValue: value
  })
} else {
  dep.notify()
}
```

有新值，重新判断有无被检测的子对象，修改 `childOb` ，如果有未被观测的子对象则加上观测 `observe(newVal, false, mock)` ，最后调用 `dep.notify()`

`src/core/observer/dep.ts`
```ts
export default class Dep {
  static target?: DepTarget | null
  id: number
  subs: Array<DepTarget | null> // render Watcher 数组

  // pending subs cleanup
  _pending = false

  constructor() {
    this.id = uid++
    this.subs = []
  }

  // ...
  notify(info?: DebuggerEventExtraInfo) {
    // stabilize the subscriber list first
    const subs = this.subs.filter(s => s) as DepTarget[]
    if (__DEV__ && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }

    for (let i = 0, l = subs.length; i < l; i++) {
      const sub = subs[i]
      if (__DEV__ && info) {
        sub.onTrigger &&
        sub.onTrigger({
          effect: subs[i],
          ...info
        })
      }
      sub.update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.

Dep.target = null
const targetStack: Array<DepTarget | null | undefined> = []

export function pushTarget(target?: DepTarget | null) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget() {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

`notify` 将遍历 `subs` 数组，然后调用 `update` 方法

`src/core/observer/watcher.ts`
```ts
import { queueWatcher } from './scheduler'

update () {
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

#### `Watcher` 类 - 派发更新

```ts
export default class Watcher {
  // 精简代码
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
}
```

最终会调用 `run()` 函数，`const value = this.get()` 此处调用将走到

`src/core/observer/watcher.ts` ^watcherGet
```ts
/**
* Evaluate the getter, and re-collect dependencies.
*/
get() {
  pushTarget(this) 
  let value
  const vm = this.vm
  try {
	value = this.getter.call(vm, vm)
  } catch (e: any) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

Watcher - constructor

```ts
export default class Watcher implements DepTarget {
  // ...

  constructor(
    vm: Component | null,
    expOrFn: string | (() => any),
    cb: Function,
    options?: WatcherOptions | null,
    isRenderWatcher?: boolean
  ) {
    // ...

	// parse expression for getter
	if (isFunction(expOrFn)) {
	  this.getter = expOrFn
	} else {
	  this.getter = parsePath(expOrFn)
	  if (!this.getter) {
	    this.getter = noop
		__DEV__ &&
		  warn(
		    `Failed watching path: "${expOrFn}" ` +
		      'Watcher only accepts simple dot-delimited paths. ' +
		      'For full control, use a function instead.',
		    vm
		  )
	  }
	}
  }
}
```

做了一些判断可知 `this.getter` 是在创建 `Watcher` 实例的时候传入进来的。

那么 `Watcher` 实例在哪创建？
`src/core/instance/lifecycle.ts` ^lifecycleWatcher
```ts
// we set this to vm._watcher inside the watcher's constructor
// since the watcher's initial patch may call $forceUpdate (e.g. inside child
// component's mounted hook), which relies on vm._watcher being already defined

new Watcher(
  vm,
  updateComponent, // expOrFn
  noop,
  watcherOptions,
  true /* isRenderWatcher */
)

let updateComponent
/* istanbul ignore if */

if (__DEV__ && config.performance && mark) {
  // ...
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating) // 渲染
  }
}
```

![[Pasted image 20231214113210.png]]
### 小结

![[Pasted image 20231214092920.png]]
官网的响应式原理图：

数据 `getter` 到观察者 `Watcher` 有一条线，上面写着 `Collect as Dependency` ( 收集依赖 )，意思是：数据发生 `get` 动作时 `Watcher` 开始收集依赖。

数据 `setter` 到观察者 `Watcher` 有一条线，上面写着 `Notify` 意思是：数据发生 `set` 动作时通知 `Watcher` 。 `vue` 的 `Watcher` 收到通知后下一步是调用 `update`

`Watcher` 到 `Component Render Function` 有一条线，上面写着 `Trigger re-render` 意思是：将 `update` 后的数据传入 `Component Render Function` 执行，渲染 ( `render` )  虚拟 DOM 树 ( `Virtual DOM Tree` )

`render` 到 `getter` 有一条线，上面写着 `Touch` 意思是：实例渲染期间，触摸组件的每个属性，以便将它们全部作为依赖项进行跟踪，当依赖项的 `setter` 被触发时，它会通知观察者，从而导致组件重新渲染。








