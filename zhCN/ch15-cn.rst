.. highlight:: cl
   :linenothreshold: 0

Chapter 15 示例：推论 (Example: Inference)
***************************************************

接下来三章提供了大量的 Lisp 程序例子。选择这些例子来说明那些较长的程序所采取的形式，和 Lisp 所擅长解决的问题类型。

在这一章中我们将要写一个基于一组 ``if-then`` 规则的推论程序。这是一个经典的例子--不仅在于其经常出现在教科书上，还因为它反映了 Lisp 作为一个“符号计算”语言的本意。这个例子散发着很多早期 Lisp 程序的气息。

15.1 目标 (The Aim)
==================================

在这个程序中，我们将用一种熟悉的形式来表示信息：包含单个判断式 (predicate) 以及跟在之后的零个或多个参数组成的列表。要表示 Donald 是 Nancy 的家长，我们可以这样写：

::

   (parent donald nancy)

事实上，我们的程序是要表示一些从已有的事实作出推断的规则。我们可以这样来表示规则：

::

   (<- head body)

其中， ``head`` 是 **那么...部分** (then-part)， ``body`` 是 **如果...部分** (if-part)。在 ``head`` 和 ``body`` 中我们使用以问号为前缀的符号来表示变量。所以下面这个规则：

::
   
   (<- (child ?x ?y) (parent ?y ?x))

表示：如果 y 是 x 的家长，那么 x 是 y 的孩子；更恰当地说，我们可以通过证明 ``(parent y x)`` 来证明 ``(child x y)`` 的所表示的事实。

可以把规则中的 *body* 部分(if-part) 写成一个复杂的表达式，其中包含 ``and`` , ``or`` 和 ``not`` 等逻辑操作。所以当我们想要表达 “如果 x 是 y 的家长，并且 x 是男性，那么 x 是 y 的父亲” 这样的规则，我们可以写：

::

   (<- (father ?x ?y) (and (parent ?x ?y) (male ?x)))

一些规则可能依赖另一些规则所产生的事实。比如，我们写的第一个规则是为了证明 ``(child x y)`` 的事实。如果我们定义如下规则：

::

   (<- (daughter ?x ?y) (and (child ?x ?y) (female ?x)))

然后使用它来证明 ``(daughter x y)`` 可能导致程序使用第一个规则去证明 ``(child x y)`` 。

表达式的证明可以回溯任意数量的规则，只要它最终结束于给出的已知事实。这个过程有时候被称为反向链接 (backward-chaining)。之所以说 *反向* (backward) 是因为这一类推论先考虑 *head* 部分，这是为了在继续证明 *body* 部分之前检查规则是否有效。*链接* (chaining) 来源于规则之间的依赖关系，从我们想要证明的内容到我们的已知条件组成一个链接 (尽管事实上它更像一棵树)。

15.2 匹配 (Matching)
==================================================

我们需要有一个函数来做模式匹配以完成我们的反向链接 (back-chaining) 程序，这个函数能够比较两个包含变量的列表，它会检查在给变量赋值后是否可以使两个列表相等。举例，如果 ?x 和 ?y 是变量，那么下面两个列表：

::

   (p ?x ?y c ?x)
   (p  a  b c  a)

当 ?x = a 并且 ?y = b 时匹配，而下面两个列表：

::

   (p ?x b ?y a)
   (p ?y b  c a)

当 ?x = ?y = c 时匹配。

我们有一个 match 函数，它接受两棵树，如果这两棵数能匹配，则返回一个 关联列表 (assoc-list) 来显示他们是如何匹配的：

