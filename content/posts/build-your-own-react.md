---
title: 构建自己的React
date: 2020-12-21 23:24:17
draft: false
tags: ["react", "前端"]
slug: "React Essentials"
---

## 构建自己的React

1. The createElement Function

2. The render Function

3. Concurrent Mode

4. Fibers

5. Render and Commit Phases

6. Reconciliation

7. Function Components

8. Hooks



## 让我们开始吧！

```jsx
const element = <h1 title="foo">Hello</h1>

const container = document.getElementById("root")

ReactDOM.render(element, container)

```

上面是一段简单的react代码，创建一个h1元素，然后添加到root容器节点下。我们知道jsx会被babel解析成React.createElement，本质上是一次函数调用：

```js
const element = React.createElement(
  "h1",
  { title: "foo" },
  "Hello"
)

```

这次函数调用生成的element对象，具有两个 properties: type and props：

```js
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}

```

此外，我们知道ReactDom.render是react更改dom的方法，我们自己来实现一下更新dom：

1. 首先是根据element对象type属性，创建dom节点，即h1标签。
2. 将element对象非children所有属性赋给创建的dom节点。
3. element对象中还有一个特殊的属性children，我们将根据它创建h1元素的子节点，这里是一个文本节点。
4. 将创建好的dom节点追加到root容器节点下。

```js
const node = document.createElement(element.type)
node["title"] = element.props.title

const text = document.createTextNode("")
text["nodeValue"] = element.props.children

const container = document.getElementById("root")

node.appendChild(text)
container.appendChild(node)

```

好了，到这里热身结束，让我们步入正题，一窥React.createElement以及React.render的真面目吧！



## 实现React.createElement方法

```js
const element = React.createElement(
  "div",
  { id: "foo" },
  React.createElement("a", null, "bar"),
  React.createElement("b")
)
const container = document.getElementById("root")
ReactDOM.render(element, container)

```

调用React.createElment,对props使用扩展运算符，对children使用剩余运算符。

```js
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children,
    },
  }
}

```

创建元素对象如下:

```json
// createElement("div", null, a, b)
{
  "type": "div",
  "props": { "children": [a, b] }
}

```

遍历子元素

```js
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}

```

使用自定义的Didact.createElement取代React.createElement

```js
const Didact = {
  createElement,
}

const element = Didact.createElement(
  "div",
  { id: "foo" },
  Didact.createElement("a", null, "bar"),
  Didact.createElement("b")
)

```

让jsx支持自定义的Didact.createElement

```jsx
/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)

```

完整代码如下

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}

const Didact = {
  createElement,
}

const element = Didact.createElement(
  "div",
  { id: "foo" },
  Didact.createElement("a", null, "bar"),
  Didact.createElement("b")
)

/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)

const container = document.getElementById("root")
ReactDOM.render(element, container)

```



## 实现ReactDOM.render方法

```js
function render(element, container) {
  const dom = document.createElement(element.type)
  container.appendChild(dom)
}

```

加入一点递归

```js
function render(element, container) {
  const dom = document.createElement(element.type)
  // add
   element.props.children.forEach(child =>
    render(child, dom)
  )
    
  container.appendChild(dom)
}

```

处理文本元素

```js
const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)

```

把props属性赋给创建的node节点

```js
const isProperty = key => key !== "children" // 非children的props作为过滤条件
Object.keys(element.props)
  .filter(isProperty)
  .forEach(name => {
    dom[name] = element.props[name]
  })

```

完整代码如下:

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}

function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)

  // 非子元素属性直接赋给dom节点
  const isProperty = key => key !== "children"
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name]
    })

  // 是子元素属性则创建子元素然后生成子节点
  element.props.children.forEach(child =>
    render(child, dom)
  )

  container.appendChild(dom)
}

const Didact = {
  createElement,
  render,
}

/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)
const container = document.getElementById("root")
Didact.render(element, container)

```



## 重构-考虑递归遍历问题

递归遍历子树渲染，如果元素树庞大，则会阻塞浏览器主线程，造成用户无法输入或动画卡顿的后果。

