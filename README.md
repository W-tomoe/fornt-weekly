# vue nextTick 原理

> 为什么 vue 优先使用 promise.then 而不是 setTimeout ？

## nextTick 初版

```
export const nextTick = (function () {
 // 存储需要执行的回调函数
 var callbacks = []
 // 标识是否有 timerFunc 被推入了任务队列
 var pending = false
 // 函数指针
 var timerFunc
 // 下一个 tick 时循环 callbacks, 依次取出回调函数执行，清空 callbacks 数组
 function nextTickHandler () {
   pending = false
   var copies = callbacks.slice(0)
   callbacks = []
   for (var i = 0; i < copies.length; i++) {
     copies[i]()
   }
 }

 // 检测 MutationObserver 是否可用
 // 当执行 timerFunc 时，改变监听值，触发观测者将 nextTickHandler 推入任务队列
 if (typeof MutationObserver !== 'undefined') {
   var counter = 1
   var observer = new MutationObserver(nextTickHandler)
   var textNode = document.createTextNode(counter)
   observer.observe(textNode, {
     characterData: true
   })
   timerFunc = function () {
     counter = (counter + 1) % 2
     textNode.data = counter
   }
 } else {
   // 如果 MutationObserver 不可用
   // timerFunc 指向 setImmediate 或者 setTimeout
   const context = inBrowser
     ? window
     : typeof global !== 'undefined' ? global : {}
   timerFunc = context.setImmediate || setTimeout
 }
 // 返回的函数接受两个参数，回调函数和传给回调函数的参数
 return function (cb, ctx) {
   var func = ctx
     ? function () { cb.call(ctx) }
     : cb
   // 将构造的回调函数压入 callbacks 中
   callbacks.push(func)
   // 防止 timerFunc 被重复推入任务队列
   if (pending) return
   pending = true
   // 执行 timerFunc
   timerFunc(nextTickHandler, 0)
 }
})()
```

第一版到2.5.2版本之间，nextTick 修改了多次，修改的内容主要是 timerFunc 的实现。 

第一次修改是将 MutationObserver 替换为 postMessage， 给出的理由是 MutationObserver 在 UIWebView (iOS >= 9.3.3) 中不可靠(现在是否有问题不清楚)。后面版本中又恢复了 MutationObserver 的使用，同时对 MutationObserver 使用做了检测， 非IE环境下且是原生 MutationObserver。 


第二次改动是恢复了微任务的优先使用，timerFunc 检测顺序变为 Promise,  MutationObserver, setTimeout. 在使用 Promise 时，针对 IOS 做了特殊处理，添加空的计时器强制刷新微任务队列。 同时这一版中还有个小的改动， nextTickHandler 方法中对 callbacks 数组重置修改为

```
callbacks.length = 0
```
一个小的性能优化，减小空间消耗。


然而这个方案并没有持续多久就迎来来一次‘大’改动，微任务全部裁撤，timerFunc 检测顺序变为 setImmediate,  MessageChannel, setTimeout. 原因是微任务优先级太高了，其中一个 issues 编号为 [#6566](https://github.com/vuejs/vue/issues/6566)

这里给个demo有兴趣可以去点点看 [点我看看](https://jsbin.com/qejofexedo/1/edit?html,js,output)


尤大对此给出了回复，简而言之，点击 div.header, 触发标签 i 上绑定的事件，执行事件后
```
expand = false
countA = 1
```

然后因为微任务优先级太高，在事件冒泡到外层 div 时就已经触发，更新期间，click listener 加到了外层div， 因为 dom 结构一致，div 和 i 标签都被重用，然后 click 事件冒泡到 div， 触发了第二次更新

```
expand = true
countB = 1
```

所以出现了如图所示的尴尬结果。如果对这块想了解的更多，可以去找一下这个issue: [#6566](https://github.com/vuejs/vue/issues/6566). 这里又要提到 JS 的事件机制了，task 依次执行， UI Render 可能在 task 之间执行， 微任务在 JS 执行栈为空时会清空队列。

## 2.5.2+


```
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let microTimerFunc
let macroTimerFunc
let useMacroTask = false

if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object): ?Promise {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // 如果不传入回调函数就直接返回一个 Promise
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

这一版抽到单独文件维护，并且引入 microTimerFunc, macroTimerFunc 分别对应微任务，宏任务。
macroTimerFunc 检测顺序为 setImmediate, Messagechannel, setTimeout, 微任务首先检测 Promise, 如果不支持 Promise 就直接指向 macroTimerFunc. 对外暴露了两个方法 nextTick 和 withMacroTask. nextTick 和之前逻辑变化不大，withMacroTask 对传入的函数做一层包装，保证函数内部代码触发状态变化，执行 nextTick 的时候强制走 macroTimerFunc。
这次修改的一个主要原因是在任何地方都使用宏任务会产生一些很奇妙的问题，其中代表 issue：[#6813](https://github.com/vuejs/vue/issues/6813)。[点我看看](https://codepen.io/ericcirone/pen/pWOrMB)


这个过程也比较好理解，之前优先使用宏任务，在两个 task 之间，会进行 UI Render ,这时，li 的行内框设置失效，展示为块级框，在之后的 task 运行了 watcher.run 更新状态，再一次 UI Render 时，ul 的 display 的值切换为 none，列表隐藏。

## 2.6+
```
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc

if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
 
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

惊奇的发现似乎又回到了初版，之前因为微任务优先级太高，太快的执行导致了非预期的问题，而这次的回归主要原因是因为宏任务执行时间太靠后导致一些无法规避的问题，而微任务高优先级导致的问题是有变通的方法的，权衡之后，决定改回高优先级的微任务。


