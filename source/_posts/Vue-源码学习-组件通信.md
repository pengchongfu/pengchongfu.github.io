---
title: Vue 源码学习-组件通信
date: 2017-04-29 20:16:59
tags:
- Vue
- 前端
---

Vue 中组件化是很重要的一个概念。而组件之间通信是很常见的一个需求。

组件间通信最常见的就是父子组件之间通信了。父组件状态发生变化，向子组件分发事件；子组件状态变化，向父组件提交事件。当然有时也会遇到两个组件在组件树中相隔甚远却需要通信的情况，这时候我们可以使用一个事件总线来处理，vuex 之类的库可以看成是高级一点的事件总线。

所以我们就来看看 Vue 中是如何实现父子组件通信的。

### 父组件 -> 子组件 Prop

正如官方文档中所说的，在父组件向子组件通信的时候，应该使用 Prop 传递数据。

这个传递过程具体是怎么样的呢？如果阅读了前面的几篇博客应该会有所了解。下面我们再梳理一下。

1. 父组件状态发生变化
2. 触发 `vm._watcher` （这个 Watcher 是触发组件更新的）
3. 生成新的 VNode，并且子组件 VNode 上挂载对应 `vm` 的 `prepatch` 函数
4. 更新 DOM 的同时对子组件的 `vm` 实例进行 `prepatch`，改变 `vm` 的 props
5. 子组件的 props 变化后，订阅 props 的 Watcher 会自动执行（**异步执行**，会放在一个事件队列中）

