前言
************

本书的目的是快速及全面向你讲述 Common Lisp 的相关知识。它实际上包含两本书。前半部分用大量的例子来解释 Common Lisp 里面重要的概念。后半部分是对于 Common Lisp 的一份紧跟时代的总结，涵盖了所有 ANSI Common Lisp 的操作符。

这本书面向的读者
====================

ANSI Common Lisp 这本书适合学生或专业程序员阅读。本书假设读者阅读前没有 Lisp 的相关知识。有别的程序语言的编程经验也许会对阅读本书有帮助，但也不是必须的。本书从解释 Lisp 中最基本的概念开始，并且着重解释了Lisp初学者容易困惑的地方。

本书也可以单独作为Lisp程序设计课程的课本，也可以用在人工智能课程或程序语言设计课程中讲授Lisp的部分。想要学习 Lisp 的专业程序员肯定会很喜欢本书所采用的直截了当、注重实践的方法。那些已经在使用 Lisp 编程的人士将会在本书中学到许多有用的实例，此外，本书也是一本方便的 ANSI Common Lisp 参考书。

如何使用这本书
====================

学习 Lisp 最好的办法就是拿它来编程。而且用Lisp编程，比单纯地去学这门语言更为有趣。本书在编写上充分考虑了如何让读者尽快的入门。本书首先对 Lisp 进行简短的介绍，在此之后
第 2 章用 21 页的篇幅，介绍了着手编写 Lisp 程序时所有可能会用到的知识。
3-9 章讲解了 Lisp 里面一些重要的知识点。这些章节特别强调了一些重要的概念，比如指针在 Lisp 中扮演的角色，如何使用递归来解决问题，以及第一级函数的重要性等等。

针对那些想要有更扎实的 Lisp 技术得读者：
10-14 章包含了宏、CLOS、列表操作、程序优化，以及一些更高级的课题，比如包和读取宏。

15-17 章通过 3 个 Common Lisp 的实际应用，总结了之前章节所讲解的知识：一个是进行逻辑推理的程序，另一个是 HTML 生成器，最后一个是针对面向对象编程的嵌入式语言。

本书的最后一部分包含了 4 个附录，这些附录对所有的读者都有用：
附录 A-D 包括了一个调试程序的教程， 58 个 Common Lisp 操作符的源代码，一个关于 ANSI Common Lisp 和以前的 Lisp 语言区别的总结，以及一个包括所有 ANSI Common Lisp 操作符的参考手册。

本书的最后是一节备注。这些备注包括一些说明，一些参考条目，一些额外的代码，偶然也会有一些异端邪说。备注在文中用一个小圆圈来表示，像这样：○

.. note::

	译注: 由于小圈圈 ○ 实在太不明显了，译文中使用 λ 符号来表示备注。

`λ <http://ansi-common-lisp.readthedocs.org/en/latest/zhCN/notes-cn.html#viii-notes-viii>`_

代码
==========

虽然本书介绍的是 ANSI Common Lisp ，但是本书中的代码可以在任何版本的 Common Lisp 中运行。那些依赖 Lisp 语言新特性的例子的旁边，会有注释告诉你如何把它们运行于旧版本的 Lisp 中。

本书中所有的代码都可以在互联网上下载到。你可以在网络上找到这些代码，一些自由软件的链接，一些过去的论文，以及 Lisp FAQ 。还有很多有关 Lisp 的资源可以在此找到：
http://www.eecs.harvard.edu/onlisp/
源代码可以在此 FTP 服务器上下载：
ftp://ftp.eecs.harvard.edu:/pub/onlisp/
读者的问题和意见可以发送到 pg@eecs.harvard.edu 。

.. tip::

	译注：下载的链接都坏掉了，本书的代码可以到此下载：https://raw.github.com/acl-translation/acl-chinese/master/code/acl2.lisp

On Lisp
=============

在整本书中，我着力于指出一些 Lisp 独一无二的特性，这些特性使得 Lisp 更像 “Lisp” 。并展示一些不用 Lisp 就做不到的事情。比如说宏： Lisp 程序员可以并写出一些能够写程序的程序，而且他们也经常这么做。在所有的主流语言中，只有 Lisp 经常使用这种技术，这是因为主流的程序设计语言中，只有 Lisp 提供了能方便使用这种技术的抽象手段。我想请那些想要更进一步了解宏和其他高级 Lisp 技术的读者，读一下本书的姐妹篇： `On Lisp <http://www.paulgraham.com/onlisp.html>`_ 。

