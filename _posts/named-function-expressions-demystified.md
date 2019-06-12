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

令人惊讶的是，在网络上似乎没有充分涵盖命名函数表达式的主题。这可能就是为什么会出现这么多误解的原因。在本文中，我将尝试从理论和实践两方面总结这些奇妙的 Javascript 结构；它们的好，坏以及丑陋的部分。

简而言之，命名函数表达式仅对一件事有用——调试器（debugger）和分析器（profiler）中的描述性函数名称。好吧，也有可能使用函数名称进行递归，但你很快就会发现目前这通常是不明智的。如果你不关心调试体验，则无需担心。否则，请继续阅读以查看你将要处理的一些跨浏览器故障以及如何解决这些故障的提示。

我将首先解释一下函数表达式是什么，以及现代调试器如何处理它们。要跳转到[最终解决方案](#解决方案)请随意，它解释了如何安全地使用这些结构。

# 函数表达式与函数声明

在 ECMAScript 中创建函数对象的两种最常用方法之一是通过函数表达式或函数声明。两者的区别相当让人困惑。至少对我而言是这样。ECMA 规范表明的唯一一点是，函数声明必须总是具有标识符（或者说函数名，如果你更喜欢这种表述的话），而函数表达式也许会忽略这点：

> 函数声明 ：
> function Identifier ( FormalParameterList ~opt~ ){ FunctionBody }
>
> 函数表达式：
> function Identifier ~opt~ ( FormalParameterList ~opt~ ){ FunctionBody }

我们可以看到，当省略标识符时，“某些东西”只能是一个表达式。但是如果有标识符怎么办？如何判断它是函数声明还是函数表达式——毕竟它们看起来是相同的？ECMAScript 似乎根据上下文区分了两者。如果 `function foo(){}` 是赋值表达式的一部分, 则它被视为函数表达式。另一方面, 如果 `function foo(){}` 包含在函数体或 （顶级）程序本身中，则它被解析为函数声明。

```js
function foo(){} // 声明，因为它是程序（Program）的一部分
var bar = function foo(){}; // 表达式，因为它是赋值表达式（AssignmentExpression）的一部分

new function bar(){}; // 表达式，因为它是创建表达式（NewExpression）的一部分

(function(){
  function bar(){} // 声明，因为它是函数体（FunctionBody）的一部分
})();
```

函数表达式的一个不太明显的例子是用括号包围起来的例子——`(function foo(){})`。它是表达式的原因又是由于上下文：“(” 和 “)” 构成分组运算符（grouping operator），分组运算符只能包含表达式：

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

你可能还记得，在使用 `eval` 计算 JSON 时，字符串通常用括号包围——`eval('(' + json + ')')`。这当然是出于同样的原因而做的——括号就是分组运算符，它强制将 JSON 括号解析为表达式，而不是块：

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

> 函数声明只允许出现在程序（Program）或函数体（FunctionBody）中。从语法上说，它们不能出现在块（`{...}`）中——比如，`if`、`while` 或 `for` 语句。这是因为块只能包含语句，而不能是函数声明这样的源元素（SourceElement）。如果我们仔细查看产生规则，我们可以看到，允许表达式直接在块中的唯一方法是当它是表达式语句的一部分时。但是，表达式语句被明确定义为不以 “function” 关键字开头，这正是为什么功能声明不能直接出现在语句或块中的原因（请注意，块只是一个语句列表）。
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



# 调试中的函数名



# JScript 的 Bug



# JScript 内存管理



# 测试



# Safari 的 Bug



# SpiderMonkey 的特点



# 解决方案



# 可选解决方案



# WebKit 的显示名



# 未来的考虑



# 鸣谢

