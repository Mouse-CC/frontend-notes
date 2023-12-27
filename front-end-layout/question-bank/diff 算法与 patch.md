## 说明

- Diff 算法：一种对比新、旧虚拟节点的算法
- Patch：打补丁，把新旧节点的差异，应用到 DOM 上

数据更新时，渲染得到新的 Virtual DOM，与上一次得到的 Virtual DOM 进行 diff，记录下所有需要在 DOM 上进行的变更，然后在 patch 过程中应用到 DOM 上，实现 UI 的同步更新。

核心代码：`src/core/vdom/patch.ts`

## Virtual DOM

可以将其看作一棵模拟了 DOM 树的 JavaScript 树，用 JS 对象表示 DOM 结构，可以根据虚拟 DOM 树构建出真实的 DOM 树

比如 DOM：

```html
<div>
  <p>123</p>
</div>
```

Virtual DOM（伪代码）：

```js
var Vnode = {
  tag: "div",
  children: [{ tag: "p", text: "123" }]
}
```

Vue 的 diff 算法是基于 snabbdom 改造而来，复杂度为 O(n)

特点为：递归进行同级 vnode 的 diff，最终实现整个 DOM 树的更新

```html
<!-- 之前 -->
<div>
  <p>
	<b> aoy </b>
	<span> diff </span>
  </p>
</div>
```

移动 p 标签内部的 span

```html
<!-- 之后 -->
<div>                    <!-- 层级 1 -->
  <p>                    <!-- 层级 2 -->
	<b> aoy </b>         <!-- 层级 3 -->
  </p>
  <span> diff </span>    <!-- 层级 2 -->
</div>
```

我们可能期望将 span 直接移动到 p 的下面，这是最优操作，但实际上 diff 操作是移除 p 里面的 span，再创建一个新的 span 插到 p 的后边。因为新加的 span 在层级 2，旧的在层级 3，属于不用层级的比较。所以 diff 时做了删除 + 新增操作，你可能回想这样性能上来说不是最好的，但在比较跨层移动的开支后，对框架来说，支出太大了（耗时长 - diff 算法是须啊要频繁调用的，如果处理时间过长，用户会觉得卡顿 -> 觉得框架性能不好）。

>[!quote] 
>	源码路径：src/core/vdom/
>	主要文件：此目录下 vnode.ts

以上对于 diff 算法的初步理解，在写代码时，可以优化的的点
-  尽量避免跨层次结构的 DOM 修改
- 

## Diff 算法流程

代码主要在 `function patch(oldVnode, vnode, hydrating, removeOnly)` 函数中

### 片段 1
```js
/**如果 vnode 不存在，但是 oldVnode 存在，说明是需要销毁旧节点
 **/

if (isUndef(vnode)) {
  if (isDef(oldVnode)) invokeDestroyHook(oldVnode);
  return;
}
```

### 片段 2
```js
/**如果 vnode 存在，但是 oldVnode 不存在，说明是需要创建新节点
 **/
if (isUndef(oldVnode)) {
// oldVnode 不存在 vnode 存在，createElm
// empty mount (likely as component), create new root element
  isInitialPatch = true
  createElm(vnode, insertedVnodeQueue)
}
```

### 片段 3
当 `vnode` 和 `oldVnode` 都存在
- 当 `oldVnode` 不是真实节点，或 `vnode` 和 `oldVnode` 通过 `sameVnode()` 对比两者不是一类，则调用 `patchVnode` 进行 patch，修改现有的节点

```js
/**当 oldVnode 不是真实节点，并且 vnode 和 oldVnode 值得比较时，则调用 patchVnode 进行 patch，即直接修改现有的节点
 **/
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // patch existing root node
    patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly);
}
```

- 若 `oldVnode` 是真实节点，或 `vnode` 和 `oldVnode` 并不一样，则存储常量： `const parentElm = nodeOps.parentNode(oldElm)` ，根据 `vnode` 创建一个真实的 DOM 节点，并插入到 `parentElm` 上。

