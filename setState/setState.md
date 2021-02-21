 <code>setState</code> 到底是同步还是异步？很多人可能都有这种经历，面试的时候面试官给了你一段代码，让你说出输出的内容，比如这样：
```js
  constructor(props) {
    super(props);
    this.state = {
      data: 'data'
    }
  }

  componentDidMount() {
    this.setState({
      data: 'did mount state'
    })

    console.log("did mount state ", this.state.data);
    // did mount state data

    setTimeout(() => {
      this.setState({
        data: 'setTimeout'
      })
  
      console.log("setTimeout ", this.state.data);
    })
  }
```
而这段代码的输出结果，第一个 <code>console.log</code> 会输出 <code>data</code> ，而第二个 <code>console.log</code> 会输出 <code>setTimeout</code> 。也就是第一次 <code>setState</code> 的时候，它是异步的，第二次 <code>setState</code> 的时候，它又变成了同步的。是不是有点晕？不慌，我们去源码中看看它到底干了什么。

### 结论
先把结论放到前面，懒得看的同学可以直接看结论了。

只要你进入了 <code>react</code> 的调度流程，那就是异步的。只要你没有进入 <code>react</code> 的调度流程，那就是同步的。什么东西不会进入 <code>react</code> 的调度流程？ <code>setTimeout</code>  <code>setInterval</code>  ，直接在 <code>DOM</code>  上绑定原生事件等。这些都不会走 <code>React</code> 的调度流程，你在这种情况下调用 <code>setState</code> ，那这次 <code>setState</code> 就是同步的。
否则就是异步的。

而 <code>setState</code> 同步执行的情况下， <code>DOM</code> 也会被同步更新，也就意味着如果你多次 <code>setState</code> ，会导致多次更新，这是毫无意义并且浪费性能的。

### scheduleUpdateOnFiber
 <code>setState</code> 被调用后最终会走到 <code>scheduleUpdateOnFiber</code> 这个函数里面来，我们来看看这里面又做了什么：
```js
function scheduleUpdateOnFiber(fiber, expirationTime) {
  checkForNestedUpdates();
  warnAboutRenderPhaseUpdatesInDEV(fiber);
  var root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);

  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return;
  }

  checkForInterruption(fiber, expirationTime);
  recordScheduleUpdate(); // TODO: computeExpirationForFiber also reads the priority. Pass the
  // priority as an argument to that function and this one.

  var priorityLevel = getCurrentPriorityLevel();

  if (expirationTime === Sync) {
    if ( // Check if we're inside unbatchedUpdates
    (executionContext & LegacyUnbatchedContext) !== NoContext && // Check if we're not already rendering
    (executionContext & (RenderContext | CommitContext)) === NoContext) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      schedulePendingInteractions(root, expirationTime); // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.

      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root);
      schedulePendingInteractions(root, expirationTime);

	  // 重点!!!!!!
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    ensureRootIsScheduled(root);
    schedulePendingInteractions(root, expirationTime);
  }

  if ((executionContext & DiscreteEventContext) !== NoContext && ( // Only updates at user-blocking priority or greater are considered
  // discrete, even inside a discrete event.
  priorityLevel === UserBlockingPriority$1 || priorityLevel === ImmediatePriority)) {
    // This is the result of a discrete event. Track the lowest priority
    // discrete update per root so we can flush them early, if needed.
    if (rootsWithPendingDiscreteUpdates === null) {
      rootsWithPendingDiscreteUpdates = new Map([[root, expirationTime]]);
    } else {
      var lastDiscreteTime = rootsWithPendingDiscreteUpdates.get(root);

      if (lastDiscreteTime === undefined || lastDiscreteTime > expirationTime) {
        rootsWithPendingDiscreteUpdates.set(root, expirationTime);
      }
    }
  }
}
```
我们着重看这段代码：
```js
if (executionContext === NoContext) {
  // Flush the synchronous work now, unless we're already working or inside
  // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
  // scheduleCallbackForFiber to preserve the ability to schedule a callback
  // without immediately flushing it. We only do this for user-initiated
  // updates, to preserve historical behavior of legacy mode.
  flushSyncCallbackQueue();
}
```
 <code>executionContext</code> 代表了目前 <code>react</code> 所处的阶段，而 <code>NoContext</code> 你可以理解为是 <code>react</code> 已经没活干了的状态。而 <code>flushSyncCallbackQueue</code> 里面就会去同步调用我们的 <code>this.setState</code> ，也就是说会同步更新我们的 <code>state</code> 。所以，我们知道了，当 <code>executionContext</code> 为 <code>NoContext</code> 的时候，我们的 <code>setState</code> 就是同步的。那什么地方会改变 <code>executionContext</code> 的值呢？
 
我们随便找几个地方看看
```js
function batchedEventUpdates$1(fn, a) {
  var prevExecutionContext = executionContext;
  executionContext |= EventContext;
  ...省略
}

function batchedUpdates$1(fn, a) {
  var prevExecutionContext = executionContext;
  executionContext |= BatchedContext;
  ...省略
}
```
当 <code>react</code> 进入它自己的调度步骤时，会给这个 <code>executionContext</code> 赋予不同的值，表示不同的操作以及当前所处的状态，而 <code>executionContext</code> 的初始值就是 <code>NoContext</code> ，所以只要你不进入 <code>react</code> 的调度流程，这个值就是 <code>NoContext</code> ，那你的 <code>setState</code> 就是同步的。

