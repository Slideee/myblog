---
title: GitHub Copilot和ChatGPT使用指南
date: 2023/03/05
updated: 2023/03/05
tag: Tools
author: Slide
categories: Tools
---
本人旨在介绍一些在使用GitHub Copilot和ChatGPT这两个工具辅助编程时的一些经验和使用场景，具体的安装过程不再赘述，请参考官方文档，希望能够帮助到大家。

<!--more-->

------

GitHub Copilot为许多语言和各种框架提供了建议，但特别适用于Python, JavaScript, TypeScript, Ruby, Go, c#和c++。下面的示例使用Go，但其他语言也可以类似地工作。

1. 在Goland IDE中，创建一个新的Go (*. go)文件。
2. 在Go文件中，实现一个数组remove指定元素的方法。GitHub Copilot将自动建议一个灰色文本的类主体，如下所示。确切的建议可能有所不同。
3. 接受建议，按Tab键。

![raft1](/images/posts/openai/remove.gif)

GitHub Copilot将尝试匹配你的代码的上下文和风格，例如以下几个例子。

#### 1. 重复代码自动生成

------

根据上下文的语义自动生成重复度高的代码，只要一直tab大量重复代码快速生成，在一些复杂的上下文语义下，几乎也可以实现。

![status](/images/posts/openai/status.gif)



#### 2.根据注释生成代码

------

根据注释生成代码，例如一些工具类。

![merges](/images/posts/openai/merges.gif)



#### 3. 为方法、字段添加注释

------

为方法字段添加注释，输入一些简单的提示词就可以生成你想要的注释。

![field](/images/posts/openai/field.gif)

#### 4.代码翻译

------

使用chatGPT进行代码翻译，常规工具类方法直接转语言实现。

![trans](/images/posts/openai/trans.gif)

------

Reference：

[Getting started with GitHub Copilot in a JetBrains IDE - GitHub Docs](https://docs.github.com/en/copilot/getting-started-with-github-copilot/getting-started-with-github-copilot-in-a-jetbrains-ide)

