**为什么我使用8个空格缩进我的代码**

​	Roger Peng

​	2018/07/27

> 英文原文：[Why I Indent My Code 8 Spaces](https://simplystatistics.org/2018/07/27/why-i-indent-my-code-8-spaces/)

詹妮·布赖恩（Jenny Bryan）最近在布里斯班（Brisbane）的“Use R! 2018 会议”上做了一次精彩的演讲，主题是“代码的异味和感觉”（我建议你看一段演讲的[视频](https://youtu.be/7oyiPBjLAWY)）。她的演讲涵盖了几种检测代码“异味”的方法，以及如何通过重构修复这些异味。虽然其他编程语言有很多关于这方面的文献，但是对于 R 语言并没有很好的涵盖。

在视频版本的演讲中（而不是在幻灯片中），Jenny 提出了我的特殊缩进规则，即使用8个空格。在我的经验中，人们倾向于认为这是一种相当极端的缩进策略，可能 4 个空格就是他们能想象的上限了。但是到目前为止我已经使用 8 个空格很长时间了，并且我发现这有很多好处。

首先，我没有编造 8 空格缩进的风格。我是从 Linux 内核编码风格的文档中学到的。第一章写道：

> 制表符是 8 个字符，而这种缩进也是 8 个字符。有的异端运动试图使用 4 个（甚至 2 个！）缩进字符，这就好比试图把 PI 的值定义为 3 一样。

我发现 Linux 内核编码风格对于我的 R 语言编程十分有用，但是很多是针对 C 语言的或者不相关的。尽管如此，它也是值得一读的。

就我个人而言，我发现 8 个空格对我的老眼是有好处的。我认为我的原则是，适当的缩进空格数与我的年龄平方成正比（不过，我仍然工作在该模式下）。就这点而言，2 空格缩进的代码是很难从左对齐的文本中区别出来的。

再继续前，我得强调 8 空格缩进不能单独存在。它必须伴以右边 80 列的限制。否则，你就可以缩进到无限远而没个头了。80 列的限制迫使你将代码保持在合理范围内。此外，如果有人要在 PDP/11 上阅读你的代码也是没问题的。

但是，最重要的是，我发现 8 空格缩进可以作为一种“异味”代码的预警系统。Jenny 在她的演讲中给出了一些异味代码示例，我想我应该在这里再现它们。正如 Jenny 所描述，第一个示例由于没有使用可用的类谓词函数来测试“数值”或“整数”而受到诟病。下面是这个示例以 2 个空格缩进展示的样子。

```
bizarro <- function(x) {
  if (class(x)[[1]] == "numeric" || class(x)[[1]] == "integer") {
    -x
  } else if (class(x)[[1]] == "logical") {
    !x
  } else { 
    stop(...) 
  }
}
```

首先，if 状态从句有一点突出，因为它太长了。更好的代码也许是使用 is.numeric() 和 is.integer() 函数。

下面是使用 8 空格缩进的相同示例。

```
bizarro <- function(x) {
        if (class(x)[[1]] == "numeric" || class(x)[[1]] == "integer") {
                -x
        } else if (class(x)[[1]] == "logical") {
                !x
        } else { 
                stop(...) 
        }
}
```

第一行在右端逼近了 80 列的限制，然而，这还不算太糟。在这种情况下，你可以什么都不做，不过，缩进系统的目的是至少触发一个反应。

Jenny 演讲的下一个示例就更明显了。这里，她给出了一个使用过多 if-else 语句的教训。下面的源码使用 2 空格缩进。

```
get_some_data <- function(config, outfile) {
  if (config_ok(config)) {
    if (can_write(outfile)) {
      if (can_open_network_connection(config)) {
        data <- parse_something_from_network()
        if(makes_sense(data)) {
          data <- beautify(data)
          write_it(data, outfile)
          return(TRUE)
        } else {
          return(FALSE)
        }
      } else {
        stop("Can't access network")
      }
    } else {
      ## uhm. What was this else for again?
    }
  } else {
    ## maybe, some bad news about ... the config?
  } 
}
```

现在，公平地说，这段代码已经有点异味了（怀疑码？），不过在视觉上也许是可接受的。让我们看看使用 8 空格缩进的样子。

```
get_some_data <- function(config, outfile) {
        if (config_ok(config)) {
                if (can_write(outfile)) {
                        if (can_open_network_connection(config)) {
                                data <- parse_something_from_network()
                                if(makes_sense(data)) {
                                        data <- beautify(data)
                                        write_it(data, outfile)
                                        return(TRUE)
                                } else {
                                        return(FALSE)
                                }
                        } else {
                                stop("Can't access network")
                        }
                } else {
                        ## uhm. What was this else for again?
                }
        } else {
                ## maybe, some bad news about ... the config?
        } 
}
```

现在代码看起来相当怪异，实际上它迫切需要重构，重构就对了！只要你眨眼，5 级的嵌套是不易阅读的。

基本上就是这样。我发现使用 8 空格缩进没有什么缺点，并且还有许多好处，包括更干净、更加模块化的代码。因为视觉指示器会对大量缩进产生不利影响，所以通常会被迫编写单独的函数来处理不同的任务，而不是再缩进一个层级。这不仅具有模块化的优点，而且对于解析之类的事情是很有用的（分析单个庞大函数可能并不能提供足够信息）。