.. tip::

	On Lisp 已经由知名 Lisp 黑客 ── 田春 ── 翻译完成，可以在网络上找到。
	── 田春（知名 Lisp 黑客、Practical Common Lisp 译者）

鸣谢
==========

在所有帮助我完成这本的朋友当中，我想特别地感谢一下 Robert Morris 。他对于本书的影响，渗入了书的字句之间，因为他的贡献，本书也变得更加出色。书中有好几个例子都源于他之前所写的程序。这些程序包括 138 页的 Henley 和 249 页的模式匹配器。

我很高兴能有一个高水平的技术审稿小组：Skona Brittain, John Foderaro, Nick Levine, Peter Norvig 和 Dave Touretzky。本书中几乎所有部分都得益于它们的意见。 John Foderaro 甚至重写了本书 5.7 节中一些代码。

另外一些人通篇阅读了本书的手稿，他们是：Ken Anderson, Tom Cheatham, Richard Fateman, Steve Hain, Barry Margolin, Waldo Pacheco, Wheeler Ruml 和 Stuart Russell。特别要提一下，Ken Anderson 和 Wheeler Ruml 给了很多有用的意见。

我非常感谢 Cheatham 教授，更广泛地说，哈佛，提供我编写这本书的一些必要条件。另外也要感谢 Aiken 实验室的人员：Tony Hartman, Dave Mazieres, Janusz Juda, Harry Bochner 和 Joanne Klys。

我非常高兴能再一次有机会和 Alan Apt 合作。Prentice Hall 的员工： Alan, Mona, Pompili Shirley McGuire 和 Shirley Michaels, 能与你们共事我很高兴。

本书用 Leslie Lamport 写的 LaTeX 进行排版。LaTeX 是在 Donald Knuth 编写的 TeX 的基础上，添加了 L.A.Carr, Van Jacobson 和 Guy Steele 所编写的宏完成。书中的图表是由 John Vlissides 和 Scott Stanton 编写的 Idraw 处理而成的。本书使用 Tim Theisen 写的 Ghostview 进行打印预览。 Ghostview 是根据 L. Peter Deutsch 的 Ghostscript 创建的。

我还需要感谢其他的许多人，包括：Henry Baker, Kim Barrett, Ingrid Bassett, Trevor Blackwell, Paul Becker, Gary Bisbee, Frank Deutschmann, Frances Dickey, Rich 和 Scott Draves, Bill Dubuque, Dan Friedman, Jenny Graham, Alice Hartley, David Hendler, Mike Hewett, Glenn Holloway, Brad Karp, Sonya Keene, Ross Knights, Mutsumi Komuro, Steffi Kutzia, David Kuznick, Madi Lord, Julie Mallozzi, Paul McNamee, Dave Moon, Howard Mullings, Mark Nitzberg, Nancy Parmet 和其家人, Robert Penny, Mike Plusch, Cheryl Sacks, Hazem Sayed, Shannon Spires, Lou Steinberg, Paul Stoddard, John Stone, Guy Steele, Steve Strassmann, Jim Veitch, Dave Watkins, Idelle and Julian Weber, the Weickers, Dave Yost 和 Alan Yuille。

尤其要感谢我的父母和 Jackie。

`高德纳 <http://zh.wikipedia.org/zh-cn/%E9%AB%98%E5%BE%B7%E7%BA%B3>`_\ 给他的经典丛书起名为《计算机程序设计艺术》。在他的图灵奖获奖感言中，他解释说，这个书名源自于内心深处的潜意识 ── 潜意识告诉他，编程其实就是追求编写最优美的程序。

就像建筑设计一样，编程既是一门工程技艺也是一门艺术。一个程序要遵循数学原理，就如建筑物必须符合符合物理定律。但是建筑师的目标不仅仅是让建筑不会倒塌。他们真正的目标，几乎总是建造出美丽的建筑。

像高德纳一样，很多程序员认为编程的真正目的，不仅仅是编写出正确的程序，更重要的是写出优美的代码。几乎所有的 Lisp 高手都是这么想的。 Lisp 黑客精神可以用两句话来概括：编程应该是有趣的。程序应该是优美的。这正是我在这本书中想要传达的精神。

`保罗•格雷厄姆 (Paul Graham) <http://paulgraham.com/>`_
