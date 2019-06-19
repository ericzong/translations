---
layout: post
title: 命名函数表达式揭秘
category: 编程语言
tags: JS
excerpt: "介绍函数表达式与函数声明的区别，以及讨论其在各种环境中的非规范行为和可能引发的问题。"
author: "Eric Zong"
date: 2019-06-11 
---

* content
{:toc}

> 英文原文：[Named function expressions demystified](http://kangax.github.io/nfe/)
>
> 创建于：2009-06-17
>
> 最后更新于：2014-08-04
>
> 译者前言：本文开头就比较系统地讨论了 JS 中命名函数表达式和函数声明的区别，这部分很有参考和研究意义，建议仔细阅读。不过，随后作者用了大量的篇幅讨论 JScript 脚本语言以及多种浏览器（引擎）实现上的差异和缺陷等等。鉴于 JScript 目前应该很少使用，不感兴趣的小伙伴可以跳过相应章节。而讨论的多种浏览器实现的差异和缺陷都是基于相当陈旧的古董版本了，所以学习意义也不大。呃……是不是觉得后续的讨论方向很奇怪？如果注意到本文的写作时间，也许就很好理解了。

# 介绍

令人惊讶的是，在网络上似乎没有充分涵盖命名函数表达式的主题。这可能就是为什么会出现这么多误解的原因。在本文中，我将尝试从理论和实践两方面总结这些奇妙的 Javascript 结构；它们好的、坏的以及丑陋的部分。

简而言之，命名函数表达式仅对一件事有用——调试器（debugger）和分析器（profiler）中的描述性函数名。好吧，也有可能使用函数名进行递归，但你很快就会发现这通常是不明智的。如果你不关心调试体验，则无需担心。否则，请继续阅读以查看你将要处理的一些跨浏览器故障以及如何解决这些故障的提示。

我将首先解释一下函数表达式是什么，以及现代调试器如何处理它们。要跳转到[最终解决方案](#jscript-解决方案)请随意，它解释了如何安全地使用这些结构。

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

熟悉 JScript 差异，我们现在可以看到使用这些有缺陷的构造时内存消耗的潜在问题。让我们来看一个简单示例：

```js
var f = (function(){
  if (true) {
    return function g(){};
  }
  return function g(){};
})();
```

我们知道从这个匿名调用中返回的函数——具有 `g` 标识符的那个——被赋值给了外部的 `f`。我们也知道命名函数表达式产生多余的函数，并且该对象与返回的函数不同。这里的内存问题是由这个无关的 `g` 函数在返回函数的闭包中字面上的“陷阱”引起的。发生这种情况是因为内部函数声明在与那个讨厌的 `g` 函数相同的作用域。除非我们明确地断开对 `g` 函数的引用，否则它会继续消耗内存。

```js
var f = (function(){
  var f, g;
  if (true) {
    f = function g(){};
  }
  else {
    f = function g(){};
  }
  // 使 g 无效，以便它不再引用那个无关的函数
  g = null;
  return f;
})();
```

注意，我们也明确声明了 `g`，因此 `g = null` 赋值不会在符合规范的客户端（例如，非 JScript 的客户端）中创建一个全局的 `g` 变量。通过使 `g` 引用无效，我们允许垃圾收集器清除 `g` 引用的隐式创建的函数对象。

当处理 JScript 命名函数表达式的内存泄漏时，我决定运行一系列简单的测试来确定使 `g` 无效确实释放了内存。

# 测试

这个测试很简单。它将通过命名函数表达式创建 10000 个函数并存储在一个数组中。我会稍等一会儿，再检查内存消耗到底有多高。然后，我将使引用无效并再次重复该过程。下面是我使用的一个测试用例：

```js
function createFn(){
  return (function(){
    var f;
    if (true) {
      f = function F(){
        return 'standard';
      };
    }
    else if (false) {
      f = function F(){
        return 'alternative';
      };
    }
    else {
      f = function F(){
        return 'fallback';
      };
    }
    // var F = null;
    return f;
  })();
}

var arr = [ ];
for (var i=0; i<10000; i++) {
  arr[i] = createFn();
}
```

在 Windows XP SP2 上的 Process Explorer 中看到的结果是：

```
  IE6:

    without `null`:   7.6K -> 20.3K
    with `null`:      7.6K -> 18K

  IE7:

    without `null`:   14K -> 29.7K
    with `null`:      14K -> 27K
```

结果在某种程度上证实了我的假设——明确地将多余引用无效化的确释放了内存，但是消耗方面的差异相对微不足道。对于 10000 个函数对象而言，会有大约 3MB 的差异。当设计大型应用、长时间运行的应用或者内存有限的设备（如移动设备）时，应该牢记这一点。对于小脚本而言，这样的差异几乎是无关紧要的。

你可能以为这就结束了，但我们还没有彻底结束 :) 这里我想提一些小细节，这些细节是跟 Safari 2.x 有关的。

# Safari 的 Bug

在较老版本的 Safari（即 Safari 2.x 系列）中，甚至存在不太广为人知的命名函数表达式 bug。我在网上看到一些[声明](http://meyerweb.com/eric/thoughts/2005/07/11/safari-syntaxerror/)称 Safari 2.x 根本不支持命名函数表达式。不是这样的。Safari 确实支持，但实现中存在 bug，你马上将看到这些 bug。

在某个上下文中遇到函数表达式时，Safari 2.x 无法完全解析该程序。它不会抛出任何错误（比如，`SyntaxError`）。而只是简单退出（bail out）：

```js
(function f(){})(); // <== NFE
alert(1); // 该行不会执行到，因为前一表达式使整个程序失败了
```

在摆弄了各种测试用例后，我得出结论，如果命名函数表达式不是赋值表达式的一部分，Safari 2.x 将无法解析。赋值表达式的一些示例：

```js
// 变量声明部分
var f = 1;

// 简单赋值部分
f = 2, g = 3;

// return 语句部分
(function(){
  return (f = 2);
})();
```

这意味着将命名函数表达式放入赋值中会使 Safari “满意”：

```js
(function f(){}); // 失败

var f = function f(){}; // 有效

(function(){
  return function f(){}; // 失败
})();

(function(){
  return (f = function f(){}); // 有效
})();

setTimeout(function f(){ }, 100); // 失败

Person.prototype = {
  say: function say() { ... } // 失败
}

Person.prototype.say = function say(){ ... }; // 有效
```

这也意味着我们不能使用这样的常见模式来返回没有赋值的命名函数表达式：

```js
// 替换这种非 Safari 2.x 兼容的语法
(function(){
  if (featureTest) {
    return function f(){};
  }
  return function f(){};
})();

// 我们应该使用这个稍微冗长的替代方案：
(function(){
  var f;
  if (featureTest) {
    f = function f(){};
  }
  else {
    f = function f(){};
  }
  return f;
})();

// 或者另一种变体：
(function(){
  var f;
  if (featureTest) {
    return (f = function f(){});
  }
  return (f = function f(){});
})();

/*
  不幸的是，这样做，我们引入了对函数的额外引用，它被限制在一个返回函数的闭包中。
  为了防止额外的内存使用，我们可以将所有命名函数表达式赋值给一个单一的变量。
*/

var __temp;

(function(){
  if (featureTest) {
    return (__temp = function f(){});
  }
  return (__temp = function f(){});
})();

...

(function(){
  if (featureTest2) {
    return (__temp = function g(){});
  }
  return (__temp = function g(){});
})();

/*
  注意，后续赋值会销毁之前的引用，防止任何过多的内存使用。
*/
```

如果 Safari 2.x 兼容性很重要，我们需要确保“不兼容”的结构甚至不会出现在源代码中。这当然是非常令人不悦的，但是绝对有可能实现，特别是在了解了问题的根源时。

还值得一提的是，在 Safari 2.x 中将一个函数声明为命名函数表达式会出现另一个小问题，该函数描述不会包含函数标识符：

```js
var f = function g(){};

// 注意函数描述是如何缺少 g 标识符的
String(f); // function () { }
```

这不是什么大不了的事。正如我之前已经提到的，无论如何都不应该依赖函数反编译（decompilation）。

# SpiderMonkey 的特点

我们知道命名函数表达式的标识符仅在函数局部作用域可用。但这种“神奇”的作用域实际上是如何发生的呢？它看起来很简单。计算命名函数表达式时，会创建一个特殊对象。该对象的唯一目的是保存一个名称对应于函数标识符的属性，以及与函数本身对应的值。然后将该对象注入当前作用域链的前端，然后使用此“扩充”作用域链初始化函数。

然而，这里有趣的部分是 ECMA-262 定义这个“特殊”对象（一个包含函数标识符的对象）的方式。规范表明，从字面上说，一个“像通过 new Object() 表达式一样”创建的对象使得这个对象是内置 `Object` 构造器的实例。但是，只有一个实现——SpiderMonkey——按字面意思遵循了此规范要求。在 SpiderMonkey 中，通过扩充 `Object.prototype` 可以干扰函数局部变量：

```js
Object.prototype.x = 'outer';

(function(){

  var x = 'inner';

  /*
     foo 函数在其作用域链中有一个特殊对象用于保存标识符。那个对象实际上是 { foo: <function object> }。
     当通过作用域链解析 x 时，首先在 foo 局部上下文中搜索。未找到时，将在作用域链的下一个对象中搜索。
     该对象即是保存标识符那个——{ foo: <function object> }，而由于它继承自 Object.prototype，x 正好在这里，
     即 Object.prototype.x（值为 'outer'）。外部函数的作用域（x === 'inner' 那个）甚至从未达到。
  */

  (function foo(){

    alert(x); // 警告 'outer'

  })();
})();
```

注意，后续版本的 SpiderMonkey 实际上改变了这种行为，因为它可能被认为是一个安全漏洞。“特殊”对象不再继承自 `Object.prototype`。但是，你仍然可以在 Firefox <=3 中见到它。

实现内部对象作为全局 `Object` 实例的另一个环境是 Blackberry 浏览器。只是这一次，它是继承自 `Object.prototype` 的活动对象（Activation Object）。请注意，规范实际上没有写创建的活动对象要“像通过 new Object() 表达式一样”（与命名函数表达式的标识符持有者对象的情况一样）。它规定活动对象仅仅是一种规范机制。

那么，让我们看看 Blackberry 浏览器中会发生什么：

```js
Object.prototype.x = 'outer';

(function(){

  var x = 'inner';

  (function(){

    /*
    当依据作用域链解析 x 时，将首先搜索局部函数的活动对象。当然，那里没有 x。
    但是，由于活动对象继承自 Object.prototype，接下来将搜索 Object.prototype。
    实际上 Object.prototype.x 确实存在，因此 x 将解析为它的值——'outer'。
    与前面的示例一样，具有它自己的 x === 'inner' 的外部函数作用域（活动对象）甚至从未达到。
    */

    alert(x); // 警告 'outer'

  })();
})();
```

这可能看起来很奇怪，但真正令人不安的是，这与现有的 `Object.prototype` 成员发生冲突的可能性更大：

```js
(function(){

  var constructor = function(){ return 1; };

  (function(){

    constructor(); // 计算得到对象 {}，而不是 1

    constructor === Object.prototype.constructor; // true
    toString === Object.prototype.toString; // true

    // etc.

  })();
})();
```

这个 Blackberry 差异的解决方案很明显：避免将命名变量作为 `Object.prototype` 的属性——`toString`、`valueOf`、`hasOwnProperty` 等等。

# JScript 解决方案

```js
var fn = (function(){

  // 声明一个变量来接受函数对象的赋值
  var f;

  // 条件化地创建一个命名函数并将其引用赋值给 f
  if (true) {
    f = function F(){ };
  }
  else if (false) {
    f = function F(){ };
  }
  else {
    f = function F(){ };
  }

  // 将 null 赋值给与函数名对应的变量。
  // 这标记该函数对象（标识符引用的）可被垃圾收集
  var F = null;

  // 返回一个条件化定义的函数
  return f;
})();
```

最后，在编写类似跨浏览器的 `addEvent` 函数时，我们将如何在现实中应用这种“技术”：

```js
// 1) 用独立的作用域包围声明
var addEvent = (function(){

  var docEl = document.documentElement;

  // 2) 声明一个接受函数赋值的变量
  var fn;

  if (docEl.addEventListener) {

    // 3) 确保给函数取一个描述性的标识符
    fn = function addEvent(element, eventName, callback) {
      element.addEventListener(eventName, callback, false);
    };
  }
  else if (docEl.attachEvent) {
    fn = function addEvent(element, eventName, callback) {
      element.attachEvent('on' + eventName, callback);
    };
  }
  else {
    fn = function addEvent(element, eventName, callback) {
      element['on' + eventName] = callback;
    };
  }

  // 4) 清理 JScript 创建的 addEvent 函数
  //   确保赋值前置有 var、在函数顶部声明 addEvent
  var addEvent = null;

  // 5) 最后返回 fn 引用的函数
  return fn;
})();
```

# 替代解决方案

值得一提的是，实际上存在在调用栈中具有描述性名称的替代方法。不需要使用命名函数表达式的方法。首先，通常可以通过声明而不是表达式来定义函数。只有在不需要创建多个函数时，此选项才可用：

```js
var hasClassName = (function(){

  // 定义一些私有变量
  var cache = { };

  // 使用函数声明
  function hasClassName(element, className) {
    var _className = '(?:^|\\s+)' + className + '(?:\\s+|$)';
    var re = cache[_className] || (cache[_className] = new RegExp(_className));
    return re.test(element.className);
  }

  // 返回函数
  return hasClassName;
})();
```

在分支函数定义时，显然不起作用。然而，这里有一个有趣的模式，我首次看到是 [Tobie Langel](http://tobielangel.com/) 在使用。这个可行的方法是预先使用函数声明定义所有函数，但是赋予它们稍微不同的标识符：

```js
var addEvent = (function(){

  var docEl = document.documentElement;

  function addEventListener(){
    /* ... */
  }
  function attachEvent(){
    /* ... */
  }
  function addEventAsProperty(){
    /* ... */
  }

  if (typeof docEl.addEventListener != 'undefined') {
    return addEventListener;
  }
  else if (typeof docEl.attachEvent != 'undefined') {
    return attachEvent;
  }
  return addEventAsProperty;
})();
```

虽然这是一种优雅的方法，但它有其自身的缺点。首先，使用不同的标识符，你将失去命名一致性。它的好坏都不是很清楚。有些人可能更愿意拥有相同的名字，而有些人则不会介意不同的名字；毕竟，不同的名字通常“描述”了使用的实现。比如，在调试器中看到“attachEvent”，会让你知道这是一个基于 `attachEvent` 实现的 `addEvent`。另一方面，与实现相关的名称可能根本没有意义。如果你以这种方式提供 API 并命名“内部”函数，那么 API 的用户很容易迷失在所有的实现细节中。

该问题的解决方案可能是采用不同的命名约定。小心不要引入额外的复杂性（verbosity）。想到的一些替代方案是：

```
`addEvent`, `altAddEvent` and `fallbackAddEvent`
  // or
  `addEvent`, `addEvent2`, `addEvent3`
  // or
  `addEvent_addEventListener`, `addEvent_attachEvent`, `addEvent_asProperty`
```

这种模式的另一个小问题是增加了内存消耗。通过预先定义所有函数变体，你隐式创建了 n-1 个未使用的函数。如你所见，如果在 `document.documentElement` 中找到了 `attachEvent`，那么 `addEventListener` 和 `addEventAsProperty` 都不会真正使用。然而，它们已经消耗了内存；由于与 JScript 古怪的命名表达式相同的原因,内存永远不会释放——两个函数都会被“捕获（trap）”到返回的函数闭包中。

增加的消耗无乎不算是什么问题。如果像 Prototype.js 这样的库要使用这个模式，那么额外创建的函数对象不会超过 100-200 个。只要函数不是以这种方式重复创建（运行时），而只有一次（在加载时），你可能不应该担心。

# WebKit 的显示名

WebKit 团队采取了一种稍微不同的方法。不好的函数描述令人沮丧——不论匿名的还是命名的——WebKit 引入了“特殊” 的 `displayName` 属性（本质上是一个字符串），当一个函数为其赋值时，它会在调试器/分析器中显示以代替该函数的“名称”。Francisco Tolmasky [详细解释](http://www.alertdebugging.com/2009/04/29/building-a-better-javascript-profiler-with-webkit/)了该解决方案的基本原理和实现。

# 对未来的讨论

即将推出的 ECMAScript 版本——ECMA-262，第 5 版——引入了所谓的严格模式。严格模式的目的是为了禁止语言中某些被认为是脆弱、不可靠或危险的部分。这部分其中之一是 `arguments.callee`，“被禁止”很可能是出于安全考虑。在严格模式下，访问 `arguments.callee` 会导致 `TypeError`（参见 10.6 节）。我提及严格模式的原因是因为无法在第 5 版中使用 `arguments.callee` 进行递归，这很可能会导致命名函数表达式的使用增加。理解它们的语义和 bug 会变得更加重要。

```js
// 以前，你可以使用 arguments.callee
(function(x) {
  if (x <= 1) return 1;
  return x * arguments.callee(x - 1);
})(10);

// 在严格模式下，一个替代方案是使用命名函数表达式
(function factorial(x) {
  if (x <= 1) return 1;
  return x * factorial(x - 1);
})(10);

// 或者只是回到稍微不那么灵活的函数声明
function factorial(x) {
  if (x <= 1) return 1;
  return x * factorial(x - 1);
}
factorial(10);
```

# 鸣谢

Richard Cornford，是第一批[解释关于命名函数表达式的 JScript bug](http://groups.google.com/group/comp.lang.javascript/msg/5b508b03b004bce8) 的人之一。 Richard 解释了本文中提到的大多数 bug。我强烈建议阅读他的解释。我还要感谢 Yann-Erwan Perio 和 Douglas Crockford 早在 2003 年就[提到并讨论了 comp.lang.javascript 中的命名函数表达式问题](http://groups.google.com/group/comp.lang.javascript/msg/03d53d114d176323)。

John-David Dalton，提供有关“最终解决方案”的有用建议。  

Tobie Langel，在“替代解决方案”中提出的想法。

Garrett Smith 和 Dmitry A. Soshnikov 进行了各种补充和更正。

有关俄语 ECMAScript 函数详细说明，请参考 [Dmitry A. Soshnikov 的这篇文章](http://dmitrysoshnikov.com/ecmascript/ru-chapter-5-functions/)。

> 译者后记
>
> 最终翻译完了！！！本文应该是目前为止翻译最长的一篇，前后用掉了好几天的业余时间。
>
> 由于文风比较学术范儿，又在辨析概念间的区别，很多词句结构也比较复杂，翻译的学术性和准确性要求都较高，翻译难度真心不小。
>
> 本着翻译力求准确的宗旨，绝大部分内容都采用直译；部分不符合中文语言习惯的句子略有调整，或使用意译的方式；极少部分句子由于多种原因，含义不易理解，翻译相当困难，一般根据上下文推测其含义，因此，翻译或与作者本意有出入，欢迎指正。