```js
const oldElm = oldVnode.elm;
const parentElm = nodeOps.parentNode(oldElm);

// create new node
createElm(
    vnode,
    insertedVnodeQueue,
    // extremely rare edge case: do not insert if old element is in a
    // leaving transition. Only happens when combining transition +
    // keep-alive + HOCs. (#4590)
    oldElm._leaveCb ? null : parentElm,
    nodeOps.nextSibling(oldElm)
);
```

### 最终函数返回 `return vnode.elm；`

patch 中的对 node 做 diff 的入口，片段 3 `patchVnode()` ：

```js
/**如果 oldVnode 和 vnode 完全一致，则可认为没有变化，return；
 **/
if (oldVnode === vnode) {
    return;
}
```

```js
/**
 * vnode.elm  表示当前虚拟节点对应的真实dom节点的引用
 * vnode,oldVnode指向同一个真实 DOM 的引用
 **/
const elm = (vnode.elm = oldVnode.elm);
```

```js
/**如果新旧 vnode 都是静态的，同时它们的 key 相同（代表同一节点），并且新的 vnode 是 clone 或者是标记了 once（标记 v-once 属性，只渲染一次），那么只需要替换 elm 以及 componentInstance 即可。
 **/
if (
    isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
) {
    vnode.componentInstance = oldVnode.componentInstance;
    return;
}
```

```js
/**如果 oldNode,vnode 结点均有 children 子节点，则对子节点进行 diff 操作，调用 updateChildren 更新子节点
 **/
updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly);
```

```js
/**如果只有 vnode 节点存在子节点，那么先清空 elm 的文本内容，然后为当前节点加入子节点
 **/
addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
```

```js
/**如果只有 oldVnode 节点有子节点的时候，则移除所有 elm 的子节点
 **/
removeVnodes(elm, oldCh, 0, oldCh.length - 1);
```

```js
/**如果 vnode 节点没有 text 文本,但是与 oldVode 节点 text 不一样时，直接替换这段文本
 **/
nodeOps.setTextContent(elm, vnode.text);
```

### 重点看 `updateChildren` 函数：

如何更新子节点

```js
/**
 *  oldCh:oldVNode chirdren
 *  newCh:newVNode chirdren
 *  StartIdx   开头索引
 *  EndIdx     结束索引
 *  vnode中的key，可以作为节点的标识
 * **/
```

1. OldCh 和 newCh 各有两个头尾的变量 StartIdx 和 EndIdx，它们的 2 个变量相互比较，一共有 4 种比较方式
2. 如果 4 种比较都没匹配，就会用 key 进行比较：拿 newStartVnode 的 key 值在 oldCh 中遍历，如果存在相同 key 的节点并且与 newStartVnode 是相同节点，则移动 Dom 节点，否则创建新元素
3. 在比较的过程中，变量会往中间靠，一旦 `StartIdx` > `EndIdx` 表明 oldCh 和 newCh 至少有一个已经遍历完了，就会结束比较

代码：

```js
let oldStartIdx = 0;                     // 老，起始节点 定位
let newStartIdx = 0;                     // 新，起始节点 定位
let oldEndIdx = oldCh.length - 1;        // 老，末尾节点 定位
let oldStartVnode = oldCh[0];
let oldEndVnode = oldCh[oldEndIdx];
let newEndIdx = newCh.length - 1;        // 新，末尾节点 定位
let newStartVnode = newCh[0];
let newEndVnode = newCh[newEndIdx];
let oldKeyToIdx, idxInOld, vnodeToMove, refElm;
```

前提：
`while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx)` 时遍历的过程
==在 oldCh 起始索引没有移动到末尾并且 newCh 起始索引没有移动到末尾== 成立时循环一直执行。

索引图：

![[Pasted image 20231222095351.png]]

#### 1.1

如果 `oldStartVnode` (老的起始节点不存在)，如果 `oldStartVnode` (老的起始节点存在)
`oldEndVnode` (老的末尾节点不存在)，移动索引往中间靠拢。

