
## vue 2 存在的问题

- 代码架构 - 配置列表（过时 out）
- 性能优化空间 - 更新的方式去 patch ，生成 AST 树。优化已有的 API
- API 在大型项目中的可维护性
- 浏览器真的需要版本吗？- 对老浏览器的兼容，`objectDefineProperty` 数据劫持将被 `proxy` 替代

## vue 3 面临的挑战

- 太多 breaking changes（破坏性升级）-  复杂的修改，升级原有项目将变得困难
- 生态的影响
- 发布节奏 - 过早发布 demo 版本，并没有让很多人能接受

对比引发的问题：vue 3 与 vue 2 的区别？对 vue 2 项目重构为 vue 3 项目需要关注的点？vue 2 转到 vue 3 你经历过哪些坑？哪些周边生态受到了影响？

## vue 3 的优化

### 结构上的优化 - TS + monorepo

代码仓库发展历程：

monolith -> 单一代码仓库
multirepo -> 多代码仓库
monorepo -> 原子结构仓库

问题：

幽灵依赖 - 依赖瞬息万变
依赖耗时 - 安装时间过长

解决方案 - pnpm

pnpm 的工作空间有效解决上述问题

```js
import Vue from 'vue'

// Vue.mixin ... 
```

引入使用上的区分

```ts
import { createApp } from 'vue'
```

### 性能上的优化

#### 移除了很多使用率较低的 api

#### tree - shaking => 对产物打包进行了优化

#### 编译优化

- compiler => 静态节点分析做初步 AST 树 => patchFlag 分析初步 AST 树给节点打上标签，并对同一节点做同样的操作（统一操作）=> 生成最终 AST 树 => 转为 render () 函数并执行渲染。

#### 数据劫持优化

- vue 2 做深度响应需要 `$set` 、`$delete` 、`$nextTick` 等操作，对数组的大多数方法需要重写，深层对象的数据劫持耗费性能。这些 Proxy 都做得更好

## 模板编译

