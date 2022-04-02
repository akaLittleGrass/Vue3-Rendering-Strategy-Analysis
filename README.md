# Vue3 渲染策略分析

原创文章，首发于美团移动大前端月刊（内部刊）

[toc]

## 简介

&ensp;&ensp;&ensp;&ensp;前端宝藏框架 Vue 如今已经来到 3.x 的版本。Vue3 自 2018 年 12 月开始原型设计，2019 年 1 月开启 RFC 征求意见，2020 年 4 月发布 beta 测试版，并于 2020 年 9 月发布了 alpha 正式版，距今已一年有余

&ensp;&ensp;&ensp;&ensp;相对于 Vue2，Vue3 在框架性能提升的方面做出了一些新的尝试，并且提供了组合式 API、自定义的渲染 API 以及更好的 TypeScript、Tree-Shaking 的支持，更带来了 Fragment、Teleport、Suspense 等新特性，本文将给大家带来 Vue3 中渲染策略的介绍及其算法层面的分析

## Vue3 的渲染策略

### 编译时的性能优化

我们知道，Vue 组件的更新会经历这么一个过程：

<img width=650 src="https://s2.loli.net/2022/04/02/nP1r23ljyKT9XeO.png" >

patch 方法会将新老 VNode 节点进行比对，然后将根据两者的比较结果最小化地修改视图，其核心在于 diff 算法，比较两个节点的标签、属性、子节点是否一致

对于 diff 这个过程，Vue3 会对动态的节点添加 patchFlag 标记，更新视图的时候只对 patchFlag 大于 0 的节点进行 diff。对于不参与更新的元素会做静态提升，只会被创建一次，渲染的时候直接复用，免去重复创建，优化内存占用。并且绑定事件行为的时候使用监听缓存，不会再去追踪事件函数的变化

在 Vue3 中，形如<span>2333</span>这样纯粹的静态节点，由于它不会参与视图的更新，会把它提升到渲染函数体之外，这样一来这个节点只会在应用启动的时候被创建一次，后面如果需要可以直接复用，给它的 patchFlag 是-1，表示不需要对它进行 diff

patchFlag 的值由于下面 PatchFlags 枚举中描述各种特性的数值的累加

```typescript
const enum PatchFlags {
  TEXT = 1, // 动态的文本节点
  CLASS = 1 << 1, // 2 动态的 class
  STYLE = 1 << 2, // 4 动态的 style
  PROPS = 1 << 3, // 8 动态属性，不包括类名和样式
  FULL_PROPS = 1 << 4, // 16 动态 key，当 key 变化时需要完整的 diff 算法做比较
  HYDRATE_EVENTS = 1 << 5, // 32 表示带有事件监听器的节点
  STABLE_FRAGMENT = 1 << 6, // 64 一个不会改变子节点顺序的 Fragment
  KEYED_FRAGMENT = 1 << 7, // 128 带有 key 属性的 Fragment
  UNKEYED_FRAGMENT = 1 << 8, // 256 子节点没有 key 的 Fragment
  NEED_PATCH = 1 << 9, // 512 只有非props需要patch的，比如`ref`
  DYNAMIC_SLOTS = 1 << 10, // 1024 动态 solt
  HOISTED = -1, // 表示它是静态节点，它的内容永远不会改变，对于hydrate的过程中，不会需要再对其子节点进行diff
  BAIL = -2, // 一个特殊的标志，指代差异算法，表示一个节点的diff应该结束
}
```

<span>{{ msg }}</span>这样的动态节点 patchFlag 便为 1，表示它是一个动态的文本节点

<span :class="info">233</span>这样带有动态 class 的节点 patchFlag 为 2

<span :class="detail" :id="detail">{{ detail }}</span>这样同时带有动态文本、class、id 的节点 patchFlag 则为 1+2+8=11，以此类推

