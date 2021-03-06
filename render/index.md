俗话说得好, 知其然更要知其所以然, 我相信现在大部人都用过<code>react</code>, 但是对其中的原理又了解多少?今天这篇文章就是为了介绍<code>react</code>的原理。文中不会涉及过多的源码, 但是里面的内容都是依据源码(v16.13.1)解析而来, 大家可以把这篇文章当作一篇科普类文章来阅读。如果发现错误也欢迎在评论区指正。

## react 是什么
在<code>react</code>的官网上, 它介绍自己是**用于构建用户界面的 JavaScript 库**, 那么它干的事情其实非常简单, 就是在<code>DOM</code>结点和你之间搭了一座桥, 帮你屏蔽了一些底层的操作, 让你能够用更简单的方式去改变你的用户界面。

那这座桥是怎么搭起来的？其实也很简单, 即使它再复杂, 也是<code>js</code>的产物。所以只要你了解<code>js</code>的语法, 你也能写出一个<code>react</code>。但是重点就在于你如何组织你的代码、如何实现这些功能。打个比方, 我们平时开的车, 很复杂。 你根本不会去关注它的内部结构是什么, 又是怎么组织起来的。但是把车的各个零件拆开摆在你面前, 你会知道, 哦这个是铁, 那个是橡胶, 这些零件我都认识。同时，了解了车的内部原理才能更好的开车，遇见什么小毛病自己就给修了。

那么今天, 我们就把<code>react</code>拆开来看一看, 看看它到底是怎么把我们写的代码最终渲染到浏览器上的。


## 直捣黄龙
我们直接跳过最开始那些花里胡哨的步骤, 来看看<code>react</code>最后做的操作, <code>removeAttribute</code>熟悉吗? <code>removeChild</code>熟悉吗?整到最后还是会回归到这些基础的<code>api</code>，所以，不要对它怀着一颗恐惧的心态, 其实，也就是那么回事。
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

#### JSX 到 js 对象
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
<code>fiber</code>对象是什么?本质上就是一个<code>js</code>对象。但是它更加的复杂。
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

 每一个组件就是一个<code>fiber</code>对象，各个组件之间连接起来就是一颗<code>fiber</code>树，和<code>DOM</code>树很像，可以说<code>fiber</code>树和<code>DOM</code>树是对应起来的。为什么要搞这样一个<code>fiber</code>树？你可曾听闻过<code>diff</code>算法。<code>fiber</code>树就是<code>js</code>对象，所以操纵它会比操纵<code>DOM</code>结点要快不少。我们把<code>diff</code>的操作放在<code>fiber</code>树上,最后再得出真正需要更新的结点映射到真实<code>DOM</code>，优化性能。
 
 其实最重要的一个原因就是这是<code>react</code>为了避免在计算<code>Virtual Dom</code>的时候一直占用<code>js</code>线程导致无法响应其他事件特意推出的新架构，今天我们不在这里谈关于这个的内容，我们首先把<code>react</code>的主流程搞明白。
 #### fiber 对象的 diff 算法
 <code>diff</code>算法的目的是什么？找出可以复用的<code>fiber</code>结点，节约创建新结点的时间，找到真正需要更新的部分。我们用图说话
 
