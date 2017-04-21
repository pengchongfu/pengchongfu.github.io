---
title: Vue 源码学习-概览
date: 2017-04-21 17:21:15
tags:
- Vue
- 前端
---

话说从开始写前端到现在也已经一年了，虽然每天写着业务代码，可是对 Vue 的核心还是不太了解。

只能在写出 bug 时有那么一点点的感悟，因此决定学学 Vue 的源码，希冀能获取一些人生的经验。

### 项目概览

**Vue 版本为2.2.6之后的 dev 分支*

文件夹 | 说明
---- | ----
.github | github 相关说明
benchmarks | 一些实例
build | build 配置文件
dist | build 产生的文件
examples | 一些例子
flow | flow 文件
packages | 不同平台的 compiler 文件
src | 源码核心文件
test | 测试文件
types | 类型定义

可以看出最核心的代码应该是 `src` 文件夹内的代码。而 Vue 初始化一个 app 的方法为 `new Vue()`，因此首先得找出 `Vue` 函数的定义位置。
为了简化阅读源码的难度，直接选择 web-runtime-esm 版本（ compiler 可以熟悉 runtime 之后再学习）

因此我们从 build 命令开始一步一步地寻找：

1. package.json `"build": "node build/build.js"`
2. build/build.js 读取 `./config` 内容循环 build
3. build/config.js 可以看到 web-runtime-esm 的入口文件为 `web/runtime.js`
4. src/platforms/web/runtime.js 从 `./runtime/index` 导入
5. src/platforms/web/runtime/index.js 从 `core/index` 导入，同时在 Vue 原型链上注册 `__patch__` 和 `$mount` 函数
6. src/core/index.js 从 `./instance/index` 导入，同时 `initGlobalAPI`
7. src/core/instance/index.js 这里定义了 Vue 函数，并且 Vue 函数执行了 `this._init(options)`，所以这是初始化的关键。

先跳过 Vue 实例初始化的过程，接下来我们对后边的几个函数进行详细的分析。

### Vue 函数初始化流程

#### initMixin

这个函数只做了一件事，那就是向 Vue 的原型链上注册了 `_init` 函数，而在 Vue 实例 init 的过程中，分别执行了

- initLifecycle
- initEvents
- initRender
- initInjections
- initState
- initProvide

并且在这个过程中执行了 `beforeCreate` 和 `created` 的回调，并且最后执行了 `$mount` 函数将实例挂载到 DOM 上。
关于 Vue 实例初始化的过程待会再进行学习，先继续分析 Vue 函数的初始化

#### stateMixin

首先定义了 Vue 原型链上的 `$data` 和 `$props`，并设置 get 和 set 函数。

接着定义了 `$set` 和 `$del`。这两个方法主要是自己定义一些响应式的对象。

最后是 `$watch` 函数，提供了动态创建 Watcher 的方法

#### eventsMixin

主要是往 Vue 的原型链上注册了 `$on` `$once` `off` `emit` 函数。这几个函数与 Vue 里边的指令关系十分密切。

#### lifecycleMixin

注册了 `_update` `$forceUpdate` `$destroy` 三个函数。

其中 `_update` 函数十分重要，是接收 VNode（也就是虚拟 DOM）更新 DOM 的重要部分。
而 `_update` 函数中除了执行 `_beforeUpdate` 回调以外， 最主要的功能是通过 `__patch__` 更新 DOM。

`$forceUpdate` 函数执行了 `vm._watcher.update()`，从其他代码可知，组件实例上的 `_watcher` 是一个专门用于更新 DOM 的 Watcher。

`$destroy` 函数执行了 `beforeDestroy` 和 `destroyed` 回调。并且从父实例中移除了自己，销毁了实例上的 Watcher，更新了 DOM。

#### renderMixin

同样是往原型链上添加一些函数。

`$nextTick` 这个函数可以看成是在当前事件循环之后执行回调的一个函数，也就是 `setTimeout(callback, 0)`

`_render` 这个函数同样是很关键的一个函数，通过 Vue 实例生成 VNode。

到这里 Vue 函数的初始化过程就基本完成了，可以看出基本是在 Vue 原型链上注册一些函数，而这些函数会在 Vue 实例的生命周期中用到。

那 `__patch__` 和 `$mount` 为什么不在这个过程中呢？主要是因为 Vue 会有不同平台（web 和 weex）以及不同的版本（runtime 或者 compiler）。

### Vue 实例初始化流程

#### initLifecycle

在实例上定义一些变量，如 `$parent` `$root` `children`，一些标识符也在这里定义 `_isMounted` `_isDestroyed`。同时 `_watcher` 也被声明。

#### initEvents

貌似是从父组件继承一些 Listener。

#### initRender

定义 `$createElement` 函数

#### initInjections

用得比较少，略过

#### initState

这里是 Vue 响应式的核心部分。核心代码都在 `src/core/instance/state.js` 中

#### initProvide

略过

#### $mount

这是和 DOM 更新息息相关的部分。初始化过程中，得通过这个函数挂载到 DOM 上，而核心代码为 `src/core/instance/lifecycle.js` 的 `mountComponent` 函数。

在这个函数里边使得 `vm._watcher` 等于一个 Watcher，而这个 Watcher 会在必要的时候执行，调用更新函数 `vm._update(vm._render(), hydrating)` 以更新 DOM。

### 总结

以上就是 Vue 的一个大致概览。这么看下来 Vue 源码的这几个部分需要细细阅读：

1. 数据监听部分
2. Dom 更新部分
3. 组件之间的交互部分