在 `src/core/vdom/create-component.js` 中，有一个 `componentVNodeHooks` 对象。对象上挂载了 `init` `prepatch` 等钩子函数，会在组件 init 以及 patch 时执行。我们在这两个函数以及其他一些函数上加上标记，看看下面的这个[例子](https://codepen.io/pengchongfu/pen/qmrejN)会是如何的执行顺序。

```
<div id="app">
</div>
```

```
Vue.component("child", {
  props: {
    number: {
      default: 0
    }
  },
  render(h) {
    return h("span", "father number is " + this.number);
  }
});

new Vue({
  el: "#app",
  data() {
    return {
      number: 1
    };
  },
  render(h) {
    return h("div", {}, [
      h("h1", "father number is " + this.number),
      h("child", {
        props: {
          number: this.number
        }
      })
    ]);
  },
  mounted() {
    setTimeout(() => {
      this.number = 2;
    }, 3000);
  }
});
```

```
_render start       // 父组件开始 `_render`
_render end         // 父组件 `_render` 结束，生成 h1 child div 3个 VNode
_update start div   // 父组件开始 `_update`
init                // 创建 3个 VNode 对应的 DOM，遇到 child 组件时，执行 `init`
_render start       // 子组件 child 开始 `_render`
_render end         // 子组件 child `_render` 结束，生成 span 1个 VNode
_update start span  // 子组件 child 开始 `_update`
_update end span    // 子组件 child `_update` 结束
_update end div     // 父组件 `_update` 结束
                    // 3秒后计时器触发
_render start       // 父组件 `vm._watcher` 触发，开始 `_render`
_render end         // 父组件 `_render` 结束，生成 h1 child div 3个 VNode
_update start div   // 父组件开始 `_update`
prepatch            // child 组件 `vm` 已经存在，所以不会执行 `init`，执行 `prepatch`，更改 props
_update end div     // 父组件 `_update` 结束（Vue 会把要执行的 Watcher 放进一个队列中，在下一个事件循环中执行，因此此处并不会马上执行 child 组件的 `_render`）
_render start       // 子组件 child 开始 `_render`
_render end         // 子组件 child `_render` 结束，生成 span 1个 VNode
_update start span  // 子组件 child 开始 `_update`
_update end span    // 子组件 child `_update` 结束
```

### 子组件 -> 父组件 自定义事件

提到事件，不可避免的和 DOM 的原生事件联系上了。而在官方文档中，也是通过自定义事件讲解子组件向父组件通信的。

#### Vue 自定义事件

`Vue.prototype.$on` `Vue.prototype.$emit` 的代码都在 `src/core/instance/events.js` 中，代码很短，直接贴上来了。

```
const hookRE = /^hook:/
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      this.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
    // optimize hook:event cost by using a boolean flag marked at registration
    // instead of a hash lookup
    if (hookRE.test(event)) {
      vm._hasHookEvent = true
    }
  }
  return vm
}
```

```
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  if (process.env.NODE_ENV !== 'production') {
    const lowerCaseEvent = event.toLowerCase()
    if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
      tip(
        `Event "${lowerCaseEvent}" is emitted in component ` +
        `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
        `Note that HTML attributes are case-insensitive and you cannot use ` +
        `v-on to listen to camelCase events when using in-DOM templates. ` +
        `You should probably use "${hyphenate(event)}" instead of "${event}".`
      )
    }
  }
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    for (let i = 0, l = cbs.length; i < l; i++) {
      cbs[i].apply(vm, args)
    }
  }
  return vm
}
```

逻辑很简单，`$on` 将事件名称和回调 push 到 `vm._events` 中，`$emit` 将对应事件名称的回调从 `vm._events` 中取出然后执行。

或许看到这里会有一个小小的疑惑。直觉上，`$on` 应该是在父组件实例上，`$emit` 在子组件上，并且会沿着组件树向上层传递，这样感觉和浏览器原生的 listener 比较接近。

但其实并不是这样的。`$on` 其实也是在子组件实例上的，在父组件中定义只是为了回调能获取**父组件的上下文**。正如代码所表明的，Vue 中的自定义事件只能在父与子之间定义，超过一个层级的事件应该用其他方式来实现。

例如可以定义一个空的 Vue 实例，把 `$on` `$emit` 都绑定在它上边，这样它就成为了一个事件总线。或者直接采用 vuex 来管理状态。

#### Vue DOM 原生事件

接下来我们看看 DOM 原生事件是怎么绑定在组件上的。

代码位置为 `src/platforms/web/runtime/modules/events.js`

```
function updateDOMListeners (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  const on = vnode.data.on || {}
  const oldOn = oldVnode.data.on || {}
  target = vnode.elm
  normalizeEvents(on)
  updateListeners(on, oldOn, add, remove, vnode.context)
}
```

其实就是在生成或者更新 DOM 时，根据新旧的 VNode 更新 listeners。具体实现有兴趣的可以去看看代码。

或许看到这里你会有个疑问，为什么是 `vnode.data.on` 而不是 `vnode.data.nativeOn`。其实这是因为在 `createComponent` 生成 VNode 这一步的时候。执行了这样几行代码

```
const listeners = data.on
data.on = data.nativeOn
```

这样的话 `vnode.data.on` 里边就是 native 事件了。所以我们再看一个[例子](https://codepen.io/pengchongfu/pen/mmmPgZ)

```
<div id="app">
</div>
```

```
Vue.component("child", {
  render(h) {
    return h("button", {
      domProps: {
        innerHTML: "button"
      }
    });
  },
  mounted() {
    setTimeout(() => {
      this.$emit("click");
    }, 3000);
  }
});

new Vue({
  el: "#app",
  methods: {
    eventClick() {
      console.log("child event click");
    },
    nativeClick() {
      console.log("child native click");
    }
  },
  render(h) {
    return h("div", [
      h("child", {
        on: {
          click: this.eventClick
        },
        nativeOn: {
          click: this.nativeClick
        }
      })
    ]);
  }
});
```

例子很简单，`nativeOn` 放的是原生 DOM 事件。 `on` 放的是 Vue 自定义事件。

### 总结

以上就是 Vue 实现组件间通信的原理了。总的来说还是挺直观的。当然组件之间离得远的话或者状态多的话就得用 vuex 了。那就是另外一个大坑了。。