诸如<span @click="handleClick"></span>这样使用 v-on 绑定了函数的节点，Vue3 会生成一个内联的函数去引用 Vue 实例上的 handleClick 属性，并将这个内联函数缓存起来，后续 DOM 更新的时候会直接从缓存当中读取，相当于每次使用的都是同一个函数，不会被更新，因此这个带有事件绑定的节点就成了一个静态节点，即便我们绑定的函数是箭头函数，形如<span @click="() => foo()"></span>，其结果也是一样的

优化之后的编译结果：

<img width=850 src="https://s2.loli.net/2022/04/02/ui6XJjmt4GrVcSg.png" >

在实际的 patch 当中，会将节点的 patchFlag 与 patchFlags 枚举按位与，直接对比动态的部分，patch 的过程更加地细化

```typescript
const patchElement = (
  n1: VNode,
  n2: VNode,n
  optimized: boolean
) => {
  const el = (n2.el = n1.el!)
  let { patchFlag, dynamicChildren, dirs } = n2
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.FULL_PROPS) {
      patchProps(el, n2)
    } else {
      if (patchFlag & PatchFlags.CLASS) {
        if (oldProps.class !== newProps.class) {
          hostPatchProp(el, 'class', null, newProps.class)
        }
      }
      if (patchFlag & PatchFlags.STYLE) {
        hostPatchProp(el, 'style', null, newProps.class)
      }
      if (patchFlag & PatchFlags.PROPS) {
        //...
        hostPatchProp(el, key, prev, next)
        //...
      }
    }
    if (patchFlag & PatchFlags.TEXT) {
      if (n1.children !== n2.children) {
        hostSetElementText(el, n2.children as string)
      }
    }
  } else if (!optimized && dynamicChildren == null) {
    patchProps(el,n2)
  }
  //.....
}
```

### DOM Diff 算法

接下来我们看看 Vue3 的 DOM Diff 具体是怎么做的

所谓 DOM diff，无非就是递归地去对比两棵树，寻找当中不同的子节点

<img width=700 src="https://s2.loli.net/2022/04/02/DGPafIdyLheSqNv.png" >

在 Vue3 当中，对比子节点的方法是这样的

```javascript
const patchChildren: PatchChildrenFn = (
  n1,
  n2
) => {
  const c1 = n1 && n1.children
  const c2 = n2.children
  const { patchFlag } = n2
  if (patchFlag > 0) { // 说明是动态节点
    if (patchFlag & PatchFlags.KEYED_FRAGMENT) { // 有key
      patchKeyedChildren(
        c1 as VNode[],
        c2 as VNodeArrayChildren,
      )
      return
    } else if (patchFlag & PatchFlags.UNKEYED_FRAGMENT) { // 无key
      patchUnkeyedChildren(
        c1 as VNode[],
        c2 as VNodeArrayChildren
      )
      return
    }
  }
}
```

我们直接来看看它具体干了什么

首先，对于 patchFlag>0 的动态节点，如果没有 key，直接暴力一一比对，直接调用 patch 用新节点更新旧节点，相当于不 diff（可见 key 之重要）

<img width=750 src="https://s2.loli.net/2022/04/02/aYVvnfLSmezjchQ.png" >

在有 key 的情况下，分为以下这么几个步骤：

#### 1.从头部开始比对

我们用一个 while 循环从新旧节点序列的头部开始一一比对，遇到不同的节点退出循环，在这个过程中用 i 代表当前对比位置的下标，e1、e2 代表两个序列尾部的下标，c2、c1 分别为新旧的节点序列

代码描述是这样的:

```javascript
while (i <= e1 && i <= e2) {
  const n1 = c1[i];
  const n2 = c2[i];
  if (isSameVNodeType(n1, n2)) {
    patch(n1, n2);
  } else break;
  i++;
}
```

<img width=500 src="https://s2.loli.net/2022/04/02/BUnNaedKvJcoODj.png" >

#### 2.从尾部开始比对

