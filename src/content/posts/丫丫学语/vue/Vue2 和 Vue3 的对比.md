---
title: Vue2 和 Vue3 的对比
published: 2026-10-22
description: data / methods / computed / watch / 生命周期在两版的写法对照，初始化流程里「谁能访问谁」的顺序差别，以及 this 的位置和兼容边界
image: "api"
tags: [Vue]
category: vue
draft: false
---

# Vue2 和 Vue3 的对比

同一个组件两套写法。写法差异往下追一层是初始化顺序，再往下追一层是响应式的实现方式，所以从响应式看起比较省事。

## 写法对照

| | Vue2（Options API） | Vue3（Composition API） |
|---|---|---|
| 状态 | `data() { return { count: 0 } }` | `const count = ref(0)` |
| 方法 | `methods: { inc() { this.count++ } }` | `const inc = () => count.value++` |
| 派生 | `computed: { double() { return this.count * 2 } }` | `const double = computed(() => count.value * 2)` |
| 监听 | `watch: { count(n, o) {} }` | `watch(count, (n, o) => {})` |
| 自动监听 | 无对应写法 | `watchEffect(() => {})` |
| 生命周期 | `created` / `mounted` | `setup` 内 / `onMounted` |
| 逻辑复用 | mixins，命名冲突和来源不明 | 组合式函数 `useXxx()` |

## Vue2 的加载顺序

```
new Vue(options)
  → mergeOptions（data 策略合并、methods 覆盖、钩子合并成数组）
  → initLifecycle / initEvents / initRender
  → beforeCreate
  → initInjections
  → initState：props → methods → data → computed → watch
  → initProvide
  → created
  → $mount → render → patch → mounted
```

`initState` 里那五步的顺序不是随意排的，它直接决定谁能访问谁：

- `props` 最先初始化，所以 `data` 里能读 `this.xxx`（props）
- `methods` 排在 `data` 前，所以 `data` 里能调 methods
- `computed` 排在 `data` 后，所以计算属性能依赖 data
- `watch` 排最后，因为 `immediate` 会立刻执行回调，而回调里通常要读 data 和 computed

反向访问都不成立：data 里读不到 computed，props 里读不到 data。

`data` 在这一步被 `observe` 递归处理，用 `Object.defineProperty` 逐个属性重写 getter/setter。所以对象有多大，初始化就要遍历多大；数组的索引赋值和 `length` 变化拦不住，得用 `Vue.set` 或 `splice`。

## Vue3 的加载顺序

```
createApp(options).mount()
  → createVNode → setupComponent
  → setup() 执行（此时 this 未就绪）
  → beforeCreate
  → 注入 → methods → data → computed → watch → provide
  → created
  → 建立渲染 effect（renderEffect）
  → patch → mounted
```

两处要留意。

**`setup` 跑在 `beforeCreate` 之前。** 所以 `setup` 里 `this` 是 `undefined`，拿不到 data、methods、`$refs`。Composition API 本来就走显式引用，不用 `this`。

**`initState` 的五步顺序在兼容层里被原样保留。** Vue3 的 Options API 跑在 Composition API 之上（`applyOptions`），老代码的访问关系不变。混用两种写法时，`setup` 先于 `data`，所以 `setup` 里拿不到 `data`；在 `data` 里访问 `setup` 的返回值属于未定义行为，不要依赖。

响应式换成了 Proxy，包在 `data` 返回的对象外层。Proxy 是惰性的，嵌套对象在第一次访问时才创建代理，初始化成本跟对象大小无关。新增属性、索引赋值、`delete` 都能拦截。

## computed 的实现差异

Vue2 的 `initComputed` 用 `Object.defineProperty` 把计算属性定义在 `vm` 上，getter 里判断 `watcher.dirty`：

- `dirty === true` 才真正求值，求值时把 `Dep.target` 指向这个 computed watcher 来收集依赖
- 依赖变化只执行 `update()`，把 `dirty` 置 true，不立刻重算
- 下次访问才重算，缓存就是这么来的

Vue3 的 `ComputedRefImpl` 内部包一个 `ReactiveEffect`，`get value` 时判断 `_dirty`，依赖触发时置脏并通知订阅者。

差别落在两点：Vue2 的 computed 依附 `vm`，Vue3 的是独立 `ref`，可以脱离组件使用，也能当普通值传给子组件或放进数组。另外 Vue3 的 computed 默认只读，要可写必须传 get 和 set：

