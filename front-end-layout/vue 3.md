
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

1. 词法分析
    核心函数 [[#^baseParse]]
2. 指令和语法转换阶段
    获取转换后的节点信息 [[#^getBaseTran]]
3. 可执行函数生成阶段

### template 转为预备 AST

### 预备 AST 区分不同 vue 类型的节点，将这些节点根据类型做转换，最终生成完整的 AST

### 完整的 AST 进行不同类型的组装生成渲染函数

- 主编译入口 `/packages/compiler-core/src/index.ts`

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
  const ast = isString(template) ? baseParse(template, options) : template

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
  // 创建根模板
  return createRoot(
    parseChildren(context, TextModes.DATA, []),
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
	    if (s.length === 1) {
		  // 只有单标签
          emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 1)
        } else if (s[1] === '!') {
	        
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


`/packages/compiler-core/src/compile.ts` ^getBaseTransformPreset
```ts
export function getBaseTransformPreset(
  prefixIdentifiers?: boolean
  ): TransformPreset {
  return [
    // 转化为 vue 类型的节点
    [
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
	  // 元素处理 - dynamic
	  transformElement,
	  trackSlotScopes,
	  // 文本处理 - static
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


手写：
`effectReactive.js

```js
// 1. 总入口 数据劫持 => 依赖收集 + 派发更新

const reactive = target => {

	return new Proxy(target, {
		get() {
			// 依赖收集
			track(target, key, receiver)
			const res = Reflect.set(target, key, receiver)
			return res
		},
		set() {
			// 派发更新
			trigger(target, key, value, receiver)
			const res = Reflect.set(target, key, value, receiver)
			return res
		}
	})
}

// 2. 依赖收集

const targetMap = new Map()
const track = (target, key) => {
	let _depMap = targetMap.get(target)
	if (!_depMap) {
		_depMap = new Map()
		targetMap.set(target, _depMap)
	}

	let _deps = _depMap.get(key)
	if (!_deps) {
		_deps = new Set()
		_depMap.set(key, _deps)
	}

	// 保存当前正在执行的 effect
	_deps.add(activeEffect)
}

// 3. 派发更新

const trigger = (target, key) => {
	const _depMap = targetMap.get(target)
	if (!_depMap) return
}

// 4. 副作用函数

const effectStack = []
let activeEffect

const effect = fn => {
	const effectFn = () => {
		try {
			effectStack.push(effectFn)
			activeEffect = effectFn
			return fn
		} finally {
			
		}
	}
}
```





