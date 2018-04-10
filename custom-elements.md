# 自定义元素的好处

**这是编写一个 JavaScript 框架系列的第六章。本章，我将会讨论自定义元素的好处和它们在现代前端框架核心内的可能角色。**

## 组件的时代

近些年组件风靡整个网络。所有的现代前端框架诸如 React，Vue 或者 Polymer - 都使用基于模块化的组件。它们提供了不同的 API 并且底层工作方式不一致，然而他们和其它的最新的框架有一些相同的以下功能。

- 他们有一组API，用于定义组件并按名称或者选择器来注册它们。
- 他们提供生命周期钩子，可以用来设置组件逻辑和同步状态视图

直到最近，这些功能还缺少了一个简单的原生 API ，但是这随着 [Custom Elements spec](https://www.w3.org/TR/custom-elements/) 的定稿而改变。自定义元素可以涵盖以上功能，但它们并不总是完美的匹配。让咱们走着瞧^.^。

## 自定义元素

自定义元素是 [Web Components standard](https://www.w3.org/standards/techs/components#w3c_all) 的一部分，它在 2011 被提议，且在最近稳定前出台了的两个不同规范。最终定稿感觉是一个简单原生的组件化框架替代品而不是框架作者的工具。它为定义组件提供了一个漂亮的高阶 API， 但它缺少不需用垫片的功能（需要兼容插件来支持）。

如果您还不熟悉自定义元素，请在继续之前查看本文。

### 自定义元素 API

自定义元素 API 是基于 ES6 类的。元素可以由原生的 HTML 元素或者自定义元素继承而来，并且它们可以用新的属性和方法扩展。他们也可以重写一系列的方法-定义在规范中-可以作为他们生命周期的钩子。

```
class MyEelement extends HTMLElement {
  // these are standard hooks, called on certain events
  constructor() { ... }
  connectedCallback () { ... }
  disconnectedCallback () { ... }
  adoptedCallback () { ... }
  attributeChangedCallback (attrName, oldVal, newVal) { ... }

  // these are custom methods and properties
  get myProp () { ... }
  set myProp () { ... }
  myMethod () { ... }
}

// this registers the Custom Element
customElements.define('my-element', MyElement)
```

在定义之后，这些元素可以在 HTML 或者 JavaScript 代码中以名称实例化。

`<my-element></my-element>`

基于类的 API 非常简洁，但是在我看来，它缺少灵活性。作为框架作者，我更加喜欢弃用的 [v0 API](https://www.html5rocks.com/en/tutorials/webcomponents/customelements/)，它是基于老旧的经典原型方法的。

```
const MyElementProto = Object.create(HTMLElement.prototype)
// native hooks
MyElementProto.attachedCallback = ...
MyElementProto.detachedCallback = ...

// custom properties and methods
MyElementProto.myMethod = ...
document.registerElement('my-element', { prototype: MyElementProto })
```

它大概不够优雅，但是它可以把 ES6 和 ES6 规范之前的代码很好地整合在一起。从另一方面说，把 ES6 规范之前的代码和类代码混合在一起使用会是相当复杂的。

比如，我想要有能力控制组件继承哪个 HTML 接口。ES6 类使用静态的 `extends` 关键字来继承，并且它们要求开发者输入 `MyClass extends ChosenHTMLInterface`。

这非常不适用于我目前的使用情况，因为 NX 基于中间件函数而不是类。在 NX 中，可以用 `element` 配置属性来设置接口，接口接受一个有效的 HTML元素名称比如 - `button`。

```
nx.component({element: 'button'})
	.register('my-button')
```

为了达到这一目标，我不得不使用基于原型的系统来模仿 ES6 类。长话短说，操作起来比人所能想的要让人蛋疼，并且它需要不需垫片的 ES6 `Reflect.construct` 和性能杀手 `Object.setPrototypeOf` 函数。

```
function MyElement() {
  return Reflect.construct(HTMLELEMENT, [], MyElement)
}
const myProto = MyElement.prototype
Object.setPrototypeOf(myProto, HTMLElement.prototype)
Object.setPrototypeOf(MyElement, HTMLElement)
myProto.connectedCallback = ...
myProto.disconnectedCallback = ...
customElements.define('my-element', MyElement)
```

这只是我发现在使用 ES6 类的很困难的情况之一。对于日常应用，我觉得他们是非常好的，但是当我想要很装逼地充分利用这门语言的功能的时候，我更倾向于使用原型继承。

### 生命周期

自定义元素拥有五个生命周期钩子，会在特定事件触发的时候同步调用。

- `constructor` 会在元素的实例化的过程被调用
- `connectedCallback` 会在元素被挂载到 DOM 的时候调用
- `disconnectedCallback` 会在元素被从 DOM 中移除的时候调用
- `adoptedCallback` 会在当使用 `importNode` 或者 `cloneNode` 把元素挂载到一个新的文档之中的时候调用
- `attributeChangedCallback` 会在当被监听的元素属性发生变化的时候被调用

`constructor` 和 `connectedCallback` 非常适合创建组件状态和逻辑，而 `attributeChangedCallback` 可以被用来以 HTML 属性来显示组件的属性，反之亦然。`disconnectedCallback` 用来在组件销毁后清理内存。

整合在一起，这些涵盖了一系列很好的功能，但是我仍然忽略了 `beforeDisconnected` 和 `childrenChanged` 回调。`beforeDisconnected` 钩子适用于简单的离开动画，然而除了封装或者大幅修改 DOM 是无法实现它的。

`childrenChanged` 钩子对于桥接状态和视图是非常重要的。看下以下示例：

```
nx.component()
	.use((elem, state) => state.name = 'World')
	.register('my-element')
```

```
<my-component>
	<p>Hello: ${name}<p>
</my-component>
```

这是一个简单的模板片段，把 `name` 属性值从状态插入到视图中。当用户决定置换 `p` 元素为其它元素时，框架会接收到改变的通知。它不得不清理老的 `p` 元素内容，然后把插值插入到新内容中。`childrenChanged` 可能不会公开为开发者钩子，但是知道何时组件内容发生改变是一个框架必备的功能。

如我所述，自定义元素缺少一个 `childrenChanged` 回调，但是可以使用老旧的 [MutationObserver API](https://developer.mozilla.org/en/docs/Web/API/MutationObserver) 来实现。MutationObservers 也为老浏览器提供了 `connectedCallback`，`disconnectedCallback` 和 `attributeChangedCallback` 钩子的替代品。

```
// create an observer instance
const observer = new MutationObserver(onMutations)

function onMutations (mutations) {
  for (let mutation of mutations) {
    // handle mutation.addedNodes, mutation.removedNodes, mutation.attributeName and mutation.oldValue here
  }
}

// listen for attribute and child mutations on `MyComponentInstance` and all of its ancestors
observer.observe(MyComponentInstance, {
  attributes: true,
  childList: true,
  subtree: true
})
```

除了自定义元素的简洁 API，这将会产生一些自定义元素必要性的问题。

下一章节，我将会阐述 MutationObservers 和 自定义元素的一些关键区别以及使用的场景。

## 自定义元素 VS MutationObservers

当发生 DOM 改变的时候，自定义元素回调会同步调用，而 MutationObservers 收集这些改变并异步调用其中的一批。对于组织逻辑这并不是什么大问题，但是它会在内存清理阶段引发一些不可预见的 bugs。当待处理的数据还存在时，有一个小的时间间隔是危险的。

另一个重要的区别是， MutationObservers 没有进入 shadow DOM 边界。监听 shadow DOM 里面的改变需要自定义元素或者手动为 shadow 根目录添加一个 MutationObserver。如果你从来没有听说过 shadow DOM，你可以在 [here ](https://developers.google.com/web/fundamentals/getting-started/primers/shadowdom) 查看更多。

最后，他们提供了略有不同的挂钩。自定义元素有 `adoptedCallback` 钩子，然而 `MutationObservers` 可以在任意层次监听文本的改变和子元素的改变。

综上所述，把这两者的最好的方面结合起来使用是一个好主意。

## 把自定义元素 和 MutationObservers 结合起来

因为自定义元素还没有被广泛支持，所以必须使用 MutationObservers 来检测 DOM 改变。主要有两种选择。

- 在自定义元素的基础上创建 api ，然后使用 MutationObservers 来作兼容
- 使用 MutationObservers 创建 api ，然后当需要的时候使用自定义元素来添加一些性能改进

我选择后者，因为 MutationObservers 是检测子元素改变的必要条件，即使在完全支持自定义元素的浏览器中也是如此。
 

我为下一版本的 NX 使用的系统简单地在旧浏览器的文档添加一个 MutationObserver。然而在现代浏览器中，该系统使用自定义元素为最顶层的组件创建钩子，并且在他们的 `connectedCallback` 钩子中添加一个 `MutationObserver`。这个 MutationObserver 可以用来扮演在组件内部检测进一步的 DOM 变化的角色。

它只查找文档中由框架控制的部分中的更改。对应的代码大概如下。

```
function registerRoot (name) {
  if ('customElements' in window) {
    registerRootV1(name)
  } else if ('registerElement' in document) {
    registerRootV0(name)
  } else {
     // add a MutationObserver to the document
  }
}

function registerRootV1 (name) {
  function RootElement () {
    return Reflect.construct(HTMLElement, [], RootElement)
  }
  const proto = RootElement.prototype
  Object.setPrototypeOf(proto, HTMLElement.prototype)
  Object.setPrototypeOf(RootElement, HTMLElement)
  proto.connectedCallback = connectedCallback
  proto.disconnectedCallback = disconnectedCallback
  customElements.define(name, RootElement)
}

function registerRootV0 (name) {
  const proto = Object.create(HTMLElement)
  proto.attachedCallback = connectedCallback
  proto.detachedCallback = disconnectedCallback
  document.registerElement(name, { prototype: proto })
}

function connectedCallback (elem) {
  // add a MutationObserver to the root element
}

function disconnectedCallback (elem) {
// remove the MutationObserver from the root element
}
```

这会为现代浏览器带来性能的好处，因为他们只需处理极少的 DOM 变化。

## 结论

总而言之，重构NX可以很容易地使用没有很大性能影响的自定义元素，但是自定义元素在某些情况仍然会带来性能的提升。我需要从他们之中得到的有用的干货即是一个灵活的底层 API 和大量的同步生命周期钩子。