同样地，我们再从两个序列的尾部用 while 循环一一比对，遇到不同的节点退出循环

<img width=500 src="https://s2.loli.net/2022/04/02/LhF2s1blGRPBzdy.png" >

代码描述:

```javascript
while (i <= e1 && i <= e2) {
  const n1 = c1[e1];
  const n2 = c2[e2];
  if (isSameVNodeType(n1, n2)) {
    patch(n1, n2);
  } else break;
  e1--;
  e2--;
}
```

#### 3.同序列挂载（新的比旧的多）

假设 DOM 变更的情况是在第二个节点的后面新增了一个节点，那么经过这两步之后，我们得到了这样一个结果：

<img width=500 src="https://s2.loli.net/2022/04/02/rVNJEHohuBmTMKj.png" >

显然，在这种情况下，我们只需要把新增的 f 节点在相应的位置挂载上去就行了

代码：

```javascript
if (i > e1) {
  if (i <= e2) {
    while (i <= e2) {
      patch(null, c2[i]);
      i++;
    }
  }
}
```

#### 4.同序列卸载（旧的比新的多）

我们把上一步的假设反过来，如果是旧的子节点序列比新的多了一个节点，那么类似地，只要把旧序列中多出的节点卸载掉就可以了

<img width=460 src="https://s2.loli.net/2022/04/02/uAMBdCbJU3gtpIV.png" >

```javascript
else if (i > e2) {
  while (i <= e1) {
    unmount(c1[i])
    i++
  }
}
```

以上两种属于比较简单的情况，即在连续的区域内新增或删除节点。在这两种情况中我们从新旧两个子节点序列的两端分别向中间遍历对比，遇到不同的节点停止对比，新旧序列中总会有一个序列被遍历完，另一个序列中未遍历到的节点就是多出来的

下面我们来看一种比较复杂的情况，新旧序列都没有遍历完

#### 4.同序列卸载（旧的比新的多）

假设当对比停止时，新旧序列都没有遍历完，我们把未遍历到的部分称为未知序列，如下图所示：

<img width=600 src="https://s2.loli.net/2022/04/02/iVM7wWID9lfxquS.png" >

我们可以发现新节点序列相比于旧节点序列，c、d、e 三个节点更换了位置，e 来到了 c 的前面，q 换成了 f

我们可以把旧序列中 c、d、e、q 全都卸载，然后在相应的位置上把 e、c、d、f 四个节点依次挂载，也能如预期地更新视图，但这样显然不是最优的方案，因为 c、d、e 三个节点都是可以复用的，我们把 e 移动到 c 的前面，把 q 卸载掉并挂载 f 即可完成视图的更新，那么我们怎么把这个更高效的方案抽象出来呢，这里我们需要用到一种数据结构——最长递增子序列

最长递增子序列要求数值要依次递增，并且长度尽可能的大，例如在数组[0, 1, 0, 3, 2]中，最长递增子序列是[0, 1, 2]

具体到我们的问题当中，我们需要的是新序列中的节点在旧序列中下标的最长递增子序列，在该子序列中的节点保持不动，其他节点执行移动或挂载操作

我们把不存在的节点的下标定为-1，那么新序列中的 e、c、d、f 四个节点在旧序列中的下标就分别为[4, 2, 3, -1]，显然最长递增子序列为[2, 3]，说明 c、d 两个节点的位置保持不变，其余节点 e 需要移动，q 需要卸载并挂载 f

<img width=600 src="https://s2.loli.net/2022/04/02/zGLvpN7wxSl2X5i.png" >

```javscript
const map = {e:2, c:3, d:4, h:5} // 新节点序列中，未知序列的节点与其下标的映射
const newIndexToOldIndexMap= [0, 0, 0, 0]
for (i = s1; i <= e1; i++) {
      const prevChild = c1[i]
      newIndex = map.get(prevChild.key)
      newIndexToOldIndexMap[newIndex - s2] = i + 1
}
```

