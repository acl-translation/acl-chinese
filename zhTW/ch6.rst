.. highlight:: cl

第六章：函數
***************************************************

理解函數是理解 Lisp 的關鍵之一。概念上來說，函數是 Lisp 的核心所在。實際上呢，函數是你手邊最有用的工具之一。

6.1 全局函數 (Global Functions)
==================================

謂詞 ``fboundp`` 告訴我們，是否有個函數的名字與給定的符號綁定。如果一個符號是函數的名字，則 ``symbol-name`` 會返回它：

::

  > (fboundp '+)
  T
  > (symbol-function '+)
  #<Compiled-function + 17BA4E>

可通過 ``symbol-function`` 給函數配置某個名字：

::

  (setf (symbol-function 'add2)
    #'(lambda (x) (+ x 2)))

新的全局函數可以這樣定義，用起來和 ``defun`` 所定義的函數一樣：

::

  > (add2 1)
  3

實際上 ``defun`` 做了稍微多的工作，將某些像是

::

  (defun add2 (x) (+ x 2))

翻譯成上述的 ``setf`` 表達式。使用 ``defun`` 讓程式看起來更美觀，並或多或少幫助了編譯器，但嚴格來說，沒有 ``defun`` 也能寫程式。

通過把 ``defun`` 的第一個實參變成這種形式的列表 ``(setf f)`` ，你定義了當 ``setf`` 第一個實參是 ``f`` 的函數呼叫時，所會發生的事情。下面這對函數把 ``primo`` 定義成 ``car`` 的同義詞：

::

  (defun primo (lst) (car lst))

  (defun (setf primo) (val lst)
    (setf (car lst) val))

在函數名是這種形式 ``(setf f)`` 的函數定義中，第一個實參代表新的數值，而剩餘的實參代表了傳給 ``f`` 的參數。

現在任何 ``primo`` 的 ``setf`` ，會是上面後者的函數呼叫：

::

  > (let ((x (list 'a 'b 'c)))
      (setf (primo x) 480)
      x)
  (480 b c)

不需要爲了定義 ``(setf primo)`` 而定義 ``primo`` ，但這樣的定義通常是成對的。

由於字串是 Lisp 表達式，沒有理由它們不能出現在程式碼的主體。字串本身是沒有副作用的，除非它是最後一個表達式，否則不會造成任何差別。如果讓字串成爲 ``defun`` 定義的函數主體的第一個表達式，

::

  (defun foo (x)
    "Implements an enhanced paradigm of diversity"
    x)

那麼這個字串會變成函數的文檔字串（documentation string）。要取得函數的文檔字串，可以通過呼叫 ``documentation`` 來取得：

::

  > (documentation 'foo 'function)
  "Implements an enhanced paradigm of diversity"

6.2 區域函數 (Local Functions)
===============================

通過 ``defun`` 或 ``symbol-function`` 搭配 ``setf`` 定義的函數是全局函數。你可以像存取全局變數那樣，在任何地方存取它們。定義區域函數也是有可能的，區域函數和區域變數一樣，只在某些上下文內可以存取。

區域函數可以使用 ``labels`` 來定義，它是一種像是給函數使用的 ``let`` 。它的第一個實參是一個新區域函數的定義列表，而不是一個變數規格說明的列表。列表中的元素爲如下形式：

::

  (name parameters . body)

而 ``labels`` 表達式剩餘的部份，呼叫 ``name`` 就等於呼叫 ``(lambda parameters . body)`` 。

::

  > (labels ((add10 (x) (+ x 10))
             (consa  (x) (cons 'a x)))
      (consa (add10 3)))
  (A . 13)

``label`` 與 ``let`` 的類比在一個方面上被打破了。由 ``labels`` 表達式所定義的區域函數，可以被其他任何在此定義的函數引用，包括自己。所以這樣定義一個遞迴的區域函數是可能的：

::

  > (labels ((len (lst)
               (if (null lst)
                   0
                   (+ (len (cdr lst)) 1))))
      (len '(a b c)))
  3

5.2 節展示了 ``let`` 表達式如何被理解成函數呼叫。 ``do`` 表達式同樣可以被解釋成呼叫遞迴函數。這樣形式的 ``do`` :

::

  (do ((x a (b x))
       (y c (d y)))
      ((test x y) (z x y))
    (f x y))

等同於

::

  (labels ((rec (x y)
             (cond ((test x y)
                    (z x y))
                   (t
                    (f x y)
                    (rec (b x) (d y))))))
    (rec a c))

這個模型可以用來解決，任何你仍然對於 ``do`` 行爲仍有疑惑的問題。

6.3 參數列表 (Parameter Lists)
================================

2.1 節我們示範過，有了前序表達式， ``+`` 可以接受任何數量的參數。從那時開始，我們看過許多接受不定數量參數的函數。要寫出這樣的函數，我們需要使用一個叫做剩餘（ *rest* ）參數的東西。

如果我們在函數的形參列表裡的最後一個變數前，插入 ``&rest`` 符號，那麼當這個函數被呼叫時，這個變數會被設成一個帶有剩餘參數的列表。現在我們可以明白 ``funcall`` 是如何根據 ``apply`` 寫成的。它或許可以定義成：

::

  (defun our-funcall (fn &rest args)
    (apply fn args))

我們也看過運算子中，有的參數可以被忽略，並可以預設設成特定的值。這樣的參數稱爲選擇性參數（optional parameters）。（相比之下，普通的參數有時稱爲必要參數「required parameters」) 如果符號 ``&optional`` 出現在一個函數的形參列表時，

::

  (defun pilosoph (thing &optional property)
    (list thing 'is property))

那麼在 ``&optional`` 之後的參數都是選擇性的，預設爲 ``nil`` :

::

  > (philosoph 'death)
  (DEATH IS NIL)

我們可以明確指定預設值，通過將預設值附在列表裡給入。這版的 ``philosoph``

::

  (defun philosoph (thing &optional (property 'fun))
    (list thing 'is property))

有著更鼓舞人心的預設值：

::

  > (philosoph 'death)
  (DEATH IS FUN)

選擇性參數的預設值可以不是常數。可以是任何的 Lisp 表達式。若這個表達式不是常數，它會在每次需要用到預設值時被重新求值。

一個關鍵字參數（keyword parameter）是一種更靈活的選擇性參數。如果你把符號 ``&key`` 放在一個形參列表，那在 ``&key`` 之後的形參都是選擇性的。此外，當函數被呼叫時，這些參數會被識別出來，參數的位置在哪不重要，而是用符號標籤（譯註: ``:`` ）識別出來：

::

  > (defun keylist (a &key x y z)
      (list a x y z))
  KEYLIST

  > (keylist 1 :y 2)
  (1 NIL 2 NIL)

  > (keylist 1 :y 3 :x 2)
  (1 2 3 NIL)

和普通的選擇性參數一樣，關鍵字參數預設值爲 ``nil`` ，但可以在形參列表中明確地指定預設值。

關鍵字與其相關的參數可以被剩餘參數收集起來，並傳遞給其他期望收到這些參數的函數。舉例來說，我們可以這樣定義 ``adjoin`` ：

::

  (defun our-adjoin (obj lst &rest args)
    (if (apply #'member obj lst args)
        lst
        (cons obj lst)))

由於 ``adjoin`` 與 ``member`` 接受一樣的關鍵字，我們可以用剩餘參數收集它們，再傳給 ``member`` 函數。

5.2 節介紹過 ``destructuring-bind`` 宏。在通常情況下，每個模式（pattern）中作爲第一個參數的子樹，可以與函數的參數列表一樣複雜：

::

  (destructuring-bind ((&key w x) &rest y) '((:w 3) a)
    (list w x y))
  (3 NIL (A))

6.4 範例：實用函數 (Example: Utilities)
=========================================

2.6 節提到過，Lisp 大部分是由 Lisp 函陣列成，這些函數與你可以自己定義的函數一樣。這是程式語言中一個有用的特色：你不需要改變你的想法來配合語言，因爲你可以改變語言來配合你的想法。如果你想要 Common Lisp 有某個特定的函數，自己寫一個，而這個函數會成爲語言的一部分，就跟內建的 ``+`` 或 ``eql`` 一樣。

有經驗的 Lisp 程式設計師，由上而下（top-down）也由下而上 (bottom-up)地工作。當他們朝著語言撰寫程式的同時，也打造了一個更適合他們程式的語言。通過這種方式，語言與程式結合的更好，也更好用。

寫來擴展 Lisp 的運算子稱爲實用函數（utilities）。當你寫了更多 Lisp 程式時，會發現你開發了一系列的程式，而在一個項目寫過許多的實用函數，下個項目裡也會派上用場。

專業的程式設計師常發現，手邊正在寫的程式，與過去所寫的程式有很大的關聯。這就是軟體重用讓人聽起來很吸引人的原因。但重用已經被聯想成物件導向程式設計。但軟體不需要是面向物件的才能重用 ── 這是很明顯的，我們看看程式語言（換言之，編譯器），是重用性最高的軟體。

要獲得可重用軟體的方法是，由下而上地寫程式，而程式不需要是面向物件的才能夠由下而上地寫出。實際上，函數式風格相比之下，更適合寫出重用軟體。想想看 ``sort`` 。在 Common Lisp 你幾乎不需要自己寫排序程式； ``sort`` 是如此的快與普遍，以致於它不值得我們煩惱。這才是可重用軟體。

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

**圖 6.1 實用函數**

你可以通過撰寫實用函數，在程式裡做到同樣的事情。圖 6.1 挑選了一組實用的函數。前兩個 ``single?`` 與 ``append1`` 函數，放在這的原因是要示範，即便是小程式也很有用。前一個函數 ``single?`` ，當實參是只有一個元素的列表時，返回真。

::

  > (single? '(a))
  T

而後一個函數 ``append1`` 和 ``cons`` 很像，但在列表後面新增一個元素，而不是在前面:

::

  > (append1 '(a b c) 'd)
  (A B C D)

下個實用函數是 ``map-int`` ，接受一個函數與整數 ``n`` ，並返回將函數應用至整數 ``0`` 到 ``n-1`` 的結果的列表。

這在測試的時候非常好用（一個 Lisp 的優點之一是，互動環境讓你可以輕鬆地寫出測試）。如果我們只想要一個 ``0`` 到 ``9`` 的列表，我們可以：

::

  > (map-int #'identity 10)
  (0 1 2 3 4 5 6 7 8 9)

然而要是我們想要一個具有 10 個隨機數的列表，每個數介於 0 至 99 之間（包含 99），我們可以忽略參數並只要:

::

  > (map-int #'(lambda (x) (random 100))
             10)
  (85 50 73 64 28 21 40 67 5 32)

``map-int`` 的定義說明了 Lisp 構造列表的標準做法（idiom）之一。我們創建一個累積器 ``acc`` ，初始化是 ``nil`` ，並將之後的物件累積起來。當累積完畢時，反轉累積器。 [1]_

我們在 ``filter`` 中看到同樣的做法。 ``filter`` 接受一個函數與一個列表，將函數應用至列表元素上時，返回所有非 ``nil`` 元素:

::

  > (filter #'(lambda (x)
                (and (evenp x) (+ x 10)))
            '(1 2 3 4 5 6 7))
  (12 14 16)

另一種思考 ``filter`` 的方式是用通用版本的 ``remove-if`` 。

圖 6.1 的最後一個函數， ``most`` ，根據某個評分函數（scoring function），返回列表中最高分的元素。它返回兩個值，獲勝的元素以及它的分數:

::

  > (most #'length '((a b) (a b c) (a)))
  (A B C)
  3

如果平手的話，返回先馳得點的元素。

注意圖 6.1 的最後三個函數，它們全接受函數作爲參數。 Lisp 使得將函數作爲參數傳遞變得便捷，而這也是爲什麼，Lisp 適合由下而上程式設計的原因之一。成功的實用函數必須是通用的，當你可以將細節作爲函數參數傳遞時，要將通用的部份抽象起來就變得容易許多。

本節給出的函數是通用的實用函數。可以用在任何種類的程式。但也可以替特定種類的程式撰寫實用函數。確實，當我們談到宏時，你可以凌駕於 Lisp 之上，寫出自己的特定語言，如果你想這麼做的話。如果你想要寫可重用軟體，看起來這是最靠譜的方式。

6.5 閉包 (Closures)
=======================================

函數可以如表達式的值，或是其它物件那樣被返回。以下是接受一個實參，並依其型別返回特定的結合函數：

::

  (defun combiner (x)
    (typecase x
      (number #'+)
      (list #'append)
      (t #'list)))

在這之上，我們可以創建一個通用的結合函數:

::

  (defun combine (&rest args)
    (apply (combiner (car args))
           args))

它接受任何型別的參數，並以適合它們型別的方式結合。（爲了簡化這個例子，我們假定所有的實參，都有著一樣的型別。）

::

  > (combine 2 3)
  5
  > (combine '(a b) '(c d))
  (A B C D)

2.10 小節提過詞法變數（lexical variables）只在被定義的上下文內有效。伴隨這個限制而來的是，只要那個上下文還有在使用，它們就保證會是有效的。

如果函數在詞法變數的作用域裡被定義時，函數仍可引用到那個變數，即便函數被作爲一個值返回了，返回至詞法變數被創建的上下文之外。下面我們創建了一個把實參加上 ``3`` 的函數：

::

  > (setf fn (let ((i 3))
               #'(lambda (x) (+ x i))))
  #<Interpreted-Function C0A51E>
  > (funcall fn 2)
  5

當函數引用到外部定義的變數時，這外部定義的變數稱爲自由變數（free variable）。函數引用到自由的詞法變數時，稱之爲閉包（closure）。 [2]_ 只要函數還存在，變數就必須一起存在。

閉包結合了函數與環境（environment）；無論何時，當一個函數引用到周圍詞法環境的某個東西時，閉包就被隱式地創建出來了。這悄悄地發生在像是下面這個函數，是一樣的概念:

::

  (defun add-to-list (num lst)
    (mapcar #'(lambda (x)
                (+ x num))
            lst))

這函數接受一個數字及列表，並返回一個列表，列表元素是元素與傳入數字的和。在 lambda 表達式裡的變數 ``num`` 是自由的，所以像是這樣的情況，我們傳遞了一個閉包給 ``mapcar`` 。

一個更顯著的例子會是函數在被呼叫時，每次都返回不同的閉包。下面這個函數返回一個加法器（adder）:

::

  (defun make-adder (n)
    #'(lambda (x)
        (+ x n)))

它接受一個數字，並返回一個將該數字與其參數相加的閉包（函數）。

::

  > (setf add3 (make-adder 3))
  #<Interpreted-Function COEBF6>
  > (funcall add3 2)
  5
  > (setf add27 (make-adder 27))
  #<Interpreted-Function C0EE4E>
  > (funcall add27 2)
  29

我們可以產生共享變數的數個閉包。下面我們定義共享一個計數器的兩個函數:

::

  (let ((counter 0))
    (defun reset ()
      (setf counter 0))
    (defun stamp ()
      (setf counter (+ counter 1))))

這樣的一對函數或許可以用來創建時間戳章（time-stamps）。每次我們呼叫 ``stamp`` 時，我們獲得一個比之前高的數字，而呼叫 ``reset`` 我們可以將計數器歸零:

::

  > (list (stamp) (stamp) (reset) (stamp))
  (1 2 0 1)

你可以使用全局計數器來做到同樣的事情，但這樣子使用計數器，可以保護計數器被非預期的引用。

Common Lisp 有一個內建的函數 ``complement`` 函數，接受一個謂詞，並返回謂詞的補數（complement）。比如：

::

  > (mapcar (complement #'oddp)
            '(1 2 3 4 5 6))
  (NIL T NIL T NIL T)

有了閉包以後，很容易就可以寫出這樣的函數：

::

  (defun our-complement (f)
    #'(lambda (&rest args)
        (not (apply f args))))

如果你停下來好好想想，會發現這是個非凡的小例子；而這僅是冰山一角。閉包是 Lisp 特有的美妙事物之一。閉包開創了一種在別的語言當中，像是不可思議的程式設計方法。

6.6 範例：函數構造器 (Example: Function Builders)
=====================================================

Dylan 是 Common Lisp 與 Scheme 的混合物，有著 Pascal 一般的語法。它有著大量返回函數的函數：除了上一節我們所看過的 `complement` ，Dylan 包含: ``compose`` 、 ``disjoin`` 、 ``conjoin`` 、 ``curry`` 、 ``rcurry`` 以及 ``always`` 。圖 6.2 有這些函數的 Common Lisp 實現，而圖 6.3 示範了一些從定義延伸出的等價函數。

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

**圖 6.2 Dylan 函數建構器**

首先， ``compose`` 接受一個或多個函數，並返回一個依序將其參數應用的新函數，即，

::

  (compose #'a #'b #'c)

返回一個函數等同於

::

  #'(lambda (&rest args) (a (b (apply #'c args))))

這代表著 ``compose`` 的最後一個實參，可以是任意長度，但其它函數只能接受一個實參。

下面我們建構了一個函數，先給取參數的平方根，取整後再放回列表裡，接著返回:

::

  > (mapcar (compose #'list #'round #'sqrt)
            '(4 9 16 25))
  ((2) (3) (4) (5))

接下來的兩個函數， ``disjoin`` 及 ``conjoin`` 同接受一個或多個謂詞作爲參數： ``disjoin`` 當任一謂詞返回真時，返回真，而 ``conjoin`` 當所有謂詞返回真時，返回真。

::

  > (mapcar (disjoin #'integerp #'symbolp)
            '(a "a" 2 3))
  (T NIL T T)

::

  > (mapcar (conjoin #'integerp #'symbolp)
            '(a "a" 2 3))
  (NIL NIL NIL NIL)

若考慮將謂詞定義成集合， ``disjoin`` 返回傳入參數的聯集（union），而 ``conjoin`` 則是返回傳入參數的交集（intersection）。

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

**圖 6.3 某些等價函數**

函數 ``curry`` 與 ``rcurry`` （“right curry”）精神上與前一小節的 ``make-adder`` 相同。兩者皆接受一個函數及某些參數，並返回一個期望剩餘參數的新函數。下列任一個函數等同於 ``(make-adder 3)`` :

::

  (curry #'+ 3)
  (rcurry #'+ 3)

當函數的參數順序重要時，很明顯可以看出 ``curry`` 與 ``rcurry`` 的差別。如果我們 ``curry #'-`` ，我們得到一個用其參數減去某特定數的函數，

::

  (funcall (curry #'- 3) 2)
  1

而當我們 ``rcurry #'-`` 時，我們得到一個用某特定數減去其參數的函數:

::

  (funcall (rcurry #'- 3) 2)
  -1

最後， ``always`` 函數是 Common Lisp 函數 ``constantly`` 。接受一個參數並原封不動返回此參數的函數。和 ``identity`` 一樣，在很多需要傳入函數參數的情況下很有用。

6.7 動態作用域 (Dynamic Scope)
====================================================

2.11 小節解釋過區域與全局變數的差別。實際的差別是詞法作用域（lexical scope）的詞法變數（lexical variable），與動態作用域（dynamic scope）的特別變數（special variable）的區別。但這倆幾乎是沒有區別，因爲區域變數幾乎總是是詞法變數，而全局變數總是是特別變數。

在詞法作用域下，一個符號引用到上下文中符號名字出現的地方。區域變數預設有著詞法作用域。所以如果我們在一個環境裡定義一個函數，其中有一個變數叫做 ``x`` ，

::

  (let ((x 10))
    (defun foo ()
      x))

則無論 ``foo`` 被呼叫時有存在其它的 ``x`` ，主體內的 ``x`` 都會引用到那個變數:

::

  > (let ((x 20)) (foo))
  10

而動態作用域，我們在環境中函數被呼叫的地方尋找變數。要使一個變數是動態作用域的，我們需要在任何它出現的上下文中宣告它是 ``special`` 。如果我們這樣定義 ``foo`` ：

::

  (let ((x 10))
    (defun foo ()
      (declare (special x))
      x))

則函數內的 ``x`` 就不再引用到函數定義裡的那個詞法變數，但會引用到函數被呼叫時，當下所存在的任何特別變數 ``x`` :

::

  > (let ((x 20))
      (declare (special x))
      (foo))
  20

新的變數被創建出來之後， 一個 ``declare`` 呼叫可以在程式的任何地方出現。 ``special`` 宣告是獨一無二的，因爲它可以改變程式的行爲。 13 章將討論其它種類的宣告。所有其它的宣告，只是給編譯器的建議；或許可以使程式運行的更快，但不會改變程式的行爲。

通過在頂層呼叫 ``setf`` 來配置全局變數，是隱式地將變數宣告爲特殊變數:

::

  > (setf x 30)
  30
  > (foo)
  30

在一個檔案裡的程式碼，如果你不想依賴隱式的特殊宣告，可以使用 ``defparameter`` 取代，讓程式看起來更簡潔。

動態作用域什麼時候會派上用場呢？通常用來暫時給某個全局變數賦新值。舉例來說，有 11 個變數來控制物件印出的方式，包括了 ``*print-base*`` ，預設是 ``10`` 。如果你想要用 16 進制顯示數字，你可以重新綁定 ``*print-base*`` :

::

  > (let ((*print-base* 16))
      (princ 32))
  20
  32

這裡顯示了兩件事情，由 ``princ`` 產生的輸出，以及它所返回的值。他們代表著同樣的數字，第一次在被印出時，用 16 進制顯示，而第二次，因爲在 ``let`` 表達式外部，所以是用十進制顯示，因爲 ``*print-base*`` 回到之前的數值， ``10`` 。

6.8 編譯 (Compilation)
========================================

Common Lisp 函數可以獨立被編譯或挨個檔案編譯。如果你只是在頂層輸入一個 ``defun`` 表達式：

::

  > (defun foo (x) (+ x 1))
  FOO

許多實現會創建一個直譯的函數（interpreted function）。你可以將函數傳給 ``compiled-function-p`` 來檢查一個函數是否有被編譯:

::

  > (compiled-function-p #'foo)
  NIL

若你將 ``foo`` 函數名傳給 ``compile`` :

::

  > (compile 'foo)
  FOO

則這個函數會被編譯，而直譯的定義會被編譯出來的取代。編譯與直譯函數的行爲一樣，只不過對 ``compiled-function-p`` 來說不一樣。

你可以把列表作爲參數傳給 ``compile`` 。這種 ``compile`` 的用法在 161 頁 (譯註: 10.1 小節)。

有一種函數你不能作爲參數傳給 ``compile`` ：一個像是 ``stamp`` 或是 ``reset`` 這種，在頂層明確使用詞法上下文輸入的函數 (即 ``let`` ) [3]_ 在一個檔案裡面定義這些函數，接著編譯然後載入檔案是可以的。直譯的程式會有這樣的限制是實作的原因，而不是因爲在詞法上下文裡明確定義函數有什麼問題。

通常要編譯 Lisp 程式不是挨個函數編譯，而是使用 ``compile-file`` 編譯整個檔案。這個函數接受一個檔案名，並創建一個原始碼的編譯版本 ── 通常會有同樣的名稱，但不同的擴展名。當編譯過的檔案被載入時， ``compiled-function-p`` 應給所有定義在檔案內的函數返回真。

當一個函數包含在另一個函數內時，包含它的函數會被編譯，而且內部的函數也會被編譯。所以 ``make-adder`` (108 頁)被編譯時，它會返回編譯的函數:

::

  > (compile 'make-adder)
  MAKE-ADDER
  > (compiled-function-p (make-adder 2))
  T

6.9 使用遞迴 (Using Recursion)
================================================

比起多數別的語言，遞迴在 Lisp 中扮演了一個重要的角色。這主要有三個原因：

1. 函數式程式設計。遞迴演算法有副作用的可能性較低。

2. 遞迴資料結構。 Lisp 隱式地使用了指標，使得遞迴地定義資料結構變簡單了。最常見的是用在列表：一個列表的遞迴定義，列表爲空表，或是一個 ``cons`` ，其中 ``cdr`` 也是個列表。

3. 優雅性。Lisp 程式設計師非常關心它們的程式是否美麗，而遞迴演算法通常比迭代演算法來得優雅。

學生們起初會覺得遞迴很難理解。但 3.9 節指出了，如果你想要知道是否正確，不需要去想遞迴函數所有的呼叫過程。

同樣的如果你想寫一個遞迴函數。如果你可以描述問題是怎麼遞迴解決的，通常很容易將解法轉成程式。要使用遞迴來解決一個問題，你需要做兩件事：

1. 你必須要示範如何解決問題的一般情況，通過將問題切分成有限小並更小的子問題。

2. 你必須要示範如何通過 ── 有限的步驟，來解決最小的問題 ── 基本用例。

如果這兩件事完成了，那問題就解決了。因爲遞迴每次都將問題變得更小，而一個有限的問題終究會被解決的，而最小的問題僅需幾個有限的步驟就能解決。

舉例來說，下面這個找到一個正規列表（proper list）長度的遞迴算法，我們每次遞迴時，都可以找到更小列表的長度：

1. 在一般情況下，一個正規列表的長度是它的 ``cdr`` 加一。

2. 基本用例，空列表長度爲 ``0`` 。

當這個描述翻譯成程式時，先處理基本用例；但公式化遞迴演算法時，我們通常從一般情況下手。

前述的演算法，明確地描述了一種找到正規列表長度的方法。當你定義一個遞迴函數時，你必須要確定你在分解問題時，問題實際上越變越小。取得一個正規列表的 ``cdr`` 會給出 ``length`` 更小的子問題，但取得環狀列表（circular list）的 ``cdr`` 不會。

這裡有兩個遞迴算法的範例。假定參數是有限的。注意第二個範例，我們每次遞迴時，將問題分成兩個更小的問題：

第一個例子， ``member`` 函數，我們說某物是列表的成員，需滿足：如果它是第一個元素的成員或是 ``member`` 的 ``cdr`` 的成員。但空列表沒有任何成員。

第二個例子， ``copy-tree`` 一個 ``cons`` 的 ``copy-tree`` ，是一個由 ``cons`` 的 ``car`` 的 ``copy-tree`` 與 ``cdr`` 的 ``copy-tree`` 所組成的。一個原子的 ``copy-tree`` 是它自己。

一旦你可以這樣描述算法，要寫出遞迴函數只差一步之遙。

某些算法通常是這樣表達最自然，而某些算法不是。你可能需要翻回前面，試試不使用遞迴來定義 ``our-copy-tree`` (41 頁，譯註: 3.8 小節)。另一方面來說，23 頁 (譯註: 2.13 節) 迭代版本的 ``show-squares`` 可能更容易比 24 頁的遞迴版本要容易理解。某些時候是很難看出哪個形式比較自然，直到你試著去寫出程式來。

如果你關心效率，有兩個你需要考慮的議題。第一，尾遞迴（tail-recursive），會在 13.2 節討論。一個好的編譯器，使用迴圈或是尾遞迴的速度，應該是沒有或是區別很小的。然而如果你需要使函數變成尾遞迴的形式時，或許直接用迭代會更好。

另一個需要銘記在心的議題是，最顯而易見的遞迴算法，不一定是最有效的。經典的例子是費氏函數。它是這樣遞迴地被定義的，

  1. Fib(0) = Fib(1) = 1

  2. Fib(n) = Fib(n-1)+Fib(n-2)

直接翻譯這個定義，

::

  (defun fib (n)
    (if (<= n 1)
        1
        (+ (fib (- n 1))
           (fib (- n 2)))))

這樣是效率極差的。一次又一次的重複計算。如果你要找 ``(fib 10)`` ，這個函數計算 ``(fib 9)`` 與 ``(fib 8)`` 。但要計算出 ``(fib 9)`` ，它需要再次計算 ``(fib 8)`` ，等等。

下面是一個算出同樣結果的迭代版本:

::

  (defun fib (n)
    (do ((i n (- i 1))
         (f1 1 (+ f1 f2))
         (f2 1 f1))
        ((<= i 1) f1)))

迭代的版本不如遞迴版本來得直觀，但是效率遠遠高出許多。這樣的事情在實踐中常發生嗎？非常少 ── 這也是爲什麼所有的教科書都使用一樣的例子 ── 但這是需要注意的事。

Chapter 6 總結 (Summary)
============================

1. 命名函數是一個存在符號的 ``symbol-function`` 部分的函數。 ``defun`` 宏隱藏了這樣的細節。它也允許你定義文檔字串（documentation string），並指定 ``setf`` 要怎麼處理函數呼叫。

2. 定義區域函數是有可能的，與定義區域變數有相似的精神。

3. 函數可以有選擇性參數（optional）、剩餘（rest）以及關鍵字（keyword）參數。

4. 實用函數是 Lisp 的擴展。他們是由下而上編程的小規模範例。

5. 只要有某物引用到詞法變數時，它們會一直存在。閉包是引用到自由變數的函數。你可以寫出返回閉包的函數。

6. Dylan 提供了構造函數的函數。很簡單就可以使用閉包，然後在 Common Lisp 中實現它們。

7. 特別變數（special variable）有動態作用域 (dynamic scope)。

8. Lisp 函數可以單獨編譯，或（更常見）編譯整個檔案。

9. 一個遞迴演算法通過將問題細分成更小丶更小的子問題來解決問題。

Chapter 6 練習 (Exercises)
==================================

1. 定義一個 ``tokens`` 版本 (67 頁)，接受 ``:test`` 與 ``:start`` 參數，預設分別是 ``#'constituent`` 與 ``0`` 。(譯註: 67 頁在 4.5 小節)

2. 定義一個 ``bin-search`` (60 頁)的版本，接受 ``:key`` , ``:test`` , ``start`` 與 ``end`` 參數，有著一般的意義與預設值。(譯註: 60 頁在 4.1 小節)

3. 定義一個函數，接受任何數目的參數，並返回傳入的參數。

4. 修改 ``most`` 函數 (105 頁)，使其返回 2 個數值，一個列表中最高分的兩個元素。(譯註: 105 頁在 6.4 小節)

5. 用 ``filter`` (105 頁) 來定義 ``remove-if`` （沒有關鍵字）。(譯註: 105 頁在 6.4 小節)

6. 定義一個函數，接受一個參數丶一個數字，並返回目前傳入參數中最大的那個。

7. 定義一個函數，接受一個參數丶一個數字，若傳入參數比上個參數大時，返回真。函數第一次呼叫時應返回 ``nil`` 。

8. 假設 ``expensive`` 是一個接受一個參數的函數，一個介於 0 至 100 的整數（包含 100)，返回一個耗時的計算結果。定義一個函數 ``frugal`` 來返回同樣的答案，但僅在沒見過傳入參數時呼叫 ``expensive`` 。

9. 定義一個像是 ``apply`` 的函數，但在任何數字印出前，預設用 8 進制印出。


.. rubric:: 腳註

.. [1] 在這個情況下， ``nreverse`` (在 222 頁描述)和 ``reverse`` 做一樣的事情，但更有效率。

.. [2] “閉包”這個名字是早期的 Lisp 方言流傳而來。它是從閉包需要在動態作用域裡實現的方式衍生而來。

.. [3] 以前的 ANSI Common Lisp， ``compile`` 的第一個參數也不能是一個已經編譯好的函數。
