.. highlight:: cl
   :linenothreshold: 0

第十四章：进阶议题
**************************************************

本章是选择性阅读的。本章描述了 Common Lisp 里一些更深奥的特性。Common Lisp 像是一个冰山：大部分的功能对于那些永远不需要他们的多数用户是看不见的。你或许永远不需要自己定义包 (Package)或读取宏 (read-macros)，但当你需要时，有些例子可以让你参考是很有用的。

14.1 类型标识符 (Type Specifiers)
==================================

类型在 Common Lisp 里不是对象。举例来说，没有对象对应到 ``integer`` 这个类型。我们像是从 ``type-of`` 函数里所获得的，以及作为传给像是 ``typep`` 函数的参数，不是一个类型，而是一个类型标识符 (type specifier)。

一个类型标识符是一个类型的名称。最简单的类型标识符是像是 ``integer`` 的符号。这些符号形成了 Common Lisp 里的类型层级。在层级的最顶端是类型 ``t`` –– 所有的对象皆为类型 ``t`` 。而类型层级不是一棵树。从 ``nil`` 至顶端有两条路，举例来说：一条从 ``atom`` ，另一条从 ``list`` 与 ``sequence`` 。

一个类型实际上只是一个对象集合。这意味著有多少类型就有多少个对象的集合：一个无穷大的数目。我们可以用原子的类型标识符 (atomic type specifiers)来表示某些集合：比如 ``integer`` 表示所有整数集合。但我们也可以建构一个复合类型标识符 (compound type specifiers)来参照到任何对象的集合。

举例来说，如果 ``a`` 与 ``b`` 是两个类型标识符，则 ``(or a b)`` 表示分别由 ``a`` 与 ``b`` 类型所表示的联集 (union)。也就是说，一个类型 ``(or a b)`` 的对象是类型 ``a`` 或 类型 ``b`` 。

如果 ``circular?`` 是一个对于 ``cdr`` 为环状的列表返回真的函数，则你可以使用适当的序列集合来表示： [1]_

::

	(or vector (and list (not (satisfies circular?))))

某些原子的类型标识符也可以出现在复合类型标识符。要表示介于 1 至 100 的整数（包含），我们可以用：

::

	(integer 1 100)

这样的类型标识符用来表示一个有限的类型 (finite type)。

在一个复合类型标识符里，你可以通过在一个参数的位置使用 ``*`` 来留下某些未指定的信息。所以

::

	(simple-array fixnum (* *))

描述了指定给 ``fixnum`` 使用的二维简单数组 (simple array)集合，而

::

	(simple-array fixnum *)

描述了指定给 ``finxnum`` 使用的简单数组集合 (前者的超类型 「supertype」)。尾随的星号可以省略，所以上个例子可以写为：

::

	(simple-array fixnum)

若一个复合类型标识符没有传入参数，你可以使用一个原子。所以 ``simple-array`` 描述了所有简单数组的集合。

如果有某些复合类型标识符你想重复使用，你可以使用 ``deftype`` 定义一个缩写。这个宏与 ``defmacro`` 相似，但会展开成一个类型标识符，而不是一个表达式。通过表达

