.. highlight:: cl

附录 B：Lisp in Lisp
***************************************************

这个附录包含了 58 个最常用的 Common Lisp 操作符。因为如此多的 Lisp 是（或可以）用 Lisp 所写成，而由于 Lisp 程序（或可以）相当精简，这是一种方便解释语言的方式。

这个练习也证明了，概念上 Common Lisp 不像看起来那样庞大。许多 Common Lisp 操作符是有用的函式库；要写出所有其它的东西，你所需要的操作符相当少。在这个附录的这些定义只需要:

``apply`` ``aref`` ``backquote`` ``block`` ``car`` ``cdr`` ``ceiling`` ``char=`` ``cons`` ``defmacro`` ``documentation`` ``eq`` ``error`` ``expt`` ``fdefinition`` ``function`` ``floor`` ``gensym`` ``get-setf-expansion`` ``if`` ``imagpart`` ``labels`` ``length`` ``multiple-value-bind`` ``nth-value`` ``quote`` ``realpart`` ``symbol-function`` ``tagbody`` ``type-of`` ``typep`` ``=`` ``+`` ``-`` ``/`` ``<`` ``>``

这里给出的代码作为一种解释 Common Lisp 的方式，而不是实现它的方式。在实际的实现上，这些操作符可以更高效，也会做更多的错误检查。为了方便参找，这些操作符本身按字母顺序排列。如果你真的想要这样定义 Lisp，每个宏的定义需要在任何呼叫它们的代码之前。


