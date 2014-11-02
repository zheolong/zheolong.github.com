---
layout: post
title: "shell programming: debugging"
description: ""
category: Shell
tags: []
---
{% include JB/setup %}

<!-- 目录 -->
* toc
{:toc}

<!-- 正文 -->
引用自：[Shell脚本调试技术](http://www.ibm.com/developerworks/cn/linux/l-cn-shell-debug/)

# 总结 #

首先使用“-n”选项检查语法错误，然后使用“-x”选项跟踪脚本的执行，使用
“-x”选项之前，别忘了先定制PS4变量的值来增强“-x”选项的输出信息，至少
应该令其输出行号信息(先执行export PS4='+[$LINENO]'，更一劳永逸的办法是
将这条语句加到您用户主目录的.bash_profile文件中去)，这将使你的调试之旅
更轻松。也可以利用trap,调试钩子等手段输出关键调试信息，快速缩小排查错误
的范围，并在脚本中使用“set -x”及“set +x”对某些代码块进行重点跟踪。
这样多种手段齐下，相信您已经可以比较轻松地抓出您的shell脚本中的臭虫了。
如果您的脚本足够复杂，还需要更强的调试能力，可以使用shell调试器bashdb，
这是一个类似于GDB的调试工具，可以完成对shell脚本的断点设置，单步执行，
变量观察等许多功能，使用bashdb对阅读和理解复杂的shell脚本也会大有裨益。
关于bashdb的安装和使用，不属于本文范围，您可参阅
http://bashdb.sourceforge.net/上的文档并下载试用。

请访问： [GNU的bash主页](http://www.gnu.org/software/bash/bash.html)
请下载和试用 [Shell调试器bashdb](http://bashdb.sourceforge.net/)

# 修改Shell源代码 #

## echo或print输出 ##
在脚本中使用echo输出变量的值，zsh和ksh可以使用print

## trap ##

**作用** 捕获指定的信号并执行预定义的命令
**语法** `trap 'command' signal`
**解释**
其中signal是要捕获的信号，command是捕获到指定的信号之后所要执行的命令。
可以用kill –l命令看到系统中全部可用的信号名，捕获信号后所执行的命令可以
是任何一条或多条合法的shell语句，也可以是一个函数名。shell脚本在执行时，
会产生三个所谓的“伪信号”，(之所以称之为“伪信号”是因为这三个信号是由
shell产生的，而其它的信号是由操作系统产生的)，通过使用trap命令捕获这三
个“伪信号”并输出相关信息对调试非常有帮助。

<table>
   <tr>
      <td>信号名</td>
      <td>何时产生</td>
   </tr>
   <tr>
      <td>EXIT</td>
      <td>从一个函数中退出或整个脚本执行完毕</td>
   </tr>
   <tr>
      <td>ERR</td>
      <td>当一条命令返回非零状态时(代表命令执行不成功)</td>
   </tr>
   <tr>
      <td>DEBUG</td>
      <td>脚本中每一条命令执行之前</td>
   </tr>
</table>

通过捕获EXIT信号,我们可以在shell脚本中止执行或从函数中退出时，输出某些
想要跟踪的变量的值，并由此来判断脚本的执行状态以及出错原因,其使用方法是：
trap 'command' EXIT　或　trap 'command' 0

通过捕获ERR信号,我们可以方便的追踪执行不成功的命令或函数，并输出相关的
调试信息，以下是一个捕获ERR信号的示例程序，其中的$LINENO是一个shell的内
置变量，代表shell脚本的当前行号。

{% highlight bash %}
$ cat -n exp1.sh
     1  ERRTRAP()
     2  {
     3    echo "[LINE:$1] Error: Command or function exited with status $?"
     4  }
     5  foo()
     6  {
     7    return 1;
     8  }
     9  trap 'ERRTRAP $LINENO' ERR
    10  abc
    11  foo
{% endhighlight %}

输出结果:
{% highlight c %}
$ sh exp1.sh
exp1.sh: line 10: abc: command not found
[LINE:10] Error: Command or function exited with status 127
[LINE:11] Error: Command or function exited with status 1
{% endhighlight %}

**通过捕获DEBUG信号，只需一条trap语句完成全程跟踪**

通过捕获DEBUG信号来跟踪变量的示例程序:

{% highlight bash %}
$ cat –n exp2.sh
     1  #!/bin/bash
     2  trap 'echo “before execute line:$LINENO, a=$a,b=$b,c=$c”' DEBUG
     3  a=1
     4  if [ "$a" -eq 1 ]
     5  then
     6     b=2
     7  else
     8     b=1
     9  fi
    10  c=3
    11  echo "end"
{% endhighlight %}

输出结果:
{% highlight bash %}
$ sh exp2.sh
before execute line:3, a=,b=,c=
before execute line:4, a=1,b=,c=
before execute line:6, a=1,b=,c=
before execute line:10, a=1,b=2,c=
before execute line:11, a=1,b=2,c=3
end
{% endhighlight %}

## tee ##

 在shell脚本中管道以及输入输出重定向使用得非常多，在管道的作用下，一些
 命令的执行结果直接成为了下一条命令的输入。如果我们发现由管道连接起来的
 一批命令的执行结果并非如预期的那样，就需要逐步检查各条命令的执行结果来
 判断问题出在哪儿，但因为使用了管道，这些中间结果并不会显示在屏幕上，给
 调试带来了困难，此时我们就可以借助于tee命令了。

tee命令会从标准输入读取数据，将其内容输出到标准输出设备,同时又可将内容
保存成文件。例如有如下的脚本片段，其作用是获取本机的ip地址：

{% highlight bash %}
ipaddr=`/sbin/ifconfig | grep 'inet addr:' | grep -v '127.0.0.1'
| cut -d : -f3 | awk '{print $1}'` 
#注意=号后面的整句是用反引号(数字1键的左边那个键)括起来的。
echo $ipaddr
{% endhighlight %}

运行这个脚本，实际输出的却不是本机的ip地址，而是广播地址,这时我们可以借
助tee命令，输出某些中间结果，将上述脚本片段修改为：


{% highlight c %}
ipaddr=`/sbin/ifconfig | grep 'inet addr:' | grep -v '127.0.0.1'
| tee temp.txt | cut -d : -f3 | awk '{print $1}'`
echo $ipaddr
{% endhighlight %}

之后，将这段脚本再执行一遍，然后查看temp.txt文件的内容：

{% highlight c %}
$ cat temp.txt
inet addr:192.168.0.1  Bcast:192.168.0.255  Mask:255.255.255.0
{% endhighlight %}

 我们可以发现中间结果的第二列(列之间以:号分隔)才包含了IP地址，而在上面
 的脚本中使用cut命令截取了第三列，故我们只需将脚本中的cut -d : -f3改为
 cut -d : -f2即可得到正确的结果。

具体到上述的script例子，我们也许并不需要tee命令的帮助，比如我们可以分段
执行由管道连接起来的各条命令并查看各命令的输出结果来诊断错误，但在一些
复杂的shell脚本中，这些由管道连接起来的命令可能又依赖于脚本中定义的一些
其它变量，这时我们想要在提示符下来分段运行各条命令就会非常麻烦了，简单
地在管道之间插入一条tee命令来查看中间结果会更方便一些。

## "调试钩子" ##

在C语言程序中，我们经常使用DEBUG宏来控制是否要输出调试信息，在shell脚本
中我们同样可以使用这样的机制，如下列代码所示：

{% highlight c %}
if [ “$DEBUG” = “true” ]; then
echo “debugging”  #此处可以输出调试信息
fi
{% endhighlight %}

 这样的代码块通常称之为“调试钩子”或“调试块”。在调试钩子内部可以输出
 任何您想输出的调试信息，使用调试钩子的好处是它是可以通过DEBUG变量来控
 制的，在脚本的开发调试阶段，可以先执行export DEBUG=true命令打开调试钩
 子，使其输出调试信息，而在把脚本交付使用时，也无需再费事把脚本中的调试
 语句一一删除。

如果在每一处需要输出调试信息的地方均使用if语句来判断DEBUG变量的值，还是
显得比较繁琐，通过定义一个DEBUG函数可以使植入调试钩子的过程更简洁方便，
如下面代码所示:

{% highlight c %}
$ cat –n exp3.sh
     1  DEBUG()
     2  {
     3  if [ "$DEBUG" = "true" ]; then
     4      $@　　
     5  fi
     6  }
     7  a=1
     8  DEBUG echo "a=$a"
     9  if [ "$a" -eq 1 ]
    10  then
    11       b=2
    12  else
    13       b=1
    14  fi
    15  DEBUG echo "b=$b"
    16  c=3
    17  DEBUG echo "c=$c"
{% endhighlight %}

在上面所示的DEBUG函数中，会执行任何传给它的命令，并且这个执行过程是可以
通过DEBUG变量的值来控制的，我们可以把所有跟调试有关的命令都作为DEBUG函
数的参数来调用，非常的方便。


# 使用shell的执行选项 #

 -n 只读取shell脚本，但不实际执行
-x 进入跟踪方式，显示所执行的每一条命令
-c "string" 从strings中读取命令

“-n”可用于测试shell脚本是否存在语法错误，但不会实际执行命令。在shell
脚本编写完成之后，实际执行之前，首先使用“-n”选项来测试脚本是否存在语
法错误是一个很好的习惯。因为某些shell脚本在执行时会对系统环境产生影响，
比如生成或移动文件等，如果在实际执行才发现语法错误，您不得不手工做一些
系统环境的恢复工作才能继续测试这个脚本。

 “-c”选项使shell解释器从一个字符串中而不是从一个文件中读取并执行
shell命令。当需要临时测试一小段脚本的执行结果时，可以使用这个选项，如下
所示：sh -c 'a=1;b=2;let c=$a+$b;echo "c=$c"'

"-x"选项可用来跟踪脚本的执行，是调试shell脚本的强有力工具。“-x”选项使
shell在执行脚本的过程中把它实际执行的每一个命令行显示出来，并且在行首显
示一个"+"号。 "+"号后面显示的是经过了变量替换之后的命令行的内容，有助于
分析实际执行的是什么命令。 “-x”选项使用起来简单方便，可以轻松对付大多
数的shell调试任务,应把其当作首选的调试手段。

如果把本文前面所述的trap ‘command’ DEBUG机制与“-x”选项结合起来，我
们 就可以既输出实际执行的每一条命令，又逐行跟踪相关变量的值，对调试相当
有帮助。

仍以前面所述的exp2.sh为例，现在加上“-x”选项来执行它：

{% highlight c %}
$ sh –x exp2.sh
+ trap 'echo "before execute line:$LINENO, a=$a,b=$b,c=$c"' DEBUG
++ echo 'before execute line:3, a=,b=,c='
before execute line:3, a=,b=,c=
+ a=1
++ echo 'before execute line:4, a=1,b=,c='
before execute line:4, a=1,b=,c=
+ '[' 1 -eq 1 ']'
++ echo 'before execute line:6, a=1,b=,c='
before execute line:6, a=1,b=,c=
+ b=2
++ echo 'before execute line:10, a=1,b=2,c='
before execute line:10, a=1,b=2,c=
+ c=3
++ echo 'before execute line:11, a=1,b=2,c=3'
before execute line:11, a=1,b=2,c=3
+ echo end
end
{% endhighlight %}

 在上面的结果中，前面有“+”号的行是shell脚本实际执行的命令，前面有
 “++”号的行是执行trap机制中指定的命令，其它的行则是输出信息。

shell的执行选项除了可以在启动shell时指定外，亦可在脚本中用set命令来指定。
"set -参数"表示启用某选项，"set +参数"表示关闭某选项。有时候我们并不需
要在启动时用"-x"选项来跟踪所有的命令行，这时我们可以在脚本中使用set命令，
如以下脚本片段所示：

{% highlight c %}
set -x　　　 #启动"-x"选项 
要跟踪的程序段 
set +x　　　　 #关闭"-x"选项
{% endhighlight %}

set命令同样可以使用上一节中介绍的调试钩子—DEBUG函数来调用，这样可以避免
脚本交付使用时删除这些调试语句的麻烦，如以下脚本片段所示：

{% highlight c %}
DEBUG set -x　　　 #启动"-x"选项 
要跟踪的程序段 
DEBUG set +x　　　 #关闭"-x"选项
{% endhighlight %}

# 对"-x"选项的增强 #

 "-x"执行选项是目前最常用的跟踪和调试shell脚本的手段，但其输出的调试信
 息仅限于进行变量替换之后的每一条实际执行的命令以及行首的一个"+"号提示
 符，居然连行号这样的重要信息都没有，对于复杂的shell脚本的调试来说，还
 是非常的不方便。幸运的是，我们可以巧妙地利用shell内置的一些环境变量来
 增强"-x"选项的输出信息，下面先介绍几个shell内置的环境变量：

$LINENO

代表shell脚本的当前行号，类似于C语言中的内置宏__LINE__

$FUNCNAME

函数的名字，类似于C语言中的内置宏__func__,但宏__func__只能代表当前所在
的函数名，而$FUNCNAME的功能更强大，它是一个数组变量，其中包含了整个调用
链上所有的函数的名字，故变量${FUNCNAME[0]}代表shell脚本当前正在执行的函
数的名字，而变量${FUNCNAME[1]}则代表调用函数${FUNCNAME[0]}的函数的名字，
余者可以依此类推。

$PS4

主提示符变量$PS1和第二级提示符变量$PS2比较常见，但很少有人注意到第四级
提示符变量$PS4的作用。我们知道使用“-x”执行选项将会显示shell脚本中每一
条实际执行过的命令，而$PS4的值将被显示在“-x”选项输出的每一条命令的前
面。在Bash Shell中，缺省的$PS4的值是"+"号。(现在知道为什么使用"-x"选项
时，输出的命令前面有一个"+"号了吧？)。

利用$PS4这一特性，通过使用一些内置变量来重定义$PS4的值，我们就可以增强
"-x"选项的输出信息。例如先执行export PS4='+{$LINENO:${FUNCNAME[0]}} ',
然后再使用“-x”选项来执行脚本，就能在每一条实际执行的命令前面显示其行
号以及所属的函数名。

以下是一个存在bug的shell脚本的示例，本文将用此脚本来示范如何用“-n”以
及增强的“-x”执行选项来调试shell脚本。这个脚本中定义了一个函数
isRoot(),用于判断当前用户是不是root用户，如果不是，则中止脚本的执行

{% highlight c %}
$ cat –n exp4.sh
     1  #!/bin/bash
     2  isRoot()
     3  {
     4          if [ "$UID" -ne 0 ]
     5                  return 1
     6          else
     7                  return 0
     8          fi
     9  }
    10  isRoot
    11  if ["$?" -ne 0 ]
    12  then
    13          echo "Must be root to run this script"
    14          exit 1
    15  else
    16          echo "welcome root user"
    17          #do something
    18  fi
{% endhighlight %}

首先执行sh –n exp4.sh来进行语法检查，输出如下：

{% highlight c %}
$ sh –n exp4.sh
exp4.sh: line 6: syntax error near unexpected token `else'
exp4.sh: line 6: `      else'
{% endhighlight %}

发现了一个语法错误，通过仔细检查第6行前后的命令，我们发现是第4行的if语
句缺少then关键字引起的(写惯了C程序的人很容易犯这个错误)。我们可以把第4
行修改为if [ "$UID" -ne 0 ]; then来修正这个错误。再次运行sh –n exp4.sh
来进行语法检查，没有再报告错误。接下来就可以实际执行这个脚本了，执行结
果如下：

{% highlight c %}
$ sh exp4.sh
exp2.sh: line 11: [1: command not found
welcome root user
{% endhighlight %}

尽管脚本没有语法错误了，在执行时却又报告了错误。错误信息还非常奇怪“[1: command not found”。现在我们可以试试定制$PS4的值，并使用“-x”选项来跟踪：

{% highlight c %}
$ export PS4='+{$LINENO:${FUNCNAME[0]}} '
$ sh –x exp4.sh
+{10:} isRoot
+{4:isRoot} '[' 503 -ne 0 ']'
+{5:isRoot} return 1
+{11:} '[1' -ne 0 ']'
exp4.sh: line 11: [1: command not found
+{16:} echo 'welcome root user'
welcome root user
{% endhighlight %}

从输出结果中，我们可以看到脚本实际被执行的语句，该语句的行号以及所属的函数名也被打印出来，从中可以清楚的分析出脚本的执行轨迹以及所调用的函数的内部执行情况。由于执行时是第11行报错，这是一个if语句，我们对比分析一下同为if语句的第4行的跟踪结果：

{% highlight c %}
+{4:isRoot} '[' 503 -ne 0 ']'
+{11:} '[1' -ne 0 ']'
{% endhighlight %}

 可知由于第11行的
 [号后面缺少了一个空格，导致[号与紧挨它的变量$?的值1被shell解释器看作了一个整体，并试着把这个整体视为一个命令来执行，故有“[1: command not found”这样的错误提示。只需在[号后面插入一个空格就一切正常了。

shell中还有其它一些对调试有帮助的内置变量，比如在Bash Shell中还有BASH_SOURCE, BASH_SUBSHELL等一批对调试有帮助的内置变量，您可以通过man sh或man bash来查看，然后根据您的调试目的,使用这些内置变量来定制$PS4，从而达到增强“-x”选项的输出信息的目的。

<!-- 代码块(注意修改语言) -->
{% highlight c %}

{% endhighlight %}
