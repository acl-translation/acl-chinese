.. highlight:: cl
   :linenothreshold: 0

Chapter 12 结构 (Structure)
**************************************************

3.3 节中介绍了 Lisp 如何使用指针允许我们将任何值放到任何地方。这种说法是完全有可能的，但这并不一定都是好事。

例如，一个对象可以是自身的元素。这样是好还是坏，要看它是程序员有意设计的还是无心这样写的。

12.1 共享结构 (Shared Structure)
==================================

多个列表可以共享 conse 。在最简单的情况下，一个列表可能是另一个列表的一部分。如

::

	> (setf part (list 'b 'c))
	(B C)
	> (setf whole (cons 'a part))
	(A B C) 

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.1.png

图 12.1 共享结构

其中，第一个 ``cons`` 是第二个 ``cons`` 的一部分 (事实上，是的第二个 ``cons`` 的 ``cdr`` )。在这样的情况下，我们说，这两个列表共享结构 (Shared Structure)。这两个列表的基本结构见图 12.1 所示。

用 ``tailp`` 函数来检测这种情况。将两个列表作为它的输入参数，如果第一个列表是第二个列表的一部分时，则返回T ：

::

	> (tailp part whole)
	T

我们可以把它想像成：

::

	(defun our-tailp (x y)
	   (or (eql x y)
	       (and (consp y)
		    (our-tailp x (cdr y)))))

定义表明，每个列表是它自己一个尾巴 (tail)， ``nil`` 是每一个正规列表的尾巴。

在更复杂的情况下，两个列表可以共享一个可以使彼此都不是对方的尾巴的结构。在这种情况下时，他们都有一个共同的尾巴，如图 12.2 所示。我们可以构建这种情况如下：

::

	(setf part (list 'b 'c)
	      whole1 (cons 1 part)
	      whole2 (cons 2 part))

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.2.png

图 12.2 被共享的尾巴

现在 ``whole1`` 和 ``whole2`` 共享结构，但是它们彼此都不是对方的一部分。 

当存在嵌套列表时，重要的是要区分是列表共享结构 (lists sharing structure)，还是列表的元素共享结构 (elements sharing structure)。顶层列表结构 (Top-level list structure) 是指，直接构成列表的 conses ，不包含组成它的元素的 conses 。图 12.3 是一个嵌套列表的顶层列表结构。

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.3.png

图 12.3 顶层列表结构

两个 conses 是否共享结构取决于我们把它们看作是列表还是 `树 (tree) <http://zh.wikipedia.org/wiki/%E6%A0%91_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)>`_ 。可能存在两个嵌套列表，把它们看作树时，它们共享结构，而看作列表时，它们不共享结构。图 12.4 构建了这种情况，两个列表以一个元素的形式包含了同一个列表，代码如下：

::

	(setf element (list 'a 'b)
	      holds1 (list 1 element 2)
	      holds2 (list element 3))

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.4.png

图 12.4 共享子树

虽然 ``holds1`` 的第二个元素和 ``holds2`` 的第一个元素共享结构 (其实是相同的)，但是把它们看成列表时，``holds1`` 和 ``holds2`` 不共享结构。仅当两个列表共享顶层列表结构时，才能被看作是作为列表共享结构，而 ``holds1`` 和 ``holds2`` 没有共享顶层列表结构。

如果我们想避免共享结构，可以使用复制。函数 ``copy-list`` 可以这样定义：

::

    (defun our-copy-list (lst)
       (if (null lst)
           nil
           (cons (car lst) (our-copy-list (cdr lst)))))

它返回一个不与原始列表共享顶层列表结构的新列表。函数 ``copy-tree`` 可以这样定义：

::

    (defun our-copy-tree (tr)
       (if (atom tr)
            tr
            (cons (our-copy-tree (car tr))
                  (our-copy-tree (cdr tr)))))

它返回一个连原始列表的树型结构也不共享的新列表。图 12.5 显示了对一个嵌套列表使用 ``copy-list`` 和 ``copy-tree`` 的区别。

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.5.png

图 12.5 两种复制

12.2 修改 (Modification) 
==================================================

为什么要避免共享结构呢？之前讨论的共享结构问题仅仅是个智力练习，到目前为止，并没使我们在写程序的时候有什么不同。当修改一个被共享的结构是，问题出现了。如果两个列表共享结构，当我们修改了其中一个，另外一个也会无意中被修改。

上一节中，我们介绍了怎样构建一个是其它列表的尾巴的列表：

::

	(setf whole (list 'a 'b 'c)
	      tail (cdr whole))

因为这使得 ``tail`` 与 ``whole`` 的 ``cdr`` 相同，无论是修改了 ``tail`` 还是 ``whole`` 的 ``cdr`` ，我们都是修改的同一个 ``cons`` ：

::

	> (setf (second tail ) 'e)
	E
	> tail
	(B E)
	> whole
	(A B E)

当然，如果两个列表共享同一个尾巴，这样的事同样会发生。

一次修改两个对象并不总是错误的。有时候这可能正是你想要的。但如果无意的修改了共享结构，将会引入一些非常细微 bug。Lisp 程序员要培养对共享结构的意识，并且在这类错误发生时能够立刻反应过来。当一个列表神秘的改变了的时候，很有可能是因为改变了其它与之共享结构的对象。

真正危险的不是共享结构，而是改变被共享的结构。为了安全起见，干脆避免对结构使用 ``setf`` (以及相关的运算，比如：``pop``，``rplaca`` 等)，这样就不会遇到问题了。某些时候不得不修改列表结构时，要搞清楚要修改的列表的来源，确保它不要和其它不需要改变的对象共享结构。如果它和其它不需要改变的对象共享了结构，或者不能预测它的来源，那么复制一个副本来进行改变。

当你调用别人写的函数的时候要加倍小心。除非你知道它内部的操作，否则，你传入的参数时要注意以下的情况：

1.它对你传入的参数可能会有破坏性的操作

2.你传入的参数可能被保存起来，如果你调用了一个函数，然后又修改了之前作为参数传入该函数的对象，那么你也就改变了函数已保存起来作为它用的对象[1]。

在这两种情况下，解决的方法是传入一个副本。

在 Common Lisp 中，被调用的函数在遍历列表结构 (比如，``mapcar`` 或 ``remove-if`` 的参数)的过程中是不允许修改被遍历的结构的。评估这样的代码的重要性并没有明确的规定。

12.3 示例：队列 (Example: Queues)
================================

共享结构并不是一个总让人担心的特性。我们也可以对其加以利用的。这一节展示了怎样用共享结构来表示 `队列 (Queue) <http://zh.wikipedia.org/wiki/%E9%98%9F%E5%88%97>`_ 。队列对象是我们可以按照数据的插入顺序逐个检出数据的仓库，这个规则叫做 `先进先出 (FIFO, first in, first out) <http://zh.wikipedia.org/zh-cn/%E5%85%88%E9%80%B2%E5%85%88%E5%87%BA>`_ 。

用列表表示 `栈 (stack) <http://zh.wikipedia.org/wiki/%E6%A0%88>`_  比较容易，因为栈是从同一端插入和检出。而表示队列要困难些，因为队列的插入和检出是在不同端。为了有效的实现队列，我们需要某种方式能指向列表的两个端。

图 12.6 给出了一种可行的策略。它展示怎样表示一个含有 a，b，c 三个元素的队列。一个队列就是一个列表对 (原文：A queue is a pair of a list)，最后那个 ``cons`` 在那个相同的列表中。这个列表对分别叫做头端 (front)和尾端 (back)。如果要从队列中检出一个元素只需在其头端 ``pop``，要插入一个元素，则创建一个新的 ``cons`` ，把尾端的 ``cdr`` 设置成指向这个 ``cons`` ，然后将尾端指向这个新的 ``cons`` 。

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.6.png

图 12.6 一个队列的结构

.. figure:: https://github.com/JuanitoFatas/acl-chinese/raw/master/images/Figure-12.7.png

图 12.7 队列实现

图 12.7 中的代码实现了这一策略。其用法如下：

::

	> (setf q1 (make-queue))
	(NIL)
	> (progn (enqueue 'a q1)
		 (enqueue 'b q1)
		 (enqueue 'c q1))
	(A B C)

现在，``q1`` 的结构就如图 12.6 那样：

::

	> q1
	((A B C) C)

现在我们从队列中检出一些元素：

::

	> (dequeue q1)
	A
	> (dequeue q1)
	B
	> (enqueue 'd q1)
	(C D) 

12.4 破坏性函数 (Destructive Functions)
===================================================

Common Lisp 包含一些允许修改列表结构的函数。为了提高效率，这些函数是具有破坏性的。虽然它们可以回收作为参数传给它们的 ``conses``，但并不是因为它们的副作用而调用它们 (译者注：而是为了效率而调用它们，下面的例子能说明问题)。

比如，``delete`` 是 ``remove`` 的一个具有破坏性的版本。虽然它可以破坏作为参数传给它的列表，但它并不保证什么。在大多数的 Common Lisp 的实现中，会出现下面的情况：

::

	> (setf lst '(a r a b i a) )
	(A R A B I A)
	> (delete 'a lst )
	(R B I)
	> lst
	(A R B I)

正如 ``remove`` 一样，如果你想要副作用，应该对返回值使用 ``setf``：

::

     (setf lst (delete 'a lst)) 

破坏性函数是怎样回收传给它们的列表的呢？比如，可以考虑 ``nconc`` —— ``append`` 的破坏性版本。[2]下面是两个参数版本的实现，其清楚地展示了两个已知列表是怎样被缝在一起的：

::

	(defun nconc2 ( x y)
	    (if (consp x)
		(progn
		   (setf (cdr (last x)) y)
		    x)
		 y))

我们找到第一个列表的最后一个 *Cons* 核 (cons cells)，把它的 ``cdr`` 设置成指向第二个列表。一个正规的多参数的 ``nconc`` 可以被定义成象附录 B 中的那样。

函数 ``mapcan`` 类似 ``mapcar``，但它是用 ``nconc`` 把函数的返回值 (必须是列表) 拼接在一起的：

::

	> (mapcan #'list
		  '(a b c)
		  '(1 2 3 4))
	( A 1 B 2 C 3)

这个函数可以定义如下：

::

	(defun our-mapcan (fn &rest lsts )
	       (apply #'nconc (apply #'mapcar fn lsts)))

使用 ``mapcan`` 时要谨慎，因为它具有破坏性。它用 ``nconc`` 拼接返回的列表，所以这些列表最好不要再在其它地方使用。

这类函数在处理某些问题的时候特别有用。比如，收集树在某层上的所有子结点，如果 ``children`` 函数返回一个节点的孩子节点的列表，那么我们可以定义一个函数返回某节点的孙子节点的列表如下：

::

	(defun grandchildren (x)
	   (mapcan #'(lambda (c)
			(copy-list (children c)))
		   (children x)))

这个函数调用 ``copy-list`` 时存在一个假设  —— ``chlidren`` 函数返回的是一个保存在某个地方的列表，而不是构建了一个新的列表。

一个 ``mapcan`` 的无损变种可以这样定义：

::

	(defun mappend (fn &rest lsts )
	    (apply #'append (apply #'mapcar fn lsts))) 

如果使用 ``mappend`` 函数，那么 ``grandchildren`` 的定义就可以省去 ``copy-list`` ：

::

	(defun grandchildren (x)
	   (mappend #'children (children x)))

12.5 Example: Binary Search
=======================================

12.6 Example: Doubly-Linked Lists
=======================================

12.7 Circular Structure
==================================================

12.8 Constant Structure
=======================================

Chapter 12 总结 (Summary)
============================

Chapter 12 练习 (Exercises)
==================================

.. rubric:: 脚注

.. [1] 比如，在 Common Lisp 中，修改一个被用作符号名的字符串被认为是一种错误，因为内部的定义并没声明它是从参数复制来的，所以必须假定修改传入内部的任何参数中的字符串来创建新的符号是错误的。
.. [2] n 代表 "non-consing"。一些具有破坏性的函数以 n 开头。