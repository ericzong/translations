---
layout: post
title: 警惕接受多个相同类型的参数的函数
category: 编程理论
tags: 代码规范
excerpt: 介绍接受多个相同类型的参数的函数可能的缺陷以及规避方案。
author: Eric Zong
date: 2020-01-26 23:00
---

* content
{:toc}

> 英文原文：[Be wary of functions which take several parameters of the same type](https://dave.cheney.net/2019/09/24/be-wary-of-functions-which-take-several-parameters-of-the-same-type)
>
> 原文作者：[Dave Cheney](https://dave.cheney.net/)
>
> 原文日期：2019-09-24

---

> API 应易于使用且难以误用。
>
> ——Josh Bloch

来看一个看似简单但难以正确使用的例子，它是一个接受两个或以上相同类型的参数的 API。让我们比较两个函数签名：

```go
func Max(a, b int) int
func CopyFile(to, frome string) error
```

这两个函数之间的区别是什么？显然，一个函数返回两个数字中最大的那个，另一个函数复制一个文件，但这不是重要的事情。

```go
Max(8, 10) // 10
Max(10, 8) // 10
```

`Max` 是*可交换的*；其参数的顺序并不重要。无论我比较八和十或十和八，八和十中的最大值都是十。

但是，此特性不适用于 `CopyFile`。

```go
CopyFile("/tmp/backup", "presentation.md")
CopyFile("presentation.md", "/tmp/backup")
```

这些语句中哪一个备份了你的演讲稿，哪一个用上周的版本覆盖了你的演讲稿？如果不查阅文档，你不知道。代码审查者不查阅文档也无法知道你使用的参数顺序是否正确。

一般的建议是尽量避免这种情况。就像长参数列表一样，模糊的参数列表是一种设计臭味。

# 挑战

当这种情况不可避免时，我解决这类问题的方法是引入一个帮助器类型，该类型将负责正确调用 `CopyFile`。

打印源码

```go
func (src Source) CopyTo(dest string) error {
	return CopyFile(dest, string(src))
}

func main() {
	var from Source = "presentation.md"
	from.CopyTo("/tmp/backup")
}
```

这样，`CopyFile` 始终能被正确调用，并且，鉴于其较差的 API 可能会被设为私有，从而进一步降低误用的可能性。

你能提出一个更好的解决方案吗？