```js
const full = computed({
  get: () => first.value + ' ' + last.value,
  set: (v) => ([first.value, last.value] = v.split(' ')),
})
```

## watch 的实现差异

Vue2 走 `this.$watch(expOrFn, cb, options)`，创建一个 user watcher：

- `expOrFn` 支持字符串路径（`'a.b.c'`，内部用 `parsePath` 解析）、函数、数组（多源）
- `deep` 靠 `traverse` 递归读一遍所有属性，用「读」触发依赖收集
- `immediate` 让回调在创建时先跑一次

Vue3 走 `watch(source, cb, options)`，`source` 可以是 `ref`、getter 函数、`reactive` 对象、数组。多了三样东西：

- **`flush`**：`'pre'`（默认，组件更新前）、`'post'`（更新后，能拿到新 DOM）、`'sync'`。要在回调里读 DOM，必须改成 `post`
- **返回 `stop` 句柄**：Vue2 里叫 `unwatch`，行为一样
- **`onCleanup`**：在回调里注册清理函数，处理异步竞态

```js
watch(keyword, async (kw, _, onCleanup) => {
  let cancelled = false
  onCleanup(() => (cancelled = true))
  const res = await search(kw)
  if (!cancelled) list.value = res
})
```

`watchEffect` 是 Vue3 独有的：不指定 source，自动收集依赖，且立即执行一次。依赖一堆、懒得逐个列的时候用它，代价是依赖关系不写在代码里。

## 生命周期的对应关系

| 时机 | Vue2 Options API | Vue3 组合式 | 备注 |
|---|---|---|---|
| 实例创建前 | `beforeCreate` | `setup` 开头 | 无 `onBeforeCreate` |
| 实例创建后 | `created` | `setup` 结尾 | 无 `onCreated` |
| 挂载前 | `beforeMount` | `onBeforeMount` | DOM 还没生成 |
| 挂载后 | `mounted` | `onMounted` | `$refs` 可用 |
| 更新前 | `beforeUpdate` | `onBeforeUpdate` | DOM 还是旧值 |
| 更新后 | `updated` | `onUpdated` | |
| 激活 | `activated` | `onActivated` | 组件需在 `KeepAlive` 内 |
| 停用 | `deactivated` | `onDeactivated` | 同上 |
| 卸载前 | `beforeDestroy` | `onBeforeUnmount` | 改名 |
| 卸载后 | `destroyed` | `onUnmounted` | 改名 |
| 捕获后代错误 | `errorCaptured` | `onErrorCaptured` | 返回 `false` 阻止继续向上传播 |
| 服务端预取 | `serverPrefetch` | `onServerPrefetch` | |
| 依赖收集（仅开发） | 无 | `onRenderTracked` | |
| 触发渲染（仅开发） | 无 | `onRenderTriggered` | |

`beforeCreate` 和 `created` 在 Vue3 的 Options API 里照样会执行，写 `created() {}` 有效。组合式那一列没有对应函数，因为 `setup` 本身就落在 `beforeCreate` 之前开始、`created` 之前结束，两个时机被它一并覆盖。

四处结构性差别值得单独记。

**第一处：`setup` 承担了 `created` 的活，但没有 `this`。** 原来写在 `created` 里的初始化逻辑搬到 `setup` 顶层，把 `this.$route` 换成 `useRoute()`，`this.$store` 换成 `useStore()`。只有依赖 DOM 的部分需要挪进 `onMounted`。

**第二处：组合式钩子必须在 `setup` 同步执行期间调用。** 放进 `await` 之后，或任何异步回调里，都会报 `onMounted is called when there is no active component instance`。Options API 没有这个限制，因为钩子是声明式的，框架构造实例时读取一次。写法调整很直接：把 `await` 放进 `onMounted` 的回调内部，而不是把 `onMounted` 放在 `await` 后面。

**第三处：同一个钩子可以注册多次，按注册顺序执行。** Options API 每个钩子只能挂一个函数。这带来一个新问题：组合式函数内部注册的钩子，调用方看不见。`useXxx()` 里可能悄悄注册了 `onMounted`，排查执行时序时要翻进函数体。

