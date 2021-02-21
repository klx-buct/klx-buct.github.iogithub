react hooks推出也有很长一段时间了，我相信很多项目的代码里面都有着 <code>hooks</code> 的身影。那么你在用的时候有没有问过自己，为什么一个函数能记住状态？为什么 <code>hook</code> 写在if else中会有 <code>warning</code> ？下面我们来一点点的扒一扒 <code>hook</code> 的实现原理。
### hooks
目前官方提供的 <code>hook</code> 有下面几种：
#### 基础 Hook
- useState
- useEffect
- useContext
#### 额外的 Hook
- useReducer
- useCallback
- useMemo
- useRef
- useImperativeHandle
- useLayoutEffect
- useDebugValue

这些 <code>hook</code> 的作用可以参阅[官网文档](https://zh-hans.reactjs.org/docs/hooks-reference.html)，他们实现的功能不外乎这几种：  
1. 在函数中可以记住当前状态
2. 实现缓存，能够在整个生命周期内维持变量
3. 一些副作用操作可以根据某些条件判断是否执行
4. 实现了ref

我们用的最多的可能就是前面两种，那么他们到底是如何实现这些功能的呢？不慌，看看源码就知道了。

### 为什么必须在函数顶层使用hooks
```
React Hook "useState" is called conditionally. React Hooks must be called in the exact same order in every component render
```
我相信很多人都看见过这句话，这是你没有在函数顶层使用 <code>hook</code> 的时候 <code>react</code> 抛出的一个错误，那为什么 <code>react</code> 有这种限制呢？我们来看看 <code>react</code> 第一次创建 <code>hook</code> 时干了什么。

#### hook的存储
```js
function mountMemo(nextCreate, deps) {
  var hook = mountWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
 <code>react</code> 在每个 <code>hook</code> 第一次运行时，总会有一句
```js
var hook = mountWorkInProgressHook();
```
这个函数是在干嘛呢？
```js
function mountWorkInProgressHook() {
  var hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null
  };
  //workInProgressHook 是当前最新生成的 hook
  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber$1.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}
