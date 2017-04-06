# React Fiber Architecture ([原文](https://github.com/acdlite/react-fiber-architecture))

## 简介

React Fiber 是对 React 核心算法的重新实现，目前正在进行中，[进度传送门](http://isfiberreadyyet.com/)。这是 React 团队过去两年研究的成果。

React Fiber 的目标是增强 React 在动画、布局和手势等各个领域的适应性。它的最重要的特性是 **incremental rendering（增量渲染）**：它能够将渲染工作拆分成多块并将这些任务块分散到多个帧执行。

其他的核心特性还包括当新的更新到来时暂停、中止或恢复工作的能力；为不同类型的更新设置优先级的能力；以及新的并发原语。

### 关于这篇文档

Fiber 介绍了一些难以直接通过阅读代码理解的新概念。这篇文档源于我在跟踪 React 项目中对 Fiber 的实现时所收集整理的一些笔记。当它慢慢积累，我意识到它将也是一份对他人有用的资源。

我将使用尽可能清楚语言来完成这篇文档，并且通过明确定义关键属于来避免行话。

请注意我并非来自 React 团队，并不从任何权威的角度来发言。**这不是一篇官方文档**，但是我邀请了 React 团队的成员就本文档的准确性进行了 review。

Fiber 的开发仍在进行中，[进度传送门](http://isfiberreadyyet.com/)。**Fiber 是一个正在开发中的项目，因此在完成前很有可能进行重大的重构。** 我对本文档设计的尝试也同时正在尝试中，十分欢迎改进和意见。

我的目标是在你阅读了本文以后，你将对 Fiber 有足够的理解，[follow along as it's implemented](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber), 甚至最终能够给 React 做贡献。

### 阅读前的准备

我强烈建议你在继续阅读前熟悉下列的资源：

- [React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - "Component" is often an overloaded term. A firm grasp of these terms is crucial.
- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - A high-level description of React's reconciliation algorithm.
- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - A description of the conceptual model of React without implementation burden. Some of this may not make sense on first reading. That's okay, it will make more sense with time.
- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - Pay special attention to the section on scheduling. It does a great job of explaining the *why* of React Fiber.

## 回顾一些概念

如果你还没有准备好，请查看上一章节。

在我们深入了解新家伙前，让我们复习一些概念。

### 什么是 reconciliation？

<dl>
  <dt>reconciliation</dt>
  <dd>React 用来比较两棵树的算法，它决定树中的哪一部分需要被改变。</dd>

  <dt>update</dt>
  <dd>用来渲染 React 应用的数据的改变。通常是 `setState` 的结果。最终将导致一次重新渲染</dd>
</dl>

React API 的核心理念认为更新就好像造成了整个应用的重新渲染。这允许开发者声明式地推导，而不用担心应用如何从一个状态高效的过渡到另一个状态（A 到 B，B 到 C，C 到 A 等等）。

实际上，对于每个更改重新渲染整个应用程序只适用于最微不足道的应用程序。在实际的应用中，这是对性能十分巨大的耗费。React 对此有大量的优化，来保证很好的性能。这些优化的很大一部分是一个被称为 **reconciliation** 的过程的一部分。 

Reconciliation 是被大家广泛知晓的 "virtual DOM" 背后的算法。更加高层的描述如下：当渲染一个 React 应用，会有一棵节点树生成并保存在内存中。这棵树随后被刷新到渲染环境。举例来说，我们以浏览器环境的应用为例，它会转换为一个 DOM 操作的集合。当应用更新的时候（通常是通过 `setState` 触发），一棵新的树生成。新树将会和先前的树作比较，并计算出更新应用所需要的操作。

尽管 Fiber 是对协调器（译注：这里的协调器可以理解为 Virtual DOM）从零开始的重写，但是高层的算法[described in the React docs](https://facebook.github.io/react/docs/reconciliation.html) 将是非常相似的。核心点如下：

- 不同的组件类型被认为将生成本质上不同的树。React 将不尝试去进行差异比较，而是直接完全替换旧的树。
- 对于列表的差异比较使用 key 来优化性能。Keys 应当是稳定、可预测且唯一的。

### Reconciliation versus rendering

DOM 仅仅是 React 支持的一个渲染环境，通过 React Native 它还可以支持原生 iOS 和 Android 页面的渲染。（这也是为什么 “Virtual DOM” 是一个错误的称呼。）

React 之所以能够支持如此多的渲染环境，主要是因为它被设计为 reconciliation 和渲染两个过程分离。协调器做了计算两棵树差异的工作；渲染器则会使用计算得到的信息来更新实际的应用。

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

> 在 React 目前的实现中，React 递归地遍历树并且调用 render 方法在 event loop 的一次迭代中更新整棵树。然而在将来，它将延迟一些更新来防止掉帧。
>
> 这是 React 设计中一个公共的主题。一些受欢迎的库实现了“推”的方法，计算会在新数据到来的时候被执行。但是 React 坚持“拉”的方法，计算能够被推迟直到需要的时候。
>
> React 不是一个通用的数据处理库。它只是设计来构建用户界面的库。我们认为知晓哪些计算是现在就相关的，哪些不是是在应用程序中有独一无二位置的内容。
>
> 如果一些东西在屏幕外，我们可以推迟任何与其相关的逻辑。如果数据到来的速度比帧率块，那么我们可以合并并批量更新。我们可以对来自用户与界面互动的工作（如一个按钮点击的动画）和相对不太重要的背后工作（如远程加载数据渲染新的内容）进行优先级的排序来阻止掉帧。

总结一下核心点如下：

- 在 UI 中，并非所有的更新都需要立即生效。实际上，这样做是浪费的，可能会造成掉帧从而影响用户体验。
- 不同类型的更新有不同的优先级。一个动画更新通常需要执行得比来自数据的更新更快。
- 一个以“推”为基础的方案要求应用程序（你，工程师）来决定如何调度工作。而一个以“拉”为核心的方案允许框架（如：React）更智能，来为你做这些决定。

React 目前没有享受调度带来的优势。一个更新将会导致整个子树被立即重新渲染。 重写 React 的核心算法来享受调度的优势是 Fiber 背后的驱动思想。

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

在处理 UIs 时，一次性执行过多的任务将会造成动画掉帧，看起来不稳定的问题。
并且，随着一些更近的更新的到来，一些任务通常是被代替不需要的。这是对 UI 组件和函数的类比被打破的地方，因为组件比起函数通常有更多具体的问题要考虑。

现代浏览器（以及 React Native）实现了能够帮助定位这个确切问题的 API：`requestIdleCallback` 可以在浏览器处于闲置状态时调度一个低优先级的函数去执行。而 `requestAnimationFrame` 调度一个高优先级的函数在下一个动画帧被执行。问题在于，为了使用这些 APIs，你需要一种方式将渲染任务拆分成增量的单元。如果你只依赖于调用栈，它将一直工作直到调用栈为空。

如果我们可以自定义调用堆栈的行为来优化 UI 渲染，那不是很好吗？如果我们可以随意中断调用栈并手动操作栈帧，那不是很好吗？

这是 React Fiber 的目标。Fiber is 是专门为 React 组件重新实现的调用栈。你可以认为一个简单的 fiber 是一个 **虚拟栈帧**。

重新实现调用栈的好处是你可以[把栈帧保存在内存中](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)并且以你想要的方式在你想要的时候执行。这对于实现我们调度中提到的目标是至关重要的。

除了调度之外，手动处理栈帧还会释放诸如并发和错误边界等功能的潜力。我们将在以后的章节中介绍这些主题。

在下一章，我们将更多地了解下一个 fiber 的结构。

### fiber 的结构

*Note: as we get more specific about implementation details, the likelihood that something may change increases. Please file a PR if you notice any mistakes or outdated information.*

In concrete terms, a fiber is a JavaScript object that contains information about a component, its input, and its output.

A fiber corresponds to a stack frame, but it also corresponds to an instance of a component.

Here are some of the important fields that belong to a fiber. (This list is not exhaustive.)

#### `type` and `key`

The type and key of a fiber serve the same purpose as they do for React elements. (In fact, when a fiber is created from an element, these two fields are copied over directly.)

The type of a fiber describes the component that it corresponds to. For composite components, the type is the function or class component itself. For host components (`div`, `span`, etc.), the type is a string.

Conceptually, the type is the function (as in `v = f(d)`) whose execution is being tracked by the stack frame.

Along with the type, the key is used during reconciliation to determine whether the fiber can be reused.

#### `child` and `sibling`

These fields point to other fibers, describing the recursive tree structure of a fiber.

The child fiber corresponds to the value returned by a component's `render` method. So in the following example

```js
function Parent() {
  return <Child />
}
```

The child fiber of `Parent` corresponds to `Child`.

The sibling field accounts for the case where `render` returns multiple children (a new feature in Fiber!):

```js
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

The child fibers form a singly-linked list whose head is the first child. So in this example, the child of `Parent` is `Child1` and the sibling of `Child1` is `Child2`.

Going back to our function analogy, you can think of a child fiber as a [tail-called function](https://en.wikipedia.org/wiki/Tail_call).

#### `return`

The return fiber is the fiber to which the program should return after processing the current one. It is conceptually the same as the return address of a stack frame. It can also be thought of as the parent fiber.

If a fiber has multiple child fibers, each child fiber's return fiber is the parent. So in our example in the previous section, the return fiber of `Child1` and `Child2` is `Parent`.

#### `pendingProps` and `memoizedProps`

Conceptually, props are the arguments of a function. A fiber's `pendingProps` are set at the beginning of its execution, and `memoizedProps` are set at the end.

When the incoming `pendingProps` are equal to `memoizedProps`, it signals that the fiber's previous output can be reused, preventing unnecessary work.

#### `pendingWorkPriority`

A number indicating the priority of the work represented by the fiber. The [ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) module lists the different priority levels and what they represent.

With the exception of `NoWork`, which is 0, a larger number indicates a lower priority. For example, you could use the following function to check if a fiber's priority is at least as high as the given level:

```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

*This function is for illustration only; it's not actually part of the React Fiber codebase.*

The scheduler uses the priority field to search for the next unit of work to perform. This algorithm will be discussed in a future section.

#### `alternate`

<dl>
  <dt>flush</dt>
  <dd>To flush a fiber is to render its output onto the screen.</dd>

  <dt>work-in-progress</dt>
  <dd>A fiber that has not yet completed; conceptually, a stack frame which has not yet returned.</dd>
</dl>

At any time, a component instance has at most two fibers that correspond to it: the current, flushed fiber, and the work-in-progress fiber.

The alternate of the current fiber is the work-in-progress, and the alternate of the work-in-progress is the current fiber.

A fiber's alternate is created lazily using a function called `cloneFiber`. Rather than always creating a new object, `cloneFiber` will attempt to reuse the fiber's alternate if it exists, minimizing allocations.

You should think of the `alternate` field as an implementation detail, but it pops up often enough in the codebase that it's valuable to discuss it here.

#### `output`

<dl>
  <dt>host component</dt>
  <dd>The leaf nodes of a React application. They are specific to the rendering environment (e.g., in a browser app, they are `div`, `span`, etc.). In JSX, they are denoted using lowercase tag names.</dd>
</dl>

Conceptually, the output of a fiber is the return value of a function.

Every fiber eventually has output, but output is created only at the leaf nodes by **host components**. The output is then transferred up the tree.

The output is what is eventually given to the renderer so that it can flush the changes to the rendering environment. It's the renderer's responsibility to define how the output is created and updated.

## 将来的章节

That's all there is for now, but this document is nowhere near complete. Future sections will describe the algorithms used throughout the lifecycle of an update. Topics to cover include:

- how the scheduler finds the next unit of work to perform.
- how priority is tracked and propagated through the fiber tree.
- how the scheduler knows when to pause and resume work.
- how work is flushed and marked as complete.
- how side-effects (such as lifecycle methods) work.
- what a coroutine is and how it can be used to implement features like context and layout.

## 相关视频
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)
- [Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs&t=471s)
