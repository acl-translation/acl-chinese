.. highlight:: cl

第十七章：示例：对象
**************************************************

在本章里，我们将使用 Lisp 来自己实现面向对象语言。这样子的程序称为嵌入式语言 (\ *embedded language*\ )。嵌入一个面向对象语言到 Lisp 里是一个绝佳的例子。同時作为一个 Lisp 的典型用途，並演示了面向对象的抽象是如何多自然地在 Lisp 基本的抽象上构建出来。

17.1 继承 (Inheritance)
==================================

11.10 小节解释过通用函数与消息传递的差别。

在消息传递模型里，

1. 对象有属性，

2. 并回应消息，

3. 并从其父类继承属性与方法。

当然了，我们知道 CLOS 使用的是通用函数模型。但本章我们只对于写一个迷你的对象系统 (minimal object system)感兴趣，而不是一个可与 CLOS 匹敌的系统，所以我们将使用消息传递模型。

我们已经在 Lisp 里看过许多保存属性集合的方法。一种可能的方法是使用哈希表来代表对象，并将属性作为哈希表的条目保存。接著可以通过 ``gethash`` 来存取每个属性：

::

	(gethash 'color obj)

由于函数是数据对象，我们也可以将函数作为属性保存起来。这表示我们也可以有方法；要调用一个对象特定的方法，可以通过 ``funcall`` 一下哈希表里的同名属性：

::

	(funcall (gethash 'move obj) obj 10)

我们可以在这个概念上，定义一个 Smalltalk 风格的消息传递语法，

::

	(defun tell (obj message &rest args)
	  (apply (gethash message obj) obj args))

所以想要一个对象 ``obj`` 移动 10 单位，我们可以说：

::

	(tell obj 'move 10)

事实上，纯 Lisp 唯一缺少的原料是继承。我们可以通过定义一个递归版本的 ``gethash`` 来实现一个简单版，如图 17.1 。现在仅用共 8 行代码，便实现了面向对象编程的 3 个基本元素。

::

	(defun rget (prop obj)
	  (multiple-value-bind (val in) (gethash prop obj)
	    (if in
	        (values val in)
	        (let ((par (gethash :parent obj)))
	          (and par (rget prop par))))))

	(defun tell (obj message &rest args)
	  (apply (rget message obj) obj args))

**图 17.1：继承**

让我们用这段代码，来试试本来的例子。我们创建两个对象，其中一个对象是另一个的子类：

::

	> (setf circle-class (make-hash-table)
	        our-circle   (make-hash-table)
	        (gethash :parent our-circle) circle-class
	        (gethash 'radius our-circle) 2)
	2

``circle-class`` 对象会持有给所有圆形使用的 ``area`` 方法。它是接受一个参数的函数，该参数为传来原始消息的对象：

::

	> (setf (gethash 'area circle-class)
	        #'(lambda (x)
	            (* pi (expt (rget 'radius x) 2))))
	#<Interpreted-Function BF1EF6>

现在当我们询问 ``our-circle`` 的面积时，会根据此类所定义的方法来计算。我们使用 ``rget`` 来读取一个属性，用 ``tell`` 来调用一个方法：

::

	> (rget 'radius our-circle)
	2
	T
	> (tell our-circle 'area)
	12.566370614359173

在开始改善这个程序之前，值得停下来想想我们到底做了什么。仅使用 8 行代码，我们使纯的、旧的、无 CLOS 的 Lisp ，转变成一个面向对象语言。我们是怎么完成这项壮举的？应该用了某种秘诀，才会仅用了 8 行代码，就实现了面向对象编程。

的确有一个秘诀存在，但不是编程的奇技淫巧。这个秘诀是，Lisp 本来就是一个面向对象的语言了，甚至说，是种更通用的语言。我们需要做的事情，不过就是把本来就存在的抽象，再重新包装一下。

17.2 多重继承 (Multiple Inheritance)
==================================================

到目前为止我们只有单继承 ── 一个对象只可以有一个父类。但可以通过使 ``parent`` 属性变成一个列表来获得多重继承，并重新定义 ``rget`` ，如图 17.2 所示。

在只有单继承的情况下，当我们想要从对象取出某些属性，只需要递归地延著祖先的方向往上找。如果对象本身没有我们想要属性的有关信息，可以检视其父类，以此类推。有了多重继承后，我们仍想要执行同样的搜索，但这件简单的事，却被对象的祖先可形成一个图，而不再是简单的树给复杂化了。不能只使用深度优先来搜索这个图。有多个父类时，可以有如图 17.3 所示的层级存在： ``a`` 起源于 ``b`` 及 ``c`` ，而他们都是 ``d`` 的子孙。一个深度优先（或说高度优先）的遍历结果会是 ``a`` , ``b`` , ``d``, ``c`` , ``d`` 。而如果我们想要的属性在 ``d`` 与 ``c`` 都有的话，我们会获得存在 ``d`` 的值，而不是存在 ``c`` 的值。这违反了子类可覆写父类提供缺省值的原则。

如果我们想要实现普遍的继承概念，就不应该在检查其子孙前，先检查该对象。在这个情况下，适当的搜索顺序会是 ``a`` , ``b`` , ``c`` , ``d`` 。那如何保证搜索总是先搜子孙呢？最简单的方法是用一个对象，以及按正确优先顺序排序的，由祖先所构成的列表。通过调用 ``traverse`` 开始，建构一个列表，表示深度优先遍历所遇到的对象。如果任一个对象有共享的父类，则列表中会有重复元素。如果仅保存最后出现的复本，会获得一般由 CLOS 定义的优先级列表。（删除所有除了最后一个之外的复本，根据 183 页所描述的算法，规则三。）Common Lisp 函数 ``delete-duplicates`` 定义成如此作用的，所以我们只要在深度优先的基础上调用它，我们就会得到正确的优先级列表。一旦优先级列表创建完成， ``rget`` 根据需要的属性搜索第一个符合的对象。

我们可以通过利用优先级列表的优点，举例来说，一个爱国的无赖先是一个无赖，然后才是爱国者：

::

	> (setf scoundrel           (make-hash-table)
	        patriot             (make-hash-table)
	        patriotic-scoundrel (make-hash-table)
	        (gethash 'serves scoundrel) 'self
	        (gethash 'serves patriot) 'country
	        (gethash :parents patriotic-scoundrel)
	                 (list scoundrel patriot))
	(#<Hash-Table C41C7E> #<Hash-Table C41F0E>)
	> (rget 'serves patriotic-scoundrel)
	SELF
	T

到目前为止，我们有一个强大的程序，但极其丑陋且低效。在一个 Lisp 程序生命周期的第二阶段，我们将这个初步框架提炼成有用的东西。

17.3 定义对象 (Defining Objects)
================================

第一个我们需要改善的是，写一个用来创建对象的函数。我们程序表示对象以及其父类的方式，不需要给用户知道。如果我们定义一个函数来创建对象，用户将能够一个步骤就创建出一个对象，并指定其父类。我们可以在创建一个对象的同时，顺道构造优先级列表，而不是在每次当我们需要找一个属性或方法时，才花费庞大代价来重新构造。

如果我们要维护优先级列表，而不是在要用的时候再构造它们，我们需要处理列表会过时的可能性。我们的策略会是用一个列表来保存所有存在的对象，而无论何时当某些父类被改动时，重新给所有受影响的对象生成优先级列表。这代价是相当昂贵的，但由于查询比重定义父类的可能性来得高许多，我们会省下许多时间。这个改变对我们的程序的灵活性没有任何影响；我们只是将花费从频繁的操作转到不频繁的操作。

图 17.4 包含了新的代码。 `λ <http://acl.readthedocs.org/en/latest/zhCN/notes-cn.html#notes-273>`_ 全局的 ``*objs*`` 会是一个包含所有当前对象的列表。函数 ``parents`` 取出一个对象的父类；相反的 ``(setf parents)`` 不仅配置一个对象的父类，也调用 ``make-precedence`` 来重新构造任何需要变动的优先级列表。这些列表与之前一样，由 ``precedence`` 来构造。

用户现在不用调用 ``make-hash-table`` 来创建对象，调用 ``obj`` 来取代， ``obj`` 一步完成创建一个新对象及定义其父类。我们也重定义了 ``rget`` 来利用保存优先级列表的好处。


::

	(defvar *objs* nil)

	(defun parents (obj) (gethash :parents obj))

	(defun (setf parents) (val obj)
	  (prog1 (setf (gethash :parents obj) val)
	         (make-precedence obj)))

	(defun make-precedence (obj)
	  (setf (gethash :preclist obj) (precedence obj))
	  (dolist (x *objs*)
	    (if (member obj (gethash :preclist x))
	        (setf (gethash :preclist x) (precedence x)))))

	(defun obj (&rest parents)
	  (let ((obj (make-hash-table)))
	    (push obj *objs*)
	    (setf (parents obj) parents)
	    obj))

	(defun rget (prop obj)
	  (dolist (c (gethash :preclist obj))
	    (multiple-value-bind (val in) (gethash prop c)
	      (if in (return (values val in))))))

**图 17.4：创建对象**

17.4 函数式语法 (Functional Syntax)
===================================================

另一个可以改善的空间是消息调用的语法。 ``tell`` 本身是无谓的杂乱不堪，这也使得动词在第三顺位才出现，同时代表著我们的程序不再可以像一般 Lisp 前序表达式那样阅读:

::

	(tell (tell obj 'find-owner) 'find-owner)

我们可以使用图 17.5 所定义的 ``defprop`` 宏，通过定义作为函数的属性名称来摆脱这种 ``tell`` 语法。若选择性参数 ``meth?`` 为真的话，会将此属性视为方法。不然会将属性视为槽，而由 ``rget`` 所取回的值会直接返回。一旦我们定义了属性作为槽或方法的名字，

::

	(defmacro defprop (name &optional meth?)
	  `(progn
	     (defun ,name (obj &rest args)
	       ,(if meth?
	          `(run-methods obj ',name args)
	          `(rget ',name obj)))
	     (defun (setf ,name) (val obj)
	       (setf (gethash ',name obj) val))))

	(defun run-methods (obj name args)
	  (let ((meth (rget name obj)))
	    (if meth
	        (apply meth obj args)
	        (error "No ~A method for ~A." name obj))))

**图 17.5: 函数式语法**

::

	(defprop find-owner t)

我们就可以在函数调用里引用它，则我们的代码读起来将会再次回到 Lisp 本来那样：

::

	(find-owner (find-owner obj))

我们的前一个例子在某种程度上可读性变得更高了：

::

	> (progn
	    (setf scoundrel           (obj)
	          patriot             (obj)
	          patriotic-scoundrel (obj scoundrel patriot))
	    (defprop serves)
	    (setf (serves scoundrel) 'self
	          (serves patriot) 'country)
	    (serves patriotic-scoundrel))
	SELF
	T

17.5 定义方法 (Defining Methods)
=======================================

到目前为止，我们借由叙述如下的东西来定义一个方法：

::

	(defprop area t)

	(setf circle-class (obj))

	(setf (area circle-class)
	      #'(lambda (c) (* pi (expt (radius c) 2))))

::

	(defmacro defmeth (name obj parms &rest body)
	  (let ((gobj (gensym)))
	    `(let ((,gobj ,obj))
	       (setf (gethash ',name ,gobj)
	             (labels ((next () (get-next ,gobj ',name)))
	               #'(lambda ,parms ,@body))))))

	(defun get-next (obj name)
	  (some #'(lambda (x) (gethash name x))
	        (cdr (gethash :preclist obj))))

**图 17.6 定义方法。**

在一个方法里，我们可以通过给对象的 ``:preclist`` 	的 ``cdr``  获得如内置 ``call-next-method`` 方法的效果。所以举例来说，若我们想要定义一个特殊的圆形，这个圆形在返回面积的过程中印出某个东西，我们可以说：

::

	(setf grumpt-circle (obj circle-class))

	(setf (area grumpt-circle)
	      #'(lambda (c)
	          (format t "How dare you stereotype me!~%")
	          (funcall (some #'(lambda (x) (gethash 'area x))
	                         (cdr (gethash :preclist c)))
	                   c)))

这里 ``funcall`` 等同于一个 ``call-next-method`` 调用，但他..

图 17.6 的 ``defmeth`` 宏提供了一个便捷方式来定义方法，并使得调用下个方法变得简单。一个 ``defmeth`` 的调用会展开成一个 ``setf`` 表达式，但 ``setf`` 在一個 ``labels`` 表达式里定义了 ``next`` 作为取出下个方法的函数。这个函数与 ``next-method-p`` 类似（第 188 页「譯註: 11.7 節」），但返回的是我们可以调用的东西，同時作為 ``call-next-method`` 。 `λ <http://acl.readthedocs.org/en/latest/zhCN/notes-cn.html#notes-273>`_ 前述两个方法可以被定义成：

::

	(defmeth area circle-class (c)
	  (* pi (expt (radius c) 2)))

	(defmeth area grumpy-circle (c)
	  (format t "How dare you stereotype me!~%")
	  (funcall (next) c))

顺道一提，注意 ``defmeth`` 的定义也利用到了符号捕捉。方法的主体被插入至函数 ``next`` 是局部定义的一个上下文里。


17.6 实例 (Instances)
=======================================

到目前为止，我们还没有将类别与实例做区别。我们使用了一个术语来表示两者，\ *对象*\ (\ *object*\ )。将所有的对象视为一体是优雅且灵活的，但这非常没效率。在许多面向对象应用里，继承图的底部会是复杂的。举例来说，模拟一个交通情况，我们可能有少于十个对象来表示车子的种类，但会有上百个对象来表示特定的车子。由于后者会全部共享少数的优先级列表，创建它们是浪费时间的，并且浪费空间来保存它们。

图 17.7 定义一个宏 ``inst`` ，用来创建实例。实例就像其他对象一样（现在也可称为类别），有区别的是只有一个父类且不需维护优先级列表。它们也没有包含在列表 ``*objs**`` 里。在前述例子里，我们可以说：

::

	(setf grumpy-circle (inst circle-class))

由于某些对象不再有优先级列表，函数 ``rget`` 以及 ``get-next`` 现在被重新定义，检查这些对象的父类来取代。获得的效率不用拿灵活性交换。我们可以对一个实例做任何我们可以给其它种对象做的事，包括创建一个实例以及重定义其父类。在后面的情况里， ``(setf parents)`` 会有效地将对象转换成一个“类别”。

17.7 新的实现 (New Implementation)
==================================================

我们到目前为止所做的改善都是牺牲灵活性交换而来。在这个系统的开发后期，一个 Lisp 程序通常可以牺牲些许灵活性来获得好处，这里也不例外。目前为止我们使用哈希表来表示所有的对象。这给我们带来了超乎我们所需的灵活性，以及超乎我们所想的花费。在这个小节里，我们会重写我们的程序，用简单向量来表示对象。

::

	(defun inst (parent)
	  (let ((obj (make-hash-table)))
	    (setf (gethash :parents obj) parent)
	    obj))

	(defun rget (prop obj)
	  (let ((prec (gethash :preclist obj)))
	    (if prec
	        (dolist (c prec)
	          (multiple-value-bind (val in) (gethash prop c)
	            (if in (return (values val in)))))
	      (multiple-value-bind (val in) (gethash prop obj)
	        (if in
	            (values val in)
	            (rget prop (gethash :parents obj)))))))

	(defun get-next (obj name)
	  (let ((prec (gethash :preclist obj)))
	    (if prec
	        (some #'(lambda (x) (gethash name x))
	              (cdr prec))
	      (get-next (gethash obj :parents) name))))

**图 17.7: 定义实例**

这个改变意味著放弃动态定义新属性的可能性。目前我们可通过引用任何对象，给它定义一个属性。现在当一个类别被创建时，我们会需要给出一个列表，列出该类有的新属性，而当实例被创建时，他们会恰好有他们所继承的属性。

在先前的实现里，类别与实例没有实际区别。一个实例只是一个恰好有一个父类的类别。如果我们改动一个实例的父类，它就变成了一个类别。在新的实现里，类别与实例有实际区别；它使得将实例转成类别不再可能。

在图 17.8-17.10 的代码是一个完整的新实现。图片 17.8 给创建类别与实例定义了新的操作符。类别与实例用向量来表示。表示类别与实例的向量的前三个元素包含程序自身要用到的信息，而图 17.8 的前三个宏是用来引用这些元素的：

::

	(defmacro parents (v) `(svref ,v 0))
	(defmacro layout (v) `(the simple-vector (svref ,v 1)))
	(defmacro preclist (v) `(svref ,v 2))

	(defmacro class (&optional parents &rest props)
	  `(class-fn (list ,@parents) ',props))

	(defun class-fn (parents props)
	  (let* ((all (union (inherit-props parents) props))
	         (obj (make-array (+ (length all) 3)
	                          :initial-element :nil)))
	    (setf (parents obj)  parents
	          (layout obj)   (coerce all 'simple-vector)
	          (preclist obj) (precedence obj))
	    obj))

	(defun inherit-props (classes)
	  (delete-duplicates
	    (mapcan #'(lambda (c)
	                (nconc (coerce (layout c) 'list)
	                       (inherit-props (parents c))))
	            classes)))

	(defun precedence (obj)
	  (labels ((traverse (x)
	             (cons x
	                   (mapcan #'traverse (parents x)))))
	    (delete-duplicates (traverse obj))))

	(defun inst (parent)
	  (let ((obj (copy-seq parent)))
	    (setf (parents obj)  parent
	          (preclist obj) nil)
	    (fill obj :nil :start 3)
	    obj))

**图 17.8: 向量实现：创建**

1. ``parents`` 字段取代旧实现中，哈希表条目里 ``:parents`` 的位置。在一个类别里， ``parents`` 会是一个列出父类的列表。在一个实例里， ``parents`` 会是一个单一的父类。

2. ``layout`` 字段是一个包含属性名字的向量，指出类别或实例的从第四个元素开始的设计 (layout)。

3. ``preclist`` 字段取代旧实现中，哈希表条目里 ``:preclist`` 的位置。它会是一个类别的优先级列表，实例的话就是一个空表。

因为这些操作符是宏，他们全都可以被 ``setf`` 的第一个参数使用（参考 10.6 节）。

``class`` 宏用来创建类别。它接受一个含有其基类的选择性列表，伴随著零个或多个属性名称。它返回一个代表类别的对象。新的类别会同时有自己本身的属性名，以及从所有基类继承而来的属性。

::

	> (setf *print-array* nil
	        gemo-class (class nil area)
	        circle-class (class (geom-class) radius))
	#<Simple-Vector T 5 C6205E>

这里我们创建了两个类别： ``geom-class`` 没有基类，且只有一个属性， ``area`` ； ``circle-class`` 是 ``gemo-class`` 的子类，并添加了一个属性， ``radius`` 。 [1]_ ``circle-class`` 类的设计

::

	> (coerce (layout circle-class) 'list)
	(AREA RADIUS)

显示了五个字段里，最后两个的名称。 [2]_

``class`` 宏只是一个 ``class-fn`` 的介面，而 ``class-fn`` 做了实际的工作。它调用 ``inherit-props`` 来汇整所有新对象的父类，汇整成一个列表，创建一个正确长度的向量，并适当地配置前三个字段。（ ``preclist`` 由 ``precedence`` 创建，本质上 ``precedence`` 没什么改变。）类别馀下的字段设置为 ``:nil`` 来指出它们尚未初始化。要检视 ``circle-class`` 的 ``area`` 属性，我们可以：

::

	> (svref circle-class
	         (+ (position 'area (layout circle-class)) 3))
	:NIL

稍后我们会定义存取函数来自动办到这件事。

最后，函数 ``inst`` 用来创建实例。它不需要是一个宏，因为它仅接受一个参数：

::

	> (setf our-circle (inst circle-class))
	#<Simple-Vector T 5 C6464E>

比较 ``inst`` 与 ``class-fn`` 是有益学习的，它们做了差不多的事。因为实例仅有一个父类，不需要决定它继承什么属性。实例可以仅拷贝其父类的设计。它也不需要构造一个优先级列表，因为实例没有优先级列表。创建实例因此与创建类别比起来来得快许多，因为创建实例在多数应用里比创建类别更常见。

::

	(declaim (inline lookup (setf lookup)))

	(defun rget (prop obj next?)
	  (let ((prec (preclist obj)))
	    (if prec
	        (dolist (c (if next? (cdr prec) prec) :nil)
	          (let ((val (lookup prop c)))
	            (unless (eq val :nil) (return val))))
	        (let ((val (lookup prop obj)))
	          (if (eq val :nil)
	              (rget prop (parents obj) nil)
	              val)))))

	(defun lookup (prop obj)
	  (let ((off (position prop (layout obj) :test #'eq)))
	    (if off (svref obj (+ off 3)) :nil)))

	(defun (setf lookup) (val prop obj)
	  (let ((off (position prop (layout obj) :test #'eq)))
	    (if off
	        (setf (svref obj (+ off 3)) val)
	        (error "Can't set ~A of ~A." val obj))))

**图 17.9: 向量实现：存取**

现在我们可以创建所需的类别层级及实例，以及需要的函数来读写它们的属性。图 17.9 的第一个函数是 ``rget`` 的新定义。它的形状与图 17.7 的 ``rget`` 相似。条件式的两个分支，分别处理类别与实例。

1. 若对象是一个类别，我们遍历其优先级列表，直到我们找到一个对象，其中欲找的属性不是 ``:nil`` 。如果没有找到，返回 ``:nil`` 。

2. 若对象是一个实例，我们直接查找属性，并在没找到时递回地调用 ``rget`` 。

``rget`` 与 ``next?`` 新的第三个参数稍后解释。现在只要了解如果是 ``nil`` ， ``rget`` 会像平常那样工作。

函数 ``lookup`` 及其反相扮演著先前 ``rget`` 函数里 ``gethash`` 的角色。它们使用一个对象的 ``layout`` ，来取出或设置一个给定名称的属性。这条查询是先前的一个复本：

::

	> (lookup 'area circle-class)
	:NIL

由于 ``lookup`` 的 ``setf`` 也定义了，我们可以给 ``circle-class`` 定义一个 ``area`` 方法，通过：

::

	(setf (lookup 'area circle-class)
	      #'(lambda (c)
	          (* pi (expt (rget 'radius c nil) 2))))

在这个程序里，和先前的版本一样，没有特别区别出方法与槽。一个“方法”只是一个字段，里面有著一个函数。这将很快会被一个更方便的前端所隐藏起来。

::

	(declaim (inline run-methods))

	(defmacro defprop (name &optional meth?)
	  `(progn
	     (defun ,name (obj &rest args)
	       ,(if meth?
	            `(run-methods obj ',name args)
	            `(rget ',name obj nil)))
	     (defun (setf ,name) (val obj)
	       (setf (lookup ',name obj) val))))

	(defun run-methods (obj name args)
	  (let ((meth (rget name obj nil)))
	    (if (not (eq meth :nil))
	        (apply meth obj args)
	        (error "No ~A method for ~A." name obj))))

	(defmacro defmeth (name obj parms &rest body)
	  (let ((gobj (gensym)))
	    `(let ((,gobj ,obj))
	       (defprop ,name t)
	       (setf (lookup ',name ,gobj)
	             (labels ((next () (rget ,gobj ',name t)))
	               #'(lambda ,parms ,@body))))))

**图 17.10: 向量实现：宏介面**

图 17.10 包含了新的实现的最后部分。这个代码没有给程序加入任何威力，但使程序更容易使用。宏 ``defprop`` 本质上没有改变；现在仅调用 ``lookup`` 而不是 ``gethash`` 。与先前相同，它允许我们用函数式的语法来引用属性：

::

	> (defprop radius)
	(SETF RADIUS)
	> (radius our-circle)
	:NIL
	> (setf (radius our-circle) 2)
	2

如果 ``defprop`` 的第二个选择性参数为真的话，它展开成一个 ``run-methods`` 调用，基本上也没什么改变。

最后，函数 ``defmeth`` 提供了一个便捷方式来定义方法。这个版本有三件新的事情：它隐含了 ``defprop`` ，它调用 ``lookup`` 而不是 ``gethash`` ，且它调用 ``regt`` 而不是 278 页的 ``get-next`` (译注: 图 17.7 的 ``get-next`` )来获得下个方法。现在我们理解给 ``rget`` 添加额外参数的理由。它与 ``get-next`` 非常相似，我们同样通过添加一个额外参数，在一个函数里实现。若这额外参数为真时， ``rget`` 取代 ``get-next`` 的位置。

现在我们可以达到先前方法定义所有的效果，但更加清晰：

::

	(defmeth area circle-class (c)
	  (* pi (expt (radius c) 2)))

注意我们可以直接调用 ``radius`` 而无须调用 ``rget`` ，因为我们使用 ``defprop`` 将它定义成一个函数。因为隐含的 ``defprop`` 由 ``defmeth`` 实现，我们也可以调用 ``area`` 来获得 ``our-circle`` 的面积：

::

	> (area our-circle)
	12.566370614359173

17.8 分析 (Analysis)
=======================================

我们现在有了一个适合撰写实际面向对象程序的嵌入式语言。它很简单，但就大小来说相当强大。而在典型应用里，它也会是快速的。在一个典型的应用里，操作实例应比操作类别更常见。我们重新设计的重点在于如何使得操作实例的花费降低。

在我们的程序里，创建类别既慢且产生了许多垃圾。如果类别不是在速度为关键考量时创建，这还是可以接受的。会需要速度的是存取函数以及创建实例。这个程序里的没有做编译优化的存取函数大约与我们预期的一样快。 `λ <http://acl.readthedocs.org/en/latest/zhCN/notes-cn.html#notes-284>`_ 而创建实例也是如此。且两个操作都没有用到构造 (consing)。除了用来表达实例的向量例外。会自然的以为这应该是动态地配置才对。但我们甚至可以避免动态配置实例，如果我们使用像是 13.4 节所提出的策略。

我们的嵌入式语言是 Lisp 编程的一个典型例子。只不过是一个嵌入式语言就可以是一个例子了。但 Lisp 的特性是它如何从一个小的、受限版本的程序，进化成一个强大但低效的版本，最终演化成快速但稍微受限的版本。

Lisp 恶名昭彰的缓慢不是 Lisp 本身导致（Lisp 编译器早在 1980 年代就可以产生出与 C 编译器一样快的代码），而是由于许多程序员在第二个阶段就放弃的事实。如同 Richard Gabriel 所写的，

	要在 Lisp 撰写出性能极差的程序相当简单；而在 C 这几乎是不可能的。 `λ <http://acl.readthedocs.org/en/latest/zhCN/notes-cn.html#notes-284-2>`_

这完全是一个真的论述，但也可以解读为赞扬或贬低 Lisp 的论点：

1. 通过牺牲灵活性换取速度，你可以在 Lisp 里轻松地写出程序；在 C 语言里，你没有这个选择。

2. 除非你优化你的 Lisp 代码，不然要写出缓慢的软件根本易如反掌。

你的程序属于哪一种解读完全取决于你。但至少在开发初期，Lisp 使你有牺牲执行速度来换取时间的选择。

有一件我们示例程序没有做的很好的事是，它不是一个称职的 CLOS 模型（除了可能没有说明难以理解的 ``call-next-method`` 如何工作是件好事例外）。如大象般庞大的 CLOS 与这个如蚊子般微小的 70 行程序之间，存在多少的相似性呢？当然，这两者的差别是出自于教育性，而不是探讨有多相似。首先，这使我们理解到“面向对象”的广度。我们的程序比任何被称为是面向对象的都来得强大，而这只不过是 CLOS 的一小部分威力。

我们程序与 CLOS 不同的地方是，方法是属于某个对象的。这个方法的概念使它们与对第一个参数做派发的函数相同。而当我们使用函数式语法来调用方法时，这看起来就跟 Lisp 的函数一样。相反地，一个 CLOS 的通用函数，可以派发它的任何参数。一个通用函数的组件称为方法，而若你将它们定义成只对第一个参数特化，你可以制造出它们是某个类或实例的方法的错觉。但用面向对象编程的消息传递模型来思考 CLOS 最终只会使你困惑，因为 CLOS 凌驾在面向对象编程之上。

CLOS 的缺点之一是它太庞大了，并且 CLOS 费煞苦心的隐藏了面向对象编程，其实只不过是改写 Lisp 的这个事实。本章的例子至少阐明了这一点。如果我们满足于旧的消息传递模型，我们可以用一页多一点的代码来实现。面向对象编程不过是 Lisp 可以做的小事之一而已。更发人深省的问题是，Lisp 除此之外还能做些什么？

.. rubric:: 脚注

.. [1] 当类别被显示时， ``*print-array*`` 应当是 ``nil`` 。 任何类别的 ``preclist`` 的第一个元素都是类别本身，所以试图显示类别的内部结构会导致一个无限循环。

.. [2] 这个向量被 coerced 成一个列表，只是为了看看里面有什么。有了 ``*print-array*`` 被设成 ``nil`` ，一个向量的内容应该不会显示出来。