::

   (defun match (x y &optional binds)
     (cond
      ((eql x y) (values binds t))
      ((assoc x binds) (match (binding x binds) y binds))
      ((assoc y binds) (match x (binding y binds) binds))
      ((var? x) (values (cons (cons x y) binds) t))
      ((var? y) (values (cons (cons y x) binds) t))
      (t
       (when (and (consp x) (consp y))
         (multiple-value-bind (b2 yes)
                              (match (car x) (car y) binds)
           (and yes (match (cdr x) (cdr y) b2)))))))

   (defun var? (x)
     (and (symbolp x)
          (eql (char (symbol-name x) 0) #\?)))

   (defun binding (x binds)
     (let ((b (assoc x binds)))
       (if b
           (or (binding (cdr b) binds)
               (cdr b)))))

::

   > (match '(p a b c a) '(p ?x ?y c ?x))
   ((?Y . B) (?X . A))
   T
   > (match '(p ?x b ?y a) '(p ?y b c a))
   ((?Y . C) (?X . ?Y))
   T
   > (match '(a b c) '(a a a))
   NIL

当 ``match`` 函数逐个元素地比较它的参数时候，它把 ``binds`` 参数中的值分配给变量，这被称为绑定 (bindings)。如果成功匹配， ``match`` 函数返回生成的绑定；否则，返回 ``nil`` 。当然并不是所有成功的匹配都会产生绑定，我们的 ``match`` 函数就像 ``gethash`` 函数那样返回第二个值来表明匹配成功：

::

   > (match '(p ?x) '(p ?x))
   NIL
   T

如果 ``match`` 函数像上面那样返回 ``nil`` 和 ``t`` ，表明这是一个没有产生绑定的成功匹配。下面用中文来描述 ``match`` 算法是如何工作的：

1. 如果 x 和 y 在 ``eql`` 上相等那么它们匹配；否则，
2. 如果 x 是一个已绑定的变量，并且绑定匹配 y ，那么它们匹配；否则，
3. 如果 y 是一个已绑定的变量，并且绑定匹配 x ，那么它们匹配；否则，
4. 如果 x 是一个未绑定的变量，那么它们匹配，并且为 x 建立一个绑定；否则，
5. 如果 y 是一个未绑定的变量，那么它们匹配，并且为 y 建立一个绑定；否则，
6. 如果 x 和 y 都是 ``cons`` ，并且它们的 ``car`` 匹配，由此产生的绑定又让 ``cdr`` 匹配，那么它们匹配。

下面是一个例子，按顺序来说明以上六种情况：

::

   > (match '(p ?v  b ?x  d (?z ?z))
            '(p  a ?w  c ?y ( e  e))
	    '((?v . a) (?w . b)))
   ((?Z . E) (?Y . D) (?X . C) (?V . A) (?W . B))
   T

``match`` 函数通过调用 ``binding`` 函数在一个绑定列表中寻找变量（如果有的话）所关联的值。这个函数必须是递归的，因为有这样的情况 “匹配建立一个绑定列表，而列表中变量只是间接关联到它的值： ``?x`` 可能被绑定到一个包含 ``(?x . ?y)`` 和 ``(?y . a)`` 的列表”：

::

   > (match '(?x a) '(?y ?y))
   ((?Y . A) (?X . ?Y))
   T

先匹配 ``?x`` 和 ``?y`` ，然后匹配 ``?y`` 和 ``a`` ，我们间接确定 ``?x`` 是 ``a`` 。

15.3 回答查询 (Answering Queries)
================================

在介绍了绑定的概念之后，我们可以更准确的说一下我们的程序将要做什么：它得到一个可能包含变量的表达式，根据我们给定的事实和规则返回使它正确的所有绑定。比如，我们只有下面这个事实：

::

   (parent donald nancy)

然后我们想让程序证明：

::

   (parent ?x ?y)

它会返回像下面这样的表达：

::

   (((?x . donald) (?y . nancy)))

它告诉我们只有一个可以让这个表达式为真的方法： ``?x`` 是 ``donald`` 并且 ``?y`` 是 ``nancy`` 。

在通往目标的路上，我们已经有了一个的重要部分：一个匹配函数。
下面是用来定义规则的一段代码：

::

   (defvar *rules* (make-hash-table))

   (defmacro <- (con &optional ant)
     `(length (push (cons (cdr ',con) ',ant)
                    (gethash (car ',con) *rules*))))

规则将被包含于一个叫做 ``*rules*`` 的哈希表，通过头部 (head) 的判断式构建这个哈系表。这样做加强了我们无法使用判断式中的变量的限制。虽然我们可以通过把所有这样的规则放在分离的列表中来消除限制，但是如果这样做，当我们需要证明某件事的时侯不得不和每一个列表进行匹配。

我们将要使用同一个宏 ``<-`` 去定义事实 (facts)和规则 (rules)。一个事实将被表示成一个没有 *body* 部分的规则。这和我们对规则的定义保持一致。一个规则告诉我们你可以通过证明 *body* 部分来证明 *head* 部分，所以没有 *body* 部分的规则意味着你不需要通过证明任何东西来证明 *head* 部分。这里有两个对应的例子：

::

   > (<- (parent donald nancy))
   1
   > (<- (child ?x ?y) (parent ?y ?x))
   1

调用 ``<-`` 返回的是给定判断式下存储的规则数量；用 ``length`` 函数来包装 ``push`` 能使我们免于看到顶层 (toplevel)中的一大堆返回值。

下面是我们的推论程序所需的大多数代码：

::

   (defun prove (expr &optional binds)
     (case (car expr)
       (and (prove-and (reverse (cdr expr)) binds))
       (or  (prove-or (cdr expr) binds))
       (not (prove-not (cadr expr) binds))
       (t   (prove-simple (car expr) (cdr expr) binds))))

   (defun prove-simple (pred args binds)
     (mapcan #'(lambda (r)
                 (multiple-value-bind (b2 yes)
                                      (match args (car r)
                                             binds)
                   (when yes
                     (if (cdr r)
                         (prove (cdr r) b2)
                         (list b2)))))
             (mapcar #'change-vars
                     (gethash pred *rules*))))

   (defun change-vars (r)
     (sublis (mapcar #'(lambda (v) (cons v (gensym "?")))
                     (vars-in r))
             r))

   (defun vars-in (expr)
     (if (atom expr)
         (if (var? expr) (list expr))
	 (union (vars-in (car expr))
                (vars-in (cdr expr)))))

上面代码中的 ``prove`` 函数是推论进行的枢纽。它接受一个表达式和一个可选的绑定列表作为参数。如果表达式不包含逻辑操作，它调用 ``prove-simple`` 函数，前面所说的链接 (chaining)正是在这个函数里产生的。这个函数查看所有拥有正确判断式的规则，并尝试对每一个规则的 *head* 部分和它想要证明的事实做匹配。对于每一个匹配的 *head* ，使用匹配所产生的新的绑定在 *body* 上调用 ``prove`` 。对 ``prove`` 的调用所产生的绑定列表被 ``mapcan`` 收集并返回：

::

   > (prove-simple 'parent '(donald nancy) nil)
   (NIL)
   > (prove-simple 'child '(?x ?y) nil)
   (((#:?6 . NANCY) (#:?5 . DONALD) (?Y . #:?5) (?X . #:?6)))

以上两个返回值指出有一种方法可以证明我们的问题。（一个失败的证明将返回 nil。）第一个例子产生了一组空的绑定，第二个例子产生了这样的绑定： ``?x`` 和 ``?y`` 被（间接）绑定到 ``nancy`` 和 ``donald`` 。

顺便说一句，这是一个很好的例子来实践 2.13 节提出的观点。因为我们用函数式的风格来写这个程序，所以可以交互式地测试每一个函数。

第二个例子返回的值里那些 *gensyms* 是怎么回事？如果我们打算使用含有变量的规则，我们需要避免两个规则恰好包含相同的变量。如果我们定义如下两条规则：

::

   (<- (child ?x ?y) (parent ?y ?x))

   (<- (daughter ?y ?x) (and (child ?y ?x) (female ?y)))

第一条规则要表达的意思是：对于任何的 x 和 y， 如果 y 是 x 的家长，则 x 是 y 的孩子。第二条则是：对于任何的 x 和 y， 如果 y 是 x 的孩子并且 y 是女性，则 y 是 x 的女儿。在每一条规则内部，变量之间的关系是显著的，但是两条规则使用了相同的变量并非我们刻意为之。

如果我们使用上面所写的规则，它们将不会按预期的方式工作。如果我们尝试证明“ a 是 b 的女儿”，匹配到第二条规则的 *head* 部分时会将 a 绑定到 ?y ，将 b 绑定到 ?x。我们无法用这样的绑定匹配第一条规则的 *head* 部分：

::

   > (match '(child ?y ?x)
            '(child ?x ?y)
	    '((?y . a) (?x . b)))
   NIL

为了保证一条规则中的变量只表示规则中各参数之间的关系，我们用 *gensyms* 来代替规则中的所有变量。这就是 ``change-vars`` 函数的目的。一个 *gensym* 不可能在另一个规则中作为变量出现。但是因为规则可以是递归的，我们必须防止出现一个规则和自身冲突的可能性，所以在定义和使用一个规则时都要调用 ``chabge-vars`` 函数。

现在只剩下定义用以证明复杂表达式的函数了。下面就是需要的函数：

::

   (defun prove-and (clauses binds)
     (if (null clauses)
         (list binds)
	 (mapcan #'(lambda (b)
                     (prove (car clauses) b))
                 (prove-and (cdr clauses) binds))))

   (defun prove-or (clauses binds)
     (mapcan #'(lambda (c) (prove c binds))
             clauses))

   (defun prove-not (clause binds)
     (unless (prove clause binds)
       (list binds)))

操作一个 ``or`` 或者 ``not`` 表达式是非常简单的。操作 ``or`` 时，我们提取在 ``or`` 之间的每一个表达式返回的绑定。操作 ``not`` 时，当且仅当在 ``not`` 里的表达式产生 ``none`` 时，返回当前的绑定。

``prove-and`` 函数稍微复杂一点。它像一个过滤器(filter)，它用之后的表达式所建立的每一个绑定来证明第一个表达式。这将导致 ``and`` 里的表达式以相反的顺序被求值。除非调用 ``prove`` 中的 ``prove-and`` 函数则会先逆转它们。

现在我们有了一个可以工作的程序，但它不是很友好。必须要解析 ``prove-and`` 返回的绑定列表是令人厌烦的，它们会变得更长随着规则变得更加复杂。下面有一个宏来帮助我们更愉快地使用这个程序：

::

   (defmacro with-answer (query &body body)
     (let ((binds (gensym)))
       `(dolist (,binds (prove ',query))
          (let ,(mapcar #'(lambda (v)
                            `(,v (binding ',v ,binds)))
                        (vars-in query))
            ,@body))))

它接受一个 ``query`` （不被求值）和若干表达式构成的 ``body`` 作为参数，把 ``query`` 所生成的每一组绑定的值赋给 ``query`` 中对应的模式变量，并计算 ``body`` 。

::

   > (with-answer (parent ?x ?y)
       (format t "~A is the parent of ~A.~%" ?x ?y))
   DONALD is the parent of NANCY.
   NIL

这个宏帮我们做了解析绑定的工作，同时为我们在程序中使用 ``prove`` 提供了一个便捷的方法。下面是这个宏展开的情况：

::

   (with-answer (p ?x ?y)
     (f ?x ?y))

   ;;将被展开成下面的代码

   (dolist (#:g1 (prove '(p ?x ?y)))
     (let ((?x (binding '?x #:g1))
           (?y (binding '?y #:g1)))
       (f ?x ?y)))

下面是使用它的一个例子：

::

   (<- (parent donald nancy))
   (<- (parent donald debbie))
   (<- (male donald))
   (<- (father ?x ?y) (and (parent ?x ?y) (male ?x)))
   (<- (= ?x ?y))
   (<- (sibling ?x ?y) (and (parent ?z ?x)
                            (parent ?z ?y)
			    (not (= ?x ?y))))

   ;;我们可以像下面这样做出推论

   > (with-answer (father ?x ?y)
       (format t "~A is the father of ~A.~%" ?x ?y))
   DONALD is the father of DEBBIE.
   DONALD is the father of NANCY.
   NIL
   > (with-answer (sibling ?x ?y))
       (format t "~A is the sibling of ~A.~%" ?x ?y))
   DEBBLE is the sibling of NANCY.
   NANCY is the  sibling of DEBBIE.
   NIL

15.4 分析 (Analysis)
===================================================

看上去，我们在这一章中写的代码是用简单自然的方式去实现这样一个程序。事实上，它非常低效。我们在这里是其实是做了一个解释器。我们能够把这个程序做得像一个编译器。

这里做一个简单的描述。基本的思想是把整个程序打包到两个宏 ``<-`` 和 ``with-answer`` ，把已有程序中在 *运行时* 做的多数工作搬到 *宏展开时* （在 10.7 节的 ``avg`` 可以看到这种构思的雏形) 用函数取代列表来表示规则，我们不在运行时用 ``prove`` 和 ``prove-and`` 这样的函数来解释表达式，而是用相应的函数把表达式转化成代码。当一个规则被定义的时候就有表达式可用。为什么要等到使用的时候才去分析它呢？这同样适用于和 ``<-`` 调用了相同的函数来进行宏展开的 ``with-answer`` 。

听上去好像比我们已经写的这个程序复杂很多，但其实可能只是长了两三倍。想要学习这种技术的读者可以看 *On Lisp* 或者 *Paradigms of Artificial Intelligence Programming* ，其中包含了一些使用这种风格写的程序例子。