```js
//2种情况，只移动索引

if (isUndef(oldStartVnode)) {
    oldStartVnode = oldCh[++oldStartIdx]; // Vnode has been moved left
} else if (isUndef(oldEndVnode)) {
    oldEndVnode = oldCh[--oldEndIdx];
}
```

#### 1.2 短路优化 - (针对四种情况)

Old 和 newCH 各自的头和尾的变量 `oldStartIdx` | `oldEndIdx` | `newStartIdx` | `newEndIdx` ，选两个相互比较，一共有以下四种比较方式，索引在比较过程中不断向中间靠拢。

```js

else if (sameVnode(oldStartVnode, newStartVnode)) {
    //不需要对dom进行移动
    patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
    oldStartVnode = oldCh[++oldStartIdx];
    newStartVnode = newCh[++newStartIdx];
} else if (sameVnode(oldEndVnode, newEndVnode)) {
    //不需要对dom进行移动
    patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
    oldEndVnode = oldCh[--oldEndIdx];
    newEndVnode = newCh[--newEndIdx];
}
```

如图

##### 老数组的开头与新数组的开头

新数组末尾有剩余节点则添加
![[Pasted image 20231222103724.png]]

从左往右对比，老数组的下标先到底了 (oldStartIdx 等于 oldEndIdx)，新数组还没有，则创建剩下没对比的节点 (对真实 DOM 添加)

老数组末尾有剩余节点则删除
![[Pasted image 20231222104543.png]]

从左往右对比，新数组的下标先到底了 (newStartIdx 等于 newEndIdx)，老数组还没有，则删除老数组剩下没对比的节点 (对真实 DOM 删除)

##### 老数组的末尾与新数组的末尾

新数组的开头有剩余节点则添加
![[Pasted image 20231222105755.png]]

从右往左对比完，老数组的下标先到头了 (oldEndIdx 等于 oldStartIdx)，新数组还没有，则创建剩下没对比的节点 (对真实 DOM 添加)

老数组的开头节点还有剩余则删除
![[Pasted image 20231222110132.png]]

从右往左对比完，新数组的下标先到头了 (newEndIdx 等于 newStartIdx)，老数组还没有，则删除老数组开头没有比对的节点 (对真实 DOM 删除)

```js
else if (sameVnode(oldStartVnode, newEndVnode)) {
    /** 当 oldStartVnode，newEndVnode 是相同节点，说明 oldStartVnode.el 跑到oldEndVnode.el 的后边了。
     **/
    patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
    canMove &&
        nodeOps.insertBefore(
            parentElm,
            oldStartVnode.elm,
            nodeOps.nextSibling(oldEndVnode.elm)
        );
    oldStartVnode = oldCh[++oldStartIdx];
    newEndVnode = newCh[--newEndIdx];
}
```

##### 老数组的开头与新数组的结尾
![[Pasted image 20231222110551.png]]

如果老数组的开头节点与新数组的结尾节点对比成功，则继续递归对比 (oldStartVnode 向右 → ，newEndVnode  ← 向左 ) 对比结束后将上图 A 节点移动到末尾。

```js
else if (sameVnode(oldEndVnode, newStartVnode)) {
	/** 当 oldEndVnode，newStartVnode 是相同节点，说明 oldEndVnode.el 跑到 oldStartVnode.el 的前面了。
	**/
	patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
	canMove &&
        nodeOps.insertBefore(
            parentElm,
            oldEndVnode.elm,
            nodeOps.nextSibling(oldStartVnode.elm)
        );
    oldEndVnode = oldCh[--oldEndIdx];
    newStartVnode = newCh[++newEndIdx];
}
```

##### 老数组的末尾与新数组的开头
![[Pasted image 20231222111321.png]]

如果老数组的末尾节点与新数组的开头节点对比成功，则继续递归对比 (oldEndVnode ← 向左，newStartVnode 向右 →) 对比结束后将上图 D 节点移动到开头