::

	(defun -abs (n)
	  (if (typep n 'complex)
	      (sqrt (+ (expt (realpart n) 2) (expt (imagpart n) 2)))
	      (if (< n 0) (- n) n)))

::

	(defun -adjoin (obj lst &rest args)
	  (if (apply #'member obj lst args) lst (cons obj lst)))

::

	(defmacro -and (&rest args)
	  (cond ((null args) t)
	        ((cdr args)  `(if ,(car args) (-and ,@(cdr args))))
	        (t           (car args))))

::

	(defun -append (&optional first &rest rest)
	  (if (null rest)
	      first
	      (nconc (copy-list first) (apply #'-append rest))))

::

	(defun -atom (x) (not (consp x)))

::

	(defun -butlast (lst &optional (n 1))
	  (nreverse (nthcdr n (reverse lst))))

::

	(defun -cadr (x) (car (cdr x)))

::

	(defmacro -case (arg &rest clauses)
	  (let ((g (gensym)))
	    `(let ((,g ,arg))
	       (cond ,@(mapcar #'(lambda (cl)
	                           (let ((k (car cl)))
	                             `(,(cond ((member k '(t otherwise))
	                                       t)
	                                      ((consp k)
	                                       `(member ,g ',k))
	                                      (t `(eql ,g ',k)))
	                               (progn ,@(cdr cl)))))
	                       clauses)))))

::

	(defun -cddr (x) (cdr (cdr x)))

::

	(defun -complement (fn)
	  #'(lambda (&rest args) (not (apply fn args))))

::

	(defmacro -cond (&rest args)
	  (if (null args)
	      nil
	      (let ((clause (car args)))
	        (if (cdr clause)
	            `(if ,(car clause)
	                 (progn ,@(cdr clause))
	                 (-cond ,@(cdr args)))
	            `(or ,(car clause)
	                 (-cond ,@(cdr args)))))))

::

	(defun -consp (x) (typep x 'cons))

::

	(defun -constantly (x) #'(lambda (&rest args) x))

::

	(defun -copy-list (lst)
	  (labels ((cl (x)
	             (if (atom x)
	                 x
	                 (cons (car x)
	                       (cl (cdr x))))))
	    (cons (car lst)
	          (cl (cdr lst)))))

::

	(defun -copy-tree (tr)
	  (if (atom tr)
	      tr
	      (cons (-copy-tree (car tr))
	            (-copy-tree (cdr tr)))))

::

	(defmacro -defun (name parms &rest body)
	  (multiple-value-bind (dec doc bod) (analyze-body body)
	    `(progn
	       (setf (fdefinition ',name)
	             #'(lambda ,parms
	                 ,@dec
	                 (block ,(if (atom name) name (second name))
	                   ,@bod))
	             (documentation ',name 'function)
	             ,doc)
	       ',name)))

::

	(defun analyze-body (body &optional dec doc)
	  (let ((expr (car body)))
	    (cond ((and (consp expr) (eq (car expr) 'declare))
	           (analyze-body (cdr body) (cons expr dec) doc))
	          ((and (stringp expr) (not doc) (cdr body))
	           (if dec
	               (values dec expr (cdr body))
	               (analyze-body (cdr body) dec expr)))
	          (t (values dec doc body)))))

::

这个定义不完全正确，参见 ``let``

::

	(defmacro -do (binds (test &rest result) &rest body)
	  (let ((fn (gensym)))
	    `(block nil
	       (labels ((,fn ,(mapcar #'car binds)
	                   (cond (,test ,@result)
	                         (t (tagbody ,@body)
	                            (,fn ,@(mapcar #'third binds))))))
	         (,fn ,@(mapcar #'second binds))))))

::

	(defmacro -dolist ((var lst &optional result) &rest body)
	  (let ((g (gensym)))
	    `(do ((,g ,lst (cdr ,g)))
	         ((atom ,g) (let ((,var nil)) ,result))
	       (let ((,var (car ,g)))
	         ,@body))))

::

	(defun -eql (x y)
	  (typecase x
	    (character (and (typep y 'character) (char= x y)))
	    (number    (and (eq (type-of x) (type-of y))
	                    (= x y)))
	    (t         (eq x y))))

::

	(defun -evenp (x)
	  (typecase x
	    (integer (= 0 (mod x 2)))
	    (t       (error "non-integer argument"))))

::

	(defun -funcall (fn &rest args) (apply fn args))

::

	(defun -identity (x) x)

这个定义不完全正确：表达式 ``(let ((&key 1) (&optional 2)))`` 是合法的，但它产生的表达式不合法。

::

	(defmacro -let (parms &rest body)
	  `((lambda ,(mapcar #'(lambda (x)
	                         (if (atom x) x (car x)))
	                     parms)
	      ,@body)
	    ,@(mapcar #'(lambda (x)
	                  (if (atom x) nil (cadr x)))
	              parms)))

::

	(defun -list (&rest elts) (copy-list elts))

::

	(defun -listp (x) (or (consp x) (null x)))

::

	(defun -mapcan (fn &rest lsts)
	  (apply #'nconc (apply #'mapcar fn lsts)))

::

	(defun -mapcar (fn &rest lsts)
	  (cond ((member nil lsts) nil)
	        ((null (cdr lsts))
	         (let ((lst (car lsts)))
	           (cons (funcall fn (car lst))
	                 (-mapcar fn (cdr lst)))))
	        (t
	         (cons (apply fn (-mapcar #'car lsts))
	               (apply #'-mapcar fn
	                      (-mapcar #'cdr lsts))))))

::

	(defun -member (x lst &key test test-not key)
	  (let ((fn (or test
	                (if test-not
	                    (complement test-not))
	                    #'eql)))
	    (member-if #'(lambda (y)
	                   (funcall fn x y))
	               lst
	               :key key)))

::

	(defun -member-if (fn lst &key (key #'identity))
	  (cond ((atom lst) nil)
	        ((funcall fn (funcall key (car lst))) lst)
	        (t (-member-if fn (cdr lst) :key key))))

::

	(defun -mod (n m)
	  (nth-value 1 (floor n m)))

::

	(defun -nconc (&optional lst &rest rest)
	  (if rest
	      (let ((rest-conc (apply #'-nconc rest)))
	        (if (consp lst)
	            (progn (setf (cdr (last lst)) rest-conc)
	                   lst)
	            rest-conc))
	      lst))

::

	(defun -not (x) (eq x nil))
	(defun -nreverse (seq)
	  (labels ((nrl (lst)
	             (let ((prev nil))
	               (do ()
	                   ((null lst) prev)
	                 (psetf (cdr lst) prev
	                        prev      lst
	                        lst       (cdr lst)))))
	           (nrv (vec)
	             (let* ((len (length vec))
	                    (ilimit (truncate (/ len 2))))
	               (do ((i 0 (1+ i))
	                    (j (1- len) (1- j)))
	                   ((>= i ilimit) vec)
	                 (rotatef (aref vec i) (aref vec j))))))
	    (if (typep seq 'vector)
	        (nrv seq)
	        (nrl seq))))

::

	(defun -null (x) (eq x nil))

::

	(defmacro -or (&optional first &rest rest)
	  (if (null rest)
	      first
	      (let ((g (gensym)))
	        `(let ((,g ,first))
	           (if ,g
	               ,g
	               (-or ,@rest))))))

这两个 Common Lisp 没有，但这里有几的定义会需要用到。

::

	(defun pair (lst)
	  (if (null lst)
	      nil
	      (cons (cons (car lst) (cadr lst))
	            (pair (cddr lst)))))

	(defun -pairlis (keys vals &optional alist)
	  (unless (= (length keys) (length vals))
	    (error "mismatched lengths"))
	  (nconc (mapcar #'cons keys vals) alist))

::

	(defmacro -pop (place)
	  (multiple-value-bind (vars forms var set access)
	                       (get-setf-expansion place)
	    (let ((g (gensym)))
	      `(let* (,@(mapcar #'list vars forms)
	              (,g ,access)
	              (,(car var) (cdr ,g)))
	         (prog1 (car ,g)
	                ,set)))))

::

	(defmacro -prog1 (arg1 &rest args)
	  (let ((g (gensym)))
	    `(let ((,g ,arg1))
	       ,@args
	       ,g)))

::

	(defmacro -prog2 (arg1 arg2 &rest args)
	  (let ((g (gensym)))
	    `(let ((,g (progn ,arg1 ,arg2)))
	       ,@args
	       ,g)))

::

	(defmacro -progn (&rest args) `(let nil ,@args))

::

	(defmacro -psetf (&rest args)
	  (unless (evenp (length args))
	    (error "odd number of arguments"))
	  (let* ((pairs (pair args))
	         (syms (mapcar #'(lambda (x) (gensym))
	                       pairs)))
	    `(let ,(mapcar #'list
	                   syms
	                   (mapcar #'cdr pairs))
	       (setf ,@(mapcan #'list
	                       (mapcar #'car pairs)
	                       syms)))))

::

	(defmacro -push (obj place)
	  (multiple-value-bind (vars forms var set access)
	                       (get-setf-expansion place)
	    (let ((g (gensym)))
	      `(let* ((,g ,obj)
	              ,@(mapcar #'list vars forms)
	              (,(car var) (cons ,g ,access)))
	         ,set))))

::

	(defun -rem (n m)
	  (nth-value 1 (truncate n m)))

	(defmacro -rotatef (&rest args)
	  `(psetf ,@(mapcan #'list
	                    args
	                    (append (cdr args)
	                            (list (car args))))))

::

	(defun -second (x) (cadr x))

	(defmacro -setf (&rest args)
	  (if (null args)
	      nil
	      `(setf2 ,@args)))

::

	(defmacro setf2 (place val &rest args)
	  (multiple-value-bind (vars forms var set)
	                       (get-setf-expansion place)
	    `(progn
	       (let* (,@(mapcar #'list vars forms)
	              (,(car var) ,val))
	         ,set)
	       ,@(if args `((setf2 ,@args)) nil))))

::

	(defun -signum (n)
	  (if (zerop n) 0 (/ n (abs n))))

::

	(defun -stringp (x) (typep x 'string))

::

	(defun -tailp (x y)
	  (or (eql x y)
	      (and (consp y) (-tailp x (cdr y)))))

::

	(defun -third (x) (car (cdr (cdr x))))

::

	(defun -truncate (n &optional (d 1))
	  (if (> n 0) (floor n d) (ceiling n d)))

::

	(defmacro -typecase (arg &rest clauses)
	  (let ((g (gensym)))
	    `(let ((,g ,arg))
	       (cond ,@(mapcar #'(lambda (cl)
	                           `((typep ,g ',(car cl))
	                             (progn ,@(cdr cl))))
	                       clauses)))))

::

	(defmacro -unless (arg &rest body)
	  `(if (not ,arg)
	       (progn ,@body)))

::

	(defmacro -when (arg &rest body)
	  `(if ,arg (progn ,@body)))

::

	(defun -1+ (x) (+ x 1))

::

	(defun -1- (x) (- x 1))

::

	(defun ->= (first &rest rest)
	  (or (null rest)
	      (and (or (> first (car rest)) (= first (car rest)))
	           (apply #'->= rest))))