**第四处：父子执行顺序两版一致，但组合式里更难看出。** 挂载是父 `beforeMount` → 子 `beforeMount` → 子 `mounted` → 父 `mounted`，卸载是父 `beforeUnmount` → 子 `beforeUnmount` → 子 `unmounted` → 父 `unmounted`。父先开始、子先结束，所以子组件 `mounted` 时父组件的 DOM 还没就绪，依赖父级布局的测量要放到 `nextTick` 或父的 `onMounted` 里。

`$nextTick` 在 Vue3 里是独立导入的 `nextTick`，不再是实例方法。清理逻辑的配对方式没变，只是两个钩子都要从 `vue` 引入：

```js
import { onMounted, onBeforeUnmount } from 'vue'

let timer = null

onMounted(() => {
  timer = setInterval(poll, 5000)
})

onBeforeUnmount(() => {
  clearInterval(timer)
})
```

`onBeforeUnmount` 里 DOM 还在，`onUnmounted` 里已经从文档移除，需要读 `$el` 的操作放在前者。

## this 在 Vue3 里的位置

| 位置 | `this` 是什么 |
|---|---|
| Options API 的选项里 | 组件实例的公开代理，`$refs`、`$emit` 可用 |
| `setup()` 函数体内 | `undefined` |
| `<script setup>` 顶层 | `undefined` |

后两行是同一个原因。`setup` 跑在 `beforeCreate` 之前，实例还没构造完，官方没有给它一个半成品 `this`。调用 `setup(props, context)` 时也没有任何绑定动作，而 ESM 模块处于严格模式，普通函数调用时 `this` 的默认值就是 `undefined`。`<script setup>` 顶层是模块作用域，规则一样。

第一行那个 `this` 是 `PublicInstanceProxyHandlers` 生成的 Proxy，和 instance 是两个对象，`this === instance` 为 `false`。它的 `get` 陷阱按 `setupState → data → props → ctx → appContext.config.globalProperties` 的顺序查找，`this.$router` 取的是最后一层。每个 `this.xxx` 都要走一遍这串查找，`<script setup>` 里的闭包变量是一次直接引用，性能差别出在这里。

`this` 缺席带来的连带影响有两个。

一是 `methods` 里不能再写箭头函数。箭头函数不绑定 `this`，取的是外层作用域的值，在模块里就是 `undefined`，`inc: () => this.count++` 会直接报错。Options API 的 method 必须写成传统函数。Composition API 里箭头函数变多，根子也在这一条：`const inc = () => count.value++` 捕捉的是外层声明的 `count`，不需要 `this`，所以怎么写都行。

二是模板里不能写 `this`。Vue2 编译出来是 `with(this){ ... }`，`{{ this.msg }}` 恰好能工作；Vue3 编译成 `_ctx.msg`，写 `this.msg` 会变成 `_ctx.this.msg`，取到 `undefined`。从 Vue2 复制模板片段时可能带过来。

确实需要实例时用 `getCurrentInstance()`，限定在插件、组合式函数封装、需要读 `appContext` 的库里。业务组件拿它去取 `$refs` 或 `$router`，等于把刚省下的那层代理又绕回去，前者有 `ref()`，后者有 `useRouter()`。

## Vue3 兼容 Vue2 的写法吗

分四层看，结论不一样。

| 层 | 状态 | 涉及内容 |
|---|---|---|
| 语法层 | 兼容 | `data` / `methods` / `computed` / `watch` / `props` / 生命周期 |
| 行为层 | 兼容但有变化 | `v-if` 与 `v-for` 优先级、`$attrs` 内容、多根节点、`$children` |
| API 层 | 已移除 | `$on` / `$off` / `$once`、`filters`、`.sync`、`functional`、`$set` |
| 生态层 | 需换代 | Element UI、Vuex、vue-router 3、IE11 |

语法层的兼容由 `applyOptions` 实现，它把各个选项翻译成 `setup` 里的调用序列，顺序沿用 `initState` 的 props → methods → data → computed → watch。顺序不变，老代码里「`data` 用 methods、`computed` 用 data」的访问关系就不变，`$refs`、`$emit`、`$watch`、`$nextTick` 也都还在。

有一个前提容易忽略：字符串模板需要带编译器的构建。打包工具默认把 `vue` 解析到 `vue.runtime.esm-bundler.js`，这个版本不含编译器，写 `template: '<div>{{ msg }}</div>'` 会报错。单文件组件的模板在构建期就编译完了，不受影响；CDN 引完整版、`new Vue({ template })` 这类写法必须换成 `vue/dist/vue.esm-bundler.js`。

