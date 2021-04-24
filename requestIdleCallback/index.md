# 如何才能更好的压榨浏览器?
各位点进来的兄弟姐妹, 浏览器这么兢兢业业的为你服务, 为何你还要想方设法去压榨它, 你的内心不会觉得愧疚吗?!

好了, 开个玩笑, 言归正传, 我们今天的主角是 <code>requestAnimationFrame</code> 和 <code>requestIdleCallback</code> 。那么它们和压榨浏览器有什么关系呢?借助它们我们可以深入浏览器内部渲染的生命周期, 利用好一切我们可以利用的时间, 榨干浏览器的每分每秒。

## requestAnimationFrame
首先我们来说说 <code>requestAnimationFrame</code>，从这个名字就可以看出来, 它可以用在动画上, 我们先来考虑这样一个场景, 用<code>js</code>实现一个<code>div</code>向右滑动的动画，我们第一反应可能是利用 setInterval
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <style>
    #animation {
      width: 100px;
      height: 100px;
      background-color: red;
      position: absolute;
    }
  </style>
</head>
<body>
  <div id="animation" style="left: 10px"></div>
  <script>
    setInterval(() => {
      const div = document.querySelector('#animation');
      const left = div.getBoundingClientRect().left;
      div.style.left = `${left+2}px`;
    }, 100);
  </script>
</body>
</html>
```
so easy, 我们来看一眼效果

看起来其实还行？但是如果你仔细看的话就会发现, 它有一点不自然, 有一种掉帧的感觉。这是为什么呢，因为如果帧数是60fps，那么就是1s内刷新60次, 也就是 1/60 s刷新一次, 但是我们的时间设置为了100ms，这个时间和浏览器的刷新时间对不上，所以造成了一种动画不够流畅的感觉，我们可以调整间隔，但是最好的方式就是让浏览器自己控制自己，让我们的函数每一帧只执行一次，因为执行多了没必要，执行少了不行，一次刚刚好。<code>requestAnimationFrame</code> 就是用来干这个事情的，它接受一个函数，这个函数一帧内只会被调用一次，调用时间由浏览器决定，所以我们把代码改一下。
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <style>
    #animation {
      width: 100px;
      height: 100px;
      background-color: red;
      position: absolute;
    }
  </style>
</head>
<body>
  <div id="animation" style="left: 10px"></div>
  <script>
    // setInterval(() => {
    //   const div = document.querySelector('#animation');
    //   const left = div.getBoundingClientRect().left;
    //   div.style.left = `${left+2}px`;
    // }, 100);

    function animation() {
      requestAnimationFrame(() => {
        const div = document.querySelector('#animation');
        const left = div.getBoundingClientRect().left;
        div.style.left = `${left+2}px`;
        animation();
      })
    }

    animation();
  </script>
</body>
</html>
```
因为每次只会执行一次, 所以我们需要递归调用这个函数, 我们再来看看效果

怎么样, 是不是感觉流畅很多, 脏活累活丢给浏览器干就行了, 用<code>setInterval</code> 还得算算间隔多少比较合适, 没必要, <code>requestAnimationFrame</code>简单又实用。

## requestIdleCallback
下面我们来说说 <code>requestIdleCallback</code>，它的作用是什么呢？我们知道，浏览器的渲染是有一套流程的，比如A、B、C、D四个步骤，一次渲染就是A->B->C->D，然后我们就能在界面上看见渲染出来的东西了，想了解具体的内容可以藏考我之前写的[一篇文章](https://juejin.cn/post/6844904121292570632)。那如果浏览器1s渲染60帧，一帧花费的时间是 1/60 s，结果还没花完，活就干完了，这咋办。浏览器表示：那不是刚好摸鱼吗？那资本家会放轻易过你吗？不行，每一秒都得压榨干净，你没事干了我可以给事情给你干，<code>requestIdleCallback</code>，通过它我们就能给浏览器派更多的活，让它在这种空闲时间有事情可干。

<code>requestIdleCallback</code> 接受一个函数作为参数，这个函数会在浏览器空闲的时候被调用，同时这个函数还接受一个参数，更具体的用法还是参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)，我这里就不作更多的介绍了。那通过它能干什么？我们知道<code>react</code>更新了<code>fiber</code>架构，这是个什么玩意，简单来说，以前如果一个组件非常庞大，下面的子组件非常多，它也只能一次性渲染完，无法中断。此时如果来了其他的任务，就会被阻塞，所以在用户看来某些任务无法及时得到响应，那就是界面一卡一卡的，掉帧了。

而更新了<code>fiber</code>架构之后呢，<code>react</code>可以做到中断更新，去响应一些优先级更高的任务，我们举一个简单的小例子：
```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <script>
    const taskList = [
      {
        id: 1,
        msg: 'first task'
      },
      {
        id: 2,
        msg: 'second task'
      },
      {
        id: 3,
        msg: 'third task'
      },
      {
        id: 4,
        msg: 'four task'
      }
    ]

    function wait(time) {
      const now = Date.now();

      while (Date.now() - now < time) {}
    }

    function oldExecuteTask(list = taskList) {
      list.forEach(item => {
        console.log("execute task", item.msg);
        wait(1000);
      })
    }

    function newExecuteTask(list = taskList) {
      requestIdleCallback(() => {
        const firstList = list[0];
        console.log(`execute task ${firstList.msg}`);
        wait(1000);

        list.length > 1 && newExecuteTask(list.slice(1));
      }, { timeout: 2000 })
    }
  </script>
</head>

<body>
  <button onclick="oldExecuteTask()">execute old task</button>
  <button onclick="newExecuteTask()">execute new task</button>
  <button onclick="(() => {console.log('other task')})()">other task</button>
</body>

</html>
```
此时我们有一个任务队列<code>taskList</code>，当我们执行这个任务队列时，我们去点击<code>other task</code>按钮，此时因为之前的任务没有执行完，老<code>react</code>是无法响应的，而新<code>react</code>因为有中断机制所以可以响应，我们首先看看老<code>react</code>的表现

当无法中断时，我疯狂点击按钮也不会有<code>console</code>打出来，当任务执行完毕后才打出了之前的<code>console</code>，而用了<code>requestIdleCallback</code>之后我们每次只执行一个任务，下个任务再次通过<code>requestIdleCallback</code>来调用，保证执行我们任务的同时还能响应一些其他的任务，我们看效果

可以看到，我们在执行任务的同时还能响应其他的任务，这就和<code>react</code>的新架构类似。所以，通过<code>requestIdleCallback</code>，我们就能更好的利用好各种碎片时间来执行优先级不那么高的任务，不要让浏览器白白浪费掉多余的时间。

## 结语
其实，在我看来，这两个api给我们提供了深入浏览器内部生命周期的机会，也就是给了我们很大的发挥空间，让我们能够能够更好的去压榨浏览器了。目前来看<code>requestIdleCallback</code>的兼容性还不是非常好，所以希望用在实际生产环境的同学还需要仔细调研一下。