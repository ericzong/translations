---
layout: post
title: 如何处理纯函数式JavaScript的不良副作用
category: 编程语言
tags: 语言 JS
excerpt: 
author: Eric Zong
date: 
---

* content
{:toc}

> 英文原文：[How to deal with dirty side effects in your pure functional JavaScript](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)
>
> 原文作者：James Sinclair
>
> 原文日期：2018-08-07

看来，你已经开始涉足函数编程了。过不了多久你就会碰到纯函数的概念。而且，随着你的继续，你会发现函数式程序员似乎对其很着迷。“纯函数是你编码的原因”，他们说。“纯函数不太可能像开始一场热核战争一样。”“纯函数为你提供引用透明性[^T1]。”然后继续。他们也没有错。纯函数是一件好事。但是有一个问题……

一个纯函数是一个没有副作用的函数。[^1]但是如果你对编程有所了解，你就知道副作用才是关键（whole point）。如果没人有办法读取 𝜋，为什么还要费心计算到 100 位？将它在某个地方打印出来，我们需要写到控制台，或者将数据发送到打印机，或者其他人可以读取的地方。如果你不能输入任何数据到数据库，那它还有什么用？我们需要从输入设备读取数据，以及从网络请求信息。我们不可能毫无副作用地做到这些。然而，函数编程是建立在纯函数的基础上的。那么，函数式程序员是如何设法完成这些事情的呢？

简而言之，他们做数学家做的事：作弊。

现在，当我说他们作弊的同时，他们又严格遵守了这些规则。但是他们发现了这些规则中的漏洞，并将它们扩大到足以驱使一群大象通过。他们有两种两种主要的方法：

1. 依赖注入，或者我谓的，把问题抛到一边；
2. 使用效果函子（effect functor），而我认为这极度拖延。[^2]

# 依赖注入

依赖注入是我们首个处理副作用的方法。在这种方法中，我们把代码中的所有不纯粹（impurity），塞进函数参数中。然后我们可以把它们当作其他函数的责任。为了解释我是什么意思，让我们来看一些代码：[^3]

```js
// logSomething :: String -> String
function logSomething(something) {
    const dt = (new Date()).toISOString();
    console.log(`${dt}: ${something}`);
    return something;
}
```

我们的 `logSomething()` 函数有两个不纯粹的来源：它创建了一个 `Date()` 并记录到了控制台。因此，它不仅仅执行了 IO，当你运行时，它将给出一个每毫秒都不同的结果。那么，你如何使得这个函数变纯粹呢？使用依赖注入，我们抽取所有不纯粹并使其成为一个函数参数。因此，我们的函数要抽取三个参数，而不是抽取一个参数：

```js
// logSomething: Date -> Console -> String -> *
function logSomething(d, cnsl, something) {
    const dt = d.toIsoString();
    return cnsl.log(`${dt}: ${something}`);
}
```

然后要调用它，我们必须自己明确地传入不纯粹的部分：

```js
const something = "Curiouser and curiouser!"
const d = new Date();
logSomething(d, console, something);
// ⦘ Curiouser and curiouser!
```

现在，你也许会想：“这也太傻了。我们所做的只是把问题抛到了上一级。这仍然跟以前一样不纯粹。”你是对的。总体上说，这是一个漏洞。

（https://youtu.be/9ZSoJDUD_bU [^T2]）

这就像假装无知：“哦，不，警官，我不知道在那个“`cnsl`”对象上调用 `log()` 会执行 IO。其他人刚刚把它传给了我。我不知道它从哪里来。”这似乎有点蹩脚。

不过，这并不像看起来那么愚蠢。请注意我们的 `logSomething()` 函数。如果你想让它做不纯粹的事情，你必须让它不纯粹。我们同样可以简单地传递不同的参数：

```js
const d = {toISOString: () => '1865-11-26T16:00:00.000Z'};
const cnsl = {
    log: () => {
        // do nothing
    },
};
logSomething(d, cnsl, "Off with their heads!");
//  ￩ "Off with their heads!"
```