```js
element.props.children.forEach(child =>
   render(child, dom)
 )

```

解决思路：将浏览器渲染子树工作更加粒化（如何颗粒化，不影响程序运行，这里有一部分工作），通过工作循环（workLoop）确定下一个小的工作单元，采用闲时回调（requestIdleCallback），提供deadline来避免渲染阻塞主线程。

react 使用 *[scheduler package](https://github.com/facebook/react/tree/master/packages/scheduler)* 替代requestIdleCallback。

```js
let nextUnitOfWork = null

function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}

requestIdleCallback(workLoop)

function performUnitOfWork(nextUnitOfWork) {
  // TODO
}

```

```js
while (nextUnitOfWork) {    
  nextUnitOfWork = performUnitOfWork(   
    nextUnitOfWork  
  ) 
}

```

## Fibers-细化render工作

performUnitOfWork不仅要执行当前工作，还要返回下一个工作单元，所以为了方便寻找下一个工作单元，从而将渲染子树的工作更加粒化，我们需要一种数据结构：firber tree（纤维树）:

 符合深度优先遍历



![fiber tree](https://pomb.us/static/a88a3ec01855349c14302f6da28e2b0c/ac667/fiber1.png)



每个react元素对应一个fiber，每个fiber对应一个工作单元，每个工作单元要做三件事：

1. 将正在处理的fiber对应的react元素添加到dom上。

2. 为正在处理的fiber对应的react元素的子元素创建fibers。

3. 选择工作单元下一个要处理的fiber。

如何选择下一个要处理的fiber，优先级如下:

1. 当前元素子元素 （child）

2. 兄弟元素 （sibling）

3. 堂亲（uncle）

4. 向上查找直到根 （root）

开始前，封装fiber创建dom方法：

```js
function createDom(fiber) {
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

首先，初始化第一个工作单元，即root fiber

```js
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}

let nextUnitOfWork = null

```

进入工作循环（workLoop）后，当浏览器主线程空闲，则会调用performUnitOfWork处理fiber：

```js
function performUnitOfWork(fiber) {
  // 1.创建dom元素
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
    
  // 添加到父元素上
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }

  // 2.为子元素创建fibers  
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null

  while (index < elements.length) {
    const element = elements[index]

    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }
    
    // 当前元素的第一个子元素为child;第二个子元素为第一个子元素的sibling，以此类推...
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }

    prevSibling = newFiber
    index++
  }
    
  // 3.寻找下一个工作单元要处理的fiber
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}

```

以上，就是performUnitOfWork需要做的工作。

## 呈现和提交阶段

performUnitOfWork遗留一个问题：

浏览器可以中断我们的渲染过程，而我们是以fiber为工作单元来处理dom树的渲染，每处理一个fiber则渲染该fiber对应的dom元素，因此我们渲染得到的dom树可能是不完整的，然而我们并不希望用户看到残缺的页面，那么应该如何来处理呢？

解决思路：我们知道如果找不到下一个fiber，那么我们已经完成fiber树的构建，为了渲染完整的用户页面，我们可以在fiber树构建完成后，一次性渲染整棵dom树。

保持root fiber的引用：

```js
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}

let nextUnitOfWork = null
let wipRoot = null

```

完成fiber树的构建,开始渲染：

```js
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }

  requestIdleCallback(workLoop)
}
```

递归遍历渲染整棵dom树:

```js
function commitRoot() {
  commitWork(wipRoot.child)
  wipRoot = null
}

function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

```



## 协调

当页面渲染完毕初始化成功，我们又如何处理页面的更新呢?页面更新包括了dom节点的新加，删除，属性更新，这时候reconciliation(协调)出现了，那么是谁和谁进行协调呢，其实就是render函数接收的最新的element生成的fiber树和上一次提交给dom的旧的fiber树做协调。而这里的协调策略是：

1. 新旧fiber元素类型相同，此时dom节点结构未发生变化，无需删除旧的dom节点，只需要进行属性更新。
2. 新旧fiber元素类型不同，此时dom节点结构发生改变，需要进行旧的dom节点的删除。
3. 此外，还需要根据新的fiber元素新增dom节点。

## 