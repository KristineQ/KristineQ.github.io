---
title: React可中断渲染和Fiber
date: 2021-11-04 09:42:01
updated: 2021-11-04 09:42:01
categories:
  - '笔记'
tags:
  - 'React'
---

那么我们从`render()`开始。

早版本 React 的 render 函数，首先把 react element 解析成真正的 dom，然后用`document.appendChild(dom)`的方式，挂载到父节点上，也是 React 之前被诟病的低性能的设计：渲染一旦开始就无法停止，直到渲染了完整的 dom 树，可能会阻塞主线程很久，如果浏览器需要做高优先级的事情，比如处理用户输入或保持动画流畅，它必须等到渲染完成。

<!-- more -->

## Concurrent Mode

为了解决问题，React 提出了 **Concurrent Mode**：把整个 dom tree 渲染过程分成一个个小的 unit，允许浏览器在执行完一个小的 unit 之后打断渲染，进行交互动画等高优操作，然后等空闲时再继续执行接下来的 unit。

使用浏览器提供的`requestIdleCallback()` api，可以实现这个设计。api 的作用类似于`setTimeOut()`，`setTimeOut()`第二个参数是在某个时间后执行，而`requestIdleCallback()` 的“第二个参数”可以类比为在浏览器空闲的时候执行。 React 内部用 **scheduler** 实现和这个 api 相似的功能，用如下递归方式实现连续调用 `requestIdleCallback()`

```JavaScript
const nextUnitOfWork = null;
const workLoop = () => {
  while (nextUnitOfWork) {
    nextUnitOfWork = preformUnitOfWork();
  }
  requestIdleCallback(workLoop);
};
requestIdleCallback(workLoop);
```

这其实是一个 “可被打断的 render”（ interruptible render）的设计，意味着我们可以在`while(nextUnitOfWork)`的条件判断里加任何 false 条件跳出 while，然后等下一个空闲时间时进行 `requestIdleCallback()`

