# 浅析 Vue3 响应式系统

## 前言

好像是某次直播上，有同学提问尤大，如果要阅读源码，现在是看 Vue2 还是 Vue3？尤大的回答是**Vue3**，原因在于其重构之后，项目结构更加清晰明了。因此还是更推荐阅读`Vue3`的源码。

查看`Vue3`代码结构，采用 monorepo 的方式进行代码管理，将不同模块拆分到 packages 目录中， 其中就包含了及其重要的响应式模块（[@vue/reactivity](https://www.npmjs.com/package/@vue/reactivity)），该模块也可以在项目中单独引入，不与 Vue 绑定。众所周知，`Vue3`响应式由`Vue2`的`defineProperty`改为`Proxy`。作为`Composition API`的核心，我们有必要来了解一下内部实现原理。

### @vue/reactivity 常用 API (来自官网🤪)

-   reactive：返回对象的响应式副本。
-   ref：接收一个内布值并返回响应式可变的 ref 对象。ref 对象具有指向内部值的单个 property。
-   isProxy：检查对象是否是由 `reactive` 或 `readonly` 创建的 proxy。
-   isReactive；检查对象是否由 `reactive` 创建的响应式代理。
-   isRef：检查值是否为一个 ref 对象。
-   computed：接受一个 getter 函数，并根据 getter 的返回值返回一个不可变的响应式 ref 对象，或者，接受一个具有 `get` 和 `set` 函数的对象，用来创建可写的 ref 对象。
-   watch：`watch` API 与选项式 API [this.$watch](https://v3.cn.vuejs.org/api/instance-methods.html#watch) (以及相应的 [watch](https://v3.cn.vuejs.org/api/options-data.html#watch) 选项) 完全等效。`watch` 需要侦听特定的数据源，并在单独的回调函数中执行副作用。默认情况下，它也是惰性的——即回调仅在侦听源发生变化时被调用。
-   effect：副作用函数，传入函数并执行，进行依赖收集。

## 前置学习

### Proxy

**Proxy** 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。

#### 参数

- target 要使用`Proxy`包装的目标对象（可以是任何类型的对象，包括原生数组，函数，甚至另一个代理）。
- handler 一个通常以函数作为属性的对象，各属性中的函数分别定义了在执行各种操作时代理对象的行为。以下列举常用的`handler`对象属性

![proxy.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1eefc1e04c8349a99e827c22522c6068~tplv-k3u1fbpfcp-watermark.image?)

## 来看一个简单的栗子

假设有一个场景，通过点击按钮同步当前时间，并且记录点击次数，如果是在 Vue 中，数据双向绑定，很快就能实现。如果是使用原生 Javascript 来实现的话，要怎么做呢？


![html.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e17abe29a0345af8628c06781bf9bcb~tplv-k3u1fbpfcp-watermark.image?)


![normal-js.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/840bcd990d9947e58670ab418600fbd7~tplv-k3u1fbpfcp-watermark.image?)

引入`@vue/reactivity`的话呢？


![reactive-js.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17e2fa0143f740d0ad24d9d8653a3a7e~tplv-k3u1fbpfcp-watermark.image?)

## 实现一个最小化响应式模型

从上面代码中，也就是说只需要`reactive`、`effect`就能实现上面这个简单的模型，开搞！

![reactivity.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d78a0416e7d4ceab08c1c7aedbdcb66~tplv-k3u1fbpfcp-watermark.image?)

简单来看还是通过收集依赖，变更时通知更新

执行流程大概如上图(只是缺少了左侧更新渲染虚拟 DOM 部分)，首先通过 reactive 创建响应式对象，在 get 操作、set 操作进行拦截，对象 get 操作时，进行依赖收集，对象 set 操作时，进行依赖更新。通过 effect 副作用函数，传入响应式回调函数，当响应式数据变化时就会触该回调函数。具体代码如下：

![reactive.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81c3c892546d46d5b93801ab5b19ea28~tplv-k3u1fbpfcp-watermark.image?)


![track.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ec59c4d19cb43b3a9fc6e20a18b84dd~tplv-k3u1fbpfcp-watermark.image?)


![trigger.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2d18fc167d04d979664644ff492c2c0~tplv-k3u1fbpfcp-watermark.image?)


![effect.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d5a7318a3344dee8485172f0327cdc2~tplv-k3u1fbpfcp-watermark.image?)

以上极大简化了 `@vue/reactivity` 代码，只是完成了一个基于上面这个例子的最小化模型，`@vue/reactivity`还提供了其他的 API ，例如 `ref`、`computed`、`shallowReactive`(创建一个响应式代理，它跟踪其自身 property 的响应性，但不执行嵌套对象的深层响应式转换 (暴露原始值))。通过上面一个这个简化的版本再去看完整版，应该会稍微清晰一点。

希望通过本文抛砖引玉，继续深入，建议大家阅读[完整源码](https://github.com/vuejs/vue-next/tree/master/packages/reactivity)看完整版源码的时候，也能够继续深入有所学习。例如，平时提到 proxy，大都只是对对象（字面意思 `Object`）进行代理，处理其`get`、`set`操作，但是不知道大家有没有想过，如果针对数组的话，proxy 的操作又是怎样？针对数组还有这么多常用方法，这可不像简单对象的`get`、`set`操作，又要怎么处理对应的响应式呢？阅读完整源码，你就能找到答案，反正我看完之后就是


![em.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07792e0d01e9476b8af52ca73c5a6f6b~tplv-k3u1fbpfcp-watermark.image?)


## 参考资料

[查看原文代码](https://github.com/JW-LINNN/custom-vue-reactivity)

[Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

[@vue/reactivity](https://github.com/vuejs/vue-next/tree/master/packages/reactivity)


