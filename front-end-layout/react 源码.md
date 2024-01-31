# 源码学习

## 关于 react 源码的一些实际情况：

- React 源码很多人没有读过
- 读过的和 react 项目开发质量没有强相关
- 就算不读源码，依然能回答得好问题
- 读了，面试也不一定能过
- 大厂考查源码，更多是关注基础能力
  - 你对工具，有没有深入探究的热情，追求极致
  - 对当前前端技术发展，有无了解
  - Render / setState 的时候，发生了什么
- 读源码对你有什么帮助？
  - 心里有数
  - 思想借鉴
  - 应付面试

## 本章主旨

- 搞定 react 整体工作流程
- 后续有 RN 内容，帮助更好的理解 react

# React 版本

## v15 stack reconciler - 栈递归遍历

## 16 .9 ~ 17.0.2 fiber reconciler

- 做一个异步可中断的更新
- 17.0.2，只是把数据结构给用户，整了一些不稳定但能用的特性，让用户使用。

## 17.0.2

- Legacy 模式：CRA 默认创建的，我有 Fiber 的结构，但我不会打断 - `xxx.old.js`
- Concurrent 模式：可以实现高优先级打断低优先级 - `xxx.new.js`

## 18

- Concurrent 模式 ++

>[!tip] 建议：
>- 阅读 17.0.2 版本的代码，至少还是能做调试的
>- 了解 18 的特性和相关能力

# Start：

## 看一段代码

```jsx
const App = () => {
  return (
    <div>
      <h2>hi</h2>
      <p>{ text }</p>
      {
        [...data].map(item => <Children data={item} key ={item.id} />)
      }
    </div>
  )
}
```

- 当 data 发什么变化，如何最快判断出来，并完成更新
- React 来说，要保持运行时的灵活性，最好办法，就是==从头再来一遍==看看和之前有哪些区别。

## React 中有哪些数据结构

>[!tip]
>四个：
>1. CreateElement 生成的 v-dom
>2. Current Fiber
>3. WorkinProgress Fiber
>4. 真实 DOM

### v-dom / element

```jsx
function App() {
  return (
    <div className="app">
      <div id="list">
        <ul>
          <li>list 1</li>
          <li>list 2</li>
          <li>list 3</li>
        </ul>
      </div>
      <h2>hello luyi</h2>
    </div>
  );
}
```

- 函数，执行结果是什么？

```js
function App() {
  return (
    // 1. 
    React.createElement("div", { className: "app" }, 
      // 2.
      React.createElement("div", { id: "list" }, 
        // 3.
        React.createElement("ul", null, 
          // 4.
          React.createElement("li", null, "list 1"), 
          React.createElement("li", null, "list 2"), 
          React.createElement("li", null, "list 3")
        )
      )
      // 2
      React.createElement("h2", null, "hello luyi"),
    )
  )
}

<!------------------->

// 数据结构 element
const element = {
  type: "div",
  props: {
    className: "app",
    children: [
      { type: "div", 
        props: {
          id: "list"
        },
        children: [
          { type: "ul",
            children: [...]
          }
        ]
      },
      { type: "h2", 
        props: {
          text: "hello luyi"
        } 
      }
    ]
  }
}
```

### Fiber

- 很大程度上，是和 element 对应的，但不完全是，比如 function App 这个节点，App 本身，也是一个 fiber 节点。
- 本质上就是一个数据结构 tree-node

```js
FiberNode = {
  tag, // 标明节点类型
  key,
  type, // dom 元素的类型节点

  // fiber 组成的是一棵树，但是从数据结构的角度来说，是一个链表
  return, // 指向父节点
  child, // 指向第一个子节点
  sibling, // 指向兄弟节点

  // pendingProps, memorizedProps, updateQueue, memorizeState

  // effect
  effectTag, // 用来收集 effect
  nextEffect, // 指向下一个 effect
  firstEffect, // 第一个 effect
  lastEffect, // 最后一个 effect

  alternate, // 双缓存树，current Fiber 对应 workInprogress Fiber
}
```

#### Fiber 图

![[react 源码 2024-01-25 16.02.21.excalidraw]]

### 双缓存架构

![[react 源码 2024-01-26 10.19.28.excalidraw]]

- Fiber 只是中间态，当内容发生变化，生成 `work in progress` 部分，`effectList` 会把变化记录下来。实际 `DOM` 渲染后，`work in progress` 将变成 `current` ，之前的 `current` 将被删除 ...

