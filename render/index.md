# react 到底干了什么？
俗话说得好, 知其然更要知其所以然, 我相信现在大部人都用过<code>react</code>, 但是对其中的原理又了解多少?今天这篇文章就是为了介绍<code>react</code>的原理。本文不会涉及过多的源码, 但是文中的内容都是依据源码(v16.13.1)解析而来, 大家可以把本文当作一篇科普类文章来阅读。如果发现错误也欢迎在评论区指正。

## react 是什么
在<code>react</code>的官网上, 它介绍自己是**用于构建用户界面的 JavaScript 库**, 那么它干的事情其实非常简单, 就是在<code>DOM</code>结点和你之间搭了一座桥, 帮你屏蔽了一些底层的操作, 让你能够用更简单的方式去改变你的用户界面。

那这座桥是怎么搭起来的？其实很简单, 即使它再复杂, 它也是<code>js</code>的产物, 所以只要你了解<code>js</code>的语法, 你也能写出一个<code>react</code>。但是重点就在于你如何组织你的代码、如何实现这些功能。打个比方, 我们平时开的车, 很复杂。 你平时根本不会去关注它的内部结构是什么, 又是怎么组织起来的。但是把车的各个零件拆开摆在你面前, 你会知道, 哦这个是铁, 那个是橡胶, 这些零件我都认识。

那么今天, 我们就把<code>react</code>拆开来看一看, 看看它到底是怎么把我们写的代码最终渲染到浏览器上的。


## 直捣黄龙
我们直接跳过最开始那些花里胡哨的步骤, 直接来看看<code>react</code>最后做的操作, <code>removeAttribute</code>熟悉吗? <code>removeChild</code>熟悉吗?整到最后还是会回归到这些基础的<code>api</code>，所以，不要对它怀着一颗恐惧的心态, 其实，也就是那么回事。
```js
if (value === null) {
  node.removeAttribute(_attributeName);
} else {
  node.setAttribute(_attributeName,  '' + value);
}

while (node.firstChild) {
  node.removeChild(node.firstChild);
}

while (svgNode.firstChild) {
  node.appendChild(svgNode.firstChild);
}
```

## 干货开始
好了下面就开始我们今天的正文, <code>react</code>到底干了什么。

### 从 JSX 到 用户界面
我们在<code>react</code>里面写的最多的东西就是<code>JSX</code>了，这是个什么玩意？啥也不是，你可以把它理解为<code>react</code>的一种规定，你按着它的规定写，它就能理解你写的东西，你不按它的规定写，它就不能理解你写的东西。

#### JSX 到 js对象
<code>react</code>会怎么处理我们写的<code>JSX</code>?它会通过一个函数叫做<code>createElement</code>来把我们写的东西变成一个固定的结构
```js
var element = {
  // This tag allows us to uniquely identify this as a React Element
  $$typeof: REACT_ELEMENT_TYPE,
  // Built-in properties that belong on the element
  type: type,
  key: key,
  ref: ref,
  props: props,
  // Record the component responsible for creating this element.
  _owner: owner
};
```
这玩意在<code>react</code>内部就相当于一个<code>DOM</code>结点。那我们为什么要写<code>JSX</code>？

其实你完全可以手动调用<code>createElement</code>来创建结点, 或者<code>react</code>官方直接只接受上文的<code>element</code>对象这种写法来形容一个组件。那<code>react</code>就凉了，这么难用的东西谁会用，所以写<code>JSX</code>只是为了方便你理解，而<code>createElement</code>是把方便你理解的东西转换为方便<code>react</code>理解的东西。

