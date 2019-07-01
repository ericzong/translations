---
layout: post
title: 如何处理纯函数式JavaScript的不良副作用
category: 编程语言
tags: 语言 JS
excerpt: 介绍JS相关的函数式编程以及其副作用的消除方法。
author: Eric Zong
date: 2019-07-01 12:30
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

[如果你被击中，那是你自己的错（辛普森一家）](https://youtu.be/9ZSoJDUD_bU) [^T2]

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

## 依赖注入的缺点

以这种方式创建大型的复杂应用程序是可能的。我知道因为我已经[做到了](https://www.squiz.net/technology/squiz-workplace)。测试变得容易，并且使得每个函数的依赖明确。但是，确实也有某些缺点。主要的缺点是，你最终会得到如下冗长的函数签名：

```js
function app(doc, con, ftch, store, config, ga, d, random) {
    // Application code goes here
 }

app(document, console, fetch, store, config, ga, (new Date()), Math.random);
```

这还不算太糟，除了你还有参数钻取的问题。在一个非常低级的函数中，你可能需要一些这样的参数。因此，你必须通过函数调用的许多层向下传递参数。这很烦人。例如，你可能需要通过 5 层中间函数传递日期。这些中间函数都不使用日期对象。这不是世界末日。能够看到这些显式的依赖关系是很好的。但还是很烦人。还有另一种方法…

# 懒惰函数

让我们看看函数式程序员利用的第二个漏洞。事情是这样开始的：副作用直到它真正发生才算副作用。听起来晦涩难懂，我知道。让我们试着说得更清楚一点。考虑以下代码：

```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}
```

我知道这是个愚蠢的例子。如果我们在代码中想要一个零，我们可以直接写出来。我知道你，温柔的读者，绝不会用 JavaScript 编写控制核武器的代码。但这有助于说明这一点。这很显然是不纯粹的代码。它将日志输出到控制台，并且也许会引发核战争。想象一下我们想要零。想象一个场景，在导弹发射后我们想计算一些东西。我们可能需要启动倒计时器之类的东西。在这种情况下，提前计划好如何进行计算是完全合理的。我们要非常小心那些导弹何时起飞。我们不想把我们的计算混为一谈，以免他们意外发射导弹。那么，如果我们将 `fZero()` 包装在另一个仅返回它的函数中会怎么样呢？有点像安全包装。

```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}

// returnZeroFunc :: () -> (() -> Number)
function returnZeroFunc() {
    return fZero;
}
```

我可以运行 `returnZeroFunc()` 任意多次，只要我不调用返回值，我（理论上）是安全的。我的代码不会发射任何核导弹。

```js
const zeroFunc1 = returnZeroFunc();
const zeroFunc2 = returnZeroFunc();
const zeroFunc3 = returnZeroFunc();
// No nuclear missiles launched.
```

现在，让我们更正式地定义纯函数。然后我们可以更详细地检查 `returnZeroFunc()` 函数。函数是纯函数，如果：

1. 它没有明显的副作用；
2. 它是引用透明的。也就是说，给定相同的输入，它总是返回相同的输出。

让我们检查一下 `returnZeroFunc()`。它有副作用吗？好吧，我们仅仅确定调用 `returnZeroFunc()` 不会发射任何核导弹。除非你执行到调用返回的函数的额外步骤，否则什么都不会发生。所以，这没有副作用。

`returnZeroFunc()` 是引用透明的吗？也就是说，给定相同的输入它是否总是返回相同的值？按照目前的编写方式，我们可以测试它：

```js
zeroFunc1 === zeroFunc2; // true
zeroFunc2 === zeroFunc3; // true
```

但是这还不十分纯粹。我们的 `returnZeroFunc()` 函数引用了一个其作用域外的变量。为了解决这个问题，我们可以用以下方式重写：

```js
// returnZeroFunc :: () -> (() -> Number)
function returnZeroFunc() {
    function fZero() {
        console.log('Launching nuclear missiles');
        // Code to launch nuclear missiles goes here
        return 0;
    }
    return fZero;
}
```

现在我们的函数是纯的了。但是，JavaScript 在这里对我们有点不利。我们不能再用 === 来证实引用透明性了。这是因为 `returnZeroFunc()` 将总是返回一个新的函数引用。但是你可以通过检查代码来检验引用透明性。我们的 `returnZeroFunc()` 函数每次只返回同一个函数。

这是一个整洁的（neat）小漏洞。但是我们真的能把它用于真正的代码吗？回答是肯定的。但是在我们讨论你如何在实践中做到这一点之前，让我们把这个想法推进一点。回到我们危险的 `fZero()` 函数：

```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}
```

让我们试着使用 `fZero()` 返回的零，但不要引发热核战争。

我们将创建一个函数，它获取 `fZero()` 最终返回的零，并加 1：

```js
// fIncrement :: (() -> Number) -> Number
function fIncrement(f) {
    return f() + 1;
}

fIncrement(fZero);
// ⦘ Launching nuclear missiles
// ￩ 1
```

哎哟。我们意外地引发了热核战争。让我们再试一次。这次，我们不会返回一个数字。相反，我们将返回一个最终会返回一个数字的函数：

```js
// fIncrement :: (() -> Number) -> (() -> Number)
function fIncrement(f) {
    return () => f() + 1;
}

fIncrement(zero);
// ￩ [Function]
```

唷。危机避免了。让我们继续。有了这两个函数，我们可以创建一大堆“最终数字”：

```js
const fOne   = fIncrement(zero);
const fTwo   = fIncrement(one);
const fThree = fIncrement(two);
// And so on…
```

我们还可以创建一组 `f*` (函数)来处理最终的值：

```js
// fMultiply :: (() -> Number) -> (() -> Number) -> (() -> Number)
function fMultiply(a, b) {
    return () => a() * b();
}

// fPow :: (() -> Number) -> (() -> Number) -> (() -> Number)
function fPow(a, b) {
    return () => Math.pow(a(), b());
}

// fSqrt :: (() -> Number) -> (() -> Number)
function fSqrt(x) {
    return () => Math.sqrt(x());
}

const fFour = fPow(fTwo, fTwo);
const fEight = fMultiply(fFour, fTwo);
const fTwentySeven = fPow(fThree, fThree);
const fNine = fSqrt(fTwentySeven);
// No console log or thermonuclear war. Jolly good show!
```

你看到我们在这里做了什么吗？任何我们想用常规数字做的事情，我们都可以用最终数字来做。数学家称之为“[同构](https://en.wikipedia.org/wiki/Isomorphism)”。我们总是可以通过把一个常规数字插入一个函数，把它变成一个最终数字。通过调用函数，我们可以得到最终数字。换句话说，我们有一个数字和最终数字之间的映射。这比听起来更令人兴奋。我保证。我们很快就会回到这个概念。

这个函数包装是一个合法的策略。只要我们愿意，我们可以一直隐藏背后的函数。只要我们从未真正调用过这些函数，它们理论上都是纯的。没有人会发动战争。在常规（无核）代码中，我们最终实际上想要这些副作用。将所有东西包装在一个函数中可以让我们精确地控制这些影响。我们明确地决定这些副作用何时发生。但是，到处敲这些括号是很痛苦的。并且创建每个函数的新版本很烦人。我们能使用如 `Math.sqrt()` 这样好用的语言内置函数。如果有一种方法可以将这些普通函数用于我们的延迟值，那就太好了。进入效果函子。

# 效果函子（Effect Functor）

对我们来说，效果函子只不过是一个对象，我们把延迟函数放在里面。所以，我们将把 `fZero` 函数放入一个效果对象中。但是，在此之前，让我们把压力降低一个等级：

```js
// zero :: () -> Number
function fZero() {
    console.log('Starting with nothing');
    // Definitely not launching a nuclear strike here.
    // But this function is still impure.
    return 0;
}
```

现在我们创建一个构造函数，为我们创建一个效果对象：

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {};
}
```

到目前为止没什么可看的。让我们使它做些有用的事情。我们想在我们的效果中使用常规的 `fZero()` 函数。我们将编写一个使用常规函数的方法，并最终将其应用于我们的延迟值。我们将执行这个方法但不触发其效果。我们称之为映射。这是因为它创建常规函数和效果函数之间的映射。它可能看起来像这样：

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        }
    }
}
```

现在，如果你注意的话，你可能会想知道 `map()`。它看起来很可疑，比如构成方面。我们稍后再谈这个。现在，让我们试一试：

```js
const zero = Effect(fZero);
const increment = x => x + 1; // A plain ol' regular function.
const one = zero.map(increment);
```

嗯。我们真的没有办法看到发生了什么。让我们修改效果，这样我们就有办法“拉动触发器”，可以这么说：

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
    }
}

