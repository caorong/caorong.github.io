---
layout: post
title: "little schemer in Clojure chapter 9 Lambda the Ultimate"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 9 Lambda the Ultimate](http://juliangamble.com/blog/2012/09/15/the-little-schemer-in-clojure-chapter-9-lambda-the-ultimate-deriving-the-y-combinator/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第九章

注，后文讲用 TLS 代替 The Little Schemer


这一章是本书最重要的地方，前几章，基本一天能看一章，这一章看了一个礼拜。你也许听过 Paul Graham 的 天使投资公司，[Ycombinator](http://ycombinator.com/) － 这一章我们不仅仅解释 YCombinator 并在 Clojure中实现它。

首先，神马是 `YCombinator` 理论？` The YCombinator is a way to do recursion in a language that doesn’t support recursion. Instead recursion is defined as a set of rewrite rules.` 也就是说他是一章方法，可以在不支持递归的语言里面使用递归。用一系列重写规则代替递归。

不是吧，就我所知不支持递归的语言要追溯到60年代了。现在再说这个还有神马意义马？其实，仅仅为了上面的理由是没有意义的，他的意义在于构造lazy function 以及 数据结构 － Clojure 里面非常实用的东西 － 并且在别的lisp语言里比较稀有。

让我们开始吧。

`rember-f` 它包含一个list，一个atom， 一个比较函数，如下解释

```Clojure
;demonstrating passing functions around
(def rember-f
  (fn [test? a l]
    (cond
      (null? l) '()
      true (cond
             (test? (first l) a) (rest l)
             true (cons (first l) (rember-f test? a (rest l)))))))
 
(println (rember-f = '(pop corn) '(lemonade (pop corn) and (cake))))
;//=>(lemonade and (cake))
```

我们把 （＝） 作为比较函数，用来去掉list 里面于a相同的atom


```Clojure
; 简化版
(def rember-f
  (fn [test? a l]
    (cond
      (null? l) '()
      (test? (first l) a) (rest l)
      true (cons (first l) (rember-f test? a (rest l))))))
 
(println (rember-f = '(pop corn) '(lemonade (pop corn) and (cake))))
;//=>(lemonade and (cake))
```

现在我们要讲讲 curried（柯里化 scala也有强调）。他的名字是根据数学家 [Haskell Curry](http://en.wikipedia.org/wiki/Haskell_Curry)而来的。 curring 一个函数的意思是，将原本需要2个入参的函数改造成1个入参，并仍然返回一个结果函数，（只有一个入参的函数）


```Clojure
(def eq?-c
  (fn [a]
    (fn [x]
      (= x a))))
 
(println (eq?-c 'lemonade))
=> (println (eq?-c 'lemonade))
#<Chapter9LambdaTheUltimate$eq_QMARK__c$fn__974 Chapter9LambdaTheUltimate$eq_QMARK__c$fn__974@2a2a2ae9>
(println ((eq?-c 'lemonade) 'coke))
;//=> false
(println ((eq?-c 'lemonade) 'lemonade))
;//=> true
 
(def eq?-salad (eq?-c 'salad))
 
(println (eq?-salad 'tuna))
;//=>false
(println (eq?-salad 'salad))
;//=>true
```

curry 版本的 rember-f 

```Clojure
(def rember-f
  (fn [test?]
    (fn [a l]
      (cond
        (null? l) '()
        (test? (first l) a) (rest l)
        true (cons (first l) ((rember-f test?) a (rest l)))))))
 
(println ((rember-f =) 'tuna '(tuna salad is good)))
;//=>(salad is good)
 
(def rember-eq? (rember-f =))
(println (rember-eq? 'tuna '(tuna salad is good)))
;//=>(salad is good)
```

curry化 insert-L

```Clojure
(def insertL-f
  (fn [test?]
    (fn [new old l]
      (cond
        (null? l) '()
        (test? (first l) old) (cons new (cons old (rest l)))
        true (cons (first l) ((insertL-f test?) new old (rest l)))))))
 
(println ((insertL-f =) 'creamy 'latte '(a hot cup of latte)))
```

```Clojure
(def insertR-f
  (fn [test?]
    (fn [new old l]
      (cond
        (null? l) '()
        (test? (first l) old) (cons old (cons new (rest l)))
        true (cons (first l) ((insertR-f test?) new old (rest l)))))))
 
(println ((insertR-f =) 'cake 'cheese '(new york cheese)))
```

你有没有发现上面的 `insertR` 和 `insertL` 的中间代码都非常相似。下面我们将用 `insert-g` 函数替代上面重复代码。用 `seqL` 和 `seqR` 分别代替不重复部分的代码。


```Clojure
(def seqL
  (fn [new old l]
    (cons new (cons old l))))
 
(def seqR
  (fn [new old l]
    (cons old (cons new l))))
 
(def insert-g
  (fn [seqarg]
    (fn [new old l]
      (cond
        (null? l) '()
        (= (first l) old) (seqarg new old (rest l))
        true (cons (first l) ((insert-g seqarg) new old (rest l)))))))
 
(def insertL (insert-g seqL))
(println (insertL 'creamy 'latte '(a hot cup of latte)))
;//=>(a hot cup of creamy latte)
(def insertR (insert-g seqR))
(println (insertR 'cake 'cheese '(new york cheese)))
;//=>(new york cheese cake)
```

利用上面的提取公因式的代码 重构 `insertL` 

```Clojure
(def insertL
  (insert-g
    (fn [new old l]
      (cons new (cons old l)))))
(println (insertL 'creamy 'latte '(a hot cup of latte)))
;//=>(a hot cup of creamy latte)
```

现在回看 [Chapter 3](/clojure/2015/02/12/the-little-schemer-in-clojure-chapter-3.html)的 `subst`

```Clojure
(def subst
  (fn [new old l]
    (cond
      (null? l) '()
      (= (first l) old) (cons new (rest l))
      true (cons (first l) (subst new old (rest l))))))
 
(println (subst 'espresso 'latte '(a hot cup of latte)))
;//=>(a hot cup of espresso)
```

有没有发现subst和 `insertL`, `insertR` 很类似，它仅仅是没有把old cons进去。所以我们同样可以用 `insert-g` 重构。

```Clojure
(def seqS
  (fn [new old l]
    (cons new l)))
 
(def subst (insert-g seqS))
 
(println (subst 'espresso 'latte '(a hot cup of latte)))
;//>(a hot cup of espresso)
```

下面重构 `rember`

```Clojure
(def seqrem
  (fn [new old l]
    l))
 
(def rember
  (fn [a l]
    ((insert-g seqrem) nil a l)))
 
(println (rember 'hot '(a hot cup of espresso)))
;//=>(a cup of espresso)
```

第十要义：

**"Abstract functions with common structures into a single function"**

就是提取公因式，并提取出来作为独立函数。


现在回头看 [Chapter 7](/clojure/2015/02/22/the-little-schemer-in-clojure-chapter-7.html) 的 `value`

```Clojure
(def number_?
  (fn [a]
    (cond
      (null? a) false
      (number? a) true
      true false)))
 
(def first-sub-exp
  (fn [aexp]
    (first (rest aexp))))
 
(def second-sub-exp
  (fn [aexp]
    (first (rest (rest aexp)))))
 
(def operator
  (fn [aexp]
    (first aexp)))
 
(use 'clojure.math.numeric-tower)
 
(def value
  (fn [aexp]
    (cond
      (number_? aexp) aexp
      (= (operator aexp) '+) (+ (value (first-sub-exp aexp)) (value  (second-sub-exp aexp)))
      (= (operator aexp) '*) (* (value (first-sub-exp aexp)) (value (second-sub-exp aexp)))
      (= (operator aexp) 'exp) (expt (value (first-sub-exp aexp)) (value (second-sub-exp aexp))))))
 
(println (value '(+ 1 1)))
;//=>2
```

现在通过使用一个 可以将一个符号字符串 返回成相应函数的函数：

```Clojure
(def atom-to-function
  (fn [x]
    (cond
      (= x '+ ) +
      (= x '* ) *
      (= x 'exp ) expt )))
 
(def value
  (fn [aexp]
    (cond
      (number_? aexp) aexp
      true ((atom-to-function (operator aexp))
             (value (first-sub-exp aexp))
             (value (second-sub-exp aexp))))))
 
(println (value '(+ 1 1)))
;//=> 2
```

现在是不是变短一些了。


现在回到 [Chapter 8](/clojure/2015/03/04/the-little-schemer-in-clojure-chapter-8.html) 看 `subset?` 和 `intersect?`

```Clojure
(def member?
  " a is a member of lat?"
  (fn [a lat]
    (cond
      (null? lat) false
      :else (or
        (= (first lat) a)
        (member? a (rest lat)))) ))
 
(def subset?
  (fn [set1 set2]
  "set1 is subset of set2 ?"
    (cond
      (null? set1) true
      :else (and
             (member? (first set1) set2)
             (subset? (rest set1) set2)))))
(println (subset? '(a b c) '(b c d)))
;//=>false
(println (subset? '(b c) '(b c d)))
;//=>true
 
(def intersect?
  "if set1 has at least one atom in set2"
  (fn [set1 set2]
    (cond
      (null? set1) false
      :else (or
             (member? (first set1) set2)
             (intersect? (rest set1) set2)))))
 
(println (intersect? '(a b c) '(b c d)))
;//=>true
```

我们发现上面的函数仅仅在 else里面的 or/and 以及递归终止时返回 true/false 不同，抽象之


```Clojure
(def set-f?
  (fn [logical? const]
    (fn [set1 set2]
      (cond
        (null? set1) const
        :else (logical?
               (member? (first set1) set2)
               ((set-f? logical? const) (rest set1) set2))))))
 
;(def subset? (set-f? and true))
;(def intersect? (set-f? or nil))
; note - doesn't work yet
```

但上面无法工作

```
CompilerException java.lang.RuntimeException: Can't take value of a macro: #'clojure.core/and, compiling:(/Users/caorong/Documents/workspace_clojure/little-schemer/src/little_schemer/chapter8.clj:161:14) 
```

于是我们不得不重新定义 `and` 或者自己要用到的函数

```Clojure
(def and-prime
  (fn [x y]
    (and x y)))
 
(def or-prime
  (fn [x y]
    (or x y)))
; still doesn't work
 
(def or-prime
  (fn [x set1 set2]
    (or x (intersect? (rest set1) set2))))
 
(def and-prime
  (fn [x set1 set2]
    (and x (subset? (rest set1) set2))))
 
(def set-f?
  (fn [logical? const]
    (fn [set1 set2]
      (cond
        (null? set1) const
        true (logical?
               (member? (first set1) set2)
               set1 set2)))))
 
(def intersect? (set-f? or-prime false))
(def subset? (set-f? and-prime true))
 
(println (intersect?  '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>true
(println (subset? '(banana butter) '(breakfast toasted banana bread with butter for breakfast)))
;//=>true
```

这里需要小心 `intersect?` 的递归的定义与使用， 这里我们用了个我们事先未定义的函数，然后用这个俄函数定义 `intersect?` 有点绕。

我们将 `and-prime` 和 `or-prime` 作为 `set-f` 的参数。



后面我们将继续 Y Combinator 的推导，从 [Chapter5](/clojure/2015/02/21/the-little-schemer-in-clojure-chapter-5.html)的 `multirember` 开始 ，从去掉 `cond` 开始。


```Clojure
(def multirember
  (fn [a lat]
    (cond
      (null? lat) '()
      (= (first lat) a) (multirember a (rest lat))
      true (cons (first lat) (multirember a (rest lat))))))
 
(println (multirember 'breakfast '(breakfast toasted banana bread with butter for breakfast)))
;//=>(toasted banana bread with butter for)
```

curry 化此函数，变成单一功能函数 （略蛋疼）


```Clojure
(def mrember-curry
  (fn [l]
    (multirember 'curry l)))
 
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```

非 curry 化的重写


```Clojure
(def mrember-curry
  (fn [l]
    (cond
      (null? l) '()
      (= (first l) 'curry) (mrember-curry (rest l))
      true (cons (first l) (mrember-curry (rest l))))))
 
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```

让我们对上面的函数继续 curry 化

```Clojure
(def curry-maker
  (fn [future]
    (fn [l]
      (cond
        (null? l) '()
        (= (first l) 'curry) ((curry-maker future) (rest l))
        true (cons (first l) ((curry-maker future) (rest l)))))))
 
(def mrember-curry (curry-maker 0))
;//=>(chicken with rice)
```

上面这么干虽然没意义，甚至于这个future没有任何意义，目前只是让他弄的像 `insert-g` 替代 `insertL` 一样。 -

下面是使用方法，用函数构造函数，然后用之。。

```Clojure
(def mrember-curry
  (curry-maker curry-maker))
 
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```



```Clojure
(def function-maker
  (fn [future]
    (fn [l]
      (cond
        (null? l) '()
        (= (first l) 'curry) ((future future) (rest l))
        true (cons (first l) ((future future) (rest l)))))))
 
;for yielding mrember-curry when applied to a fcuntion
 
;
(def mrember-curry
  (function-maker function-maker))
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```

现在，我们把 `future` 函数换掉，换成自己的匿名函数 

```Clojure
(def function-maker
  (fn [future]
    (fn [l]
      (cond
        (null? l) '()
        (= (first l) 'curry) ((fn [arg] ((future future) arg)) (rest l))
        true (cons (first l) ((fn [arg] ((future future) arg)) (rest l)))))))
 
(def mrember-curry
  (function-maker function-maker))
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```

当然可以用。


下面，我们将已经2此 curry 化的 function-maker 再次 curry 化...

```Clojure
(def function-maker
  (fn [future]
    ((fn [recfun]
      (fn [l]
        (cond
          (null? l) '()
          (= (first l) 'curry) (recfun (rest l))
        true (cons (first l) ((future future))))))
    (fn [arg] ((future future) arg)))))
;abstraction above to remove l
; just take my word on this for now
```

将上面的函数，拆分成 2 部分

```Clojure
(def M
  (fn [recfun]
    (fn [l]
      (cond
        (null? l) '()
        (= (first l) 'curry) (recfun (rest l))
        true (cons (first l) (recfun (rest l)))))))
 
(def function-maker
  (fn [future]
    (M (fn [arg]
         ((future future) arg)))))
```

现在可以不需要显示的 `function-maker` 引用来重构 `mrember-curry`


```Clojure
;Now we'll change this
(def mrember-curry
  (function-maker function-maker))
;to this
(def mrember-curry
  ((fn [future]
     (M (fn [arg]
          ((future future) arg))))
    (fn [future]
      (M (fn [arg]
           ((future future) arg))))))
```

将上面干的事情提取成 Y ，它将接受一个 M (函数)作为参数

```Clojure
(def Y
  (fn [M]
    ((fn [future]
       (M (fn [arg]
            ((future future) arg))))
      (fn [future]
        (M (fn [arg]
             ((future future) arg)))))))
 
(def mrember-curry (Y M))
 
(println (mrember-curry '(curry chicken with curry rice)))
;//=>(chicken with rice)
```

下面我们将把 Y-combinator 用在求length


```Clojure
;using add1 from chapter 7 not chapter 4
(def add1
  (fn [n]
    (cons '() n)))
 
; now we'll look at using the y-combinator to look at the length of a list
(def L
  (fn [recfun]
    (fn [l]
      (cond
        (null? l) '()
        true (add1 (recfun (rest l)))))))
 
(def length (Y L))
 
(println (length '(curry chicken with curry rice)))
;//=>(() () () () ()) ie 5
```

下面我们重定义了add1， 并且把 L 作为匿名函数定义

```Clojure
(def add1
  (fn [n]
    (+ 1 n)))
 
;just for the sake of it - we'll rewrite length without the L function
(def length
  (Y
    (fn [recfun]
      (fn [l]
        (cond
          (null? l) 0
          true (add1 (recfun (rest l))))))))
 
(println (length '(curry chicken with curry rice)))
;//=>5
```

不用 Y 和 L 重写length

```Clojure
(def length
  ((fn [M]
     ((fn [future]
        (M (fn [arg]
             ((future future) arg))))
     (fn [future]
       (M (fn [arg]
            ((future future) arg))))))
    (fn [recfun]
      (fn [l]
        (cond
          (null? l) 0
          true (add1 (recfun (rest l))))))))
 
(println (length '(curry chicken with curry rice)))
;//=>5
```

本章结束，但是我们想更深入下，用 ycombinator 开始定义一个惰性函数

```Clojure
;building a pair with an S-expression and a thunk leads to a stream
(def first$ first)
 
(def second$
  (fn [str]
    ((second str))))
 
; careful re use of first and second here - as yet undefined!
 
(def build
  (fn [a b]
    (cond
      true (cons a (cons b '())))))
 
(def str-maker
  (fn [next n]
    (build n (fn [] (str-maker next (next n))))))
 
(def int_ (str-maker add1 0))
 
(def even (str-maker (fn [n] (+ 2 n)) 0))
 
;sub1 from chapter 4
(def sub1
  (fn [n]
    (- n 1)))
 
(def frontier
  (fn [str n]
    (cond
      (zero? n) '()
      true (cons (first$ str) (frontier (second$ str) (sub1 n))))))
 
(frontier int_ 10)
;//=>(0 1 2 3 4 5 6 7 8 9)
```

上面的 second 其实是不停的构造新的的函数（入参一直在变化的函数）


上面我们构造了了一个简单的 惰性数据结构，下面的例子更有趣

```Clojure
(def Q
  (fn [str n]
    (cond
      (zero? (rem (first$ str) n)) (Q (second$ str) n)
      true (build (first$ str) (fn [] (Q (second$ str) n))))))
; note new function call rem - re new primitve
 
(def P
  (fn [str]
    (build (first$ str) (fn [] (P (Q str (first$ str)))))))
 
(frontier (P (second$ (second$ int_))) 10)
;//=>(2 3 5 7 11 13 17 19 23 29)
```