```
可以看到，它新声明了一个 <code>hook</code> 对象，里面各个值的含义我们暂且不去关心，在后面的代码中，先是判断了<code>workInprogressHook</code> 是否为空，这个字段其实是指向了一个最新生成的 <code>hook</code> ,如果它为空，证明我们是第一次生成 <code>hook</code> ，我们就把生成的 <code>hook</code> 赋值给 <code>workInProgressHook</code> 和 <code>currentlyRenderingFiber\$1.memoizedState</code>。（<code>currentlyRenderingFiber$1</code>是一个正在生成的 <code>FiberNode</code> 对象）。而如果已经生成过 <code>hook</code> 了，那么我们就直接让当前 <code>hook</code> 的 <code>next</code> 等于下一个 <code>hook</code> ，再修改 <code>workInprogressHook</code> 为最新生成的 <code>hook</code> 。这是典型的链表结构。我们用一张图来理解一下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa5d452745f842019bb7dbddf2f638f6~tplv-k3u1fbpfcp-watermark.image)

我们知道 <code>react</code> 更新了 <code>fiber</code> 架构，现在 <code>react</code> 渲染的时候会生成一颗<code>fiber</code>树，这颗树由很多个<code>FiberNode</code>结点组成。<code>FiberNode</code> 中有一个属性就叫做 <code>memoizedState</code> 。当然还有很多其他的属性，为了排除干扰项我们就不列出来了。

**注意： <code>hook</code> 的数据结构中也有一个 <code>memoizedState</code>，这两个不是同一个东西，大家不要搞混了。**

每个组件都会生成一个 <code>FiberNode</code> 。每个组件内使用的 <code>hook</code> 会以链表的形式挂在 <code>FiberNode</code> 的 <code>memoizedState</code> 上面。而每个 <code>FiberNode</code> 汇聚起来会变成一颗 <code>Fiber</code> 树， <code>React</code> 每次会以固定的顺序遍历这棵树，这样就把整个页面的 <code>hook</code> 都串联起来了。

所以，<code>mountWorkInProgressHook</code> 其实就是在做一个初始化的过程，把  <code>hook</code> 挂载到结点上去，再返回这个 <code>hook</code> 。

ps: <code>FiberNode</code> 也不只是单纯用这种单向的方式连接，他们其实会有指向父结点和兄弟结点的指针，同样为了减少干扰在此处没有表现出来。

#### hook的使用
那么我们初始化 <code>hook</code> 之后，再次 <code>render</code> 的时候会发生什么呢？
```js
function updateMemo(nextCreate, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var prevState = hook.memoizedState;

  ...省略
}
```
我们会发现，每次开头都有一句
```js
var hook = updateWorkInProgressHook();
```
那这个函数又是在干什么
```js
function updateWorkInProgressHook() {
  // This function is used both for updates and for re-renders triggered by a
  // render phase update. It assumes there is either a current hook we can
  // clone, or a work-in-progress hook from a previous render pass that we can
  // use as a base. When we reach the end of the base list, we must switch to
  // the dispatcher used for mounts.
  var nextCurrentHook;

  // currentHook: 已经生成的 fiber 树上的 hook，第一次是空
  if (currentHook === null) {
    // currentlyRenderingFiber$1: 正在生成的 FiberNode 结点, alternate 上挂载的是上一次已经生成完的 fiber 结点
    // 所以 current 就是上次生成的 FiberNode
    var current = currentlyRenderingFiber$1.alternate;

    if (current !== null) {
      // 我们之前说过 hooks 挂在 FiberNode 的 memoizedState 上，这里拿到第一个 hook
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    // 不是第一次，则证明已经拿到了 hook，我们只需要用 next 就能找到下一个 hook
    nextCurrentHook = currentHook.next;
  }

  var nextWorkInProgressHook;

  // workInProgressHook: 正在生成的 FiberNode 结点上的 hook，第一次为空
  if (workInProgressHook === null) {
    // currentlyRenderingFiber$1 是当前正在生成的 FiberNode
    // 所以这里 nextWorkInProgressHook 的值就是当前正在遍历的 hook，第一次让它等于 memoizedState
    nextWorkInProgressHook = currentlyRenderingFiber$1.memoizedState;
  } else {
    // 不是第一次，始终让它指向下一个 hook，如果这是最后一个，那么 nextWorkInProgressHook 就会是 null
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;
    currentHook = nextCurrentHook;
  } else {
    // 不存在的话会根据上一次的 hook 克隆一个新的 hook，挂在新的链表、FiberNode上。
    if (!(nextCurrentHook !== null)) {
      {
        throw Error( "Rendered more hooks than during the previous render." );
      }
    }

    currentHook = nextCurrentHook;
    var newHook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber$1.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  return workInProgressHook;
}
```
我在代码中加了注释，感兴趣的同学可以看看具体的代码，下面我们大体解释一下这个函数干了什么。

当 <code>react</code> 重新渲染时，会生成一个新的 <code>fiber</code> 树，而这里会根据之前已经生成的 <code>FiberNode</code>  ，拿到之前的  <code>hook</code>  ，再复制一份到新的  <code>FiberNode</code>  上，生成一个新的 <code>hooks</code> 链表。

而这个  <code>hook</code> 是怎么拿的？是去遍历 <code>hooks</code> 链表拿的，所以每次都会按顺序拿下一个 <code>hook</code> ，然后复制到新的 <code>FiberNode</code> 上。可以理解为这个 <code>updateWorkInProgressHook</code> 每次都会按顺序返回下一个 <code>hook</code> 。

拿到这个 <code>hook</code> 之后再根据我们 <code>setState</code> 的值或者其他的一些东西去更新 <code>hook</code> 对象上的属性。这一步也就是 <code>updateMemo</code> 干的事情。

#### hooks只能在顶层使用的原因
其实看到这里你就应该明白为什么 <code>hooks</code> 只能在顶层使用了，因为它会按顺序去拿<code>hook</code>，<code>react</code>也是按顺序来区分不同的 <code>hook</code> 的，它默认你不会修改这个顺序。如果你没有在顶层使用 <code>hook</code> ，打乱了每次 <code>hook</code> 调用的顺序，就会导致 <code>react</code> 无法区分出对应的 <code>hook</code> ，进而导致错误。那你说，如果我不在顶层使用 <code>hooks</code> ，但是我保证它每次都会被调用，这样行不行？行，但是为什么要给自己徒增烦恼去保证它每次都会被调用，老老实实写在顶层不好吗？

### hooks 如何实现一个函数组件能够记住之前的状态
我们知道，一个函数重复运行的时候它的变量都会被销毁，那 <code>react</code> 为什么可以记住上次的变量？因为 <code>react</code> 帮我们把这些变量存了下来。我们之前说到， <code>hook</code> 会以链表的形式被挂在 <code>FiberNode</code> 的 <code>memoizedState</code> 上，你可以把  <code>FiberNode</code> 理解为一个全局变量，它并不会被销毁。所以我们下次 <code>render</code> 的时候就能从这上面拿到上次的 <code>hook</code> ，自然也能拿到 <code>hook</code> 上携带的一些信息，再根据这些信息去 <code>render</code> 新的组件，就能实现函数组件也能有自己的状态了。而 <code>useState</code> , <code>useMemo</code> , <code>useRef</code> 这种带缓存效果的 <code>hooks</code> 的实现原理也显而易见了，我们看一个简单的 <code>useMemo</code> 

```js
function mountMemo(nextCreate, deps) {
  var hook = mountWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
在生成的时候，就是简单的调用了一下 <code>create</code> 函数生成了初始值并返回。

而在更新的时候
```js
function updateMemo(nextCreate, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var prevState = hook.memoizedState;

  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    if (nextDeps !== null) {
      var prevDeps = prevState[1];

      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }

  var nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
会判断一下我们的 <code>deps</code> 依赖是否改变（这里最底层会使用 <code>Object.is</code> 来判断是否相等）,如果改变了，那么再调用一下我们传入的  <code>create</code> 来返回最新的值，如果没有改变，那么就直接返回我们上次的值，进而实现缓存的效果。

怎么拿到上次的 <code>hook</code> ？就是通过我们之前说的<code>updateWorkInProgressHook</code>，那怎么保证两次拿的 <code>hook</code> 是同一个？这就是靠顺序保证了。
### 结语
本文只是简单叙述了 <code>hooks</code> 背后的实现方式，并没有对每个 <code>hook</code> 的具体实现方式做过多的阐述，我相信大家在了解了基本原理之后再去看各个 <code>hook</code> 的实现方式就会简单很多了。同时我后面也会再出一些关于具体 <code>hook</code> 的实现方式解析，和大家一起共同交流学习。