>[!quote] 
>React 工作流程 ( update / mount ) 就是：对比 `v-dom` 和 `current Fiber` ，来生成 `workProgress Fiber` 的过程。
>
>通过差异形成的 `effectList` ，`在 DOM` 更新之后 => 删掉 `current Fiber` ，让当前的 `workProgress Fiber` 变成 `current Fiber`
>
>删除 / 切换 `current Fibel` ：` root.current = finishedWork `

- `effectList` 是一个链表，从一些根节点出发，记录当前改动的内容，即对 `current Fiber` 变化的记录 => `nextEffect`
 
# Analysis

![[screenshot-20240126-144228.png]]

![[react 源码 2024-01-26 15.29.59.excalidraw]]

## 详述 beginWork

```js
function beginWork(current, workInProgress, renderLanes) {
  var updateLanes = workInProgress.lanes
  // ...
  if (current !== null) {
    var oldProps = current.memoizedProps // v-dom
    var newProps = workInProgress.pendingProps // current Fiber
    // 新老 props 对比
  } else {
    // ...
  }
  // TODO: This assumes that we're about to evaluate the component and process
  // the update queue. However, there's an exception: SimpleMemoComponent
  // sometimes bails out later in the begin phase. This indicates that we should
  // move this assignment out of the common path and into each branch.

  workInProgress.lanes = NoLanes;

  // 根据类型创建 Fiber
  switch (workInProgress.tag) {
    // 初始化必定走入
    case IndeterminateComponent: // 函数，但不确定是函数组件
      {
        return mountIndeterminateComponent(current, workInProgress, workInProgress.type, renderLanes);
      }
    // ...
    case FunctionComponent: // 函数组件
      {
        var _Component = workInProgress.type;
        var unresolvedProps = workInProgress.pendingProps;
        var resolvedProps = workInProgress.elementType === _Component ? unresolvedProps : resolveDefaultProps(_Component, unresolvedProps);
        return updateFunctionComponent(current, workInProgress, _Component, resolvedProps, renderLanes);
      }

    case ClassComponent: // 类组件
      {
        var _Component2 = workInProgress.type;
        var _unresolvedProps = workInProgress.pendingProps;

        var _resolvedProps = workInProgress.elementType === _Component2 ? _unresolvedProps : resolveDefaultProps(_Component2, _unresolvedProps);

        return updateClassComponent(current, workInProgress, _Component2, _resolvedProps, renderLanes);
      }


    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);

    case HostComponent: // 宿主节点：div、p、button、ul ...
    // 就是 dom 支持的节点
      // diff 逻辑一般在 updateHostComponent 中
      return updateHostComponent(current, workInProgress, renderLanes);

    case HostText:
      return updateHostText(current, workInProgress);

    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress, renderLanes);

    case HostPortal:
      return updatePortalComponent(current, workInProgress, renderLanes);
    // ...
  }
}
```

递归所有子节点
```js
// 循环
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork) {
  // ...
  
  if ( (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork$1(current, unitOfWork, subtreeRenderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork$1(current, unitOfWork, subtreeRenderLanes);
  }

  resetCurrentFiber();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

存在兄弟节点
```js
var siblingFiber = completedWork.sibling;

if (siblingFiber !== null) {
  // If there is more work to do in this returnFiber, do that next.
  workInProgress = siblingFiber;
  return;
} // Otherwise, return to the parent
```

### 抽象上述代码
```js
while (workInProgress !== null) {
  performUnitOfWork(workInProgress);
}

next = child = beginWork()

if (next !== null) {
  completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}

