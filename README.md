# Vue3 渲染策略分析

原创文章，首发于美团移动大前端月刊（内部刊）

## 简介

前端通道的宝藏框架 Vue 如今已经来到 3.x 的版本。Vue3 自 2018 年 12 月开始原型设计，2019 年 1 月开启 RFC 征求意见，2020 年 4 月发布 beta 测试版，并于 2020 年 9 月发布了 alpha 正式版，距今已一年有余。相对于 Vue2，Vue3 在框架性能提升的方面做出了一些新的尝试，并且提供了组合式 API、自定义的渲染 API 以及更好的 TypeScript、Tree-Shaking 的支持，更带来了 Fragment、Teleport、Suspense 等新特性，本文将给大家带来 Vue3 中渲染策略的介绍及其算法层面的分析

## Vue3 的渲染策略

### 编译时的性能优化

我们知道，Vue 组件的更新会经历这么一个过程：

<img width=500 src="https://s2.loli.net/2022/04/02/nP1r23ljyKT9XeO.png" >

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