const zero = Effect(fZero);
const increment = x => x + 1; // Just a regular function.
const one = zero.map(increment);

one.runEffects();
// ⦘ Starting with nothing
// ￩ 1
```

如果我们愿意，我们可以持续调用映射函数：

```js
const double = x => x * 2;
const cube = x => Math.pow(x, 3);
const eight = Effect(fZero)
    .map(increment)
    .map(double)
    .map(cube);

eight.runEffects();
// ⦘ Starting with nothing
// ￩ 8
```

现在，这是它开始变得有趣的地方。我们称之为“函子（functor）”。这意味着效果有一个映射函数，它[遵守一些规则](https://github.com/fantasyland/fantasy-land#functor)。这些规则不是你不能做的事情的规则。它们是你能做的事情的规则。它们更像特权。因为效果是函子俱乐部的一部分，所以它会做一些特定的事情。其中之一被称为“合成法则”。事情是这样的：

如果我们有一个效果 `e`，和两个函数 `f` 和 `g`。而 `e.map(g).map(f)` 等价于 `e.map(x => f(g(x)))`。

换句话说，连续做两个映射相当于组合两个函数。这意味着效果可以做这样的事情（回想一下我们上面的例子）：

```js
const incDoubleCube = x => cube(double(increment(x)));
// If we're using a library like Ramda or lodash/fp we could also write:
// const incDoubleCube = compose(cube, double, increment);
const eight = Effect(fZero).map(incDoubleCube);
```

当我们这样做的时候，我们保证会得到和三映射版本相同的结果。我们可以用它来重构我们的代码，以确信我们的代码不会被破坏。在某些情况下，我们甚至可以通过方法之间的交换来提高性能。

与数字有关的示例已经够多了。让我们做一些更像“真实”代码做的事。

## 制作效果的快捷方式

我们的效果构造函数以一个函数作为参数。这很方便，因为我们想要延迟的大多数副作用也是函数。例如，`Math.random()` 和 `console.log()` 都是这一类型的东西。但是有时我们想把一个简单的旧值加入到效果中。例如，想象我们在浏览器中将一些配置对象附加到全局窗口。我们想得到一个 a 值，但这不是一个纯粹的操作。我们可以写一个小快捷方式，让这项任务更容易：[^4]

```js
// of :: a -> Effect a
Effect.of = function of(val) {
    return Effect(() => val);
}
```

为了展示这是多么方便，想象我们正在开发一个网络应用程序。该应用程序有一些标准特性，如文章列表和用户简历。但是在 HTML 中，这些组件为不同的客户带来了变化。因为我们是聪明的工程师，所以我们决定将他们的位置存储在一个全局配置对象中。这样我们总能找到他们。例如：

```js
window.myAppConf = {
    selectors: {
        'user-bio':     '.userbio',
        'article-list': '#articles',
        'user-name':    '.userfullname',
    },
    templates: {
        'greet':  'Pleased to meet you, {name}',
        'notify': 'You have {n} alerts',
    }
};
```

现在，通过我们的 `Effect.of()` 快捷方式，我们可以快速地将我们想要的值插入到一个效果包装器中，就像这样：

```js
const win = Effect.of(window);
userBioLocator = win.map(x => x.myAppConf.selectors['user-bio']);
// ￩ Effect('.userbio')
```

## 嵌套和非嵌套效果

映射效果的内容让我们有很长的路要走。但是有时我们最终会映射一个也会返回效果的函数。我们已经定义了 `getElementLocator()`，它返回一个包含字符串的效果。如果我们确实想要定位 DOM 元素，那么我们需要调用 `document.querySelector()`——另一个不纯粹的函数。因此我们可以通过返回效果来使其纯粹：

```js
// $ :: String -> Effect DOMElement
function $(selector) {
    return Effect.of(document.querySelector(s));
}
```

现在，如果我们想把这两个函数放到一起，我们可以尝试使用 `map()`：

```js
const userBio = userBioLocator.map($);
// ￩ Effect(Effect(<div>))
```

现在我们有点尴尬。如果我们想要访问该 div，我们不得不映射一个映射我们实际想要做的事情的函数。例如，如果我们想获得 `innerHTML`，它将看起来像这样：

```js
const innerHTML = userBio.map(eff => eff.map(domEl => domEl.innerHTML));
// ￩ Effect(Effect('<h2>User Biography</h2>'))
```

让我们尝试分开一点。我们将一直回到  `userBio` 并从那里前进。这将有点乏味，但我们想要弄清楚这里将会发生什么。我们一直在使用的符号，`Effect('user-bio')	` 是有一点误导人的。如果我们把它写成代码，它看起来更像是这样：

```js
Effect(() => '.userbio');
```

这也不准确。我们真正做的更像是：

```js
Effect(() => window.myAppConf.selectors['user-bio']);
```

现在，当我们映射时，这和用另一个函数组成内部函数一样（如我们上面所见）。因此，当我们用 `$` 映射时，看起来有点像这样：

```js
Effect(() => $(window.myAppConf.selectors['user-bio']));
```

展开我们能得到：

```js
Effect(
    () => Effect.of(document.querySelector(window.myAppConf.selectors['user-bio'])))
);
```

再展开 `Effect.of` 将给我们一个更清晰的画面：

```js
Effect(
    () => Effect(
        () => document.querySelector(window.myAppConf.selectors['user-bio'])
    )
);
```

注意：所有实际做事情的代码都在最里面的函数中。没有一个泄露到外部效果。

### Join

为什么要费心把这些拼写出来？嗯，我们想去嵌套这些嵌套效果。如果我们要这么做，我们要确保在这个过程中不会带来任何不必要的副作用。为了达到效果，去嵌套的方法就是在外部函数上调用 `.runEffects()`。但这可能会让人困惑。我们已经完成了整个练习，以检查我们不会运行任何效果。所以我们将创建另一个做同样事情的函数，并将其称为 `join`。当我们去消嵌套效果时，我们使用 `join`，当我们实际上想要运行效果时，我们使用 `runEffects()`。这表明我们的意图是明确的，即使我们运行的代码是相同的。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
        join(x) {
            return f(x);
        }
    }
}
```

