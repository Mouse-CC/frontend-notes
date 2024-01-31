
什么值适合存储在 vuex ？

全局要引用的内容（特征变量），多组件需要获得的内容

`vuex` 中通过 `action` 去触发 `mutation` 去改变 `state`，为什么不直接用 `action` 去改变 `state` ？

状态流转最终的操作一定是同步的：操纵数据入库的来源过于多必定会产生冲突，后端解决这个问题（锁：lock）=> 在进入某一项写入或读取操作的时候，其他方无法进行写入或读取操作（锁定当前有且仅有一个来源在操作）在前端 vue 中 `vuex` 的 `actions` 能够接收同步/异步操作，但当 `actions` 中操作执行完开始 `mutation` 对 `state` 修改始终是同步的。

# 3 版本

`vuex3` 基于 `mixin` 做数据混入，同时将状态数据基于 `vue` 实例的响应式实时更新

```vue
<template>
  <div>
    {{ $store.state.name }}
  </div>
  // 1. 挂载 beforeCreate => Vue.mixin() 混入，$store 此时混入并将实例挂载在 store 上
  // 2. 拼装了一个 store 类，用于生成整个 store 实例功能

  get state() {
    return this._vm._data.$$state
  }
  // 3.能够实现响应式的本质 new Vue()，借助了 vue 实例的响应式能力
</template>
```

# 4 版本

基于单例模式，参数注入

```js
const store = new Store()

const rootInstance = {
  parent: null,
  provides: {
    store
  }
}

const parentInstance = {
  parant: parentInstance,
  provides: {
    store
  }
}

store.dispatch('change')
// 共享同一单例实例
```

# 3 版本源码