1. 词法分析阶段（baseParse）：
    template => AST
    (核心函数 [[#^baseParse]])
2. 指令和语法转换阶段：
     AST => 解析区分不同 vue 类型的节点 => 进行对应的转换 => 最终 AST
    (获取转换后的节点信息 [[#^getBaseTran]])
3. 可执行函数生成阶段
	 转换完的 AST 进行不同类型的组装生成渲染函数

### template 转为预备 AST

主编译入口 `/packages/compiler-core/src/index.ts`

```ts
// baseCompile 是模板的编译核心

export { baseCompile } from './compile'
```

`/packages/compiler-core/src/compile.ts`
```ts
export function baseCompile(
  template: string | RootNode,
  options: CompilerOptions = {}
): CodegenResult {
  // ...
  // template => ast

  // 判断：是文本？进函数 baseParse 做第一次转换。no，代表已经转换过，则直接赋值

  // 核心，模板返回的 AST
  
  const ast = isString(template) ? baseParse(template, options) : template

  // 核心，获取 transform 节点信息 => getBaseTransformPreset
  const [nodeTransforms, directiveTransforms] =
  getBaseTransformPreset(prefixIdentifiers)

  // ...
}
```

`/packages/compiler-core/src/parse.ts` ^baseParse
```ts
// template 可能的各种情况，分类

export const enum TextModes {
  //          | Elements | Entities | End sign              | Inside of
  // 元素
  DATA, //    | ✔        | ✔        | End tags of ancestors |
  // <textarea></textarea>
  RCDATA, //  | ✘        | ✔        | End tag of the parent | <textarea>
  // 原生模板： <script> | <noscript> | <iframe> | <style>
  RAWTEXT, // | ✘        | ✘        | End tag of the parent | <style>,<script>
  // xml 中的注释 <![CDATA[cdata]]>
  CDATA,
  // 标签属性
  ATTRIBUTE_VALUE
}

// 核心目标 - 构建 AST
// 构建基础 => 处理子节点 => 生成 AST

export function baseParse(
  content: string,
  options: ParserOptions = {}
): RootNode {
  const context = createParserContext(content, options)
  const start = getCursor(context)
  // createRoot 描述了标准 AST 内一个节点包含的属性
  // 创建根模板，由 parseChildren 函数 和 getSelection 函数，填充
  return createRoot(
    // parseChildren 遍历根模板，返回 AST 节点数据
    parseChildren(context, TextModes.DATA, []), // AST node
    // 将语句 拼成 AST node
    getSelection(context, start)
  )
}
```

`/packages/compiler-core/src/parse.ts` ^parseChildren
```ts
// 核心：将子节点转换成 AST，template 内部 子节点 => AST 阶段

function parseChildren(
  context: ParserContext,
  mode: TextModes,
  ancestors: ElementNode[]
): TemplateChildNode[] {
  // 声明部分
  const parent = last(ancestors)
  const ns = parent ? parent.ns : Namespaces.HTML
  const nodes: TemplateChildNode[] = [] // node 节点 => AST 上的某一枝干节点 nodes：node 集合，parseChildren 函数主要用于将当前这些分支内容挂在主干上

  while (!isEnd(context, mode, ancestors)) {
	__TEST__ && assert(context.source.length > 0)
    const s = context.source // 传入内容
    let node: TemplateChildNode | TemplateChildNode[] | undefined = undefined

	// 仅对 元素 以及 textarea 进入核心编辑逻辑
	if (mode === TextModes.DATA || mode === TextModes.RCDATA) {
	  // 
	  if (!context.inVPre && startsWith(s, context.options.delimiters[0])) {
        // '{{' 
        // 变量节点 => {{ dataList }}
        node = parseInterpolation(context, mode)
	  } else if (mode === TextModes.DATA && s[0] === '<') {
	    // 是元素 且 以尖括号开头
	    // 处理尖括号开头的代码：标签
	    // 
	    if (s.length === 1) {
		  // 只有尖括号 或 多写尖括号
		  // 报错，尖括号未闭合，不成对 ... 
          emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 1)
        } else if (s[1] === '!') {
	      // 尖括号后 ！可能是注释，判断注释
	      // ...
        } else if (s[1] === '/') {
	      // 后闭合标签，反标签
	      // ...
        } else if (/[a-z]/i.test(s[1])) {
	      // 解析标签，此处是元素解析了，节点赋值
	      node = parseElement(context, ancestors)
	      // ...
        } else if (s[1] === '?') {
          emitError(
		    context,
	        ErrorCodes.
	        UNEXPECTED_QUESTION_MARK_INSTEAD_OF_TAG_NAME,
            1
          )
          node = parseBogusComment(context)
        } else {
          emitError(
            context,
            ErrorCodes.INVALID_FIRST_CHARACTER_OF_TAG_NAME, 
            1
          )
        }
	  }
	}

    // 不是节点，是文本，按文本处理
	if (!node) {
      node = parseText(context, mode)
    }

    // node => 节点数组 (多层子节点) or 节点 (无子节点)
	if (isArray(node)) {
      for (let i = 0; i < node.length; i++) {
        pushNode(nodes, node[i])
      }
    } else {
      pushNode(nodes, node)
    }
  }

  // 空格处理
  // Whitespace handling strategy like v2
  let removedWhitespace = false
  if (mode !== TextModes.RAWTEXT && mode !== TextModes.RCDATA) {
    const shouldCondense = context.options.whitespace !== 'preserve'
    for (let i = 0; i < nodes.length; i++) {
      const node = nodes[i]
      if (node.type === NodeTypes.TEXT) {
        if (!context.inPre) {
          if (!/[^\t\r\n\f ]/.test(node.content)) {
            const prev = nodes[i - 1]
            const next = nodes[i + 1]
            // ...
            // 变量空格 忽略
            // 文本空格 按 HTML 处理
            // 注释 按 注释 处理
          }
        }
      }
    }
  }

  // 返回 nodes，中间部分都是对 nodes 的添加
  return removedWhitespace ? nodes.filter(Boolean) : nodes
}
```

`/packages/compiler-core/src/ast.ts` ^createRoot
```ts
export const enum NodeTypes {
  ROOT, // 根节点
  ELEMENT, // 元素
  TEXT, // 文本
  COMMENT, // 注释
  SIMPLE_EXPRESSION, //简单描述
  INTERPOLATION, // 变量
  ATTRIBUTE, // 属性
  DIRECTIVE, // 指令

  // containers - 容器类型
  COMPOUND_EXPRESSION,
  IF,
  IF_BRANCH,
  FOR,
  TEXT_CALL,

  // codegen - 代码生成
  VNODE_CALL,
  JS_CALL_EXPRESSION,
  JS_OBJECT_EXPRESSION,
  JS_PROPERTY,
  JS_ARRAY_EXPRESSION,
  JS_FUNCTION_EXPRESSION,
  JS_CONDITIONAL_EXPRESSION,
  JS_CACHE_EXPRESSION,

  // ssr codegen
  JS_BLOCK_STATEMENT,
  JS_TEMPLATE_LITERAL,
  JS_IF_STATEMENT,
  JS_ASSIGNMENT_EXPRESSION,
  JS_SEQUENCE_EXPRESSION,
  JS_RETURN_STATEMENT
}

export function createRoot(
  children: TemplateChildNode[],
  loc = locStub
  ): RootNode {
  return {
    type: NodeTypes.ROOT, // 节点类型
    children, // 子节点
    helpers: new Set(), // 对节点处理以及处理集
    components: [], // 组件
    directives: [], // 指令
    hoists: [], // 提升
    imports: [], // 引入
    cached: 0,
    temps: 0,
    codegenNode: undefined, // 生成渲染
    loc
  }
}
```

NodeType 和 TextModes 分别指代不同的节点类型

NodeType ：代表 vue 类型，是包含指令 (DIRECTIVE) 的，有文本类型 (包括 JS 代码)，非文本类型 (逻辑类型，容器类型 ...) 

TextModes：代表当前模板，确认其是普通的标签，还是原生具有含义的标签 (主要) 或是 xml 中的注释和标签中的属性 (个别判断)

### 预备 AST 区分不同 vue 类型的节点，将这些节点根据类型做转换，最终生成完整的 AST

`/packages/compiler-core/src/compile.ts` ^getBaseTransformPreset
```ts
export function getBaseTransformPreset(
  prefixIdentifiers?: boolean
  ): TransformPreset {
  return [
    // 操作 AST
    [
      /**
      * 各种 vue 指令
      */
	  transformOnce,
	  transformIf,
	  transformMemo,
	  transformFor,
	  
	  ...(__COMPAT__ ? [transformFilter] : []),
	  ...(!__BROWSER__ && prefixIdentifiers
	    ? [
			// order is important
			trackVForSlotScopes,
			transformExpression
		  ]
		: __BROWSER__ && __DEV__
		  ? [transformExpression]
		  : []),
	  transformSlotOutlet,
	  // 元素处理 - dynamic(动态类型，节点是以动态的方式呈现)
	  transformElement,
	  trackSlotScopes,
	  // 文本处理 - static(静态类型，节点是以静态的方式呈现)
	  transformText
	],
	{
	  on: transformOn,
	  bind: transformBind,
	  model: transformModel
	}
  ]
}
```

`transformText`
`/packages/compiler-core/src/transforms/transformText.ts`

需要转译的节点仅限：`NodeTypes.ROOT` 根节点 | `NodeTypes.ELEMENT` 元素节点 | `NodeTypes.FOR` 循环节点 | `NodeTypes.IF_BRANCH` 逻辑节点

`transformElement`
`/packages/compiler-core/src/transforms/transformElement.ts`

首先确认当前节点是元素节点 `NodeTypes.ELEMENT`
再从 `shouldBuildAsSlots` 开始看，此处开始节点分流 ...

### 完整的 AST 进行不同类型的组装生成渲染函数

`packages/compiler-core/src/compile.ts`
```ts
// 调用 generate， AST 进行生成、产出，产出可执行方法：render()
return generate(
  ast,
  extend({}, options, {
    prefixIdentifiers
  })
)
```

`generate` 函数  =>  `render` 函数

`packages/compiler-core/src/codegen.ts`
```ts
export function generate(
  ast: RootNode,
  options: CodegenOptions & {
	onContextCreated?: (context, CodegenContext) => void
  } = {}
): CodegenResult {
  // 内容生成器
  const context = createCodegenContext(ast, options) // 输入，AST
  
  // ...
}
```

## 基于 proxy 响应式

1. 数据劫持  |  数据响应 ：数据变化 => 函数的监听执行
    响应式入口：

2. 依赖收集 
     => 当前 `vm` 实例上挂载 `effect` (副作用)函数 
     => `activeEffect` (激活的副作用) 切换成 `effect` 函数 
     => 在 `effect` 函数中创建属性 `deps`，用于传递依赖

  2.5
     1. 结合上面的两部分可知：访问变量 => 触发对应 `get()` => 创建 `targetMap` => 创建 `depsMap` | `depsMap = targetMap.get(target)` => 转为 `deps` 数组 | `deps = [...depsMap.values()]`
     2. `depsMap` 将放入 `activeEffect` 中 - 被收集的订阅方 (==会影响到哪些地方，变量位置==)
        同时 `activeEffect` 中也会存在 `deps` 数组，用于存放关联者的 `depsMap` - 订阅的内容 (==变量修改后关联到哪些内容==)
 
3. 派发更新 (ref) :
     依赖的 `set()` 被触发 => 修改相应的属性 => 获取到 `targetMap` 的订阅方 `depsMap` | `depsMap = new Map()` => 链条传递 => 在下次渲染中清空所有执行 `effect` 

### 入口

`/packages/reactivity/src/reactive.ts`
```ts
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // ...

  // 创建了 reactive 对象(响应式核心对象)
  // 对于 target 的绑定
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}
```

```ts
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  // ...


  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object

  // 标识当前对象时候已经是响应式
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }

  // ...


  // 通过 proxy 对 target 做了一层代理
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )

  // 将所有的响应以 Map 的形式存储，相当于做了一层缓存
  proxyMap.set(target, proxy)
  return proxy
}
```

### 依赖收集

`/packages/reactivity/src/effect.ts`
```ts
// 核心响应副作用函数
export function effect<T = any>(
  fn: () => T,
  options?: ReactiveEffectOptions
): ReactiveEffectRunner {
  if ((fn as ReactiveEffectRunner).effect instanceof ReactiveEffect) {
    fn = (fn as ReactiveEffectRunner).effect.fn
  }

  const _effect = new ReactiveEffect(fn)
  if (options) {
    extend(_effect, options)
    if (options.scope) {
      recordEffectScope(_effect, options.scope)
    }
  }

  if (!options || !options.lazy) {
    _effect.run()
  }

  const runner = _effect.run.
  bind(_effect) as ReactiveEffectRunner
  
  runner.effect = _effect
  return runner
}
```

收集：
```ts
export function trackEffects(
  dep: Dep,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {

  // 1. 是否需要采集
  let shouldTrack = false
  if (effectTrackDepth <= maxMarkerBits) {
    if (!newTracked(dep)) {
      dep.n |= trackOpBit // set newly tracked
      shouldTrack = !wasTracked(dep)
    }
  } else {
    // Full cleanup mode.
    shouldTrack = !dep.has(activeEffect!)
  }

  // 收集主模块
  if (shouldTrack) {
    // activeEffect 中存在 deps，添加 dep 
    dep.add(activeEffect!)
    activeEffect!.deps.push(dep)
    if (__DEV__ && activeEffect!.onTrack) {
      activeEffect!.onTrack(
        extend(
          {
            effect: activeEffect!
          },
          debuggerEventExtraInfo!
        )
      )
    }
  }
}
```

派发：
```ts
function triggerEffect(
  effect: ReactiveEffect,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  if (effect !== activeEffect || effect.allowRecurse) {
    if (__DEV__ && effect.onTrigger) {
      effect.onTrigger(
      extend({ effect }, debuggerEventExtraInfo))
    }
    if (effect.scheduler) {
      effect.scheduler() // fn 中在微任务队列中执行的地方
    } else {
      effect.run() // fn 同步执行
    }
  }
}
```

### 派发更新

`/packages/reactivity/src/ref.ts`
```ts

```

## 手写 

`effectReactive.js`
```js
// 1. 总入口 数据劫持 => (依赖收集 + 派发更新): 上下链条

const reactive = target => {
  return new Proxy(target, {
    get(target, key, receiver) {
      // 依赖收集
      track(target, key, receiver)
      const res = Reflect.set(target, key, receiver)
      return res
    },
    set(target, key, value, receiver) {
      // 派发更新
      trigger(target, key, value, receiver)
      const res = Reflect.set(target, key, value, receiver)
      return res
    }
  })
}

// 4. 副作用函数
const effectStack = []
let activeEffect // 被激活

const effect = fn => {
  const effectFn = () => {
    try {
      effectStack.push(effectFn)
      activeEffect = effectFn 
      // 被激活的永远是当前这个 effectFn
      return fn
    } finally {
	  effectStack.pop()
	  activeEffect = effectStack[effectStack.length - 1]
    }
  }
  effectFn()

  return effectFn
}

// 2. 依赖收集
const targetMap = new Map()
const track = (target, key) => {
  // 依赖目标(订阅方)
  let _depsMap = targetMap.get(target) 
  // 尝试获取订阅对象，首次 undefined
  if (!_depsMap) {
    _depsMap = new Map() // 创建 Map，存放订阅对象
    targetMap.set(target, _depsMap) // 例如：{'data': {..}}
    // 并把 Map 挂载到 targetMap 上，targetMap(多对象集合)
  }

  // 依赖补全(依赖) - 依赖保存
  let _dep = _depsMap.get(key) 
  // 尝试获取对象属性 => 依赖，首次 undefined
  if (!_dep) {
    _dep = new Set() // 创建 Set，存放属性
    _depsMap.set(key, _dep) // 例如：{'list': eFn}
  }

  // 保存当前正在执行的 effect
  _dep.add(activeEffect) // 将激活的副作用函数添加到依赖列表，表示当前依赖正在被收集
}

// 3. 派发更新
const trigger = (target, key) => {
  // 订阅方
  const _depsMap = targetMap.get(target)
  if (!_depsMap) return

  // 订阅内容
  const _dep = _depsMap.get(key)
  if (!_dep) return

  // 派发 => 执行当前所有 effectFn
  _dep.forEach(effectFn => {
    effectFn && effectFn() // 判断当前活跃的副作用函数并执行
  })
}
```