`requestIdleCallback()`的 callback 还提供一个参数[IdleDeadline](https://developer.mozilla.org/en-US/docs/Web/API/IdleDeadline)，包含一个方法`timeRemaining()`， 此方法返回的是当前空闲时间预计还有多久结束，如果现在不空闲则为 0。所以我们可以在空闲时间马上要到 0 的时候暂停 render，然后在下个空闲时间开始的时候再继续

```JavaScript
let pauseRender = false;
while (nextUnitOfWork && !pauseRender) {
  nextUnitOfWork = performUnitWork(nextUnitOfWork)
  pauseRender = deadline.timeRemaining() < 2
}
```

## Fiber

可中断的 render 完成后，接下来，要实现 **在中断之前完成一个 unit，中断后继续完成下一个 unit** 的目标，我们就需要写一个用来拆分 unit 的处理函数`performUnitOfWork()`，他的任务是：处理完这个 unit，返回下一个 unit

```JavaScript
const performUnitOfWork = (nextUnitOfWork) => {
  // do perform dom
  return nextUnitOfWork
}
```

如果用原有的 dom element 树形的数据结构，链接树形同层级的兄弟节点查找较为困难，所以 React 使用了另外一种 Fiber 的链式数据结构重新组织 dom element。

![fiber-construct](/statics/fiber-construct.png)

每一个节点都有相邻三个节点的引用：父节点、子节点、兄弟节点，可以按照如下 interFace 来简略理解 Fiber 的类型

```TypeScript
interface Fiber {
  dom: HTMLElement | null;
  type: HTMLElementTagNameMap | null;
  return: Fiber | null;  // same to parent
  child: Fiber | null;
  sibling: Fiber | null;
  props: Iprops;
}
interface Iprops {
  children?: Array<ReactElement>;
  [propName: string]: any;
}
```

Fiber 树用深度优先方法进行遍历，如上图的箭头，hostRoot -> ClickCounter -> button -> span -> ClickCounter -> hostRoot

如果想实现`performUnitOfWork()`的功能，我们在需要在遍历的时候做以下的事情

1. 把现在的节点渲染到 dom 上
2. 把下个节点挂载到 fiber 上
3. 返回接下来需要渲染的 fiber

首先，进行 dom 的渲染，如果是 root 节点，就会有 root 的 dom，但其他的 fiber 开始没有 dom，需要根据 fiber 来 create dom

```JavaScript
const createDom = (fiber) => {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)

  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })

  return dom
}
```

创建 dom 完成后，用 appendChild，挂到 parent 上

```JavaScript
// add dom to fiber, render use appendChild to parent dom
if (!fiber.dom) { fiber.dom = createDom(fiber) }
if (fiber.parent) { fiber.parent.dom.appendChild(fiber.dom) }
```

此时的 fiber 里 children 属性，还是 react element 的类型，所以，接下来要把它的 children 都解析为 fiber，里面三个关键的指针，parent child sibling，parent 指向的就是当前被遍历的 fiber，child 指向的是 children 里的第一个 element，sibling 指向的是 child 的兄弟节点

```JavaScript
// create new fibers for each child, add to fiber child

// store the refer for last child fiber
// 这里用一个指针保存上一个sibling
let preFiberChild = null;
fiber.props.children?.forEach((child, index) => {
  const { type, props } = child
  const newFiber = {
    dom: null,
    type,
    parent: fiber,
    props,
  }

  if (index === 0) {
    // if the first child add to child
    fiber.child = newFiber
  } else {
    // else only have sibling without child
    // 在构建sibling的时候把上个fiber的sibling指针指向这里
    preFiberChild.sibling = newFiber
  }
  // 保存上一个sibling
  preFiberChild = newFiber
});
```

最后就是返回下一个需要渲染的 fiber 节点，使用深度优先遍历的话，先返回 child，如果没有 child，就先查找 sibling，然后向上查找，一直找到 root。

```JavaScript
// search for the next unit of work and return
// have child return child fiber
if (fiber.child) { return fiber.child }
// don't have child, search siblings, up to parents search uncles
let nextFiber = fiber;
while (nextFiber) {
  if (nextFiber.sibling) { return nextFiber.sibling }
  nextFiber = nextFiber.parent
}
```

## Commit

在`performUnitOfWork`实现中存在一个问题，在第一个环节即挂载 dom 的时候，使用了`parent.dom.appendChild(fiber.dom)`这样的形式，由于 concurrent mode 是可以随时打断渲染的，这样就会出现一个问题：如果当前被别的高优打断渲染，那么用户就会看到渲染一半的 dom，这样的显示是很不友好的，react 认为应该能够一次性挂载所有渲染完的东西。

所以我们删掉这行`parent.dom.appendChild(fiber.dom)`再声明一个变量`wipRoot`，用来保存正在解析的 fiber tree，在解析完成后一波 render，commit 到 root 上

```JavaScript
let wipRoot = null
// if finish wipRoot perform, commit dom tree
const workLoop = (deadline) => {
  // ...
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
    cancelIdleCallback(workLoop)
  }
}
const commitRoot = () => {
  commitWork(wipRoot.child);
  wipRoot = null
}
const commitWork = (fiber) => {
  if (!fiber) return
  fiber.parent.dom.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

至此就利用了 fiber 和`requestIdleCallback()` 实现了 **可中断渲染(Interruptible Render)** 的目标

---

文章主要参见：[build-your-own-react](https://pomb.us/build-your-own-react/)

> 其他参考：
>
> [in-depth overview of the new reconciliation algorithm in React](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
>
> [Introducing Concurrent Mode](https://reactjs.org/docs/concurrent-mode-intro.html)

🙇‍♀️ 向大佬们致敬

---

附录，完整代码：

```JavaScript
const createDom = (fiber) => {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)

  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })

  return dom
}
const performUnitOfWork = (fiber) => {
  // add dom to fiber, render use appendChild to parent dom
  if (!fiber.dom) { fiber.dom = createDom(fiber) }

  // create new fibers for each child, add to fiber child
  let preFiberChild = null; // store the refer for last child fiber
  fiber.props.children?.forEach((child, index) => {
    const { type, props } = child
    const newFiber = {
      dom: null,
      type,
      parent: fiber,
      props,
    }
    if (index === 0) {
      // if the first child add to child
      fiber.child = newFiber
    } else {
      // else only have sibling without child
      preFiberChild.sibling = newFiber
    }
    preFiberChild = newFiber
  });

  // search for the next unit of work and return
  // have child return child fiber
  if (fiber.child) { return fiber.child }
  // don't have child, search siblings, up to parents search uncles
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) { return nextFiber.sibling }
    nextFiber = nextFiber.parent
  }
}
const render = (element, container) => {
  wipRoot = {
    dom: container,
    type: null,
    parent: null,
    child: null,
    sibling: null,
    props: { children: [element], }
  };
  nextUnitOfWork = wipRoot
}
const commitRoot = () => {
  commitWork(wipRoot.child);
  wipRoot = null
}
const commitWork = (fiber) => {
  if (!fiber) return
  fiber.parent.dom.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
const workLoop = (deadline) => {
  let pauseRender = false;
  while (nextUnitOfWork && !pauseRender) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    pauseRender = deadline.timeRemaining() < 1
  }
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
    cancelIdleCallback(workLoop)
  } else {
    requestIdleCallback(workLoop)
  }
}

let nextUnitOfWork = null
let wipRoot = null

requestIdleCallback(workLoop)



export default render;
```