在这段逻辑中，我们先去遍历旧的节点序列，找到新未知序列的节点在旧序列中的下标，e、c、d、f 四个节点在旧序列中的下标分别为 4、2、3、-1（-1 表示不存在），将这些下标+1，得到数组[5, 3, 4, 0]，在这个数组中，最长递增子序列为[3, 4]，这两个元素在数组中的下标分别为 1、2，随即构建数组[1, 2]， 接着我们从后向前遍历[5, 3, 4, 0]这个数组，会遇到三种情况：

1.若当前值为 0，代表是新的节点，执行插入操作

2.当前索引在数组[1, 2]中，则不需要移动

3.将该旧节点移动到新节点所在的位置

至此，整个 Vue3 的 DOM Diff 过程就完成了

## Vue3 对比 Vue2

Vue3 中的 diff 算法涉及到复杂度为 O(NlogN)的最长递增子序列的求解，其余的遍历操作需要 O(N)，基于最长递增子序列修改 DOM 的操作亦为 O(N)，所以这整一个 diff 算法的复杂度为 O(NlogN)

在 Vue2 中，我们会对新旧的节点序列进行头 ↔ 头、尾 ↔ 尾、头 ↔ 尾、尾 ↔ 头的比较，最多需要对所有节点进行一次遍历，时间复杂度为 O(N)

接着构造旧节点的 key 映射到 index 的 map，时间复杂度为 O(N)，对每个新的头节点查找是否有对应的旧节点，时间复杂度为 O(1)，最多需要对 N 个节点进行这样的判断，时间复杂度为 O(N)

对于新节点或旧节点用完的情况，创建所有剩余新节点或删除所有剩余旧节点的时间复杂度也为 O(N)

在 Vue2 中，diff 算法总的时间复杂度为 O(N)，为什么换代之后的 Vue3 diff 算法看起来时间花费更大了呢？

其实 Vue3 追求的是尽量少的 DOM 移动次数，因为真实的 DOM 节点是非常庞大的，节点移动的性能开支其实比起一个数组问题的求解要高得多

我们来看一个例子：

<img width=600 src="https://s2.loli.net/2022/04/02/1ZQedWDnyMcGohX.png" >

用 Vue2 的策略处理这两个序列

a↔d、g↔c、a↔c、g↔d 四组节点相互比较均无一致，然后去查找 d，将之前移，

a↔e、g↔c、a↔c、g↔e 四组节点比较均无一致，去查找 e，将之前移

a↔f、g↔c、a↔c、g↔f 四组节点比较无一致，去查找 f，将之前移

最后 a↔g、g↔c、a↔c、g↔g 四组节点中 g↔g 一致，将 g 前移，至此新旧节点序列一致，共需要移动四次

<img width=750 src="https://s2.loli.net/2022/04/02/9j6t1QuNvGh54W3.png" >

而在 Vue3 中，我们求解新节点在旧节点序列中下标数组[4, 5, 6, 7, 1, 2, 3]的最长递增子序列，得到[4, 5, 6, 7]，由此得出 d、e、f、g 四个节点不需要移动，只需对 a、b、c 节点移动三次，节省了一次 DOM 操作的开支

## 总结

的来看，Vue3 相对于 Vue2，在编译和 DOM Diff 等方面对框架性能的提升做出了新的探索和努力。在大型应用中，标记动态元素的 patchFlag、将 DOM 节点复用的静态提升、类似 React useMemo 的事件缓存等特性带来的收益是很明显的，并且服务端渲染的性能也迈上了一个新的台阶。其实除了以上介绍的这些，Vue3 在 DOM 挂载、异步渲染、打包构建等其他方面也带来了许多有意思的新特性，大家可以在工程开发中多多探索和尝试，欢迎将心得收获再与我分享讨论

## 参考资料

[1] 尤雨溪 - Vue.js 3.0 Beta 分享

[2] Vue3 源码：https://github.com/vuejs/vue-next