completeUnitOfWork() {
  // ...
  do {
    // ...

    var siblingFiber = completedWork.sibling;

    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      // 有兄弟节点跳出循环
      return;
    }

    // 返回上一层
    completedWork = returnFiber; // Update the next thing we're working on in case something throws.

    workInProgress = completedWork;
  } while (completedWork !== null)
}
```

### beginWork：

使用 v-dom 和 current Fiber 来创建 workInProgress Fiber
- 期间会执行（函数组件、类组件、diff 子节点）
- 给结果打上 effectTag => 可能做（增 | 删 | 改）操作

### completeWork：

合并递归，向上合并
- 生成 effectList
- 构建真实的 dom，但不挂载在界面上。

逻辑：sibling 存在继续执行 beginWork，然后是 completeWork 因为已经没有子节点了，合并，再向上一层

### 函数 updateHostComponent

`isDirectTextChild` 用来判断只有文本
`markRef` 处理 `ref`
`reconcileChildren` 首次走 `mount` 挂载 child 的 Fiber，其他走调和，调和子节点 `reconcileChildFibers`（diff 逻辑）

## completeWork

三个阶段：
### commitBeforeMutationEffects
更新前 -
- 生命周期 `getSnapshotBeforeUpdate`
- 如果存在 `useEffect` ，则调度 `flushPassiveEffects`
### commitMutationEffects
- 首先，遍历所有的 `effect` ，看是否需要重置文本节点
- Ref 检查
- 判断是否有 `effectTag` 

切换 current Fiber
### commitLayoutEffects
- 执行 cdm 和 cdu 生命周期

# Advanced

## 浏览器帧率

一般情况下，我们看的视频，是由一张张图片动态组合起来的，当图片切换速度超过每秒 25 张，人眼就感受不到这个切换过程了。

浏览器一般可以做到，每秒 60 帧。每一 (帧 / 张图) 占 16.666 毫秒
![[react 源码 2024-01-31 10.23.27.excalidraw]]

```js
while (workInProgress !== null) {
  // 本质是个递归
  performUnitOfWork(workInProgress);
}
```

- `performUnitOfWork` 有 20 万个节点要执行
     我执行一会，询问浏览器，是否执行 `layout` 
     是，则 `layout` 先更新，然后这 20 万个节点再继续：`beginWork` -> `completeWork`
     解决上图宏任务时间过长的问题，先 layout 更新
     PS：在 React ( beginWork -> completeWork ) 宏任务执行时间最长 `5ms`

- React 的问题，其机制决定了，构建是由根节点遍历，所以 `beginWork` 和 `completeWork` 的时间，可能很长 ... 源码内的东西没有微任务

- 怎么实现？
    如果我在固定的时间内，能快速地把任务分片，并且以切换下一个宏任务的方式执行后续代码，是否就可以做到了？

```js
// 加一个条件，并且不需要让出执行权给浏览器
while (workInProgress !== null && !shouldYield()) {
  // 本质是个递归
  performUnitOfWork(workInProgress);
}

function shouldYield() {
  // 5ms 到了，或者 用户的高优先级，打断了低优先级
  return now() >= deadline || navigator.scheduling.isInputPending()
}
```

## 实现：

```js
function testTask() {
  // todo
  return () => {
    // todo
    return () => {
      // todo
      return () => {};
    };
  };
}

// ******************** 测试用例

const queue = []; // [{ cb: testTask }, ...]

let deadline;
const threshold = 5;

const now = () => performance.now();
const peek = (arr) => arr[0];
const transition = [];

// 为什么不用 setTimeout，因为 setTimeout 有 4ms 延迟
// 为什么不用微任务，为了保证 layout 执行
// 顺序上 宏 -> 微 -> RAF -> layout -> requestIdleCallback
// 宏 -> layout -> 微 -> 宏 是不行的，微任务需要在 layout 之前执行

// 使用 new MessageChannel() 为了产生一个宏任务
// 宏 -> layout -> 宏 这样顺序上就没问题了

const postMessage = (() => {
  // 当函数再次执行，应当为拿回执行权，再次执行 flush
  const { port1, port2 } = new MessageChannel();
  // splice 移除数组第一个元素，返回一个元素的数组，执行存入的 flush
  const func = () => transition.splice(0, 1).forEach((f) => f());

  // 第二个宏任务，取回执行权，继续执行
  port1.onmessage = func; // port1 接收

  // 第一个宏任务
  return () => port2.postMessage(null); // port2 发出消息
})();

function startTransition(flush) {
  transition.push(flush) && postMessage();
  // postMessage  执行 -> port2.postMessage(null)
  // 再次 则是执行接收 port1.onmessage = func
}

// 实现时间分片
function flush() {
  // 在 5ms 内让出执行权利，也就意味着，deadline = 当前时间 + 5ms
  deadline = now() + threshold;
  // 取出，要执行的任务，queue 结构：[{ cb: testTask }]
  let task = peek(queue);

  while (task && !shouldYield()) {
    const { cb } = task;
    task.cb = null; // [{ cb: null }]
    // 执行任务
    const next = cb();
    // 如果 next 是函数，表明任务还未执行完毕，也就是 workInProgress 的 beginWork 还没完
    if (next && typeof next === "function") {
      task.cb = next; // [{ cb: testTask }]
    } else {
      queue.shift();
    }
  }
  // 当 task 完成 或者 应该让出执行权
  task && startTransition(flush);
}

function shouldYield() {
  return now() >= deadline || navigator.scheduling.isInputPending();
  // navigator.scheduling.isInputPending() => 检查事件队列中是否有待处理的输入事件
}
```