#### 1.3 以上四种情况都没对比成功

开始用 key 进行比较

```js
if (isUndef(oldKeyToIdx))
    /**
     * createKeyToOldIdx 生成一个 key 与 oldVnode 的索引对应的 map，只有第一次进来 undefined 的时候会生成
     **/
    oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
/**如果 newStartVnode 存在 key，返回这个 key 在 oldVnode 中的索引值（即第几个节点，下标），否则调用 findIdxInOld 方法查找 key 在 oldVnode 中的索引值
 **/
idxInOld = isDef(newStartVnode.key)
    ? oldKeyToIdx[newStartVnode.key]
    : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
    
if (isUndef(idxInOld)) {
    //newStartVnode没有key或者是该key没有在老节点中找到，则创建一个新的节点
    createElm(
        newStartVnode,
        insertedVnodeQueue,
        parentElm,
        oldStartVnode.elm,
        false,
        newCh,
        newStartIdx
    );
} else {
    /**获取相同 key 的老节点**/
    vnodeToMove = oldCh[idxInOld];
    if (sameVnode(vnodeToMove, newStartVnode)) {
        //相同 key 值的 vnodeToMove 与 newStartVnode 是同一个节点，将vnodeToMove.elm 插入到oldStartVnode.elm前面
        patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue);
        oldCh[idxInOld] = undefined;
        canMove &&
            nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
    } else {
        // same key but different element. treat as new element
        createElm(
            newStartVnode,
            insertedVnodeQueue,
            parentElm,
            oldStartVnode.elm,
            false,
            newCh,
            newStartIdx
        );
    }
}

newStartVnode = newCh[++newStartIdx];
```

1. 当 `oldStartIdx > oldEndIdx` 时，表示 oldCh 已经先遍历结束，newCh 还没有遍历完，则 `newStartIdx = oldStartIdx` (同一个下标)，到 `newEndIdx` 这部分是新增的添加节点 (对真实 DOM 添加)

```js
if (oldStartIdx > oldEndIdx) {
    //添加新节点
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
    addVnodes(
        parentElm,
        refElm,
        newCh,
        newStartIdx,
        newEndIdx,
        insertedVnodeQueue
    );
}
```

如图：
![[Pasted image 20231222112832.png]]

2. 当 `newStartIdx > newEndIdx` 时，表示 newCh 已经先遍历结束，oldCh 还没有遍历完，则 `oldStartIdx = newStartIdx` (同一个下标)，到 `oldEndIdx` 这部分是不包含在新节点数组内的，此时应该删除这部分内容 (对真实 DOM 删除)

```js
else if (newStartIdx > newEndIdx) {
    //删除老节点
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
}
```

如图：
![[Pasted image 20231222113321.png]]

## 自测

引用一个例子，画出 diff 完整完成过程，每一步 dom 的变化都用不同颜色的线画出

题：`a, b, c, d, e` 5 个不同的元素，没设置 key 

![[Pasted image 20231222113637.png]]

解释下过程，从左往右对比，b 在四种情况下都不匹配，走创建。 e 在四种情况下都不匹配，走创建。d 在(新数组的开头与老数组的结尾) 中对比成功，位移。位移后 c 是 newCh 和 oldCH 的最后一个元素。多余的 a，b 删除。
(创建时 dom ：`b e a b c d` ，newCh ：`b e d c` ，此时 newStartIdx 在 d `newStartVnode` 与 `oldEndVnode` oldCh 末尾元素相等，dom 位移到对应位置 => `b e d a b c` ，oldStartIdx 下标目前仍然是 0，`oldEndVnode` 指向 c , `newStartVnode` 指向 c ，同上再做位移 => `b e d c a b` ，且 newStartIdx = newEndIdx 了,  `oldEndVnode` 指向 b ，oldCh 从右往左，b 和 a 分别对比 `newEndVnode` 都不成立，删除 a，b => `b e d c` )

==当我们给元素加上 key== ，b 就被复用了

![[Pasted image 20231222134152.png]]