被移除的 API 对应关系：

| Vue2 | Vue3 |
|---|---|
| `$on` / `$off` / `$once` | 实例不再是事件总线，换 `mitt` |
| `filters` | `computed` 或普通方法 |
| `.sync` 加 `model` 选项 | `v-model:propName` |
| `$children` | `$refs` |
| `$listeners` | 合并进 `$attrs` |
| `$scopedSlots` | 合并进 `$slots` |
| `$set` / `$delete` | 不再需要，Proxy 处理新增属性 |
| `Vue.observable` | `reactive` |
| `new Vue()` / `Vue.prototype` | `createApp()` / `app.config.globalProperties` |
| `<template functional>` | 普通函数组件 |
| `render(h)` 里的 `h` 参数 | 从 `vue` 导入 `h` |

行为层有四处要改代码，最后一处不会报错。

`v-if` 与 `v-for` 的优先级反转了。Vue2 里 `v-for` 先执行，`v-if` 能拿到循环变量；Vue3 里 `v-if` 先执行，同一元素上两者并用会报变量未定义。这条是升级时最常撞上的，报错信息也不直白。

`$attrs` 的内容变宽，现在包含 `class`、`style`，也包含没在 `emits` 里声明的事件监听器。用了 `inheritAttrs: false` 加 `v-bind="$attrs"` 的组件要逐个检查，否则会给内层元素挂上不该有的样式。

多根节点不再自动继承属性。Vue2 只有一个根节点，属性自然落上去；Vue3 没有唯一落点，需要显式 `v-bind="$attrs"`。原本单根、升级后想加第二个根节点的组件，要补这一行。

还有一条不报错也不警告：`beforeDestroy` 与 `destroyed` 改名成 `beforeUnmount` 与 `unmounted`。非迁移构建下老名字被静默忽略，钩子不执行。定时器没清、监听没解绑、连接没关，页面看起来一切正常，内存慢慢涨。

要迁存量项目，官方提供 `@vue/compat` 迁移构建。别名指向它之后，Vue3 运行时能跑 Vue2 代码，被移除或行为变化的特性会打印运行时警告，`configureCompat` 可以按特性单独设 `false` / `2` / `3` 逐个切换。它的体积和性能都有代价，只用于过渡期。

不改存量、只判断新代码怎么写：Options API 在 Vue3 里继续可用，是一等公民。存量组件留着不动，新组件从 `<script setup>` 起，同一个文件里两种写法混着是最差的状态，两套响应式来源，读代码要在两个位置找变量。

## 迁移时容易踩的六处

1. **`watch` 监听 `reactive` 对象，新旧值相等**。传进来的是同一个 proxy 引用，`newValue === oldValue`。要对比得自己存快照，`toRaw` 加浅拷贝，或者 `JSON.parse(JSON.stringify())`
2. **`watch` 默认浅监听**。监听 `reactive` 对象的嵌套属性必须加 `deep: true`；Vue2 里监听对象会隐式深度
3. **`computed` 变成只读**，直接赋值会报错。Vue2 里只是警告
4. **`flush` 默认 `pre`**。在回调里读 DOM 拿到的是旧值，要改 `post`
5. **生命周期改名**：`beforeDestroy` 改 `onBeforeUnmount`，`destroyed` 改 `onUnmounted`。名字变了，语义没变
6. **`methods` 里的 `this` 没了**。从 mixin 迁到组合式函数，所有 `this.xxx` 都要改成显式引用，这是迁移里改动量最大的机械劳动

## 一句话记

加载顺序上最大的差别是 `setup` 跑在 `beforeCreate` 之前，所以内部没有 `this`；而 `initState` 的五步顺序被兼容层保留，两版混用时的访问关系一致。响应式从 `Object.defineProperty` 换成 Proxy，computed 因此脱离组件实例；`watch` 新增了 `flush`、`stop` 句柄和 `onCleanup`，默认浅监听加 `pre` 时机这两条最容易写错。

兼容性上，Options API 由 `applyOptions` 完整保留，字符串模板还要挑带编译器的构建；被删的 `$on`、`filters`、`.sync`、`$set` 和整个组件库生态需要换代。`v-if` 与 `v-for` 的优先级反转、生命周期改名的静默失效，是升级时最不容易发现的两处。