## `vuex/src/store.js - class Store`
```js
export class Store {
  constructor(options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731

    // 不存在 Vue 实例，且有 window，window 上挂载了 Vue。
    // 此情况 => 以 script 标签全局挂载 vue ，才会是这个情况。
    // install => 传入 Vue，并将其赋值给全局 Vue 变量，applyMixin 为调用传入的 Vue.mixin()
    if (!Vue && typeof window !== "undefined" && window.Vue) {
      install(window.Vue);
    }

    if (__DEV__) {
      // 创建存储实例前，要用 Vue.use(Vuex)
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`);
      // vue 底层依赖 Promise，需要浏览器支持
      assert(
        typeof Promise !== "undefined",
        `vuex requires a Promise polyfill in this browser.`
      );
      // store 必须要实例化（ new ）后才能使用，此处的 this 表示实例
      assert(
        this instanceof Store,
        `store must be called with the new operator.`
      );
    }

    // options 为传递进来的参数，对其结构，vuex 本身也是支持 plugins => 插件化拓展
    const { plugins = [], strict = false } = options;

    // store internal state
    this._committing = false;

    // 处理用户自定义的 action
    // Object.create(null)，为什么不用 {} ？ => 原型链
    // 因为 Object.create(null) 创建出来的空对象更纯净
    // 即，没有原型对象
    // 
    // Object.create(null).__proto__ => undefined
    // 函数签名：Object.create(proto)
    // proto 新建对象的原型对象，以 null 为原型创建结果为 {}
    // 且上面是没有属性的 => __proto__ = undefined
    //
    // ({}).__proto__                => Object.prototype
    // Object 本身是构造函数，只有函数才有 prototype 属性
    // 值是一个有 constructor 属性的对象，不是空对象
    // Object.prototype 是原型链的尽头
    
    this._actions = Object.create(null);
    this._actionSubscribers = [];

    // mutations
    this._mutations = Object.create(null);
    this._wrappedGetters = Object.create(null);

    // 命名空间，做层级
    this._modules = new ModuleCollection(options);
    this._modulesNamespaceMap = Object.create(null);

    // 观察模式的辅助 => 观察者模式 & 发布订阅是什么概念？
    // 都是设计模式，都是流式触发后续的操作：依赖变更
    // 通过对数据劫持，劫持数据的读取和写入，拦截后数据后，对其后置的钩子进行操作。依赖收集，observe。deps 中存放多个 watcher，Vue3 中是使用 effect 副作用去通知数据的修改。
    // 发布订阅，使用某种分发机制，通过某一中心处理。订阅，异步注册过程，回调往往是在订阅者被触发后的回调。
    this._subscribers = [];
    this._watcherVM = new Vue();
    this._makeLocalGettersCache = Object.create(null);

    // bind commit and dispatch to self
    const store = this;
    const { dispatch, commit } = this;
    this.dispatch = function boundDispatch(type, payload) {
      // 确认当前 this 执行状态 => 指向实例化
      // => 确保 store 是实例化后的 => class Store
      return dispatch.call(store, type, payload);
    };
    this.commit = function boundCommit(type, payload, options) {
      return commit.call(store, type, payload, options);
    };

    // strict mode
    this.strict = strict;

    const state = this._modules.root.state;

    // 每个模块的初始化
    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root);

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state);

    // apply plugins
    plugins.forEach((plugin) => plugin(this));

    const useDevtools =
      options.devtools !== undefined ? options.devtools : Vue.config.devtools;
    if (useDevtools) {
      devtoolPlugin(this);
    }
  }

  get state() {
    return this._vm._data.$$state;
  }

  set state(v) {
    if (__DEV__) {
      assert(
        false,
        `use store.replaceState() to explicit replace store state.`
      );
    }
  }

  commit(_type, _payload, _options) {
    // check object-style commit
    const { type, payload, options } = unifyObjectStyle(
      _type,
      _payload,
      _options
    );

    const mutation = { type, payload };
    const entry = this._mutations[type];
    if (!entry) {
      if (__DEV__) {
        console.error(`[vuex] unknown mutation type: ${type}`);
      }
      return;
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator(handler) {
        handler(payload);
      });
    });

    this._subscribers
      .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
      .forEach((sub) => sub(mutation, this.state));

    if (__DEV__ && options && options.silent) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
          "Use the filter functionality in the vue-devtools"
      );
    }
  }

  dispatch(_type, _payload) {
    // check object-style dispatch
    const { type, payload } = unifyObjectStyle(_type, _payload);

    const action = { type, payload };
    const entry = this._actions[type];
    if (!entry) {
      if (__DEV__) {
        console.error(`[vuex] unknown action type: ${type}`);
      }
      return;
    }

    try {
      this._actionSubscribers
        .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
        .filter((sub) => sub.before)
        .forEach((sub) => sub.before(action, this.state));
    } catch (e) {
      if (__DEV__) {
        console.warn(`[vuex] error in before action subscribers: `);
        console.error(e);
      }
    }

    const result =
      entry.length > 1
        ? Promise.all(entry.map((handler) => handler(payload)))
        : entry[0](payload);

    return new Promise((resolve, reject) => {
      result.then(
        (res) => {
          try {
            this._actionSubscribers
              .filter((sub) => sub.after)
              .forEach((sub) => sub.after(action, this.state));
          } catch (e) {
            if (__DEV__) {
              console.warn(`[vuex] error in after action subscribers: `);
              console.error(e);
            }
          }
          resolve(res);
        },
        (error) => {
          try {
            this._actionSubscribers
              .filter((sub) => sub.error)
              .forEach((sub) => sub.error(action, this.state, error));
          } catch (e) {
            if (__DEV__) {
              console.warn(`[vuex] error in error action subscribers: `);
              console.error(e);
            }
          }
          reject(error);
        }
      );
    });
  }

  subscribe(fn, options) {
    return genericSubscribe(fn, this._subscribers, options);
  }

  subscribeAction(fn, options) {
    const subs = typeof fn === "function" ? { before: fn } : fn;
    return genericSubscribe(subs, this._actionSubscribers, options);
  }

  watch(getter, cb, options) {
    if (__DEV__) {
      assert(
        typeof getter === "function",
        `store.watch only accepts a function.`
      );
    }
    return this._watcherVM.$watch(
      () => getter(this.state, this.getters),
      cb,
      options
    );
  }

  replaceState(state) {
    this._withCommit(() => {
      this._vm._data.$$state = state;
    });
  }

  registerModule(path, rawModule, options = {}) {
    if (typeof path === "string") path = [path];

    if (__DEV__) {
      assert(Array.isArray(path), `module path must be a string or an Array.`);
      assert(
        path.length > 0,
        "cannot register the root module by using registerModule."
      );
    }

    this._modules.register(path, rawModule);
    installModule(
      this,
      this.state,
      path,
      this._modules.get(path),
      options.preserveState
    );
    // reset store to update getters...
    resetStoreVM(this, this.state);
  }

  unregisterModule(path) {
    if (typeof path === "string") path = [path];

    if (__DEV__) {
      assert(Array.isArray(path), `module path must be a string or an Array.`);
    }

    this._modules.unregister(path);
    this._withCommit(() => {
      const parentState = getNestedState(this.state, path.slice(0, -1));
      Vue.delete(parentState, path[path.length - 1]);
    });
    resetStore(this);
  }

  hasModule(path) {
    if (typeof path === "string") path = [path];

    if (__DEV__) {
      assert(Array.isArray(path), `module path must be a string or an Array.`);
    }

    return this._modules.isRegistered(path);
  }

  hotUpdate(newOptions) {
    this._modules.update(newOptions);
    resetStore(this, true);
  }

  _withCommit(fn) {
    const committing = this._committing;
    this._committing = true;
    fn();
    this._committing = committing;
  }
}
```

## `vuex/src/store.js - function install`
```js
export function install(_Vue) {
  if (Vue && _Vue === Vue) {
    if (__DEV__) {
      console.error(
        "[vuex] already installed. Vue.use(Vuex) should be called only once."
      );
    }
    return;
  }
  Vue = _Vue;
  applyMixin(Vue);
}
```
## 总结

1. `install(window.Vue)` `install` => 传入 `Vue`，并将其赋值给全局 `Vue` 变量，`applyMixin` 为调用传入的 `Vue.Mixin()`。模块初始化：`installModule(this, state, [], this._modules.root)`，调用方法 `registerMutation | registerAction | registerGetter` 给 store 加上 `_mutations | _actions | _wrappedGetters` ...
2. `resetStoreVM` 新建 `vue` 实例，对实例做了设置，将 `state` 所有值变成实例 `data: { $$state: state, }` ，`const wrappedGetters = store._wrappedGetters` 遍历 `Object.keys(obj).forEach(key => fn(obj[key], key))`：`obj` 为 `wrappedGetters` ，`fn (obj[key], key)`：
```js
   (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    // direct inline function use will lead to closure preserving oldVm.
    // using partial to return function with only arguments preserved in closure environment.

    // partial 去返回函数，闭包中保留传入的 store，fn 为上层遍历 getter key 对应的值，即函数，传入 store 帮助函数完成
    computed[key] = partial(fn, store);
    // 拦截：遍历的 key，将其放入 store.getters 中，重写 get
    Object.defineProperty(store.getters, key, {
      // 返回 实例 computed 计算后的值
      get: () => store._vm[key],
      enumerable: true, // for local getters
    });
  }

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent;
  Vue.config.silent = true;
  // computed 响应式值
  store._vm = new Vue({
    data: {
      $$state: state,
    },
    computed,
  });
  Vue.config.silent = silent;