::

	(deftype proseq ()
		'(or vector (and list (not (satisfies circular?)))))

我们定义了 ``proseq`` 作为一个新的原子类型标识符：

::

	> (typep #(1 2) 'proseq)
	T

如果你定义一个接受参数的类型标识符，参数会被视为 Lisp 形式（即没有被求值），与 ``defmacro`` 一样。所以

::

	(deftype multiple-of (n)
	  `(and integer (satisfies (lambda (x)
	                             (zerop (mod x ,n))))))

(译注: 注意上面代码是使用反引号 ````` )

定义了 `(multiple-of n)` 当成所有 ``n`` 的倍数的标识符：

::

	> (type 12 '(multiple-of 4))
	T

类型标识符会被直译 (interpreted)，因此很慢，所以通常你最好定义一个函数来处理这类的测试。

14.2 二进制流 (Binary Streams)
==================================================

第 7 章曾提及的流有二进制流 (binary streams)以及字符流 (character streams)。一个二进制流是一个整数的来源及/或终点，而不是字符。你通过指定一个整数的子类型来创建一个二进制流 –– 当你打开流时，通常是用 ``unsigned-byte`` –– 来作为 ``:element-type`` 的参数。

关于二进制流的 I/O 函数仅有两个， ``read-byte`` 以及 ``write-byte`` 。所以下面是如何定义复制一个文件的函数：

::

	(defun copy-file (from to)
	  (with-open-file (in from :direction :input
	                           :element-type 'unsigned-byte)
	    (with-open-file (out to :direction :output
	                            :element-type 'unsigned-byte)
	      (do ((i (read-byte in nil -1)
	              (read-byte in nil -1)))
	          ((minusp i))
	        (declare (fixnum i))
	        (write-byte i out)))))

仅通过指定 ``unsigned-byte`` 给 ``:element-type`` ，你让操作系统选择一个字节 (byte)的长度。举例来说，如果你明确地想要读写 7 比特的整数，你可以使用：

::

	(unsigned-byte 7)

来传给 ``:element-type`` 。

14.3 读取宏 (Read-Macros)
================================

7.5 节介绍过宏字符 (macro character)的概念，一个对于 ``read`` 有特别意义的字符。每一个这样的字符，都有一个相关联的函数，这函数告诉 ``read`` 当遇到这个字符时该怎么处理。你可以变更某个已存在宏字符所相关联的函数，或是自己定义新的宏字符。

函数 ``set-macro-character`` 提供了一种方式来定义读取宏 (read-macros)。它接受一个字符及一个函数，因此当 ``read`` 碰到该字符时，它返回调用传入函数后的结果。

Lisp 中最古老的读取宏之一是 ``'`` ，即 ``quote`` 。我们可以定义成：

::

	(set-macro-character #\'
		#'(lambda (stream char)
			(list (quote quote) (read stream t nil t))))

当 ``read`` 在一个普通的语境下遇到 ``'`` 时，它会返回在当前流和字符上调用这个函数的结果。(这个函数忽略了第二个参数，第二个参数永远是引用字符。)所以当 ``read`` 看到 ``'a`` 时，会返回 ``(quote a)`` 。

译注: ``read`` 函数接受的参数 ``(read &optional stream eof-error eof-value recursive)``

现在我们明白了 ``read`` 最后一个参数的用途。它表示无论 ``read`` 调用是否在另一个 ``read`` 里。传给 ``read`` 的参数在几乎所有的读取宏里皆相同：传入参数有流 (stream)；接著是第二个参数， ``t`` ，说明了 ``read`` 若读入的东西是 end-of-file 时，应不应该报错；第三个参数说明了不报错时要返回什么，因此在这里也就不重要了；而第四个参数 ``t`` 说明了这个 ``read`` 调用是递归的。

(译注：困惑的话可以看看 `read 的定义 <https://gist.github.com/3467235>`_ )

你可以（通过使用 ``make-dispatch-macro-character`` ）来定义你自己的派发宏字符（dispatching macro character），但由于 ``#`` 已经是一个宏字符，所以你也可以直接使用。六个 ``#`` 打头的组合特别保留给你使用： ``#!`` 、 ``#?`` 、 ``##[`` 、 ``##]`` 、 ``#{`` 、 ``#}`` 。

你可以通过调用 ``set-dispatch-macro-character`` 定义新的派发宏字符组合，与 ``set-macro-character`` 类似，除了它接受两个字符参数外。下面的代码定义了 ``#?`` 作为返回一个整数列表的读取宏。

::

	(set-dispatch-macro-character #\# #\?
	  #'(lambda (stream char1 char2)
	      (list 'quote
	            (let ((lst nil))
	              (dotimes (i (+ (read stream t nil t) 1))
	                (push i lst))
	              (nreverse lst)))))

现在 ``#?n`` 会被读取成一个含有整数 ``0`` 至 ``n`` 的列表。举例来说：

::

	> #?7
	(1 2 3 4 5 6 7)

除了简单的宏字符，最常定义的宏字符是列表分隔符 (list delimiters)。另一个保留给用户的字符组是 ``#{`` 。以下我们定义了一种更复杂的左括号：

::

	(set-macro-character #\} (get-macro-character #\)))

	(set-dispatch-macro-character #\# #\{
	  #'(lambda (stream char1 char2)
	      (let ((accum nil)
	            (pair (read-delimited-list #\} stream t)))
	        (do ((i (car pair) (+ i 1)))
	            ((> i (cadr pair))
	             (list 'quote (nreverse accum)))
	          (push i accum)))))

这定义了一个这样形式 ``#{x y}`` 的表达式，使得这样的表达式被读取为所有介于 ``x`` 与 ``y`` 之间的整数列表，包含 ``x`` 与 ``y`` ：

::

	> #{2 7}
	(2 3 4 4 5 6 7)

函数 ``read-delimited-list`` 正是为了这样的读取宏而生的。它的第一个参数是被视为列表结束的字符。为了使 ``}`` 被识别为分隔符，必须先给它这个角色，所以程序在开始的地方调用了 ``set-macro-character`` 。

如果你想要在定义一个读取宏的文件里使用该读取宏，则读取宏的定义应要包在一个 ``eval-when`` 表达式里，来确保它在编译期会被求值。不然它的定义会被编译，但不会被求值，直到编译文件被载入时才会被求值。

14.4 包 (Packages)
===================================================

一个包是一个将名字映对到符号的 Lisp 对象。当前的包总是存在全局变量 ``*package*`` 里。当 Common Lisp 启动时，当前的包会是 ``*common-lisp-user*`` ，通常称为用户包 (user package)。函数 ``package-name`` 返回包的名字，而 ``find-package`` 返回一个给定名称的包:

::

	> (package-name *package*)
	"COMMON-LISP-USER"
	> (find-package "COMMON-LISP-USER")
	#<Package "COMMON-LISP-USER" 4CD15E>

通常一个符号在读入时就被 interned 至当前的包里面了。函数 ``symbol-package`` 接受一个符号并返回该符号被 interned 的包。

::

	(symbol-package 'sym)
	#<Package "COMMON-LISP-USER" 4CD15E>

有趣的是，这个表达式返回它该返回的值，因为表达式在可以被求值前必须先被读入，而读取这个表达式导致 ``sym`` 被 interned。为了之后的用途，让我们给 ``sym`` 一个值:

::

	> (setf sym 99)
	99

现在我们可以创建及切换至一个新的包：

::

	> (setf *package* (make-package 'mine
	                                :use '(common-lisp)))
	#<Package "MINE" 63390E>

现在应该会听到诡异的背景音乐，因为我们来到一个不一样的世界了：
在这里 ``sym`` 不再是本来的 ``sym`` 了。

::

	MINE> sym
	Error: SYM has no value

为什么会这样？因为上面我们设为 99 的 ``sym`` 与 ``mine`` 里的 ``sym`` 是两个不同的符号。 [2]_ 要在用户包之外参照到原来的 ``sym`` ，我们必须把包的名字加上两个冒号作为前缀：

::

	MINE> common-lisp-user::sym
	99

所以有着相同打印名称的不同符号能够在不同的包内共存。可以有一个 ``sym`` 在 ``common-lisp-user`` 包，而另一个 ``sym`` 在 ``mine`` 包，而他们会是不一样的符号。这就是包存在的意义。如果你在分开的包内写你的程序，你大可放心选择函数与变量的名字，而不用担心某人使用了同样的名字。即便是他们使用了同样的名字，也不会是相同的符号。

包也提供了信息隐藏的手段。程序应通过函数与变量的名字来参照它们。如果你不让一个名字在你的包之外可见的话，那么另一个包中的代码就无法使用或者修改这个名字所参照的对象。

通常使用两个冒号作为包的前缀也是很差的风格。这么做你就违反了包本应提供的模块性。如果你不得不使用一个双冒号来参照到一个符号，这是因为某人根本不想让你用。

通常我们应该只参照被输出 ( *exported* )的符号。如果我们回到用户包里，并输出一个被 interned 的符号，

::

	MINE> (in-package common-lisp-user)
	#<Package "COMMON-LISP-USER" 4CD15E>
	> (export 'bar)
	T
	> (setf bar 5)
	5

我们使这个符号对于其它的包是可视的。现在当我们回到 ``mine`` ，我们可以仅使用单冒号来参照到 ``bar`` ，因为他是一个公开可用的名字：

::

	> (in-package mine)
	#<Package "MINE" 63390E>
	MINE> common-lisp-user:bar
	5

通过把 ``bar`` 输入 ( ``import`` )至 ``mine`` 包，我们就能进一步让 ``mine`` 和 ``user`` 包可以共享 ``bar`` 这个符号：

::

	MINE> (import 'common-lisp-user:bar)
	T
	MINE> bar
	5

在输入 ``bar`` 之后，我们根本不需要用任何包的限定符 (package qualifier)，就能参照它了。这两个包现在共享了同样的符号；不可能会有一个独立的 ``mine:bar`` 了。

要是已经有一个了怎么办？在这种情况下， ``import`` 调用会产生一个错误，如下面我们试著输入 ``sym`` 时便知：

::

	MINE> (import 'common-lisp-user::sym)
	Error: SYM is already present in MINE.

在此之前，当我们试着在 ``mine`` 包里对 ``sym`` 进行了一次不成功的求值，我们使 ``sym`` 被 interned 至 ``mine`` 包里。而因为它没有值，所以产生了一个错误，但输入符号名的后果就是使这个符号被 intern 进这个包。所以现在当我们试著输入 ``sym`` 至 ``mine`` 包里，已经有一个相同名称的符号了。

另一个方法来获得别的包内符号的存取权是使用( ``use`` )它：

::

	MINE> (use-package 'common-lisp-user)
	T

现在所有由用户包 (译注: common-lisp-user 包）所输出的符号，可以不需要使用任何限定符在 ``mine`` 包里使用。(如果 ``sym`` 已经被用户包输出了，这个调用也会产生一个错误。)

含有自带操作符及变量名字的包叫做 ``common-lisp`` 。由于我们将这个包的名字在创建 ``mine`` 包时作为 ``make-package`` 的 ``:use`` 参数，所有的 Common Lisp 自带的名字在 ``mine`` 里都是可视的:

::

	MINE> #'cons
	#<Compiled-Function CONS 462A3E>

在编译后的代码中, 通常不会像这样在顶层进行包的操作。更常见的是包的调用会包含在源文件里。通常，只要把 ``in-package`` 和 ``defpackage`` 放在源文件的开头就可以了，正如 137 页所示。

这种由包所提供的模块性实际上有点奇怪。我们不是对象的模块 (modules)，而是名字的模块。

每一个使用了 ``common-lisp`` 的包，都可以存取 ``cons`` ，因为 ``common-lisp`` 包里有一个叫这个名字的函数。但这会导致一个名字为 ``cons`` 的变量也会在每个使用了 ``common-lisp`` 包里是可视的。如果包使你困惑，这就是主要的原因；因为包不是基于对象而是基于名字。

14.5 Loop 宏 (The Loop Facility)
=======================================

``loop`` 宏最初是设计来帮助无经验的 Lisp 用户来写出迭代的代码。与其撰写 Lisp 代码，你用一种更接近英语的形式来表达你的程序，然后这个形式被翻译成 Lisp。不幸的是， ``loop`` 比原先设计者预期的更接近英语：你可以在简单的情况下使用它，而不需了解它是如何工作的，但想在抽象层面上理解它几乎是不可能的。

如果你是曾经计划某天要理解 ``loop`` 怎么工作的许多 Lisp 程序员之一，有一些好消息与坏消息。好消息是你并不孤单：几乎没有人理解它。坏消息是你永远不会理解它，因为 ANSI 标准实际上并没有给出它行为的正式规范。

这个宏唯一的实际定义是它的实现方式，而唯一可以理解它（如果有人可以理解的话）的方法是通过实例。ANSI 标准讨论 ``loop`` 的章节大部分由例子组成，而我们将会使用同样的方式来介绍相关的基础概念。

第一个关于 ``loop`` 宏我们要注意到的是语法 ( *syntax* )。一个 ``loop`` 表达式不是包含子表达式而是子句 (*clauses*)。這些子句不是由括号分隔出来；而是每种都有一个不同的语法。在这个方面上， ``loop`` 与传统的 Algol-like 语言相似。但其它 ``loop`` 独特的特性，使得它与 Algol 不同，也就是在 ``loop`` 宏里调换子句的顺序与会发生的事情没有太大的关联。

一个 ``loop`` 表达式的求值分为三个阶段，而一个给定的子句可以替多于一个的阶段贡献代码。这些阶段如下：

1. *序幕* (*Prologue*)。 被求值一次来做为迭代过程的序幕。包括了将变量设至它们的初始值。

2. *主体* (*Body*) 每一次迭代时都会被求值。

3. *闭幕* (*Epilogue*) 当迭代结束时被求值。决定了 ``loop`` 表达式的返回值（可能返回多个值）。

我们会看几个 ``loop`` 子句的例子，并考虑何种代码会贡献至何个阶段。

举例来说，最简单的 ``loop`` 表达式，我们可能会看到像是下列的代码：

::

	> (loop for x from 0 to 9
	        do (princ x))
	0123456789
	NIL

这个 ``loop`` 表达式印出从 ``0`` 至 ``9`` 的整数，并返回 ``nil`` 。第一个子句，

``for x from 0 to 9``

贡献代码至前两个阶段，导致 ``x`` 在序幕中被设为 ``0`` ，在主体开头与 ``9`` 来做比较，在主体结尾被递增。第二个子句，

``do (princ x)``

贡献代码给主体。

一个更通用的 ``for`` 子句说明了起始与更新的形式 (initial and update form)。停止迭代可以被像是 ``while`` 或 ``until`` 子句来控制。

::

	> (loop for x = 8 then (/ x 2)
	        until (< x 1)
	        do (princ x))
	8421
	NIL

你可以使用 ``and`` 来创建复合的 ``for`` 子句，同时初始及更新两个变量：

::

	> (loop for x from 1 to 4
	        and y from 1 to 4
	        do (princ (list x y)))
	(1 1)(2 2)(3 3)(4 4)
	NIL

要不然有多重 ``for`` 子句时，变量会被循序更新。

另一件在迭代代码通常会做的事是累积某种值。举例来说：

::

	> (loop for x in '(1 2 3 4)
	        collect (1+ x))
	(2 3 4 5)

在 ``for`` 子句使用 ``in`` 而不是 ``from`` ，导致变量被设为一个列表的后续元素，而不是连续的整数。

在这个情况里， ``collect`` 子句贡献代码至三个阶段。在序幕，一個匿名累加器 (anonymous accumulator)設為 ``nil`` ；在主体裡， ``(1+ x)`` 被累加至這個累加器，而在闭幕时返回累加器的值。

这是返回一个特定值的第一个例子。有用来明确指定返回值的子句，但没有这些子句时，一个 ``collect`` 子句决定了返回值。所以我们在这里所做的其实是重复了 ``mapcar`` 。

``loop`` 最常见的用途大概是蒐集调用一个函数数次的结果：

::

	> (loop for x from 1 to 5
	        collect (random 10))
	(3 8 6 5 0)

这里我们获得了一个含五个随机数的列表。这跟我们定义过的 ``map-int`` 情况类似 (105 页「译注: 6.4 小节。」)。如果我们有了 ``loop`` ，为什么还需要 ``map-int`` ？另一个人也可以说，如果我们有了 ``map-int`` ，为什么还需要 ``loop`` ？

一个 ``collect`` 子句也可以累积值到一个有名字的变量上。下面的函数接受一个数字的列表并返回偶数与奇数列表：

::

	(defun even/odd (ns)
	  (loop for n in ns
	        if (evenp n)
	           collect n into evens
	           else collect n into odds
	        finally (return (values evens odds))))

一个 ``finally`` 子句贡献代码至闭幕。在这个情况它指定了返回值。

一个 ``sum`` 子句和一个 ``collect`` 子句类似，但 ``sum`` 子句累积一个数字，而不是一个列表。要获得 ``1`` 至 ``n`` 的和，我们可以写：

::

	(defun sum (n)
	  (loop for x from 1 to n
	        sum x))

``loop`` 更进一步的细节在附录 D 讨论，从 325 页开始。举个例子，图 14.1 包含了先前章节的两个迭代函数，而图 14.2 演示了将同样的函数翻译成 ``loop`` 。

::

	(defun most (fn lst)
	  (if (null lst)
	      (values nil nil)
	      (let* ((wins (car lst))
	             (max (funcall fn wins)))
	        (dolist (obj (cdr lst))
	          (let ((score (funcall fn obj)))
	            (when (> score max)
	              (setf wins obj
	                    max  score))))
	        (values wins max))))

	(defun num-year (n)
	  (if (< n 0)
	      (do* ((y (- yzero 1) (- y 1))
	            (d (- (year-days y)) (- d (year-days y))))
	           ((<= d n) (values y (- n d))))
	      (do* ((y yzero (+ y 1))
	            (prev 0 d)
	            (d (year-days y) (+ d (year-days y))))
	           ((> d n) (values y (- n prev))))))

**图 14.1 不使用 loop 的迭代函数**

::

	(defun most (fn lst)
	  (if (null lst)
	      (values nil nil)
	      (loop with wins = (car lst)
	            with max = (funcall fn wins)
	            for obj in (cdr lst)
	            for score = (funcall fn obj)
	            when (> score max)
	                 (do (setf wins obj
	                           max score)
	            finally (return (values wins max))))))

	(defun num-year (n)
	  (if (< n 0)
	      (loop for y downfrom (- yzero 1)
	            until (<= d n)
	            sum (- (year-days y)) into d
	            finally (return (values (+ y 1) (- n d))))
	      (loop with prev = 0
	            for y from yzero
	            until (> d n)
	            do (setf prev d)
	            sum (year-days y) into d
	            finally (return (values (- y 1)
	                                    (- n prev))))))

**图 14.2 使用 loop 的迭代函数**

一个 ``loop`` 的子句可以参照到由另一个子句所设置的变量。举例来说，在 ``even/odd`` 的定义里面， ``finally`` 子句参照到由两个 ``collect`` 子句所创建的变量。这些变量之间的关系，是 ``loop`` 定义最含糊不清的地方。考虑下列两个表达式：

::

	(loop for y = 0 then z
	      for x from 1 to 5
	      sum 1 into z
	      finally (return y z))

	(loop for x from 1 to 5
	      for y = 0 then z
	      sum 1 into z
	      finally (return y z))

它们看起来够简单 –– 每一个有四个子句。但它们返回同样的值吗？它们返回的值多少？你若试着在标准中想找答案将徒劳无功。每一个 ``loop`` 子句本身是够简单的。但它们组合起来的方式是极为复杂的 –– 而最终，甚至标准里也没有明确定义。

由于这类原因，使用 ``loop`` 是不推荐的。推荐 ``loop`` 的理由，你最多可以说，在像是图 14.2 这般经典的例子中， ``loop`` 让代码看起来更容易理解。

14.6 状况 (Conditions)
=======================================

在 Common Lisp 里，状况 (condition)包括了错误以及其它可能在执行期发生的情况。当一个状况被捕捉时 (signalled)，相应的处理程序 (handler)会被调用。处理错误状况的缺省处理程序通常会调用一个中断循环 (break-loop)。但 Common Lisp 提供了多样的操作符来捕捉及处理错误。要覆写缺省的处理程序，甚至是自己写一个新的处理程序也是有可能的。

多数的程序员不会直接处理状况。然而有许多更抽象的操作符使用了状况，而要了解这些操作符，知道背后的原理是很有用的。

Common lisp 有数个操作符用来捕捉错误。最基本的是 ``error`` 。一个调用它的方法是给入你会给 ``format`` 的相同参数：

::

	> (error "Your report uses ~A as a verb." 'status)
	Error: Your report uses STATUS as a verb
				 Options: :abort, :backtrace
	>>

如上所示，除非这样的状况被处理好了，不然执行就会被打断。

用来捕捉错误的更抽象操作符包括了 ``ecase`` 、 ``check-type`` 以及 ``assert`` 。前者与 ``case`` 相似，要是没有键值匹配时会捕捉一个错误：

::

	> (ecase 1 (2 3) (4 5))
	Error: No applicable clause
				 Options: :abort, :backtrace
	>>

普通的 ``case`` 在没有键值匹配时会返回 ``nil`` ，但由于利用这个返回值是很差的编码风格，你或许会在当你没有 ``otherwise`` 子句时使用 ``ecase`` 。

``check-type`` 宏接受一个位置，一个类型名以及一个选择性字串，并在该位置的值不是预期的类型时，捕捉一个可修正的错误 (correctable error)。一个可修正错误的处理程序会给我们一个机会来提供一个新的值：

::

	> (let ((x '(a b c)))
			(check-type (car x) integer "an integer")
			x)
	Error: The value of (CAR X), A, should be an integer.
	Options: :abort, :backtrace, :continue
	>> :continue
	New value of (CAR X)? 99
	(99 B C)
	>

在这个例子里， ``(car x)`` 被设为我们提供的新值，并重新执行，返回了要是 ``(car x)`` 本来就包含我们所提供的值所会返回的结果。

这个宏是用更通用的 ``assert`` 所定义的， ``assert`` 接受一个测试表达式以及一个有着一个或多个位置的列表，伴随着你可能传给 ``error`` 的参数：

::

	> (let ((sandwich '(ham on rye)))
	    (assert (eql (car sandwich) 'chicken)
	            ((car sandwich))
	            "I wanted a ~A sandwich." 'chicken)
	    sandwich)
	Error: I wanted a CHICKEN sandwich.
	Options: :abort, :backtrace, :continue
	>> :continue
	New value of (CAR SANDWICH)? 'chicken
	(CHICKEN ON RYE)

要建立新的处理程序也是可能的，但大多数程序员只会间接的利用这个可能性，通过使用像是 ``ignore-errors`` 的宏。如果它的参数没产生错误时像在 ``progn`` 里求值一样，但要是在求值过程中，不管什么参数报错，执行是不会被打断的。取而代之的是， ``ignore-errors`` 表达式会直接返回两个值： ``nil`` 以及捕捉到的状况。

举例来说，如果在某个时候，你想要用户能够输入一个表达式，但你不想要在输入是语法上不合时中断执行，你可以这样写：

::

	(defun user-input (prompt)
	  (format t prompt)
	  (let ((str (read-line)))
	    (or (ignore-errors (read-from-string str))
	        nil)))

若输入包含语法错误时，这个函数仅返回 ``nil`` :

::

	> (user-input "Please type an expression")
	Please type an expression> #%@#+!!
	NIL


.. rubric:: 脚注

.. [1] 虽然标准没有提到这件事，你可以假定 ``and`` 以及 ``or`` 类型标示符仅考虑它们所要考虑的参数，与 ``or`` 及 ``and`` 宏类似。

.. [2] 某些 Common Lisp 实现，当我们不在用户包下时，会在顶层提示符前打印包的名字。
