---
layout: post
title: "latex lstlistings"
description: ""
category: LaTex 
tags: []
---
{% include JB/setup %}

<!-- 目录 -->
* toc
{:toc}

<!-- 正文 -->

## 参考资料 ##

[LaTeX/Source Code Listings](http://en.wikibooks.org/wiki/LaTeX/Source_Code_Listings)

## 防止lst代码分页 ##

使用`float`将其变成浮动环境可以防止代码被分页
{% highlight latex%}
 \begin{lstlisting}[float,caption=<Caption text>,label=<label>]
   content
 \end{lstlisting}
{% endhighlight %}

<!-- 代码块(注意修改语言) -->
{% highlight c %}
c code
{% endhighlight %}