---
layout: post
title: 
category: 
tags: 
excerpt:
author: "Eric Zong"
date: 2019-06-11 
---

* content
{:toc}

> 英文原文：[Named function expressions demystified](http://kangax.github.io/nfe/)

# 介绍

令人惊讶的是，在网络上似乎没有充分涵盖命名函数表达式的主题。这可能就是为什么会出现这么多误解的原因。在本文中，我将尝试从理论和实践两方面总结这些奇妙的 Javascript 结构；它们好的、坏的以及丑陋的部分。

简而言之，命名函数表达式仅对一件事有用——调试器（debugger）和分析器（profiler）中的描述性函数名。好吧，也有可能使用函数名进行递归，但你很快就会发现这通常是不明智的。如果你不关心调试体验，则无需担心。否则，请继续阅读以查看你将要处理的一些跨浏览器故障以及如何解决这些故障的提示。

我将首先解释一下函数表达式是什么，以及现代调试器如何处理它们。要跳转到[最终解决方案](#解决方案)请随意，它解释了如何安全地使用这些结构。

# 函数表达式与函数声明

在 ECMAScript 中创建函数对象的两种最常用方法之一是通过函数表达式或函数声明。两者的区别相当让人困惑。至少对我而言是这样。ECMA 规范表明的唯一一点是，函数声明必须总是具有标识符（或者说函数名，如果你更喜欢这种表述的话），而函数表达式也许会忽略这点：

> 函数声明 ：
> function Identifier ( FormalParameterList ~opt~ ){ FunctionBody }
>
> 函数表达式：
> function Identifier ~opt~ ( FormalParameterList ~opt~ ){ FunctionBody }

我们可以看到，当省略标识符时，“某些东西”只能是一个表达式。但是如果有标识符怎么办？如何判断它是函数声明还是函数表达式——毕竟它们看起来是相同的？ECMAScript 似乎根据上下文区分两者。如果 `function foo(){}` 是赋值表达式的一部分, 则它被视为函数表达式。另一方面, 如果 `function foo(){}` 包含在函数体或（顶级）程序本身中，则它被解析为函数声明。

```js
function foo(){} // 声明，因为它是程序（Program）的一部分
var bar = function foo(){}; // 表达式，因为它是赋值表达式（AssignmentExpression）的一部分

new function bar(){}; // 表达式，因为它是创建表达式（NewExpression）的一部分

(function(){
  function bar(){} // 声明，因为它是函数体（FunctionBody）的一部分
})();
```

函数表达式的一个不太明显的例子是用括号包围起来的例子——`(function foo(){})`。它是表达式的原因也是由于上下文：“(” 和 “)” 构成分组运算符（grouping operator），分组运算符只能包含表达式：

演示示例：

```js
function foo(){} // 函数声明
(function foo(){}); // 函数表达式：由于分组运算符

try {
  (var x = 5); // 分组运算符只能包含表达式，而不能包含语句（而 var 是）
} catch(err) {
  // 语法错误
}
```

你可能还记得，在使用 `eval` 计算 JSON 时，字符串通常用括号包围——`eval('(' + json + ')')`。这当然是出于同样的原因而做的——括号就是分组运算符，它强制将 JSON 花括号解析为表达式，而不是块：

```js
try {
  { "x": 5 }; // “{” 和 “}” 被解析为块
} catch(err) {
  // 语法错误
}

({ "x": 5 }); // 分组运算符强制 “{” 和 “}” 被解析为对象字面量
```

声明和表达式的行为有细微差别。

首先，函数声明在任何其他表达式之前被计算。即使声明位于源代码中的最后位置，它也将在其所在作用域中包含的任何其他表达式前被计算。下面的示例演示当 `alert` 执行时 `fn` 函数是怎么已经被定义的，即使函数是在其后声明的：

```js
alert(fn());

function fn() {
  return 'Hello world!';
}
```

函数声明的另一个重要特征是，有条件地声明它们是非标准化的，并且在不同的环境中不同。永远不要依赖有条件地声明函数而应使用函数表达式。

```js
// 绝不要这样做。
// 有的浏览器会将 foo 函数声明为返回 'first' 的那个
// 而其他的——返回 'second'

if (true) {
  function foo() {
    return 'first';
  }
}
else {
  function foo() {
    return 'second';
  }
}
foo();

// 使用函数表达式代替：
var foo;
if (true) {
  foo = function() {
    return 'first';
  };
}
else {
  foo = function() {
    return 'second';
  };
}
foo();
```

如果你对函数声明的实际产生规则感到好奇，请继续阅读。否则，请跳过以下摘录。

> 函数声明只允许出现在程序（Program）或函数体（FunctionBody）中。从语法上说，它们不能出现在块（`{...}`）中——比如，`if`、`while` 或 `for` 语句。这是因为块只能包含语句，而不能是函数声明这样的源元素（SourceElement）。如果我们仔细查看产生规则，我们可以看到，允许表达式直接在块中的唯一方法是当它是表达式语句的一部分时。但是，表达式语句被明确定义为不以 “function” 关键字开头，这正是为什么函数声明不能直接出现在语句或块中的原因（请注意，块只是一个语句列表）。
>
> 由于这些限制，每当函数直接出现在块中（如前面的示例），它实际上应该被视为语法错误，而不是函数声明或表达式。问题是，我所看到的实现几乎没有一个严格按照每一个规则解析这些函数（例外的是 BESEN 和 DMDScript）。它们用专有的方式来解释它们。

值得一提的是，根据规范，允许实现引入语法扩展（参见第 16 节），但仍然完全符合要求。这正是如今在这么多客户端中发生的事情。其中一些将块中的函数声明解释为任意其他函数声明——只需将它们提升到封闭作用域的顶部；另外——引入不同的语义，并遵循稍微复杂一些的规则。

# 函数语句

ECMAScript 的一种语法扩展是功能语句，目前在基于 Gecko 的浏览器中实现（在 Mac OS X 上的 Firefox 1-3.7a1pre 中测试）。不知何故，无论是好的方面或是坏的方面，这个扩展似乎并不广为人知（[MDC 提到过它们](https://developer.mozilla.org/En/Core_JavaScript_1.5_Reference:Functions#Conditionally_defining_a_function)，但非常简要）。请记住，我们在这里讨论它仅仅是为了学习目的和满足我们的好奇心；除非你正在为特定的基于 Gecko 的环境编写脚本，否则我不建议依赖这个扩展。

这里是一些该非标准结构的部分特性：

1. 函数语句允许出现在任何普通语句允许出现的地方。这当然包括块：

```js
if (true) {
  function f(){ }
}
else {
  function f(){ }
}
```

2. 函数语句被解读为任何其他语句，包括条件执行：

```js
if (true) {
  function foo(){ return 1; }
}
else {
  function foo(){ return 2; }
}
foo(); // 1
// 注意，其他客户端将 foo 视为函数声明，
// 会用第二个覆盖第一个 foo，因此，结果是“2”而不是“1”。
```

3. 在变量实例化期间不声明函数语句。它们在运行时声明，就像函数表达式。但是，一旦声明，函数语句的标识符将在函数的整个作用域可用。此标识符可用性使得函数语句不同于函数表达式（你将在下一章节看到命名函数表达式的确切行为）。

```js
// 至此, foo 还没有声明
typeof foo; // "undefined"
if (true) {
  // 一旦进入块中，foo 将被声明并在整个作用域中可用
  function foo(){ return 1; }
}
else {
  // 决不会进入这个块，并且 foo 不会重新声明
  function foo(){ return 2; }
}
typeof foo; // "function"
```

一般而言，我们可以使用标准兼容的代码模拟前面示例中函数语句的行为（不幸的是，更冗长）：

```js
var foo;
if (true) {
  foo = function foo(){ return 1; };
}
else {
  foo = function foo() { return 2; };
}
```

4. 函数语句的字符串表示类似于函数声明或命名函数表达式（在本例中包括标识符——“foo”）：

```js
if (true) {
  function foo(){ return 1; }
}
String(foo); // function foo() { return 1; }
```

5. 最后，在基于 Gecko 的早期实现（出现于 <= Firefox 3）中，函数语句覆盖函数声明时似乎有一个 Bug。早期版本不知为何在使用函数语句覆盖函数声明时会失败：

```js
// 函数声明
function foo(){ return 1; }
if (true) {
  // 使用函数语句覆盖
  function foo(){ return 2; }
}
foo(); // 在 FF<= 3 中是 1, 在 FF3.5 及后续版本中是 2

// 但是，当覆盖函数表达式时这种情况不会发现
var foo = function(){ return 1; };
if (true) {
  function foo(){ return 2; }
}
foo(); // 所有版本都是 2
```

注意，早期版本的 Safari（至少 1.2.3、2.0 - 2.0.4 和 3.0.4，以及更早的版本）以同 SpiderMonkey 相同的方式实现函数语句。除了最后有“Bug”那个示例，这一章节所有示例在这些版本的 Safari 中都产生与 Firefox 相同的结果。其他遵循相同语义的浏览器是 Blackberry one（至少在 8230、9000 和 9530 机型中）。这种行为的多样性再次证明了依赖这些扩展是多么糟糕的想法。

# 命名函数表达式

函数表达式实际上很常见。在 Web 开发中，“fork”是一种常见的模式，它基于某种特性测试的函数定义，以便获得最佳性能。因为这些 forking 通常发生在相同作用域中，所以几乎总是需要使用函数表达式。毕竟，正如我们现在所知的，不应有条件地执行函数声明：

```js
// contains 是 Garrett Smith 编写的 “APE Javascript 库”的一部分（http://dhtmlkitchen.com/ape/）
var contains = (function() {
  var docEl = document.documentElement;

  if (typeof docEl.compareDocumentPosition != 'undefined') {
    return function(el, b) {
      return (el.compareDocumentPosition(b) & 16) !== 0;
    };
  }
  else if (typeof docEl.contains != 'undefined') {
    return function(el, b) {
      return el !== b && el.contains(b);
    };
  }
  return function(el, b) {
    if (el === b) return false;
    while (el != b && (b = b.parentNode) != null);
    return el === b;
  };
})();
```

很明显，当一个函数表达式具有名字（技术上讲——标识符），则称为“命名函数表达式”。在你看到的第一个示例中——`var bar = function foo(){};`——正是如此——它是一个函数名为 `foo` 的命名函数表达式。一个需要记住的重要细节是，该名字仅在新定义函数的作用域内有效；规范要求标识符对于一个封闭作用域（enclosing scope）不应有效：

```js
var f = function foo(){
  return typeof foo; // “foo” 在内部作用域有效
};
// foo 在“外部”不可见
typeof foo; // "undefined"
f(); // "function"
```

那么，这些命名函数表达式有什么特别呢？为什么我们要给它们取名呢？

似乎命名函数有利于获得一个更令人愉快的调试体验。调试应用程序时，有一个具有描述性项目的调用栈将会有很大不同。

# 调试器中的函数名

当函数具有相应的标识符时，调试器在检查（inspect）调用堆栈时将会把该标识符作为函数名显示。一些调试器（比如 Firebug）对显示匿名函数名称有所帮助，它将匿名函数赋值给的变量名作为标识。不幸的是，这些调试器通常依赖于简单的解析规则；这种提取通常非常脆弱，并且经常产生错误结果。

让我们看一个简单的例子：

```js
function foo(){
  return bar();
}
function bar(){
  return baz();
}
function baz(){
  debugger;
}
foo();

// 在这里，当定义 3 个函数时我们均使用了函数声明
// 当调试器在 debugger 语句停止时，
// 调用栈（在 Firebug 中）看起来很有描述性：
baz
bar
foo
expr_test.html()
```

我们可以看到，`foo` 调用 `bar` 继而调用 `baz`（而 `foo` 自己是被 `expr_test.html` 文档全局作用域调用的）。真正美妙的是，即使使用的是匿名表达式，Firebug 也会设法解析函数的“名字”：

```js
function foo(){
  return bar();
}
var bar = function(){
  return baz();
}
function baz(){
  debugger;
}
foo();

// 调用栈
baz
bar()
foo
expr_test.html()
```

但是，不那么美好的是，如果一个函数表达式变得更为复杂（现实中通常总是如此），所有调试器的努力都将毫无作用；我们最终会得到一个闪亮的问号来代替函数名：

```js
function foo(){
  return bar();
}
var bar = (function(){
  if (window.addEventListener) {
    return function(){
      return baz();
    };
  }
  else if (window.attachEvent) {
    return function() {
      return baz();
    };
  }
})();
function baz(){
  debugger;
}
foo();

// 调用栈
baz
(?)()
foo
expr_test.html()
```

当一个函数被赋值给一个以上的变量时，另一个混乱的情况将会出现：

```js
function foo(){
  return baz();
}
var bar = function(){
  debugger;
};
var baz = bar;
bar = function() {
  alert('spoofed');
};
foo();

// 调用栈
bar()
foo
expr_test.html()
```

你可以看到调用栈显示 `foo` 调用了 `bar`。很明显，这不是实际发现的情况。这个混乱源于 `baz` 与另一个函数——弹出“spoofed（欺骗）”警告的那个——“交换”了引用。如你所见，这种解析，在简单情况下工作得很棒，但在复杂（non-trivial）脚本中常常是无用的。

所有的一切都归结为，命名函数表达式是得到一个真正健壮的堆栈检查（stack inspection）的唯一方法。让我们考虑用命名函数来重写前面的示例。注意那两个命名为 `bar` 的函数是如何从自动执行包装器（self-executing wrapper）中返回的：

```js
function foo(){
  return bar();
}
var bar = (function(){
  if (window.addEventListener) {
    return function bar(){
      return baz();
    };
  }
  else if (window.attachEvent) {
    return function bar() {
      return baz();
    };
  }
})();
function baz(){
  debugger;
}
foo();

// 再一次，我们有了一个描述性的调用栈！
baz
bar
foo
expr_test.html()
```

在我们开始愉快地跳舞庆祝这个圣杯发现之前，我想将我钟爱的 JScript 带入这个画面。

# JScript 的 Bug

不幸的是，JScript（即 Internet Explorer 的 ECMAScript 实现）严重搞乱了命名函数表达式。目前，很多人不推荐使用命名函数表达式，JScript 应为此负责。仍然令人痛心的是，即使是用于 Internet Explorer 8 的最新版本的 JScript——5.8——仍然表现出以下所述的每个独立的怪异行为。

让我们看看这个不恰当的实现到底有什么错误。理解其所有问题将使我们能安全地使用它们。请注意，为了清晰起见，我把这些差异分成了几个示例，即使所有的差异很可能是一个主要 bug 的结果。

## 示例 1：函数表达式标识符泄漏到封闭作用域

```js
var f = function g(){};
typeof g; // "function"
```

还记录我提过在封闭作用域内命名函数表达式的标识符是无效的吗？好吧，JScript 不遵循这个规范——在上面的示例中 `g`  解析成了一个函数对象。这是一个被观察到的最为广泛的差异。用一个额外的标识符不经意地污染封闭作用域是很危险的——而该作用域又很可能是全局的。当然，这种污染将是一个难以追踪的 bug 的根源。

## 示例 2：命名函数表达式被同时视为函数声明和函数表达式

```js
typeof g; // "function"
var f = function g(){};
```

正如我之前解释的一样，在一个特定执行上下文中，函数声明将在任意表达式之前解析。上面的示例演示了 JScript 实际上是如何将命名函数表达式视为函数声明的。你可以看到，在“实际声明”前 `g` 的解析就发生了。

这将带我们进入下一个示例：

## 示例 3：命名函数表达式创建了两个不同的函数对象！

```js
var f = function g(){};
f === g; // false

f.expando = 'foo';
g.expando; // undefined
```

这就是事情变得有趣的地方。更准确地说——简直很疯狂。这里我们看到了不得不处理两个不同对象的危险性——扩充了其中一个明显没有修改另一个；如果你决定这样使用，可能会相当麻烦，如示例所示，缓存机制将某些内容存储在 `f` 的属性中，然后试图通过 `g` 的属性访问，仅仅因为主观上以为是在使用同一个对象。

让我们看稍微复杂点的情况。

## 示例 4：函数声明顺序解析且不受条件块影响

```js
var f = function g() {
  return 1;
};
if (false) {
  f = function g(){
    return 2;
  };
}
g(); // 2
```

像这样的示例可能导致更难跟踪的 bug。这里到底发生了什么实际上是相当简单的。首先，`g` 被解析为一个函数声明，又由于在 JScript 中声明是独立于条件块的，因此 `g` 是“死亡” `if` 分支中声明的一个函数——`function g(){ return 2 }`。然后，所有的“常规”表达式被计算，而 `f` 将被另一个新创建的函数对象赋值。在计算表达式时，“死亡” `if` 分支永远不会进入，因此，`f` 保持引用第一个函数——`function g(){ return 1 }`。目前为止应该很清楚了，如果你不够小心，从 `f` 内部调用了 `g`，那么你将最终调用一个完全不相关的 `g` 函数对象。

你或许想知道，将所有这些令人混乱的不同函数对象与 `arguments.callee` 比较会如何。`callee` 是引用 `f` 还是 `g` 呢？让我们来看一看：

```js
var f = function g(){
  return [
    arguments.callee == f,
    arguments.callee == g
  ];
};
f(); // [true, false]
g(); // [false, true]
```

正如你所见，`arguments.callee` 引用正在被调用的函数。这实际上是好消息，稍后你会看到。

当在声明赋值中使用命名函数表达式时，可能见到另一个有关“意外行为”的有趣示例，但是仅当函数名与其赋值给的变量标识符相同时才会发生：

```js
(function(){
  f = function f(){};
})();
```

你也许知道，非声明赋值（不推荐，这里使用仅仅是为了演示）应该导致创建全局的 `f` 属性。在符合规范的实现中这是明确发生的。但是，JScript 的 bug 使得事情更令人困惑一点。由于命名函数表达式会被解析为函数声明（见 示例 2），这里会发生的是，在变量声明阶段 `f` 变成声明了一个局部变量。随后，当函数执行开始，赋值不再是未声明的，因此右边的 `function f(){}` 被简单地赋值给了新创建的局部变量 `f`。全局 `f` 不会创建了。

这表明如何理解 JScript 的特性可能会导致代码中完全不同的行为。

看看 JScript 的缺陷，我们需要避免什么就十分清楚了。首先，我们需要意识到泄漏的标识符（以至于不会污染封闭作用域）。其次，我们决不应该引用用作函数名的标识符；前面示例中的 `g` 就是一个麻烦的标识符。请注意，如果我们忘记 `g` 的存在，将可以避免很多歧义。这里，总是通过 `f` 或 `arguments.callee` 引用函数是一个关键。如果你使用命名表达式，应将该名称视为仅用于调试目的。最后，一个加分点是，始终清理在 NFE 声明期间错误创建的无关函数。

我认为最后这点需要一点说明：

# JScript 内存管理



# 测试



# Safari 的 Bug



# SpiderMonkey 的特点



# 解决方案



# 可选解决方案



# WebKit 的显示名



# 未来的考虑



# 鸣谢