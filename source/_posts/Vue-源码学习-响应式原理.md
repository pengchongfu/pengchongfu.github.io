---
title: Vue 源码学习-响应式原理
date: 2017-04-26 22:07:49
tags:
- Vue
- 前端
---

上一篇[文章](/post/Vue-源码学习-概览/)大致对 Vue 的整体进行了一个介绍。

Vue 作为一个前端mvvm框架最出色的地方就是对数据做了监听。当数据变化时 Vue 会自动执行相应的函数，并且更新 DOM。

免除了 jQuery 时代手动更改 DOM 的麻烦。这也是 Vue 得以在其他平台上执行的原因（weex）。

正如上一篇文章所说，Vue 的数据监听实现主要是在 `src/core/instance/state.js` 中，下面就好好看一看是如何实现数据监听的。

### 基础知识

核心为 [`Object.defineProperty()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 函数，这里就不做详细介绍了，总而言之就是当我对一个对象的某个键取值或者赋值的时候，能够执行相应的函数。
并且取值时函数的返回值为取到的值。

```
var ob = {}
Object.defineProperty(ob, 'num', {
  enumerable: true,
  configurable: true,
  get () {
    console.log('get')
    return 10
  },
  set () {
    console.log('set')
  }
})

console.log(ob.num)
// get
// 10

ob.num = 1
// set
```

依照这个思路，假设我想在接受服务器的数据之后，自动更新界面上的显示。也就是在这个值的 `set` 触发的时候，让它执行一个函数来更新 DOM 就可以了。

而这个更新的函数就是接下来要讲的 Watcher。

### Watcher

代码位置为 `src/core/observer/watcher.js`

Watcher 的构造函数为

```
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: Object
)
```

其中，`vm` 为一个组件实例，`expOrFn` 为一个表达式或函数，`cb` 为回调函数（callback），`options` 为一些额外参数。

举个例子，在 `mountComponent` 这个函数中有这样两行代码

```
...
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
...
vm._watcher = new Watcher(vm, updateComponent, noop)
```

`noop` 是一个什么都不做的函数，估计是 `no operation`。。需要更新 Watcher 的时候会以这样的顺序执行

`Watcher.update()` -> `this.run()` -> `this.get()` -> `this.cb()`

其中 `this.get()` 在这里就是 `updateComponent`。因此这样的话 DOM 就得到了更新。
那为啥既有 `expOrFn`，又有 `cb` 呢？那是因为 `cb` 只有在 `expOrFn` 返回值变化时才会执行。

在[文档](https://vuejs.org/v2/api/#vm-watch)中，`$watch` 函数是这样的

`vm.$watch( expOrFn, callback, [options] )`

我们来看一个[例子](https://codepen.io/pengchongfu/pen/OmWJzO?editors=1111)
```
var Main = {
  data: function() {
    return {
      a: 1,
      b: 2
    };
  },
  methods: {
    change() {
      this.a = 2;
      this.b = 1;
    }
  }
};
var Ctor = Vue.extend(Main);
var vm = new Ctor();

vm.$watch(
  function() {
    return this.a + this.b;
  },
  function(val) {
    console.log("new value " + val);
  },
  {
    sync: false
  }
);

vm.change();
```

可以看到控制台毫无反应，因为当执行 `expOrFn` 时发现返回值不变，所以没有执行 `cb`。

*当  `sync` 为 `true` 时，可以看到 `cb` 被调用，这其实是个优化技巧。Vue 会管理一个 Watcher 队列，当有新的 Watcher 进入队列时，若是之前已在队列中则不 `push`，比如更新 DOM 的 Watcher 没必要多次执行，最后执行即可，所以 `sync` 默认值为 `false`*

由上边的例子可以知道，当 Watcher 依赖的值变化时，会执行 `Watcher.update()`。
那该怎么知道依赖变化了呢，要不依赖告诉 Watcher，要不 Watcher 去看依赖是否变化。

一个很直观的方法就是，Watcher 轮询 `expOrFn` 中依赖的值，若是变化了执行更新。
（这样做的话直觉上就很慢，，但愿浏览器不要卡死）
所以我们还是让所依赖的值告诉我们好了。

值得注意的一点是 Watcher 初始化的时候会执行一次 `this.get()` 也就是 `expOrFn`。所以收集依赖的过程只能在这里进行。

我们做这样一个设想。每个被依赖的值维护一个数组，当对这个值执行取值操作时，会执行 `Object.defineProperty()` 定义好的 `get` 函数。
然后 `get` 函数会将正在执行的 Watcher （只可能有一个，毕竟 js 单线程）放进数组中。
当值变化时（也就是执行 `set` 函数时），将数组中的 Watcher 全部执行 `Watcher.update()` 一遍。
这样的话是不是就能实现响应式了呢？

从源码中可以看到，Vue 就是这么做的，只是这个数组更为复杂，叫 Dep。

### Dep

代码位置为 `src/core/observer/dep.js`

Dep 的构造函数为

```
constructor () {
  this.id = uid++
  this.subs = []
}
```

`this.id` 是该 Dep 的唯一标识符，`this.subs` 是一个数组，里边要放的就是之前提到的依赖于（换个说法就是订阅了）这个值的 Watcher。

代码里边的内容还是挺少的。可以看一看 `notify` 函数

```
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

其实就是之前所说的
> 将数组中的 Watcher 全部执行 `Watcher.update()` 一遍。

至于将 Watcher 加入 Dep 的过程大概是这样

`dep.depend()` -> `Dep.target.addDep(this)` -> `dep.addSub(this)`

其中 `Dep.target` 是当前正在执行的 Watcher，Watcher 将 Dep 加入自己的依赖列表，Dep 再将 Watcher 加入自己的订阅器数组。
其实直接执行 `dep.addSub(Dep.target)` 应该就可以，可是这样的话 Watcher 就收集不到依赖了，这些依赖信息还是很重要的。

所以接着我们看看 Vue 是如何将上边的流程串在一起的，也就是关键的 `Object.defineProperty()` 是如何定义的。

### defineReactive

代码位置为 `src/core/observer/index.js`

代码不是很多，直接贴上来吧

```
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          dependArray(value)
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
```

代码涉及的东西还是很多的，这里我们不细说，只看关键的地方。

- 函数执行时会生成一个 Dep 实例
- 接着执行 `Object.defineProperty` 函数
- `get` 时执行 `dep.depend()` 并 `return value` 作为返回值（Watcher 收集依赖，Dep 将 Watcher push 进数组中）
- `set` 时执行 `dep.notify()`（数组内的 Watcher 执行 `Watcher.updat()`）

### 总结

Vue 的数据监听实现主要是在 `src/core/instance/state.js` 中，`initProps` `initData` `initComputed` 是实现数据监听的函数。

以上就是 Vue 响应式的核心了，当然还有很多细节的东西没有说清楚，比如 Vue 是怎么样监听数组、对象的。这些都需要亲自看源码才能了解。