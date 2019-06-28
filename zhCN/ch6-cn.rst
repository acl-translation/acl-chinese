.. highlight:: cl

第六章：函数
***************************************************

理解函数是理解 Lisp 的关键之一。概念上来说，函数是 Lisp 的核心所在。实际上呢，函数是你手边最有用的工具之一。

6.1 全局函数 (Global Functions)
==================================

谓词 ``fboundp`` 告诉我们，是否有个函数的名字与给定的符号绑定。如果一个符号是函数的名字，则 ``symbol-function`` 会返回它：

::

  > (fboundp '+)
  T
  > (symbol-function '+)
  #<Compiled-function + 17BA4E>

可通过 ``symbol-function`` 给函数配置某个名字：

::

  (setf (symbol-function 'add2)
    #'(lambda (x) (+ x 2)))

新的全局函数可以这样定义，用起来和 ``defun`` 所定义的函数一样：

::

  > (add2 1)
  3

实际上 ``defun`` 做了稍微多的工作，将某些像是

::

  (defun add2 (x) (+ x 2))

翻译成上述的 ``setf`` 表达式。使用 ``defun`` 让程序看起来更美观，并或多或少帮助了编译器，但严格来说，没有 ``defun`` 也能写程序。

通过把 ``defun`` 的第一个实参变成这种形式的列表 ``(setf f)`` ，你定义了当 ``setf`` 第一个实参是 ``f`` 的函数调用时，所会发生的事情。下面这对函数把 ``primo`` 定义成 ``car`` 的同义词：

::

  (defun primo (lst) (car lst))

  (defun (setf primo) (val lst)
    (setf (car lst) val))

在函数名是这种形式 ``(setf f)`` 的函数定义中，第一个实参代表新的数值，而剩余的实参代表了传给 ``f`` 的参数。

现在任何 ``primo`` 的 ``setf`` ，会是上面后者的函数调用：

::

  > (let ((x (list 'a 'b 'c)))
      (setf (primo x) 480)
      x)
  (480 b c)

不需要为了定义 ``(setf primo)`` 而定义 ``primo`` ，但这样的定义通常是成对的。

由于字符串是 Lisp 表达式，没有理由它们不能出现在代码的主体。字符串本身是没有副作用的，除非它是最后一个表达式，否则不会造成任何差别。如果让字符串成为 ``defun`` 定义的函数主体的第一个表达式，

::

  (defun foo (x)
    "Implements an enhanced paradigm of diversity"
    x)

那么这个字符串会变成函数的文档字符串（documentation string）。要取得函数的文档字符串，可以通过调用 ``documentation`` 来取得：

::

  > (documentation 'foo 'function)
  "Implements an enhanced paradigm of diversity"

6.2 局部函数 (Local Functions)
===============================

通过 ``defun`` 或 ``symbol-function`` 搭配 ``setf`` 定义的函数是全局函数。你可以像存取全局变量那样，在任何地方存取它们。定义局部函数也是有可能的，局部函数和局部变量一样，只在某些上下文内可以访问。

局部函数可以使用 ``labels`` 来定义，它是一种像是给函数使用的 ``let`` 。它的第一个实参是一个新局部函数的定义列表，而不是一个变量规格说明的列表。列表中的元素为如下形式：

::

  (name parameters . body)

而 ``labels`` 表达式剩余的部份，调用 ``name`` 就等于调用 ``(lambda parameters . body)`` 。

::

  > (labels ((add10 (x) (+ x 10))
             (consa  (x) (cons 'a x)))
      (consa (add10 3)))
  (A . 13)

``labels`` 与 ``let`` 的类比在一个方面上被打破了。由 ``labels`` 表达式所定义的局部函数，可以被其他任何在此定义的函数引用，包括自己。所以这样定义一个递归的局部函数是可能的：

::

  > (labels ((len (lst)
               (if (null lst)
                   0
                   (+ (len (cdr lst)) 1))))
      (len '(a b c)))
  3

5.2 节展示了 ``let`` 表达式如何被理解成函数调用。 ``do`` 表达式同样可以被解释成调用递归函数。这样形式的 ``do`` :

::

  (do ((x a (b x))
       (y c (d y)))
      ((test x y) (z x y))
    (f x y))

等同于

::

  (labels ((rec (x y)
             (cond ((test x y)
                    (z x y))
                   (t
                    (f x y)
                    (rec (b x) (d y))))))
    (rec a c))

这个模型可以用来解决，任何你对于 ``do`` 行为仍有疑惑的问题。

6.3 参数列表 (Parameter Lists)
================================

2.1 节我们演示过，有了前序表达式， ``+`` 可以接受任何数量的参数。从那时开始，我们看过许多接受不定数量参数的函数。要写出这样的函数，我们需要使用一个叫做剩余（ *rest* ）参数的东西。

如果我们在函数的形参列表里的最后一个变量前，插入 ``&rest`` 符号，那么当这个函数被调用时，这个变量会被设成一个带有剩余参数的列表。现在我们可以明白 ``funcall`` 是如何根据 ``apply`` 写成的。它或许可以定义成：

::

  (defun our-funcall (fn &rest args)
    (apply fn args))

我们也看过操作符中，有的参数可以被忽略，并可以缺省设成特定的值。这样的参数称为选择性参数（optional parameters）。（相比之下，普通的参数有时称为必要参数「required parameters」) 如果符号 ``&optional`` 出现在一个函数的形参列表时，

::

  (defun philosoph (thing &optional property)
    (list thing 'is property))

那么在 ``&optional`` 之后的参数都是选择性的，缺省为 ``nil`` :

::

  > (philosoph 'death)
  (DEATH IS NIL)

我们可以明确指定缺省值，通过将缺省值附在列表里给入。这版的 ``philosoph``

::

  (defun philosoph (thing &optional (property 'fun))
    (list thing 'is property))

有着更鼓舞人心的缺省值：

::

  > (philosoph 'death)
  (DEATH IS FUN)

选择性参数的缺省值可以不是常量。可以是任何的 Lisp 表达式。若这个表达式不是常量，它会在每次需要用到缺省值时被重新求值。

一个关键字参数（keyword parameter）是一种更灵活的选择性参数。如果你把符号 ``&key`` 放在一个形参列表，那在 ``&key`` 之后的形参都是选择性的。此外，当函数被调用时，这些参数会被识别出来，参数的位置在哪不重要，而是用符号标签（译注: ``:`` ）识别出来：

::

  > (defun keylist (a &key x y z)
      (list a x y z))
  KEYLIST

  > (keylist 1 :y 2)
  (1 NIL 2 NIL)

  > (keylist 1 :y 3 :x 2)
  (1 2 3 NIL)

和普通的选择性参数一样，关键字参数缺省值为 ``nil`` ，但可以在形参列表中明确地指定缺省值。

关键字与其相关的参数可以被剩余参数收集起来，并传递给其他期望收到这些参数的函数。举例来说，我们可以这样定义 ``adjoin`` ：

::

  (defun our-adjoin (obj lst &rest args)
    (if (apply #'member obj lst args)
        lst
        (cons obj lst)))

由于 ``adjoin`` 与 ``member`` 接受一样的关键字，我们可以用剩余参数收集它们，再传给 ``member`` 函数。

5.2 节介绍过 ``destructuring-bind`` 宏。在通常情况下，每个模式（pattern）中作为第一个参数的子树，可以与函数的参数列表一样复杂：

::

  (destructuring-bind ((&key w x) &rest y) '((:w 3) a)
    (list w x y))
  (3 NIL (A))

6.4 示例：实用函数 (Example: Utilities)
=========================================

2.6 节提到过，Lisp 大部分是由 Lisp 函数组成，这些函数与你可以自己定义的函数一样。这是程序语言中一个有用的特色：你不需要改变你的想法来配合语言，因为你可以改变语言来配合你的想法。如果你想要 Common Lisp 有某个特定的函数，自己写一个，而这个函数会成为语言的一部分，就跟内置的 ``+`` 或 ``eql`` 一样。

有经验的 Lisp 程序员，由上而下（top-down）也由下而上 (bottom-up)地工作。当他们朝着语言撰写程序的同时，也打造了一个更适合他们程序的语言。通过这种方式，语言与程序结合的更好，也更好用。

写来扩展 Lisp 的操作符称为实用函数（utilities）。当你写了更多 Lisp 程序时，会发现你开发了一系列的程序，而在一个项目写过许多的实用函数，下个项目里也会派上用场。

专业的程序员常发现，手边正在写的程序，与过去所写的程序有很大的关联。这就是软件重用让人听起来很吸引人的原因。但重用已经被联想成面向对象程序设计。但软件不需要是面向对象的才能重用 ── 这是很明显的，我们看看程序语言（换言之，编译器），是重用性最高的软件。

要获得可重用软件的方法是，由下而上地写程序，而程序不需要是面向对象的才能够由下而上地写出。实际上，函数式风格相比之下，更适合写出重用软件。想想看 ``sort`` 。在 Common Lisp 你几乎不需要自己写排序程序； ``sort`` 是如此的快与普遍，以致于它不值得我们烦恼。这才是可重用软件。

::

  (defun single? (lst)
    (and (consp lst) (null (cdr lst))))

  (defun append1 (lst obj)
    (append lst (list obj)))

  (defun map-int (fn n)
    (let ((acc nil))
      (dotimes (i n)
        (push (funcall fn i) acc))
      (nreverse acc)))

  (defun filter (fn lst)
    (let ((acc nil))
      (dolist (x lst)
        (let ((val (funcall fn x)))
          (if val (push val acc))))
      (nreverse acc)))

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

**图 6.1 实用函数**

你可以通过撰写实用函数，在程序里做到同样的事情。图 6.1 挑选了一组实用的函数。前两个 ``single?`` 与 ``append1`` 函数，放在这的原因是要演示，即便是小程序也很有用。前一个函数 ``single?`` ，当实参是只有一个元素的列表时，返回真。

::

  > (single? '(a))
  T

而后一个函数 ``append1`` 和 ``cons`` 很像，但在列表后面新增一个元素，而不是在前面:

::

  > (append1 '(a b c) 'd)
  (A B C D)

下个实用函数是 ``map-int`` ，接受一个函数与整数 ``n`` ，并返回将函数应用至整数 ``0`` 到 ``n-1`` 的结果的列表。

这在测试的时候非常好用（一个 Lisp 的优点之一是，互动环境让你可以轻松地写出测试）。如果我们只想要一个 ``0`` 到 ``9`` 的列表，我们可以：

::

  > (map-int #'identity 10)
  (0 1 2 3 4 5 6 7 8 9)

然而要是我们想要一个具有 10 个随机数的列表，每个数介于 0 至 99 之间（包含 99），我们可以忽略参数并只要:

::

  > (map-int #'(lambda (x) (random 100))
             10)
  (85 50 73 64 28 21 40 67 5 32)

``map-int`` 的定义说明了 Lisp 构造列表的标准做法（idiom）之一。我们创建一个累积器 ``acc`` ，初始化是 ``nil`` ，并将之后的对象累积起来。当累积完毕时，反转累积器。 [1]_

我们在 ``filter`` 中看到同样的做法。 ``filter`` 接受一个函数与一个列表，将函数应用至列表元素上时，返回所有非 ``nil`` 元素:

::

  > (filter #'(lambda (x)
                (and (evenp x) (+ x 10)))
            '(1 2 3 4 5 6 7))
  (12 14 16)

另一种思考 ``filter`` 的方式是用通用版本的 ``remove-if`` 。

图 6.1 的最后一个函数， ``most`` ，根据某个评分函数（scoring function），返回列表中最高分的元素。它返回两个值，获胜的元素以及它的分数:

::

  > (most #'length '((a b) (a b c) (a)))
  (A B C)
  3

如果平手的话，返回先驰得点的元素。

注意图 6.1 的最后三个函数，它们全接受函数作为参数。 Lisp 使得将函数作为参数传递变得便捷，而这也是为什么，Lisp 适合由下而上程序设计的原因之一。成功的实用函数必须是通用的，当你可以将细节作为函数参数传递时，要将通用的部份抽象起来就变得容易许多。

本节给出的函数是通用的实用函数。可以用在任何种类的程序。但也可以替特定种类的程序撰写实用函数。确实，当我们谈到宏时，你可以凌驾于 Lisp 之上，写出自己的特定语言，如果你想这么做的话。如果你想要写可重用软件，看起来这是最靠谱的方式。

6.5 闭包 (Closures)
=======================================

函数可以如表达式的值，或是其它对象那样被返回。以下是接受一个实参，并依其类型返回特定的结合函数：

::

  (defun combiner (x)
    (typecase x
      (number #'+)
      (list #'append)
      (t #'list)))

在这之上，我们可以创建一个通用的结合函数:

::

  (defun combine (&rest args)
    (apply (combiner (car args))
           args))

它接受任何类型的参数，并以适合它们类型的方式结合。（为了简化这个例子，我们假定所有的实参，都有着一样的类型。）

::

  > (combine 2 3)
  5
  > (combine '(a b) '(c d))
  (A B C D)

2.10 小节提过词法变量（lexical variables）只在被定义的上下文内有效。伴随这个限制而来的是，只要那个上下文还有在使用，它们就保证会是有效的。

如果函数在词法变量的作用域里被定义时，函数仍可引用到那个变量，即便函数被作为一个值返回了，返回至词法变量被创建的上下文之外。下面我们创建了一个把实参加上 ``3`` 的函数：

::

  > (setf fn (let ((i 3))
               #'(lambda (x) (+ x i))))
  #<Interpreted-Function C0A51E>
  > (funcall fn 2)
  5

当函数引用到外部定义的变量时，这外部定义的变量称为自由变量（free variable）。函数引用到自由的词法变量时，称之为闭包（closure）。 [2]_ 只要函数还存在，变量就必须一起存在。

闭包结合了函数与环境（environment）；无论何时，当一个函数引用到周围词法环境的某个东西时，闭包就被隐式地创建出来了。这悄悄地发生在像是下面这个函数，是一样的概念:

::

  (defun add-to-list (num lst)
    (mapcar #'(lambda (x)
                (+ x num))
            lst))

这函数接受一个数字及列表，并返回一个列表，列表元素是元素与传入数字的和。在 lambda 表达式里的变量 ``num`` 是自由的，所以像是这样的情况，我们传递了一个闭包给 ``mapcar`` 。

一个更显着的例子会是函数在被调用时，每次都返回不同的闭包。下面这个函数返回一个加法器（adder）:

::

  (defun make-adder (n)
    #'(lambda (x)
        (+ x n)))

它接受一个数字，并返回一个将该数字与其参数相加的闭包（函数）。

::

  > (setf add3 (make-adder 3))
  #<Interpreted-Function COEBF6>
  > (funcall add3 2)
  5
  > (setf add27 (make-adder 27))
  #<Interpreted-Function C0EE4E>
  > (funcall add27 2)
  29

我们可以产生共享变量的数个闭包。下面我们定义共享一个计数器的两个函数:

::

  (let ((counter 0))
    (defun reset ()
      (setf counter 0))
    (defun stamp ()
      (setf counter (+ counter 1))))

这样的一对函数或许可以用来创建时间戳章（time-stamps）。每次我们调用 ``stamp`` 时，我们获得一个比之前高的数字，而调用 ``reset`` 我们可以将计数器归零:

::

  > (list (stamp) (stamp) (reset) (stamp))
  (1 2 0 1)

你可以使用全局计数器来做到同样的事情，但这样子使用计数器，可以保护计数器被非预期的引用。

Common Lisp 有一个内置的函数 ``complement`` 函数，接受一个谓词，并返回谓词的补数（complement）。比如：

::

  > (mapcar (complement #'oddp)
            '(1 2 3 4 5 6))
  (NIL T NIL T NIL T)

有了闭包以后，很容易就可以写出这样的函数：

::

  (defun our-complement (f)
    #'(lambda (&rest args)
        (not (apply f args))))

如果你停下来好好想想，会发现这是个非凡的小例子；而这仅是冰山一角。闭包是 Lisp 特有的美妙事物之一。闭包开创了一种在别的语言当中，像是不可思议的程序设计方法。

6.6 示例：函数构造器 (Example: Function Builders)
=====================================================

Dylan 是 Common Lisp 与 Scheme 的混合物，有着 Pascal 一般的语法。它有着大量返回函数的函数：除了上一节我们所看过的 `complement` ，Dylan 包含: ``compose`` 、 ``disjoin`` 、 ``conjoin`` 、 ``curry`` 、 ``rcurry`` 以及 ``always`` 。图 6.2 有这些函数的 Common Lisp 实现，而图 6.3 演示了一些从定义延伸出的等价函数。

::

  (defun compose (&rest fns)
    (destructuring-bind (fn1 . rest) (reverse fns)
      #'(lambda (&rest args)
          (reduce #'(lambda (v f) (funcall f v))
                  rest
                  :initial-value (apply fn1 args)))))

  (defun disjoin (fn &rest fns)
    (if (null fns)
        fn
        (let ((disj (apply #'disjoin fns)))
          #'(lambda (&rest args)
              (or (apply fn args) (apply disj args))))))

  (defun conjoin (fn &rest fns)
    (if (null fns)
        fn
        (let ((conj (apply #'conjoin fns)))
          #'(lambda (&rest args)
              (and (apply fn args) (apply conj args))))))

  (defun curry (fn &rest args)
    #'(lambda (&rest args2)
        (apply fn (append args args2))))

  (defun rcurry (fn &rest args)
    #'(lambda (&rest args2)
        (apply fn (append args2 args))))

  (defun always (x) #'(lambda (&rest args) x))

**图 6.2 Dylan 函数建构器**

首先， ``compose`` 接受一个或多个函数，并返回一个依序将其参数应用的新函数，即，

::

  (compose #'a #'b #'c)

返回一个函数等同于

::

  #'(lambda (&rest args) (a (b (apply #'c args))))

这代表着 ``compose`` 的最后一个实参，可以是任意长度，但其它函数只能接受一个实参。

下面我们建构了一个函数，先给取参数的平方根，取整后再放回列表里，接着返回:

::

  > (mapcar (compose #'list #'round #'sqrt)
            '(4 9 16 25))
  ((2) (3) (4) (5))

接下来的两个函数， ``disjoin`` 及 ``conjoin`` 同接受一个或多个谓词作为参数： ``disjoin`` 当任一谓词返回真时，返回真，而 ``conjoin`` 当所有谓词返回真时，返回真。

::

  > (mapcar (disjoin #'integerp #'symbolp)
            '(a "a" 2 3))
  (T NIL T T)

::

  > (mapcar (conjoin #'integerp #'symbolp)
            '(a "a" 2 3))
  (NIL NIL NIL NIL)

若考虑将谓词定义成集合， ``disjoin`` 返回传入参数的联集（union），而 ``conjoin`` 则是返回传入参数的交集（intersection）。

::

        cddr = (compose #'cdr #'cdr)
        nth  = (compose #'car #'nthcdr)
        atom = (compose #'not #'consp)
             = (rcurry #'typep 'atom)
          <= = (disjoin #'< #'=)
       listp = (disjoin #'< #'=)
             = (rcurry #'typep 'list)
          1+ = (curry #'+ 1)
             = (rcurry #'+ 1)
          1- = (rcurry #'- 1)
      mapcan = (compose (curry #'apply #'nconc) #'mapcar
  complement = (curry #'compose #'not)

**图 6.3 某些等价函数**

函数 ``curry`` 与 ``rcurry`` （“right curry”）精神上与前一小节的 ``make-adder`` 相同。两者皆接受一个函数及某些参数，并返回一个期望剩余参数的新函数。下列任一个函数等同于 ``(make-adder 3)`` :

::

  (curry #'+ 3)
  (rcurry #'+ 3)

当函数的参数顺序重要时，很明显可以看出 ``curry`` 与 ``rcurry`` 的差别。如果我们 ``curry #'-`` ，我们得到一个用其参数减去某特定数的函数，

::

  (funcall (curry #'- 3) 2)
  1

而当我们 ``rcurry #'-`` 时，我们得到一个用某特定数减去其参数的函数:

::

  (funcall (rcurry #'- 3) 2)
  -1

最后， ``always`` 函数是 Common Lisp 函数 ``constantly`` 。接受一个参数并原封不动返回此参数的函数。和 ``identity`` 一样，在很多需要传入函数参数的情况下很有用。

6.7 动态作用域 (Dynamic Sc​​ope)
====================================================

2.11 小节解释过局部与全局变量的差别。实际的差别是词法作用域（lexical scope）的词法变量（lexical variable），与动态作用域（dynamic scope）的特别变量（special variable）的区别。但这俩几乎是没有区别，因为局部变量几乎总是是词法变量，而全局变量总是是特别变量。

在词法作用域下，一个符号引用到上下文中符号名字出现的地方。局部变量缺省有着词法作用域。所以如果我们在一个环境里定义一个函数，其中有一个变量叫做 ``x`` ，

::

  (let ((x 10))
    (defun foo ()
      x))

则无论 ``foo`` 被调用时有存在其它的 ``x`` ，主体内的 ``x`` 都会引用到那个变量:

::

  > (let ((x 20)) (foo))
  10

而动态作用域，我们在环境中函数被调用的地方寻找变量。要使一个变量是动态作用域的，我们需要在任何它出现的上下文中声明它是 ``special`` 。如果我们这样定义 ``foo`` ：

::

  (let ((x 10))
    (defun foo ()
      (declare (special x))
      x))

则函数内的 ``x`` 就不再引用到函数定义里的那个词法变量，但会引用到函数被调用时，当下所存在的任何特别变量 ``x`` :

::

  > (let ((x 20))
      (declare (special x))
      (foo))
  20

新的变量被创建出来之后， 一个 ``declare`` 调用可以在代码的任何地方出现。 ``special`` 声明是独一无二的，因为它可以改变程序的行为。 13 章将讨论其它种类的声明。所有其它的声明，只是给编译器的建议；或许可以使程序运行的更快，但不会改变程序的行为。

通过在顶层调用 ``setf`` 来配置全局变量，是隐式地将变量声明为特殊变量:

::

  > (setf x 30)
  30
  > (foo)
  30

在一个文件里的代码，如果你不想依赖隐式的特殊声明，可以使用 ``defparameter`` 取代，让程序看起来更简洁。

动态作用域什么时候会派上用场呢？通常用来暂时给某个全局变量赋新值。举例来说，有 11 个变量来控制对象印出的方式，包括了 ``*print-base*`` ，缺省是 ``10`` 。如果你想要用 16 进制显示数字，你可以重新绑定 ``*print-base*`` :

::

  > (let ((*print-base* 16))
      (princ 32))
  20
  32

这里显示了两件事情，由 ``princ`` 产生的输出，以及它所返回的值。他们代表着同样的数字，第一次在被印出时，用 16 进制显示，而第二次，因为在 ``let`` 表达式外部，所以是用十进制显示，因为 ``*print-base*`` 回到之前的数值， ``10`` 。

6.8 编译 (Compilation)
========================================

Common Lisp 函数可以独立被编译或挨个文件编译。如果你只是在顶层输入一个 ``defun`` 表达式：

::

  > (defun foo (x) (+ x 1))
  FOO

许多实现会创建一个直译的函数（interpreted function）。你可以将函数传给 ``compiled-function-p`` 来检查一个函数是否有被编译:

::

  > (compiled-function-p #'foo)
  NIL

若你将 ``foo`` 函数名传给 ``compile`` :

::

  > (compile 'foo)
  FOO

则这个函数会被编译，而直译的定义会被编译出来的取代。编译与直译函数的行为一样，只不过对 ``compiled-function-p`` 来说不一样。

你可以把列表作为参数传给 ``compile`` 。这种 ``compile`` 的用法在 161 页 (译注: 10.1 小节)。

有一种函数你不能作为参数传给 ``compile`` ：一个像是 ``stamp`` 或是 ``reset`` 这种，在顶层明确使用词法上下文输入的函数 (即 ``let`` ) [3]_ 在一个文件里面定义这些函数，接着编译然后载入文件是可以的。这么限制直译的代码的是实作的原因，而不是因为在词法上下文里明确定义函数有什么问题。

通常要编译 Lisp 代码不是挨个函数编译，而是使用 ``compile-file`` 编译整个文件。这个函数接受一个文件名，并创建一个原始码的编译版本 ── 通常会有同样的名称，但不同的扩展名。当编译过的文件被载入时， ``compiled-function-p`` 应给所有定义在文件内的函数返回真。

当一个函数包含在另一个函数内时，包含它的函数会被编译，而且内部的函数也会被编译。所以 ``make-adder`` (108 页)被编译时，它会返回编译的函数:

::

  > (compile 'make-adder)
  MAKE-ADDER
  > (compiled-function-p (make-adder 2))
  T

6.9 使用递归 (Using Recursion)
================================================

比起多数别的语言，递归在 Lisp 中扮演了一个重要的角色。这主要有三个原因：

1. 函数式程序设计。递归演算法有副作用的可能性较低。

2. 递归数据结构。 Lisp 隐式地使用了指标，使得递归地定义数据结构变简单了。最常见的是用在列表：一个列表的递归定义，列表为空表，或是一个 ``cons`` ，其中 ``cdr`` 也是个列表。

3. 优雅性。Lisp 程序员非常关心它们的程序是否美丽，而递归演算法通常比迭代演算法来得优雅。

学生们起初会觉得递归很难理解。但 3.9 节指出了，如果你想要知道是否正确，不需要去想递归函数所有的调用过程。

同样的如果你想写一个递归函数。如果你可以描述问题是怎么递归解决的，通常很容易将解法转成代码。要使用递归来解决一个问题，你需要做两件事：

1. 你必须要示范如何解决问题的一般情况，通过将问题切分成有限小并更小的子问题。

2. 你必须要示范如何通过 ── 有限的步骤，来解决最小的问题 ── 基本用例。

如果这两件事完成了，那问题就解决了。因为递归每次都将问题变得更小，而一个有限的问题终究会被解决的，而最小的问题仅需几个有限的步骤就能解决。

举例来说，下面这个找到一个正规列表（proper list）长度的递归算法，我们每次递归时，都可以找到更小列表的长度：

1. 在一般情况下，一个正规列表的长度是它的 ``cdr`` 加一。

2. 基本用例，空列表长度为 ``0`` 。

当这个描述翻译成代码时，先处理基本用例；但公式化递归演算法时，我们通常从一般情况下手。

前述的演算法，明确地描述了一种找到正规列表长度的方法。当你定义一个递归函数时，你必须要确定你在分解问题时，问题实际上越变越小。取得一个正规列表的 ``cdr`` 会给出 ``length`` 更小的子问题，但取得环状列表（circular list）的 ``cdr`` 不会。

这里有两个递归算法的示例。假定参数是有限的。注意第二个示例，我们每次递归时，将问题分成两个更小的问题：

第一个例子， ``member`` 函数，我们说某物是列表的成员，需满足：如果它是第一个元素的成员或是 ``member`` 的 ``cdr`` 的成员。但空列表没有任何成员。

第二个例子， ``copy-tree`` 一个 ``cons`` 的 ``copy-tree`` ，是一个由 ``cons`` 的 ``car`` 的 ``copy-tree`` 与 ``cdr`` 的 ``copy-tree`` 所组成的。一个原子的 ``copy-tree`` 是它自己。

一旦你可以这样描述算法，要写出递归函数只差一步之遥。

某些算法通常是这样表达最自然，而某些算法不是。你可能需要翻回前面，试试不使用递归来定义 ``our-copy-tree`` (41 页，译注: 3.8 小节)。另一方面来说，23 页 (译注: 2.13 节) 迭代版本的 ``show-squares`` 可能更容易比 24 页的递归版本要容易理解。某些时候是很难看出哪个形式比较自然，直到你试着去写出程序来。

如果你关心效率，有两个你需要考虑的议题。第一，尾递归（tail-recursive），会在 13.2 节讨论。一个好的编译器，使用循环或是尾递归的速度，应该是没有或是区别很小的。然而如果你需要使函数变成尾递归的形式时，或许直接用迭代会更好。

另一个需要铭记在心的议题是，最显而易见的递归算法，不一定是最有效的。经典的例子是费氏函数。它是这样递归地被定义的，

  1. Fib(0) = Fib(1) = 1

  2. Fib(n) = Fib(n-1)+Fib(n-2)

直接翻译这个定义，

::

  (defun fib (n)
    (if (<= n 1)
        1
        (+ (fib (- n 1))
           (fib (- n 2)))))

这样是效率极差的。一次又一次的重复计算。如果你要找 ``(fib 10)`` ，这个函数计算 ``(fib 9)`` 与 ``(fib 8)`` 。但要计算出 ``(fib 9)`` ，它需要再次计算 ``(fib 8)`` ，等等。

下面是一个算出同样结果的迭代版本:

::

  (defun fib (n)
    (do ((i n (- i 1))
         (f1 1 (+ f1 f2))
         (f2 1 f1))
        ((<= i 1) f1)))

迭代的版本不如递归版本来得直观，但是效率远远高出许多。这样的事情在实践中常发生吗？非常少 ── 这也是为什么所有的教科书都使用一样的例子 ── 但这是需要注意的事。

Chapter 6 总结 (Summary)
============================

1. 命名函数是一个存在符号的 ``symbol-function`` 部分的函数。 ``defun`` 宏隐藏了这样的细节。它也允许你定义文档字符串（documentation string），并指定 ``setf`` 要怎么处理函数调用。

2. 定义局部函数是有可能的，与定义局部变量有相似的精神。

3. 函数可以有选择性参数（optional）、剩余（rest）以及关键字（keyword）参数。

4. 实用函数是 Lisp 的扩展。他们是由下而上编程的小规模示例。

5. 只要有某物引用到词法变量时，它们会一直存在。闭包是引用到自由变量的函数。你可以写出返回闭包的函数。

6. Dylan 提供了构造函数的函数。很简单就可以使用闭包，然后在 Common Lisp 中实现它们。

7. 特别变量（special variable）有动态作用域 (dynamic scope)。

8. Lisp 函数可以单独编译，或（更常见）编译整个文件。

9. 一个递归演算法通过将问题细分成更小丶更小的子问题来解决问题。

Chapter 6 练习 (Exercises)
==================================

1. 定义一个 ``tokens`` 版本 (67 页)，接受 ``:test`` 与 ``:start`` 参数，缺省分别是 ``#'constituent`` 与 ``0`` 。(译注: 67 页在 4.5 小节)

2. 定义一个 ``bin-search`` (60 页)的版本，接受 ``:key`` , ``:test`` , ``start`` 与 ``end`` 参数，有着一般的意义与缺省值。(译注: 60 页在 4.1 小节)

3. 定义一个函数，接受任何数目的参数，并返回传入的参数。

4. 修改 ``most`` 函数 (105 页)，使其返回 2 个数值，一个列表中最高分的两个元素。(译注: 105 页在 6.4 小节)

5. 用 ``filter`` (105 页) 来定义 ``remove-if`` （没有关键字）。(译注: 105 页在 6.4 小节)

6. 定义一个函数，接受一个参数丶一个数字，并返回目前传入参数中最大的那个。

7. 定义一个函数，接受一个参数丶一个数字，若传入参数比上个参数大时，返回真。函数第一次调用时应返回 ``nil`` 。

8. 假设 ``expensive`` 是一个接受一个参数的函数，一个介于 0 至 100 的整数（包含 100)，返回一个耗时的计算结果。定义一个函数 ``frugal`` 来返回同样的答案，但仅在没见过传入参数时调用 ``expensive`` 。

9. 定义一个像是 ``apply`` 的函数，但在任何数字印出前，缺省用 8 进制印出。


.. rubric:: 脚注

.. [1] 在这个情况下， ``nreverse`` (在 222 页描述)和 ``reverse`` 做一样的事情，但更有效率。

.. [2] “闭包”这个名字是早期的 Lisp 方言流传而来。它是从闭包需要在动态作用域里实现的方式衍生而来。

.. [3] 以前的 ANSI Common Lisp， ``compile`` 的第一个参数也不能是一个已经编译好的函数。