现在，我们的函数什么都不做（除了返回 `something` 参数）。但是它完全是纯粹的。如果你用相同的参数调用它，它每次都会返回相同的内容。而这就是关键。为了使它不纯粹，我们必须采取蓄意的行为。或者，换句话说，函数所依赖的一切都在签名中。它不会访问任何全局对象，比如 `console` 或 `Date`。这使得一切都是明确的。

同样重要的是，要注意，我们也可以将函数传递给以前不纯粹的函数。让我们看另一个示例。想象我们在某处的表单中有一个用户名。我们希望得到该表单输入的值：

```js
// getUserNameFromDOM :: () -> String
function getUserNameFromDOM() {
    return document.querySelector('#username').value;
}

const username = getUserNameFromDOM();
username;
// ￩ "mhatter"
```

在这个场景中，我们为了得到某些信息试图查询 DOM。这是不纯粹的，因为 `document` 是一个全局对象，它可能在任意时刻改变。使得我们的函数纯粹的一个方法是将全局的 `document` 对象作为参数传递。但是，我们也可以像这样传递一个 `querySelector()` 函数：

```js
// getUserNameFromDOM :: (String -> Element) -> String
function getUserNameFromDOM($) {
    return $('#username').value;
}

// qs :: String -> Element
const qs = document.querySelector.bind(document);

const username = getUserNameFromDOM(qs);
username;
// ￩ "mhatter"
```

现在，你也许又在想，“这仍然很傻呀！”我们所做的一切只是将不纯粹移出 `getUserNameFromDOM()`。它没有消失。我们仍困在另一个函数 `qs()` 中。除了让代码变长之外，它似乎没什么作用。为了替换一个不纯粹的函数，我们有了两个函数，而其中一个仍然不纯粹。

忍耐一下。想象我们要为 `getUserNameFromDOM()` 写一个测试。现在，比较一下不纯粹和纯粹的版本，哪一个更容易使用？为了让不纯粹的版本能够工作，我们需要一个全局的文档对象。最重要的是，这需要在文档某处有一个 ID 为 `username` 的元素。如果我想在浏览器之外测试它，那么我必须导入一些东西，比如 JSDOM 或一个无头浏览器。这一切都只是为了测试一个非常小的函数。但是使用第二个版本，我可以这样做：

```js
const qsStub = () => ({value: 'mhatter'});
const username = getUserNameFromDOM(qsStub);
assert.strictEqual('mhatter', username, `Expected username to be ${username}`);
```

现在，这并不意味着你不应该创建一个在真实浏览器中运行的集成测试。（或者，至少一个像 JSDOM 这样的模拟测试）。但是这个示例所做的表明 `getUserNameFromDOM()` 现在是完全可预测的。如果我们传给它 qsStub，它总会返回 `mhatter`。我们把不可预测性转移到了更小的函数 `qs` 中。

如果我们愿意，我们可以继续把这种不可预测性推得越来越远。最终，我们将它们推到代码的最边缘。因此，我们最终得到了一个由不纯粹代码组成的薄壳，它包裹着一个经过良好测试的、可预测的内核。随着你开始构建更大的应用程序，可预测性开始变得重要许多。

# 依赖注入的缺点



# 懒惰函数



---

[^1]: 这不是一个完整的定义，但目前可用。稍后我们将回到正式定义。
[^2]: 在其他语言（比如 Haskell）中这称为 IO 函子或 IO 单体。[PureScript](http://www.purescript.org/) 使用了这项效果。我发现它更具描述性。
[^3]: 熟悉类型签名的人请注意。如果我们是严格的，我们需要考虑副作用。我们稍后会讲到。

---

[^T1]: 引用透明（referential transparency），指的是函数的运行不依赖于外部变量或"状态"，只依赖于输入的参数，任何时候只要参数相同，引用函数所得到的返回值总是相同的。
[^T2]: 原文插入了一个视频……