然后，我们可以使用它来去嵌套我们的“用户传记”函数：

```js
const userBioHTML = Effect.of(window)
    .map(x => x.myAppConf.selectors['user-bio'])
    .map($)
    .join()
    .map(x => x.innerHTML);
// ￩ Effect('<h2>User Biography</h2>')
```

### Chain

这种在 `.join()` 后紧跟着执行 `.map()` 的模式经常出现。事实上，有一个快捷函数会很方便。这样，只要我们有一个返回效果的函数，我们就可以使用这个快捷方式。它保存了我们编写的 `map` 然后一遍遍地 `join`。我们会这样写：

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
        join(x) {
            return f(x);
        }
        chain(g) {
            return Effect(f).map(g).join();
        }
    }
}
```

我们称这个新函数为 `chain()` 因为它允许我们将效果链接在一起。（这也是因为标准告诉我们这样调用它。)[^5]我们获取“用户传记”内部 HTML 的代码看起来更像这样：

```js
const userBioHTML = Effect.of(window)
    .map(x => x.myAppConf.selectors['user-bio'])
    .chain($)
    .map(x => x.innerHTML);
// ￩ Effect('<h2>User Biography</h2>')
```

不幸的是，其他编程语言对此理念使用了许多不同的名称。如果你想了解它，可能会有点困惑。有时它叫做 `flatMap`。这个名字很有意义，因为我们正在做一个常规的映射，然后使用 `.join()` 使其扁平化。但是在 Haskell 中，它被赋予了一个令人困惑的名字 `bind`。因此，如果你在别处阅读，请记住 `chain`、`flatMap` 和 `bind` 是指类似的概念。

## 合并效果

还有最后一个场景，使用效果可能会有点尴尬。这就是我们想用一个函数组合两个或更多效果的地方。例如，如果我们想从 DOM 中获取用户名，会怎么样？然后将其插入到我们的应用程序配置提供的模板中？因此，我们可能有这样一个模板函数（请注意，我们正在创建一个柯里化[^6]版本）：

```js
// tpl :: String -> Object -> String
const tpl = curry(function tpl(pattern, data) {
    return Object.keys(data).reduce(
        (str, key) => str.replace(new RegExp(`{${key}}`, data[key]),
        pattern
    );
});
```

一切都很好。但是让我们来获取我们的数据：

```js
const win = Effect.of(window);
const name = win.map(w => w.myAppConfig.selectors['user-name'])
    .chain($)
    .map(el => el.innerHTML)
    .map(str => ({name: str});
// ￩ Effect({name: 'Mr. Hatter'});

const pattern = win.map(w => w.myAppConfig.templates('greeting'));
// ￩ Effect('Pleased to meet you, {name}');
```

我们已经有了一个模板函数。它接受一个字符串和一个对象，并返回一个字符串。但是我们的字符串和对象（`name` 和 `pattern`）被包装在效果中。我们想做的是将我们的 `tpl()` 函数提升到一个更高的平面，这样它就可以与效果一起工作。 

让我们先看看如果我们在模式效果上调用 `map()` 和 `tpl()` 会发生什么：

```js
pattern.map(tpl);
// ￩ Effect([Function])
```

查看类型可能会让事情变得更清楚一点。映射的类型签名如下：

map :: Effect a ~> (a -> b) -> Effect b

我们的模板函数有如下签名：

tpl :: String -> Object -> String

因此，当我们调用模式上的映射时，我们会在一个效果中得到一个部分应用的函数（记住我们柯里化了 `tpl`）。

Effect (Object -> String)

我们现在想要从模式效果内部传递值。但是我们还没有办法做到这一点。我们将编写另一种效果方法（称为 `ap()`）来解决这个问题：

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
        join(x) {
            return f(x);
        }
        chain(g) {
            return Effect(f).map(g).join();
        }
        ap(eff) {
             // If someone calls ap, we assume eff has a function inside it (rather than a value).
            // We'll use map to go inside off, and access that function (we'll call it 'g')
            // Once we've got g, we apply the value inside off f() to it
            return eff.map(g => g(f()));
        }
    }
}
```

有了这个，我们就可以执行 `.ap()` 来应用我们的模板了：

```js
const win = Effect.of(window);
const name = win.map(w => w.myAppConfig.selectors['user-name'])
    .chain($)
    .map(el => el.innerHTML)
    .map(str => ({name: str}));

const pattern = win.map(w => w.myAppConfig.templates('greeting'));

const greeting = name.ap(pattern.map(tpl));
// ￩ Effect('Pleased to meet you, Mr Hatter')
```

我们已经实现了我们的目标。但是我要坦白……问题是，我发现 `ap()` 有时会令人感到困惑。很难记得我必须先映射函数，然后再运行 `ap()`。然后我忘记了参数应用的顺序。但是有办法解决这个问题。大多数时候，我想做的是把一个普通的函数提升到应用的世界。也就是说，我有一些简单的函数，我想让它们和带有 `.ap()` 方法的效果一起工作。我们可以为自己编写一个函数：

```js
// liftA2 :: (a -> b -> c) -> (Applicative a -> Applicative b -> Applicative c)
const liftA2 = curry(function liftA2(f, x, y) {
    return y.ap(x.map(f));
    // We could also write:
    //  return x.map(f).chain(g => y.map(g));
});
```

我们称之为 `liftA2()`，因为它提升了一个包含两个参数的函数。我们也可以像这样写一个 `liftA3()`：

```js
// liftA3 :: (a -> b -> c -> d) -> (Applicative a -> Applicative b -> Applicative c -> Applicative d)
const liftA3 = curry(function liftA3(f, a, b, c) {
    return c.ap(b.ap(a.map(f)));
});
```

请注意，`liftA2` 和 `liftA3` 从未提及效果。理论上，它们可以与任何具有兼容的 `ap()` 方法的对象一起工作。 

使用 `liftA2()`，我们可以将上面的示例重写如下：

```js
const win = Effect.of(window);
const user = win.map(w => w.myAppConfig.selectors['user-name'])
    .chain($)
    .map(el => el.innerHTML)
    .map(str => ({name: str});

const pattern = win.map(w => w.myAppConfig.templates['greeting']);

const greeting = liftA2(tpl)(pattern, user);
// ￩ Effect('Pleased to meet you, Mr Hatter')
```

# 这又怎样？

在这一点上，你可能会想，“这似乎需要付出很大的努力才能避免偶尔出现的奇怪副作用。”这有什么关系？把东西放在效果里，并且让我们的头部包裹在 `ap()` 中看起来很困难。当不纯粹的代码运行良好时，为什么还要费心呢？你在现实世界中什么时候会需要这个？

> 这个函数式程序员听起来相当像一个中世纪的僧侣，否认自己生活的乐趣，希望这能让他变得高尚。 
>
> ——约翰·雨果[^7]

让我们把这些异议分成两个问题：

1. 函数的纯粹性真的重要吗？
2. 这种效果在现实世界中什么时候有用？

## 函数的纯粹性很重要

这是真的。当你孤立地看待一个小函数时，一点点不纯粹并不重要。编写 `const pattern = window.myAppConfig.templates['greeting'];` 比下面这样更快更简单：

```js
const pattern = Effect.of(window).map(w => w.myAppConfig.templates('greeting'));
```

如果这就是你所做的一切，那也是事实。副作用没关系。但这只是一行代码——在一个可能包含数千甚至数百万行代码的应用程序中。当你试图弄清楚为什么你的应用程序莫名其妙地停止工作时，函数的纯粹性开始变得更加重要。意想不到的事情发生了。你试图把问题分解开来，找出原因。在这种情况下，你能排除的代码越多越好。如果你的函数是纯粹的，那么你可以确信影响它们行为的唯一东西是传递给它的输入。这大大减少了你需要考虑的事情的数量。换句话说，它让你少思考。在大型复杂的应用程序中，这是一件大事。

## 现实世界中的效果模式

好吧。如果你正在构建一个大型复杂的应用程序，也许函数的纯粹性很重要。比如像脸书或者谷歌邮箱。但是如果你不这么做呢？让我们考虑一个将变得越来越普遍的场景。你有一些数据。不仅仅是一点点数据，而是很多数据。数百万行，在 <abbr title='Comma Separated Values'>CSV</abbr> 文本文件或巨大的数据库表中。你的任务是处理这些数据。也许你正在训练一个人工神经网络来建立一个推理模型。也许你正试图弄清楚下一个大型加密货币的进展。无论如何。问题是，完成这项工作需要大量的处理工作。

乔尔·斯波尔斯基（Joel Spolsky）令人信服地认证了，[函数式编程可以帮助我们](https://www.joelonsoftware.com/2006/08/01/can-your-programming-language-do-this/)。我们可以编写 `map` 和 `reduce` 并行运行的替代版本。函数的纯粹性使这成为可能。但这不是故事的结局。当然，你可以写一些奇特的并行处理代码。但是即使这样，你的开发机器仍然只有 4 个内核（或者如果你幸运的话，可能有 8 或 16 个）。那项工作仍然需要很长时间。除非，也就是说，你可以在一堆处理器上运行它……比如一个 GPU，或者整个集群的处理服务器。

为了使这一点有效，你需要描述你想要运行的计算。但是，你想描述它们而不是实际运行它们。听起来熟悉吗？理想情况下，你可以将描述传递给某种框架。该框架将负责读入所有数据，并分割给处理节点。然后，相同的框架将把结果收集到一起，并告诉你进展如何。TensorFlow 就是这样工作的。

> TensorFlow 是一个开源软件库，用于高性能数值计算。其灵活的体系结构允许跨多种平台（CPU、GPU、TPU）轻松部署计算，从台式机到服务器集群，再到移动和边缘设备。最初是由谷歌人工智能组织内的谷歌大脑团队的研究人员和工程师开发的，它对机器学习和深度学习有强大的支持，灵活的数值计算核心被用于许多其他科学领域。 
>
> —TensorFlow 主页[^8]

当你使用 TensorFlow 时，你不会使用你正在编写的编程语言中的常规数据类型。相反，你创造了“张量（tensor）”。如果我们想将两个数字相加，它应该是这样的：

```js
node1 = tf.constant(3.0, tf.float32)
node2 = tf.constant(4.0, tf.float32)
node3 = tf.add(node1, node2)
```

上面的代码是用 Python 编写的，但是它看起来和 JavaScript 没什么不同，是吗？就像我们的效果一样，`add` 代码直到我们告诉它运行（在这种情况下使用 `sess.run()`）才会运行：

```js
print("node3: ", node3)
print("sess.run(node3): ", sess.run(node3))
# ⦘ node3:  Tensor("Add_2:0", shape=(), dtype=float32)
# ⦘ sess.run(node3):  7.0
```

直到我们调用 `sess.run()` 我们才会得到7.0。如你所见，这和我们的延迟函数非常相似。我们提前计划好了我们的计算。然后，一旦我们准备好了，我们就扣动扳机开始一切。

# 总结

我们已经覆盖了很多内容。并且我们已经探索了两种方法来处理代码中的函数不纯粹性：

1. 依赖注入；
2. 效果函子。

依赖注入通过将代码中不纯粹的部分移出函数来工作。所以你必须把它们作为参数传入。相比之下，效果函子的工作原理是将所有东西都包装在一个函数后面。为了运行这些效果，我们必须特意运行包装函数。

这两种方法都是欺骗。它们没有完全清除不纯粹性，只是把它们推到我们代码的边缘。但这是一件好事。它明确了代码的哪些部分是不纯粹的。当试图在复杂的代码库中调试问题时，这可能是一个真正的优势。



[^1]: 这不是一个完整的定义，但目前可用。稍后我们将回到正式定义。
[^2]: 在其他语言（比如 Haskell）中这称为 IO 函子或 IO 单体。[PureScript](http://www.purescript.org/) 使用了这项效果。我发现它更具描述性。
[^3]: 熟悉类型签名的人请注意。如果我们是严格的，我们需要考虑副作用。我们稍后会讲到。
[^4]: 请注意，不同的语言对此快捷方式有不同的名称。例如，在 Haskell 中，它被称为 `pure`。我不知道为什么。
[^5]: 在这种情况下，标准是 [Fantasy Land specification for Chain](https://github.com/fantasyland/fantasy-land#chain)。
[^6]: 如果你以前没见过柯里化，看看《[A Gentle Introduction to Functional Programming in JavaScript](https://jrsinclair.com/articles/2016/gentle-introduction-to-functional-javascript-functions/#currying)》的第 4 部分。
[^7]: 约翰·休斯，1990，《为什么函数式编程很重要》，函数式编程的研究课题编辑。杜纳，爱迪生-韦斯利，第17-42 页，<https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf>

[^8]: TensorFlow<sup>TM</sup>：面向所有人的开源机器学习框架，https://www.tensorflow.org/，2018 年 5 月 12 日。

---

[^T1]: 【译者注】引用透明（referential transparency），指的是函数的运行不依赖于外部变量或"状态"，只依赖于输入的参数，任何时候只要参数相同，引用函数所得到的返回值总是相同的。
[^T2]: 【译者注】原文插入了一个视频……