### useState的setState
自从 <code>raect</code> 出了 <code>hooks</code> 之后，函数组件也能有自己的状态，那么如果我们调用它的 <code>setState</code> 也是和 <code>this.setState</code> 一样的效果吗？

对，因为 <code>useState</code> 的 <code>set</code> 函数最终也会走到 <code>scheduleUpdateOnFiber</code> ，所以在这一点上和 <code>this.setState</code> 是没有区别的。

但是值得注意的是，我们调用 <code>this.setState</code> 的时候，它会自动帮我们做一个 <code>state</code> 的合并，而 <code>hook</code> 则不会，所以我们在使用的时候要着重注意这一点。

举个🌰
```js
state = {
  data: 'data',
  data1: 'data1'
};

this.setState({ data: 'new data' });
console.log(state);
//{ data: 'new data',data1: 'data1' }

const [state, setState] = useState({ data: 'data', data1: 'data1' });
setState({ data: 'new data' });
console.log(state);
//{ data: 'new data' }
```

但是如果你自己去尝试在 <code>function</code> 组件的 <code>setTimeout</code> 中去调用 <code>setState</code> 之后，打印 <code>state</code> ，你会发现他并没有改变，这时你就会很疑惑，为什么呢？这不是同步执行的吗？

这是因为一个闭包问题，你拿到的还是上一个 <code>state</code> ，那打印出来的值自然是上一次的，此时真正的 <code>state</code> 已经被改变了。那有没有其他的方法可以观察到 <code>function</code> 函数的同步行为？有，我们下面再介绍。

### 案例分析
 <code>setTimeout</code> 、原生事件内调用 <code>setState</code> 的操作确实比较少见，但是下面这种写法一定很常见了。
```js
  fetch = async () => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve('fetch data');
      }, 300);
    })
  }

  componentDidMount() {
    (async () => {
      const data = await this.fetch();
      this.setState({data});
      console.log("data: ", this.state);
      // data: fetch data
    })()
  }
```
我们在 <code>didMount</code> 的时候发了一个请求，然后再将结果 <code>setState</code> ，这时候我们用了 <code>async/await</code> 来进行处理。

这时候我们会发现其实 <code>setState</code> 也会变成同步了，为什么呢？因为<code>componentDidMount</code>执行完毕后，就已经退出了 <code>react</code> 的调度，而我们请求的代码还没有执行完毕，等结果请求回来以后 <code>setState</code> 才会执行。<code>async</code> 函数中 <code>await</code> 后面的代码其实是异步执行的。这和我们在 <code>setTimeout</code> 中执行 <code>setState</code> 其实是一个效果，所以我们的 <code>setState</code> 就变成同步的了。

如果它变成同步会有什么坏处呢？我们来看看如果我们多次调用了 <code>setState</code> 会发生什么。
```js
this.state = {
  data: 'init data',
}

componentDidMount() {
    setTimeout(() => {
      this.setState({data: 'data 1'});
      // console.log("dom value", document.querySelector('#state').innerHTML);
      this.setState({data: 'data 2'});
      // console.log("dom value", document.querySelector('#state').innerHTML);
      this.setState({data: 'data 3'});
      // console.log("dom value", document.querySelector('#state').innerHTML);
    }, 1000)
}

render() {
  return (
    <div id="state">
      {this.state.data}
    </div>
  );
}
```
这是在浏览器运行的结果

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/912aad28300f44dc863a5503dd9b202f~tplv-k3u1fbpfcp-watermark.image)

这样来看的话，其实也并没有什么，每次刷新后最终还是会显示 <code>data 3</code> ，但是我们将代码中 <code>console.log</code> 的注释去掉，再看看：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a888eab8ec034afd9edfbd62fe2925ad~tplv-k3u1fbpfcp-watermark.image)
我们每次都能在 <code>DOM</code> 上拿到最新的 <code>state</code> ，这是因为 <code>react</code> 已经把 <code>state</code> 的修改同步更新了，但是为什么界面没有显示出来？因为对浏览器来说，渲染线程 和 js线程 是互斥的， <code>react</code> 代码运行时浏览器是没办法渲染的。所以实际上我们已经把 <code>DOM</code> 更新了，但是 <code>state</code> 又被修改了， <code>react</code> 只好再做一次更新，这样反复了三次，最后 <code>react</code> 代码执行完毕后，浏览器才把最终的结果渲染到了界面上。这也就意味着其实我们已经做了两次无用的更新。

我们把 <code>setTimeout</code> 去掉，就会发现三次都输出了 <code>init data</code> ，因为此时的 <code>setState</code> 是异步的，会把三次更新合并到一次去执行。

所以当 <code>setState</code> 变成同步的时候就要注意，不要写出让 <code>react</code> 多次更新组件的代码，这是毫无意义的。

而这里也回答了之前提出的问题，如果我们想在 <code>function</code> 函数中观察到同步流程，大家可以去试试当你在 <code>setTimeout</code> 中 <code>setState</code> 之后， <code>DOM</code> 里面的内容会不会改变。

### 结语
 <code>react</code> 已经帮助我们做了很多优化措施，但是有时候也会因为代码不同的实现方式而导致 <code>react</code> 的性能优化失效，相当于我们自己做了反优化。所以理解 <code>react</code> 的运行原理对我们日常开发确实是很有帮助的。