#### js 对象到 fiber 对象
<code>fiber</code>对象是什么?本质上还是一个<code>js</code>对象。但是它更加的复杂。
```js
function FiberNode(tag, pendingProps, key, mode) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null; // Fiber

  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;
  this.ref = null;
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;
  this.mode = mode; // Effects

  this.effectTag = NoEffect;
  this.nextEffect = null;
  this.firstEffect = null;
  this.lastEffect = null;
  this.expirationTime = NoWork;
  this.childExpirationTime = NoWork;
  this.alternate = null;

  {
    // Note: The following is done to avoid a v8 performance cliff.
    //
    // Initializing the fields below to smis and later updating them with
    // double values will cause Fibers to end up having separate shapes.
    // This behavior/bug has something to do with Object.preventExtension().
    // Fortunately this only impacts DEV builds.
    // Unfortunately it makes React unusably slow for some applications.
    // To work around this, initialize the fields below with doubles.
    //
    // Learn more about this here:
    // https://github.com/facebook/react/issues/14365
    // https://bugs.chromium.org/p/v8/issues/detail?id=8538
    this.actualDuration = Number.NaN;
    this.actualStartTime = Number.NaN;
    this.selfBaseDuration = Number.NaN;
    this.treeBaseDuration = Number.NaN; // It's okay to replace the initial doubles with smis after initialization.
    // This won't trigger the performance cliff mentioned above,
    // and it simplifies other profiler code (including DevTools).

    this.actualDuration = 0;
    this.actualStartTime = -1;
    this.selfBaseDuration = 0;
    this.treeBaseDuration = 0;
  } // This is normally DEV-only except www when it adds listeners.
  // TODO: remove the User Timing integration in favor of Root Events.


  {
    this._debugID = debugCounter++;
    this._debugIsCurrentlyTiming = false;
  }

  {
    this._debugSource = null;
    this._debugOwner = null;
    this._debugNeedsRemount = false;
    this._debugHookTypes = null;

    if (!hasBadMapPolyfill && typeof Object.preventExtensions === 'function') {
      Object.preventExtensions(this);
    }
  }
}
```

 每一个组件就是一个<code>fiber</code>对象，各个组件之间连接起来就是一颗<code>fiber</code>树，和<code>DOM</code>树很像，可以说<code>fiber</code>树和<code>DOM</code>树是对应起来的。为什么要搞这样一个<code>fiber</code>树？你可曾听闻过<code>diff</code>算法。fiber树本质上还是<code>js</code>对象，所以我们操纵它比操纵<code>DOM</code>结点要快不少，所以我们会<code>diff</code>的操作放在<code>fiber</code>树上,最后再映射到真实<code>DOM</code>，优化性能。

 #### fiber 对象的 diff 算法
 <code>diff</code>算法的目的是什么？找出可以复用的<code>fiber</code>结点，节约创建新结点的时间。我们用图说话
![](https://klx-buct.github.io/klx-buct.github.iogithub/render/oldFiber.png)
 我们这里有五个组件, 他们一起组成了一颗<code>Fiber</code>树, 在一次<code>setState</code>之后，组件<code>D</code>变成了组件<code>E</code>, 变成了下面这样
![](https://klx-buct.github.io/klx-buct.github.iogithub/render/newFiber.png)
 那第二颗<code>fiber</code>树会如何生成?<code>react</code>通过<code>diff</code>，会知道<code>App</code> 、<code>A</code>、<code>B</code>、<code>C</code>四个组件都没有改变，会复用之前的<code>fiber</code>结点。而<code>D</code>组件变成了<code>E</code>组件，<code>react</code>就会帮它重新生成一个<code>fiber</code>结点。
 
 那么<code>diff</code>的具体过程是什么？

 简单来说，如果是单个结点，只要<code>key</code>和<code>type</code>（结点类型）相同，<code>react</code>就会进行复用。不给<code>key</code>的情况下默认是<code>null</code>。在内部比较时<code>null === null</code>，所以此时在<code>type</code>相同的情况下也会进行复用。

 如果是多个结点，也是比较<code>key</code>和<code>type</code>，此时不给<code>key</code>的情况下会默认将<code>index</code>作为<code>key</code>。在这种情况下，除了按顺序比较<code>key</code>和<code>type</code>之外，还针对结点更换位置的情况做了优化，会根据<code>key</code>值去列表中寻找对应的结点，尝试复用。

 可以看出，<code>react</code>会尽可能的去复用之前的结点, 避免创建新的结点。

 #### 新 fiber 树到更新链表
 对于<code>react</code>来说，一切都是<code>props</code>，比如<code>DOM</code>结点上的<code>style</code>、<code>src</code>、<code>data-props</code>等等，而里面包裹的内容就是<code>props.children</code>，所以<code>react</code>在生成新的<code>fiber</code>后，会去对比新旧<code>props</code>的区别，如果有改变就会把它加到更新链表中去。更新链表是什么？我们再次用图说话
 ![](https://klx-buct.github.io/klx-buct.github.iogithub/render/update.png)

 <code>fiber</code>结点上有一个属性叫做<code>nextEffect</code>，它会把各个需要更新的结点连接起来，所以当<code>react</code>发现<code>props</code>有改变的时候，就会将其加入到这个链路中来。

 #### 最终渲染
 最后<code>react</code>会走到<code>commit</code>阶段，在这里它会去遍历之前生成的更新链表，然后把这些内容真正的更新到用户界面上，至此，一次<code>render</code>流程就结束了，我们看到的界面也更新了。