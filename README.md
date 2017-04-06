# React Fiber Architecture ([原文](https://github.com/acdlite/react-fiber-architecture))

> React Conf 2017 上对于 Fiber 的介绍视频，译者补充。
[Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs&t=471s)

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

## What is a fiber?

We're about to discuss the heart of React Fiber's architecture. Fibers are a much lower-level abstraction than application developers typically think about. If you find yourself frustrated in your attempts to understand it, don't feel discouraged. Keep trying and it will eventually make sense. (When you do finally get it, please suggest how to improve this section.)

Here we go!

---

We've established that a primary goal of Fiber is to enable React to take advantage of scheduling. Specifically, we need to be able to

- pause work and come back to it later.
- assign priority to different types of work.
- reuse previously completed work.
- abort work if it's no longer needed.

In order to do any of this, we first need a way to break work down into units. In one sense, that's what a fiber is. A fiber represents a **unit of work**.

To go further, let's go back to the conception of [React components as functions of data](https://github.com/reactjs/react-basic#transformation), commonly expressed as

```
v = f(d)
```

It follows that rendering a React app is akin to calling a function whose body contains calls to other functions, and so on. This analogy is useful when thinking about fibers.

The way computers typically track a program's execution is using the [call stack](https://en.wikipedia.org/wiki/Call_stack). When a function is executed, a new **stack frame** is added to the stack. That stack frame represents the work that is performed by that function.

When dealing with UIs, the problem is that if too much work is executed all at once, it can cause animations to drop frames and look choppy. What's more, some of that work may be unnecessary if it's superseded by a more recent update. This is where the comparison between UI components and function breaks down, because components have more specific concerns than functions in general.

Newer browsers (and React Native) implement APIs that help address this exact problem: `requestIdleCallback` schedules a low priority function to be called during an idle period, and `requestAnimationFrame` schedules a high priority function to be called on the next animation frame. The problem is that, in order to use those APIs, you need a way to break rendering work into incremental units. If you rely only on the call stack, it will keep doing work until the stack is empty.

Wouldn't it be great if we could customize the behavior of the call stack to optimize for rendering UIs? Wouldn't it be great if we could interrupt the call stack at will and manipulate stack frames manually?

That's the purpose of React Fiber. Fiber is reimplementation of the stack, specialized for React components. You can think of a single fiber as a **virtual stack frame**.

The advantage of reimplementing the stack is that you can [keep stack frames in memory](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/) and execute them however (and *whenever*) you want. This is crucial for accomplishing the goals we have for scheduling.

Aside from scheduling, manually dealing with stack frames unlocks the potential for features such as concurrency and error boundaries. We will cover these topics in future sections.

In the next section, we'll look more at the structure of a fiber.

### Structure of a fiber

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

## Future sections

That's all there is for now, but this document is nowhere near complete. Future sections will describe the algorithms used throughout the lifecycle of an update. Topics to cover include:

- how the scheduler finds the next unit of work to perform.
- how priority is tracked and propagated through the fiber tree.
- how the scheduler knows when to pause and resume work.
- how work is flushed and marked as complete.
- how side-effects (such as lifecycle methods) work.
- what a coroutine is and how it can be used to implement features like context and layout.

## Related Videos
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)
