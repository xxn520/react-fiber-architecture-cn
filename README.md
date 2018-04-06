# React Fiber Architecture ([原文](https://github.com/acdlite/react-fiber-architecture))

## 简介

React Fiber 是对 React 核心算法的重新实现，目前正在进行中，[进度传送门](http://isfiberreadyyet.com/)。这是 React 团队过去两年研究的成果。

React Fiber 的目标是提高其对动画，布局和手势等领域的适用性。它的最重要的特性是 **incremental rendering（增量渲染）**：它能够将渲染 work 拆分成多块并将这些任务块分散到多个帧执行。

其他的核心特性还包括当新的更新到来时暂停、中止或恢复 work 的能力；为不同类型的更新设置优先级的能力；以及新的并发原语。

### 关于这篇文档

Fiber 介绍了一些难以直接通过阅读代码理解的新颖的概念。这篇文档源于我在跟踪 React 项目中对 Fiber 的实现时所收集整理的一些笔记。随着它慢慢积累，我意识到它也可能是一份对他人有用的资源。

我会尝试尽可能使用最普通的语言，并通过明确定义的关键术语来避免一些行话。我还会尽可能大量引用一些外部的资源。

请注意我并非来自 React 团队，也不从任何权威的角度来发言。**这不是一篇官方文档**，但是我邀请了 React 团队的成员就本文档的准确性进行了 review。

Fiber 的开发仍在进行中，[进度传送门](http://isfiberreadyyet.com/)。**Fiber 是一个正在开发中的项目，因此在完成前很有可能进行重大的重构。** 我也计划在这里继续记录关于它的设计的一些内容。 非常欢迎对此提出改进和建议。

我的目标是在你阅读了本文以后，你将对 Fiber 有足够的理解，[follow along as it's implemented](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber), 甚至最终能够对 React 做出贡献。

### 阅读前的准备

我强烈建议你在继续阅读前熟悉下列的资源：

- [React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - "Component" 是一个经常被重复使用的术语。牢牢掌握这些术语至关重要.

- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - 一个对 React 的 reconciliation 算法的整体描述。

- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - A description of the conceptual model of React without implementation burden. Some of this may not make sense on first reading. That's okay, it will make more sense with time.

- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - 请特别关注 `Scheduling` 部分。它很好地解释了*为什么*使用 React Fiber。

## 回顾一些概念

如果你还没有准备好，请查看上一章节。

在我们深入研究新的内容之前，让我们回顾一下几个概念。

### 什么是 reconciliation？

<dl>
  <dt>reconciliation</dt>
  <dd>React 用来比较两棵树的算法，它确定树中的哪些部分需要被更新。</dd>

  <dt>update</dt>
  <dd>用来渲染 React 应用的数据的改变。通常是 `setState` 的结果。最终结果将是一次重新渲染</dd>
</dl>

React 的 API 的核心理念是将更新想象成为对整个应用的重新渲染。这允许开发者声明式地推导，而不用担心应用如何从一个状态高效的过渡到另一个状态（A 到 B，B 到 C，C 到 A 等等）。

实际上，对于每个更改重新渲染整个应用程序只适用于最简单的应用程序。在实际的应用中，对性能会有非常大的开销。React 对此做了优化，在保证良好的性能的同时对整个应用应用进行重新渲染。这些优化的很大一部分是一个被称为 **reconciliation** 的过程的一部分。

Reconciliation 是被大家广泛知晓的 "virtual DOM" 背后的算法。更加高层的描述如下：当渲染一个 React 应用，会生成一棵描述应用结构的节点树，并保存在内存中。这棵树随后被刷新到渲染环境。举例来说，我们的浏览器环境，它会转换为一个 DOM 操作的集合。当应用更新的时候（通常是通过 `setState` 触发），会生成一棵新的树。新树会和先前的树进行对比，并计算出更新应用所需要的操作。

尽管 Fiber 是对协调器（译注：这里的协调器可以理解为 Virtual DOM）从零开始的重写，但是高层的算法[在 React 文档中的解释](https://facebook.github.io/react/docs/reconciliation.html) 将是非常相似的。核心点如下：

- 不同的组件类型被认为将生成本质上不同的树。React 将不尝试去进行差异比较，而是直接完全替换旧的树。
- 对于列表的差异比较使用 key 来优化性能。Keys 应当是稳定、可预测且唯一的。

### Reconciliation versus rendering

DOM 仅仅是 React 支持的一个渲染环境，通过 React Native 它还可以支持原生 iOS 和 Android 页面的渲染。（这也是为什么 “Virtual DOM” 有点用词不当。）

React 之所以能够支持如此多的渲染环境，主要是因为在设计上，reconciliation 和渲染两个过程是分离的。reconciliation 做了计算两棵树差异的工作；渲染器则会使用计算得到的信息来更新实际的应用。

这样的分离做法意味着 React DOM 和 React Native 可以使用各自的渲染器，并且共享 React 核心库提供的相同的协调器。

Fiber 重新实现了协调器。这原则上来说不影响渲染器，虽然渲染器可能将需要改变去支持新的架构，来获得新架构所带来的优势。

### Scheduling

<dl>
  <dt>scheduling</dt>
  <dd>决定 work 何时应该被执行的过程。</dd>

  <dt>work</dt>
  <dd>任何计算结果都必须被执行。Work 通常是更新的结果(e.g. <code>setState</code>).
</dl>

React's [Design Principles](https://facebook.github.io/react/contributing/design-principles.html#scheduling) 文档对这一主题讲得很好，此处我将仅仅引用该文档：

> 在 React 目前的实现中，React 递归地遍历树并且调用 render 方法在 event loop 的一次迭代中更新整棵树。不过在将来，它将延迟一些更新来防止掉帧。
>
> 这是 React 设计中一个公共的主题。一些受欢迎的库实现了“推”的方法，计算会在新数据到来的时候被执行。但是 React 坚持“拉”的方法，计算能够被推迟直到需要的时候。
>
> React 不是一个通用的数据处理库。它只是设计来构建用户界面的库。我们认为知晓哪些计算是现在就相关的，哪些不是是在应用程序中有独一无二位置的内容。
>
> 如果一些东西在屏幕外，我们可以推迟任何与其相关的逻辑。如果数据到来的速度比帧率块，那么我们可以合并并批量更新。我们可以对来自用户与界面互动的工作（如一个按钮点击的动画）和相对不太重要的背后工作（如远程加载数据渲染新的内容）进行优先级的排序来阻止掉帧。

总结一下核心点如下：

- 在 UI 中，并非所有的更新都需要立即生效。实际上，这样做是浪费的，可能会造成掉帧从而影响用户体验。
- 不同类型的更新有不同的优先级。一个动画更新相比于来自数据的更新通常需要更快地被执行。
- 一个以“推”为基础的方案要求应用程序（你，工程师）来决定如何调度工作。而一个以“拉”为核心的方案允许框架（如：React）更智能，来为你做这些决定。

React 目前没有享受调度带来的优势。一个更新将会导致整个子树被立即被重新渲染。 背后驱动 Fiber 重写 React 的核心算法是为了来利用调度的优势。

---

好了，现在我们准备深度到 Fiber 的实现。下一章会比之前我们的讨论涉及更多技术方面的内容。在继续之前，请确保你很好地理解了前面的材料。

## 什么是 fiber？

我们即将讨论 React Fiber 架构的核心。Fibers 是比应用程序开发者通常认为的要底层得多的抽象。如果你在尝试理解过程中出现了瓶颈，请不要沮丧。保持尝试它将最终说得通。（当你最终理解了，请对改进这个章节给出建议）

好，让我们开始！

---

我们已经说明了 Fiber 的主要目标是让 React 能够享受到调度带来的好处。明确地说，我们需要能够做到以下几件事：

- 暂停任务并能够在之后恢复任务。
- 为不同的任务设置不同的优先级。
- 重新使用之前完成的任务。
- 如果不在需要则可以终止一个任务。

为了做到这些事，我们首先需要一个方式来拆分这些任务为一个个任务单元。从某种意义上来说，这些任务单元就是 fiber。一个 fiber 代表了一个 **任务单元**.

更进一步得说，我们回到 [React components as functions of data](https://github.com/reactjs/react-basic#transformation) 的概念, 更通俗地表示如下：

```
v = f(d)
```

因此渲染 React 应用程序就类似于调用一个包含其他函数调用的函数。这个比喻在思考 fibers 时十分地有用。

通常情况下，计算机跟踪程序执行是通过[调用栈](https://en.wikipedia.org/wiki/Call_stack)。当一个函数执行，一个新的 **栈帧** 被添加到调用栈。栈帧代表了函数执行的任务。

在处理 UIs 时，一次性执行过多的任务将会造成动画掉帧，看起来卡顿的问题。
并且，随着一些更近的更新的到来，一些任务通常是不需要的。这是对 UI 组件和函数的类比不同的地方，因为组件比起函数通常有更多具体的问题要考虑。

现代浏览器（以及 React Native）实现了能够帮助定位这个确切问题的 API：`requestIdleCallback` 可以在浏览器处于闲置状态时调度一个低优先级的函数去执行。而 `requestAnimationFrame` 调度一个高优先级的函数在下一个动画帧被执行。问题在于，为了使用这些 APIs，你需要一种方式将渲染任务拆分成增量的单元。如果你只依赖于调用栈，它将一直工作直到调用栈为空。

如果我们可以自定义调用堆栈的行为来优化 UI 渲染，那不是很好吗？如果我们可以随意中断调用栈并手动操作栈帧，那不是很好吗？

这是 React Fiber 的目标。Fiber is 是专门为 React 组件重新实现的调用栈。你可以认为一个简单的 fiber 是一个 **虚拟栈帧**。

重新实现调用栈的好处是你可以[把栈帧保存在内存中](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)并且以你想要的方式（在你想要的时候）执行。这对于实现我们调度中提到的目标是至关重要的。

除了调度之外，手动处理栈帧还会释放诸如并发和错误边界等功能的潜力。我们将在以后的章节中介绍这些主题。

在下一章，我们将更多地了解下一个 fiber 的结构。

### fiber 的结构

*注意：当我们去了解更具体的实现细节的时候，一些内容发生改变的可能性将会增加。请发起一个 PR，如果你发现了错误或者过时的信息。*

具体来说，一个 fiber 是一个包含了组件及其输入输出的 JavaScript 对象。

一个 fiber 对应于一个栈帧，但是它也对应与一个组件的实例。

一个 fiber 有以下一些重要的字段。（该列表不是完全详尽的）

#### `type` and `key`

type 和 key 在 fiber 中的用途与其在 React 元素中的用途相同。（实际上，当从一个元素创建一个 fiber，这两个字段是直接拷贝过来的。）

fiber 的 type 描述了它对应的组件。对于复合组件，type 是函数或者组件类本身。对于宿主环境的组件（`div`、`span` 等），type 是一个字符串。

从概念上来讲，type 是执行过程被栈帧跟踪记录的函数（就好像 `v = f(d)`）。

key 则与 type 一起，被用来在协调期间决定是否 fiber 可以被重新使用。

#### `child` and `sibling`

这些字段指向了其他 fibers，它们描述了 fiber 构成的递归结构的树。

孩子 fiber 对应于组件 `render` 方法返回的值。所以在下面的例子中：

```js
function Parent() {
  return <Child />
}
```

`Parent` 的孩子 fiber 对应于 `Child`。

sibling 字段是为了解释 `render` 方法返回多个子元素的情况（这是 Fiber 中的一个新特性！）:

```js
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

孩子 fibers 来自一个单链表中的头部元素。所以在上面的例子中，`Parent` 的孩子是 `Child1`，`Child1` 的兄弟是 `Child2`。

回到我们的函数类比，你可以认为一个函数 fiber 是一个[尾调用函数](https://en.wikipedia.org/wiki/Tail_call).

#### `return`

程序处理完当前 fiber 后应该返回 fiber。返回的 fiber 在概念上等同于返回栈帧的地址。

如果 fiber 有不同的子 fibers，每个孩子 fiber 是由它的父亲返回的。所以在我们之前章节的例子中，孩子 fiber `Child1` 和 `Child2` 是由 `Parent` 返回的。

#### `pendingProps` and `memoizedProps`

从概念上讲，props 是函数的参数。`pendingProps` 是 fiber 执行开始时的属性集合，`memoizedProps` 则是 fiber 执行结束时的属性集合。

当新到来的 `pendingProps` 等于 `memoizedProps`，意味着 fiber 之前的输出可以被重复使用，避免不必要的工作。

#### `pendingWorkPriority`

一个用来表示 fiber 代表的任务优先级的数字。[ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) 模块列出了不同的优先级以及它们所代表的含义。

除了优先级为 0 的 `NoWork` 以外，一个更大的数字表示更低的优先级。举例来说，你可以使用下列函数来确保一个 fiber 的优先级与给定的级别一样高：

```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

*这个函数知识为了说明；它不是 React Fiber 基础代码的一部分。*

调度器使用优先级字段来搜索下一个执行的任务单元。这个算法将在将来的章节被讨论。

#### `alternate`

<dl>
  <dt>flush</dt>
  <dd>flush 一个 fiber 就是把它的输出应用到屏幕上。</dd>

  <dt>work-in-progress</dt>
  <dd>一个 fiber 还没有完成；从概念上讲，就是一个栈帧还没有返回。</dd>
</dl>

在任何时候，一个组件实例至多对应有两个 fibers。它们是当前已经 flush 到屏幕上的 fiber，另一个是正在任务执行中的 fiber。

两个 fiber 相互交替。

fiber 的交替过程是使用一个叫做 `cloneFiber` 的函数延迟创建的。相比于创建一个新的对象，`cloneFiber` 将尝试去重复利用已存在的 fiber，来最小化分配。

你应该把 `alternate` 字段做为一个实现细节，但是它在基础代码中经常出现，在此讨论它是有价值的。

#### `output`

<dl>
  <dt>host component</dt>
  <dd>React 应用的叶子节点。它们由特定的渲染环境决定（例如在网页应用中，它们是 `div`、`span` 等）。在 JSX 中，它们用小写标签名表示。</dd>
</dl>

概念上说，fiber 输出的是函数的返回值。

每一个 fiber 最终都有输出，但是输出只包含 **host components** 叶子节点。输出的内容随后将被转移到树上。

输出的内容最终会给到渲染器，以至于改变能够被应用到真正的渲染环境上。输出如何被创建和更新则是渲染器的职责。

## 将来的章节（作者立了flag并没有写啊）

以上就是现在所有的内容，但是本文档任何地方都不接近完整。在将来的章节，将会描述一次更新贯穿整个生命周期所使用的算法。话题将被包括如下：

- 调度器如何找到下一个任务单元去执行。
- 优先级在 fiber 树结构中是如何被跟踪和传播的。
- 调度器如何知道什么时候去暂停和恢复任务。
- 任务如何被刷新和标记为完成。
- 副作用（如生命周期方法）如何工作。
- 什么是协程以及如何利用它来实现上下文和布局的特性。

## 相关视频
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)
- [Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs&t=471s)