![](https://klx-buct.github.io/klx-buct.github.iogithub/render/newFiber.png)

 这里有五个组件, 他们一起组成了一颗<code>Fiber</code>树, 在一次<code>setState</code>之后，组件<code>D</code>变成了组件<code>E</code>, 变成了下面这样
 
![](https://klx-buct.github.io/klx-buct.github.iogithub/render/oldFiber.png)

 那第二颗<code>fiber</code>树会如何生成?<code>react</code>通过<code>diff</code>，会知道<code>App</code> 、<code>A</code>、<code>B</code>、<code>C</code>四个组件都没有改变，进而复用之前的<code>fiber</code>结点。而<code>D</code>组件变成了<code>E</code>组件，<code>react</code>就会帮它重新生成一个<code>fiber</code>结点。
 
 那么<code>diff</code>的具体过程是什么？

 简单来说，如果是单个结点，只要<code>key</code>和<code>type</code>（结点类型）相同，<code>react</code>就会进行复用。不给<code>key</code>的情况下默认是<code>null</code>。在内部比较时<code>null === null</code>，所以此时在<code>type</code>相同的情况下也会进行复用。

 如果是多个结点，也是比较<code>key</code>和<code>type</code>，此时不给<code>key</code>的情况下会默认将<code>index</code>作为<code>key</code>。在这种情况下，除了按顺序比较<code>key</code>和<code>type</code>之外，还针对结点更换位置的情况做了优化，会根据<code>key</code>值去列表中寻找对应的结点，尝试复用。

 可以看出，<code>react</code>会尽可能的去复用之前的结点, 避免创建新的结点。而这里的复用指的是复用之前的<code>fiber</code>结点，并不意味着不会去更新真实<code>DOM</code>。
 
 想了解更多内容的可以参考我之前的[文章](https://juejin.cn/post/6923208323516874766)

 #### 新 fiber 树到更新链表
 我们把后面这颗<code>fiber</code>树称为新<code>fiber</code>树。
 
 对于<code>react</code>来说，一切都是<code>props</code>，比如<code>DOM</code>结点上的<code>style</code>、<code>src</code>、<code>data-props</code>等等，里面包裹的内容也是<code>props.children</code>。所以<code>react</code>在生成新的<code>fiber</code>后，会去对比新旧<code>props</code>的区别，如果有改变就会把它加到更新链表中去。更新链表是什么？我们再次用图说话

![](https://klx-buct.github.io/klx-buct.github.iogithub/render/update.png)

 <code>fiber</code>结点上有一个属性叫做<code>nextEffect</code>，它会把各个需要更新的结点连接起来，所以当<code>react</code>发现<code>props</code>有改变的时候，就会将其加入到这个链路中来。

 #### 最终渲染
 最后<code>react</code>会走到<code>commit</code>阶段，在这里它会去遍历之前生成的更新链表，然后把这些内容真正的更新到用户界面上，至此，一次<code>render</code>流程就结束了，我们看到的界面也更新了。
 
 对于我们上面这张图来说，<code>react</code>最终就会去更新<code>A</code>、<code>B</code>、<code>D</code>三个组件。
 
 也就是在这一步, <code>react</code>会去调用js操纵<code>DOM</code>的<code>api</code>来更新用户界面。

 ## 渲染过程的优化

 ### 避免重复渲染
 我们上面把<code>react</code>的整个渲染流程梳理了一遍, 那么在我们更新<code>state</code>的时候不可能把整个<code>fiber</code>树都重新更新一遍，什么时候才会触发<code>react</code>的更新流程呢？

 **只要<code>props</code>改变了，就会触发<code>react</code>的更新流程**

这句话看似简单，实则另有玄机，我们来看看这个例子：
```jsx
class App extends Component {
  state = {data: "hello world", src: logo};

  data = {data: 'test'}

  render() {
    return (
      <div>
        <Item data={{data: this.data}}/>
        <button onClick={() => this.setState({data: "new data", src: logo2})}>setState</button>
      </div>
    );
  }
}

export default App;
```
如果我点击<code>setState</code>，传给<code>Item</code>的<code>data</code>会不会更新？我们这样想，<code>data</code>是挂在<code>this</code>上的一个对象，它其实是不会改变的，那这样想起来我们两次给的<code>props</code>其实是没有改变的。

但是真实情况却是<code>Item</code>组件每次都会重新经历一遍<code>diff</code>流程，为什么呢？还记得我们之前说<code>JSX</code>会调用一个<code>createElement</code>吗，这个函数会对<code>props</code>再包装一层，生成了一个新的<code>props</code>。所以即使我们传给<code>Item</code>的值没有改变，传到<code>React</code>那里还是变了。

这个问题和<code>Context</code>的重复渲染问题很像，我们再来看下面的代码:
```jsx
//app.js
class App extends React.Component {
  state = {
    data: 'old data'
  }

  changeState = data => {
    this.setState({
      data
    })
  }

  render() {
    return (
      <CustomContext.Provider value={{data: this.state, setData: this.changeState}}>
        <div className="App">
          <ContextContent />
          <OtherContent />
        </div>
      </CustomContext.Provider>
    );
  }
}

export default App;

//ContextContent.js
export default class ContextContent extends Component {
  render() {
    console.log("Context render")
    return <CustomContext.Consumer>
        {value => (
          <div>
            <div>{value.data.data}</div>
            <button onClick={() => value.setData(Math.random())}>change</button>
          </div>
        )}
      </CustomContext.Consumer>
  }
}

//OtherContent.js
export default class OtherContent extends Component {
  render() {
    console.log("otherContent render")
    return (
      <>
        <div>other content</div>
      </>
    )
  }
}
```
我们通过<code>Context</code>给里面的组件传值，当我们改变<code>state</code>的时候，当然只希望用到了这个值的组件重新渲染，但是目前这种写法会导致什么结果呢？

![](https://klx-buct.github.io/klx-buct.github.iogithub/render/context.gif)

可以看到，我们每次改变<code>state</code>的时候，另一个无关的组件也更新了。为什么？因为我们每次都调用了<code>createElement</code>，每次都生成了一个新的<code>props</code>（即使这里我们没有给任何<code>props</code>，<code>react</code>最后拿到的<code>props</code>其实是<code>createElement</code>内部生成的）。

那么问题如何解决？我们想办法让<code>props</code>不要改变，也就是想办法不要每次都让无关组件调用<code>createElement</code>，具体解决方法我这里就不给出了，让大家思考一下。

这也是为什么<code>Context</code>使用不当会有性能问题的原因。想想在组件非常多的情况下，如果你在最外层给了一个<code>Provider</code>，又去频繁的改变它的<code>Value</code>，那么结果可想而知，每次都会重新<code>diff</code>一遍整个<code>fiber</code>树。

注意我这里说的是重新<code>diff</code>而不是重新渲染。<code>react</code>走到最后会发现其实任何东西都没有改变，所以它并不会重新渲染这个组件，也就是不会去更新用户界面上的内容。但是<code>react</code>耗时的部分其实恰恰就在<code>diff</code>，你可以理解为你做了一大堆工作来判断这个组件什么地方需要更新，最后得出的结论是这个组件并不需要更新。

### 给组件加 key 值
其实我们之前有提到过, <code>react</code>会尽可能的去复用<code>fiber</code>结点。所以即使你没有特意加<code>key</code>，只要你不随意改变<code>DOM</code>结构，<code>react</code>还是会去复用各个结点。

而在渲染数组结构时，<code>react</code>默认会拿<code>index</code>作为<code>key</code>，如果此时你随意改变各个结点的位置，可能就会导致<code>react</code>的优化失效。在这种情况下我们需要给每个结点一个真正能代表它本身的<code>key</code>，这样才能保证在顺序改变时<code>react</code>还能认出他们。

那么加了<code>key</code>之后的用处是什么？如果<code>key</code>不变的情况下还会重新渲染吗？

加了<code>key</code>之后<code>react</code>在构建<code>fiber</code>树的时候就会尝试去复用之前的结点。所以它带来的优化是<code>diff</code>过程构建结点的时候更快了，后面也会照常的对比<code>props</code>然后加入到更新链表中。

那么有没有一种方式是可以告诉<code>react</code>，我这个组件啥都没改，你别给我动它，啥也别管。<code>shouldComponentUpdate</code>就起到了一个这样的作用。

### shouldComponentUpdate
通过给组件加<code>key</code>的方法确实有效，但是<code>react</code>最终还是走了<code>diff</code>流程。

而生命周期中有一个叫做<code>shouldComponentUpdate</code>的，从名字就可以看出来：组件是否应该更新，如果我们返回<code>false</code>的话，那么<code>react</code>是不会去进行<code>diff</code>流程，也不会去对比<code>props</code>，更不会去更新用户界面上的组件了。

我们平时可以通过继承<code>PurComponent</code>组件来达到一个简单的优化效果，继承了PurComponent之后会对<code>props</code>进行一层浅比较，在<code>props</code>没有改变的情况下不会去重新走<code>diff</code>更新流程。

而我们在使用<code>function</code>组件的时候也可以通过<code>React.memo</code>达到相同的效果。

比如我在上文的<code>Context</code>例子中稍微修改一下, 把<code>Component</code>改为<code>PurComponent</code>
```js
//otherContent.js
export default class OtherContent extends PureComponent {
  render() {
    console.log("otherContent render")
    return (
      <>
        <div>other content</div>
      </>
    )
  }
}
```
我们再来看效果

![](https://klx-buct.github.io/klx-buct.github.iogithub/render/purRender.gif)

这样每次改变的时候<code>otherContent</code>组件就不会再走一遍<code>diff</code>流程了。

## react 和 原生比谁更快？
不会吧不会吧，不会现在还有人认为<code>react</code>会快过原生操作吧。我们之前说了这么多，从<code>JSX</code>到<code>fiber</code>到渲染到界面上，<code>react</code>绕了前面那么一大圈才到了最后的调用<code>js</code>更新用户界面的流程，怎么可能比你直接调用<code>js</code>去更新用户界面来的快。

我们之所以用<code>react</code>开发而不是用原生，并不是因为它快，而是因为它方便。我们能用<code>react</code>更快速高效的开发出一种新产品，而<code>react</code>虚拟<code>DOM</code>的快指的是它在给我们提供了这种便利的前提下，还能给我们保持一定的性能。但是一定是会比你直接写原生慢的。

那当然，你直接用原生写的代码可维护性和可阅读性以及开发效率上也是一定比不上<code>react</code>的

## 结语
好了，这就是今天的全部内容了，本文主要介绍了<code>react</code>是如何把我们的代码渲染到浏览器上的，其实这里面还有很多学问和细节没有在文中列出。我们再用一张图来总结<code>react</code>的整个流程

![](https://klx-buct.github.io/klx-buct.github.iogithub/render/summary.png)

其实第一次渲染和后续的渲染略有不同，我这里没有太区别开来，但是大体是这样一个流程。

同时如果文中有什么错误的地方，还请大家在评论区中指出