```

# 手写 vuex4 原理

```js
import { inject, reactive } from 'vue'

export function createStore(options) {
  return new Store(options)
}

export function useStore(injectkey = 'store') {
  return inject(injectkey)
}

class Store {
  // Vue 配置 use 的入口处安装
  install(vue, injectKey = 'store') {
    vue.provide(injectKey, this)
    vue.config.globalProperties.$store = this
  }

  constructor(options) {
    const store = this
    // 1. state 响应式
    // 相比 3 版本，reactive 用 proxy 帮助做了响应式，不再需要 computed
    store._state = reactive({
      data: options.state
    })

    // 2. 实现 getters
    const _getters = options.getters
    
    store._getters = {}
    
    forEachValue(_getters, (fn, key) => {
      new Proxy(store.getters, {
        // 实时获取具体值
        get: () => fn(store.state)
      })
      // or
      Object.defineProperty(store.getters, key, {
        enumerable: true,
        get: () => fn(store.state)
      })
    })

    // 注册 mutation，action，反复调用执行
    const _mutations = options.mutations

	store._mutations = Object.create(null)
	forEachValue(_mutations, (mutation, key) => {
	// mutation 为 mutations 下的每一条，value 为 mutation 的 payload: (state, payload) => { ... }
	  store._mutations[key] = (value) => {
	    mutation.call(store, store.state, value)
	  }
	})

    const _actions = options.actions
    
    store._actions = Object.create(null)
    forEachValue(_actions, (action, key) => {
      store._actions[key] = (value) => {
        action.call(store, store.state, value)
      }
    })
  }

  get state() {
    return this._state.data
  }

  dispatch = (type, value) => {
    this._actions[type](value)
  }

  commit = (type, value) => {
    this._mutations[type](value)
  }
}

function forEachValue(obj, fn) {
  (Object.keys(obj) || []).forEach(key => fn(obj[key], key))
}
```

### 总结

`vuex` 本质就是一层代理加上对外接口的暴露 ( `dispatch` | `commit` ...)

# Q & A

## 发版后，客户浏览器立即请求最新资源，而不是走缓存

1. Hash (最好的办法)，用一串 hash 值修改发办变更的文件名，当文件名不同，请求自然会去请求最新的文件，而不去请求缓存。
2. 云上，定时更新文件 => 强制让所有边缘节点做一次同步

## 总线与外部依赖强相关联，当依赖被销毁，总线也去除这条依赖？

实时数据 GC (垃圾回收) => state 中若干变量会跟随其清空而清空，假设：`state: { static: {}, temp: {} }` 长期留存的放在 `static` ，需要回收的放在 `temp` 当生命周期到 `beforeDestroy` 那么就去 `temp` 中清空需要清空的内容。

状态机请保持全局唯一性