---
layout: post
title: 单元测试的命名标准
category: 编程理论
tags: 测试
excerpt: 介绍单元测试中的命名问题。
author: Eric Zong
date: 2020-03-02 23:55
---

> 英文原谅：[Naming standards for unit tests](https://osherove.com/blog/2005/4/3/naming-standards-for-unit-tests.html)
>
> 原文作者：[Roy Osherove](https://osherove.com/)
>
> 原文日期：2005-04-03

*测试的基本命名包括三个主要部分：*

[工作单元\_测试状态\_期望行为]

工作单元是系统中的一个用例，该系统以一个公共方法开始，并以以下三种类型的结果之一结束：返回值/异常，改变系统行为的状态变更，或调用第三方组件（当我们使用 mock 时）。所以一个工作单元可以是一个小的方法，也可以是一个大的类，甚至是多个类。只要它在内存中运行，并且完全在我们的控制之下。

<u>示例：</u>

Public void Sum_NegativeNumberAs1stParam_ExceptionThrown()

Public void Sum_NegativeNumberAs2ndParam_ExceptionThrown ()

Public void Sum_simpleValues_Calculated ()

***原因：***

<u>***测试名称应表达特定的要求***</u>

你的单元测试名称应该表达一个特定的需求。这个需求应该以某种方式从业务需求或技术需求中派生出来。无论如何，这个需求已经被分解成足够小的部分，每个部分代表一个测试用例。如果你的测试不代表需求，为什么要编写它？为什么代码还在那里？

开发人员乐于给测试取无意义或编号的名称，以便他们能够继续工作。这导致了一个斜坡，没人有时间或精力去修正测试的名称。而且，大多数开发人员不会在他们的测试中编写好的断言消息。一个好的断言消息（如果测试失败就会显示）可以帮助你理解哪里出错了。假设编写良好的断言消息是大多数单元测试开发人员所缺乏的关键地方之一，那么剩下的唯一能拯救我们的就是测试的名称。如果测试的名称和其中的错误消息都没有足够的表达力，那么可怜的维护开发人员（通常开发测试的是同一人）就会想知道这个蠢货是怎么想的？应基于单元测试来查看一年前的代码。

***测试名称应该包括期望的输入或状态以及该输入或状态下相应的期望结果***

为了清楚地表达特定的需求，你应该考虑需要表达的两件事情：测试期间的输入值/当前状态，以及该状态下的预期结果。

示例：

给定以下方法：

**Public int Sum(params int[] values)**

你有一个需求，即不应考虑传入大于 1000 的数字进行计算（即大于 1000 的数字等于零）。

**这个名称怎么样：Sum_NumberBiggerThan1000**

这仅仅达到了一半的目的。作为测试的阅读者，我只能假设你的测试中的某些输入大于1000，但是我不理解被测试的需求是什么。

**这个名称怎么样：Sum_NumberIsIgnored**

仍不够好。虽然它确实告诉了我预期的结果，但它没有告诉我在什么情况下需要忽略它。我将不得不进入测试代码并通读以找出期望的输入是什么。

**这个名称怎么样：Sum_NumberIgnoredIfBiggerThan1000**

这就好多了。我可以通过查看测试名称来知道测试失败的含义。我不需要看代码，并且我可以确切地说出该测试试图解决的需求。

**测试名称应作为陈述或生活事实给出，以表述工作流程和输出**

单元测试关乎测试功能，同时也关乎文档。你应该总是假设其他人将在一年内阅读你的测试，而他们应该不难理解你的意图。

用简短的声明性语句编写的测试名称比机器可读的名称更具表达力，因为只有在你理解隐藏代码语言以解析其含义的情况下，它们才有意义。[^1]

人们可能首先会抱怨测试方法的名字对他们来说太长了，但最终它的可读性更强。更长的测试名称并不意味着更长的或更少优化的代码，即使它们这样做了，我们也不希望优化测试代码的性能。如果要进行优化的话，测试代码应该总是为了可读性而优化。

下面有一个不好不坏的测试名称的示例

Public void SumNegativeNumber2()

而下面有一种更好的方法来表达你的意图

Public void Sum_ExceptionThrownOnNegativeNumberAs2ndParam ()

**<u>如果测试框架需要测试名称，或者测试名称在某种程度上简化了单元测试的开发和维护，那么测试名称应该以 Test 开头。</u>**

由于当今大多数测试位于一个特定的类或模块中，它们在逻辑上定义为测试容器（测试夹具[^2]），因此仅有少数原因使你想要在测试方法前加上单词 Test：

- 单元测试框架要求如此
- 您已经制定了编码标准，其中包括该项
- 你想要在使用智能感知或其他某种搜索机制时使得导航到方法更为容易。

**<u>测试名称应包括测试方法或类的名称</u>**

通常，对于每个要测试的方法至少都有一系列的测试，因此，通常在一个测试夹具中测试一系列的方法（因此通常每个测试的类都一一个测试夹具）。这导致要求你的测试还应该反映开始测试的方法的名称，而不仅仅是它的需求。

示例：

**[MethodName_StateUnderTest_ExpectedBehavior]**

Public void Sum_NegativeNumberAs1stParam_ExceptionThrown()

Public void Sum_NegativeNumberAs2ndParam_ExceptionThrown ()

Public void Sum_simpleValues_Calculated ()

Public void Parse_OnEmptyString_ExceptionThrown()

Public void Parse_SingleToken_ReturnsEqualToeknValue ()

**<u>测试中的命名变量</u>**

变量名称应表示预期的输入和状态

将变量命名为具有通用名称的方法很容易，但是对于单元测试，你必须加倍小心地将它们命名为它们所表示的东西，例如 BAD_DATA 或 EMPTY_ARRAY 或 NON_INITIALIZED_PERSON

你无需使用大写的 CONST 为它们命名，但是该名称必须代表测试作者的意图。

检查名称是否足够好的一个好方法是查看测试方法末尾的 ASSERT。如果 ASSERT 行表示您你的需求是什么，或者很接近，那么你可能已经很好了。

比如：

Assert.AreEqual(-1, val)

Vs.

Assert.AreEqual(BAD_INITIALIZATION_CODE, ReturnCode, Method shold have returned a bad initialization code )



[^1]: 译注：原文是一个超长的句子——Test names which are written as short declarative statements are much more expressive that names which are machine readable that is they only make sense if you have an understanding of a hidden code language to dissect their meaning. ——感觉翻译不太流畅。 
[^2]: 译注：测试夹具，fixture，主要目的是建立一个固定/已知的环境状态以确保测试可重复并且按照预期方式运行。

