---
title: Vue 源码学习- DOM 更新
date: 2017-04-28 11:33:23
tags:
---

上一篇[文章](/post/Vue-源码学习-响应式原理/)介绍了 Vue 是如何实现响应式的。响应式带来的最大好处就是自动地更新 DOM，无需手动操作 DOM。所有状态都在 Vue 实例中保存。

而提到更新 DOM，我们自然会想到 react 提出的虚拟 DOM。也就是在 js 中记录 DOM 的状态。当虚拟 DOM 接收到更新请求时，会做一些优化处理，再改变 DOM。由于 DOM 操作是很慢的，所以引入虚拟 DOM 带来的开支还是可以接受的。

Vue 1.0 中是没有虚拟 DOM 的，2.0 版本中引入了这一功能，在 Vue 中，虚拟 DOM 的构造类为 VNode。

### VNode

上一篇文章中提到，更新 DOM 执行的是这样一个 Watcher

```
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

而 `Vue.prototype._render` 的返回值正是 VNode。

核心代码为 `vnode = render.call(vm._renderProxy, vm.$createElement)`
其中 `render` 就是通过 `template` 编译成的渲染函数，也就是 jsx 写法中的 `render`，`vm._renderProxy` 其实就是 `vm` 本身，`vm.$createElement` 是在 `initRender` 时定义在 `vm` 上的函数。因此，关键在于 `vm.$createElement` 的内容

#### `createElement`

代码在 `src/core/vdom/create-element.js` 中

可以看到这里进行了各种各样的 `if else` 判断。大概的逻辑是：

- 不合法的话返回 emptyVNode
- 必要的调用 `createComponent` 创建组件
- 返回 VNode

VNode 的构造函数很简单，基本就是一个大对象。所以我们重点看看 `createComponent`

#### `createComponent`

`createComponent` 这个函数主要是用于创建组件实例对应的 VNode。

在这个函数中主要实现以下几个功能：

- 调用 `Vue.extend` 生成组件构造函数
- 根据 data 的内容生成新的 `vm` 实例构造函数，在这一步最关键的是将 props 还有 on 事件定义好。（注意，这里并没有执行初始化的过程，只是生成了实例的构造函数）
- 返回在父组件中用于占位的 VNode

在更进一步地了解之后，我们可以知道其实，**每一个 VNode 对应 DOM 中的一个标签**，当然具体内容会待会细说。

### patch

生成了 VNode 之后，需要调用 `vm._update` 将 VNode 更新到 DOM 上。

在 `lifecycleMixin` 中，`vm_update` 调用 `vm.__patch__` 输入 VNode 并且返回 DOM。
在经过查找之后，发现关键代码在 `src/core/vdom/patch.js` 中

这个文件的代码有 600 行。。比较复杂，对应不同的情况会有不同的处理措施。只看看几个关键的函数

#### function patch

- 调用 `invokeDestroyHook` 删除 VNode 和 DOM
- 调用 `patchVnode` 对 VNode 进行 patch 并且更新 DOM
- 调用 `createElm` 生成 DOM 并且挂载到指定元素上

#### function createElm

这个函数主要执行生成 DOM 的工作， 关键函数为：

- `createElement` 生成 DOM（其实就是调用了浏览器的 `document.createElement`）
- `createComponent`，如果 Vnode 里是一个组件的话，会执行之前生成的实例的构造函数，进行组件的 `init` 过程，最后当然也是生成 DOM
- `createChildren`，对   `children` 数组中的 VNode 循环调用 `createElm` 生成 DOM（这一步也是 Vue DOM 更新的关键，正是有这一步，Vue 才能将整个 DOM 树给**递归**生成）

#### function patchVnode

有不同的策略：

- 静态 Vnode 复用之前生成好的 DOM，返回
- 对 VNode 对应的 vm 进行 `prepatch`（这一步很关键，就是在这一步，props 得以在组件树中进行传递，触发子组件的 Watcher），执行 `updateChildren`，返回
- 执行 `updateChildren` 返回

`patchVnode` 以及 `updateChildren` 中有一些 diff 更新策略，此处不讲

### 简单例子

用一个简单的[例子](http://codepen.io/pengchongfu/pen/ybMpew)说明一下 DOM 的生成流程

```
<div id="app">
</div>
```

```
Vue.component("com", {
  render(h) {
    return h("span", {
      domProps: {
        innerHTML: "this is com"
      }
    });
  }
});

console.log("extend component");

new Vue({
  el: "#app",
  render(h) {
    return h("div", {}, [h("h1", "this is h1"), h("com")]);
  }
});

```

假设我们在组件 `_render` `_update` 开始和结束时以及创建 VNode 时都加上标记，输出结果会是怎么样的？

输出结果为

```
new VNode: tag is                       // **不知道为啥加载 Vue 时会生成 VNode**
extend component                        // ---
_render start                           // 根实例开始执行 `_render`
new VNode: tag is undefined             // **不知道这是为什么生成的**
new VNode: tag is h1                    // 根实例生成 `children` 数组中的第一个 VNode，h1
new VNode: tag is vue-component-1-com   // 根实例生成 `children` 数组中的第二个 VNode，组件 com
new VNode: tag is div                   // 根实例生成自身 VNode， div
_render end                             // 根实例 `_render` 结束
_update start div                       // 根实例开始执行 `_update`
new VNode: tag is div                   // 清空 html 中 #app div，并且创建空的 div 代替
_render start                           // com 实例开始执行 `_render`
new VNode: tag is span                  // com 实例生成自身 VNode， span（因为 com 实例没有 children）
_render end                             // com 实例 `_render` 结束
_update start span                      // com 实例开始执行 `_update`
_update end span                        // com 实例 `_update` 结束
_update end div                         // 根实例 `_update` 结束
```

### 总结

以上就是 Vue 实现 DOM 更新的一些要点，当然还有很多细节没有提及。

其中很值得看的就是虚拟 DOM patch 的部分。合理的 patch 能够大大减少操作 DOM 浪费的时间。

当然组件间信息传递部分的实现也是十分重要的。这会在下一篇文章中介绍。