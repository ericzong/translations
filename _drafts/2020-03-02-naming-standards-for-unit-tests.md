---
layout: post
title: 单元测试的命名标准
category: 编程理论
tags: 测试
excerpt: 介绍单元测试中的命名问题。
author: Eric Zong
date: 2020-03-02 23:55
---

* content
{:toc}

> 英文原谅：[Naming standards for unit tests](https://osherove.com/blog/2005/4/3/naming-standards-for-unit-tests.html)
>
> 原文作者：Roy Osherove
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