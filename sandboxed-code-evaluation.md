## 邪恶的 eval

> `eval()` 函数评估字符串形式的  JavaScript 代码

代码评估的最常见的解决方案即使用 `eval()` 函数。由 `eval()` 评估的代码能够访问闭包和全局作用域，这会导致被称为 [code injection](https://en.wikipedia.org/wiki/Code_injection) 的安全隐患并因此让 `eval()` 成为 JavaScript 最臭名昭著的功能。

虽然让人不爽，但是在某些情况下 `eval()` 是非常有用的。大多数的现代框架需要它的功能但是因为上面提到的问题而不敢使用。结果，许多在沙箱而非全局作用域中评估字符串的替代方案出现了。沙箱防止代码访问安全数据。一般情况下它是一个简单的对象，这个对象会为评估代码替换掉全局的对象。

## 常规方案

替代 `eval()` 最常见的方式即为完全重写 - 分两步走，包括解析和解释字符串。首先解析器创建一个抽象语法树（AST），然后解释器遍历语法树并在沙箱中解释为代码。

这是被最为广泛使用的方案，但是对于如此简单的事情被认为是牛刀小用。从零开始重写所有的东西而不是为 `eval()` 打补丁会导致易出很多的 bug 并且它还要求频繁地修改以匹配最新的语言的修改。

## 替代方案

NX 试图避免重新实现原生代码。代码评估是由一个使用了一些新或者冷门的 JavaScript 功能的小型库来处理的。

本节将会循序渐进地介绍这些功能然后由它们来介绍 [nx-compile](https://github.com/RisingStack/nx-compile) 是如何运行代码的。此库含有一个被称为 `compileCode()` 的库，运行方式类似以下代码：

```
const code = compileCode('return num1 + num2')
// this logs 17 to the console
console.log(code({num1: 10, num2: 7}))

const globalNum = 12
const otherCode = compileCode('return globalNum')

// global scope access is prevented
// this logs undefined to the console
console.log(otherCode({num1: 2, num2: 3}))
```

在本章末尾，我们将会以少于 20 行的代码来实现 `compileCode` 函数。

### new Function()

> 函数构建器创建了一个新的函数对象。在 JavaScript 中，每个函数都实际上是一个函数对象。

`Function` 构造器是 `eval()` 的一个替代方案。`new Function(...args, 'funcBody') ` 评估传进来的 `'funcBody'` 字符串里面的代码并且返回一个新的函数来执行。它和 `eval()` 主要有两点区别：

- 它只会评估传入的代码一次。调用返回的函数会直接运行而无需评估。
- 它不能访问本地闭包变量，但是仍然可以访问全局作用域。

```
function compileCode(src) {
	return new Function(src)
}
```

`new Function()` 在我们的需求中是一个更好的替代 `eval()` 的方案。它有很好的性能和安全性，但是为使其可行需要屏蔽其对全局作用域的访问。

### With 关键字

> with 声明为一个声明语句拓展了作用域链

`with` 是 JavaScript 一个冷门的关键字。它造出一个半沙箱的运行环境。`with` 代码块中的代码会首先试图从传入的沙箱对象获得变量，但是如果没找到则会闭包和全局作用域中寻找。闭包作用域的访问可以用 `new Function()` 来避免，只需要处理全局作用域。

```
function compileCode(src) {
  src = 'with (sandbox) {' + src + '}'
  return new Function('sandbox', src)
}
```

`with` 内部使用 `in` 运算符。在块中访问每个变量，它会在 `沙箱中变量` 的环境中编译代码。如果在环境中找到则会从沙箱中获得变量。否则，它会在全局作用域中寻找变量。欺骗 `with` 让评估 `沙箱中变量` 一直返回真，可以防止它访问全局作用域。

![](./assets/Sandboxed_code_evaluation_simple_with_statement-1470403007416.svg)

### ES6 代理

> 代理对象是用来为诸如属性遍历和赋值的基本操作定义自定义行为

一个 ES6 `proxy` 是封装一个对象并且定义了一个陷阱函数可能被用来在那个对象上拦截基本操作。当操作发生的时候，陷阱函数会被调用。把沙箱对象封闭在一个 `Proxy` 中并且定义一个 `has` 陷阱，我们可以重写 `in` 运算符的默认行为。

```
function compileCode(src) {
  src ='with (sandbox) {' + src + '}
  const code = new Function('sandbox', src)
  
  return function(sandbox) {
    const sandboxProxy = new Proxy(sandbox, {has})
    return code(sandboxProxy)
  }
}

// this trap intercepts 'in' operations on sandboxProxy
function has(target, key) {
  return true
}
```

以上代码欺骗了 `with` 代码块。`沙箱中变量` 将会一直评估是 true 值，因为 `has` 陷阱函数会一直返回 true。`with` 代码块将永远都不会访问全局对象。

![](./assets/Sandboxed_code_evaluation_with_statement_and_proxies-1470403030877.svg)

### Symbol.unscopables

> 标记是一个唯一和不可变的数据类型可以被用作对象属性的一个标识

`Symbol.unscopables` 是一个著名的标记。一个著名的标记即是一个内置的 JavaScript 标记，可以用来代表内部语言行为。著名的标记可以被用作诸如添加或者覆写遍历或者类型转换。

> Symbol.unscopables 著名标记用来指定对象属性的值，这个对象的自身和继承的属性名称被排除在 `with` 环境绑定之外。

`Symbol.unscopables` 为对象定义了一个 `unscopable` 的属性。Unscopable 属性永远不可以被从 `with` 声明中的沙箱对象中获得，相反他们可以从闭包或者全局作用域中直接获得。`Symbol.unscopables` 是一个不常用的功能。你可以在[这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/unscopables)阅读它被提出的原因。

![](./assets/Sandboxed_code_evaluation_security_issue-1470403047129.svg)

我们可以用在沙箱的 `Proxy` 属性中定义一个 `get` 陷阱来解决以上的问题，这样可以评估 `Symbol.unscopables` 的返回值并且一直返回未定义。这将会欺骗 `with` 块的代码认为我们的沙箱对象没有 unscopable 属性。

```
function compileCode(src) {
  src = 'with(sandbox) {' + src + '}'
  const code = new Function('sandbox', src)
  
  return function(sandbox) {
    const sandboxProxy = new Proxy(sandbox, {has, get})
    return code(sandboxProxy)
  }
}

function has(target, key) {
  return true
}
  
function get(target, key) {
  if (key === Symbol.unscopables) return undefined
  return target[key]
}
```

![](./assets/with_statements_and_proxies_has_and_get_traps-1470403073125.svg)

### WeakMaps 来作缓存

现在代码是安全的但是可以提升其性能，因为当每次创建返回函数的时候都会创建一个新的代理对象。可以使用缓存来防止并且在每个相同的沙箱对象中为每个函数调用使用相同的代理。

一个代理属于一个沙箱对象，所以可以简单地把代理作为沙箱对象的一个属性。然而，这将会把我们的实现细节暴露为公有的，并且如果固化的沙箱对象以 `Object.freeze()` 函数来冻结对象这将行不通。在这个例子中使用 `WeakMap` 是一个更好的替代方案。

> WeakMap 对象是一些键/值对，这里键是被弱引用。键必须是对象而值可以是任意值。

一个 `WeakMap` 可以用来为对象添加数据而不用直接用属性来扩展对象。我们杺使用 `WeakMaps` 来间接地为沙箱对象添加缓存代理。

```
const sandboxProxies = new WeakMap()

function compileCode (src) {
	src = 'with (sandbox) {' + src + '}'
	const code = new Function('sandbox', src)
	
	return function(sandbox) {
		if (!sandboxProxies.has(sandbox)) {
      const sandboxProxy = new Proxy(sandbox, {has, get})
      sandboxProxies.set(sandbox, sandboxProxy)
		}
		return code(sandboxProxies.get(sandbox))
	}
}

function has(target, key) {
  return true
}

function get(target, key) {
  if (key === Symbol.unscopables) return undefined
  return target[key]
}
```

这样每个沙箱对象只会有一个 `Proxy`。

## 最后说明

以上的 `compileCode` 例子是一个只有 19 行代码的可用的沙箱代码评估器。如果需要找出 nx-compile 库的完整源码可以参见[这里](https://github.com/RisingStack/nx-compile)。

除了解释代码评估，本章的目标是为了展示如何利用新的 ES6 功能来替换掉已经存在的老函数的功能，而不是覆写轮子。我试图通过例子来展示 `Proxies` 和 `Symbols` 的所有功能。