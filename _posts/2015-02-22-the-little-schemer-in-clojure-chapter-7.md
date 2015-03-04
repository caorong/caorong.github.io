---
layout: post
title: "little schemer in Clojure chapter 7 Shadows"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 7 Shadows](http://juliangamble.com/blog/2012/08/31/the-little-schemer-in-clojure-chapter-7-shadows/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第七章

注，后文讲用 TLS 代替 The Little Schemer

这章开始我们主要处理数字计算。建立一个表达式，分解，然后求出他的值。

首先，需要一个函数用来检测输入的是否是一个数字。这个函数名为 `numbered`。但是首先我们将扩展 Clojure 自带的 `number?`

```Clojure
;we need to define number_? to handle true/false
(def number_?
  (fn [a]
    (cond
      (null? a) false
      (number? a) true
      true false)))

(println (number_? 3))
;//=> true
(println (number_? 'a))
;//=> false

```

`numbered` 用来检验是否是数学表达式

```Clojure
(def numbered?
  (fn [aexp]
    (cond
      (atom? aexp) (number_? aexp)
      (= (first (rest aexp)) '+)
         (and
           (numbered? (first aexp))
           (numbered? (first (rest (rest aexp)))))
      (= (first (rest aexp)) '*)
         (and
           (numbered? (first aexp))
           (numbered? (first (rest (rest aexp)))))
      (= (first (rest aexp)) 'exp)
         (and
           (numbered? (first aexp))
           (numbered? (first (rest (rest aexp)))))
      true false)))

(println (numbered? '(a b c)))
;//=> false
(println (numbered? '(3 + (4 * 5))))
;//=> true
(println (numbered? '(3 a (4 * 5))))
;//=> false
```

简化 `numbered?` 

```Clojure
(def numbered?
  (fn [aexp]
    (cond
      (atom? aexp) (number_? aexp)
      true (and
             (numbered? (first aexp))
             (numbered? (first (rest (rest aexp))))))))

(println (numbered? '(a b c)))
;//=> false
(println (numbered? '(3 + (4 * 5))))
;//=> true
(println (numbered? '(3 a (4 * 5))))
;//=> true
```

注意，在这里最后一个表达式返回的是 true ，但是之前是false。 也就是说我们是基于知道输入是一个数学表达式而写的这个函数，是不正确的。

现在看第八戒律。 这里你将递归所有的子部分包括

- 所有list里面的 sublists

- 所有数学表达式里面的子表达式

我们将对数据和代码做区分。处理数据的函数递归数据。处理代码的函数递归代码。

现在你将想到一个名声狼藉的函数 `eval` 。我们将写 eval 的第一个版本的实现，用于解释数学表达式。


```Clojure
(use 'clojure.math.numeric-tower)
; note use of new primitive expt

(def value
  (fn [aexp]
    (cond
      (number_? aexp) aexp
      (= (first (rest aexp)) '+) (+ (value (first aexp)) (value (first (rest (rest aexp)))))
      (= (first (rest aexp)) '*) (* (value (first aexp)) (value (first (rest (rest aexp)))))
      (= (first (rest aexp)) 'exp) (expt (value (first aexp)) (value (first (rest (rest aexp))))))))

(println (value '(1 + 1)))
;//=> 2
(println (value '(2 + 2)))
;//=> 4
(println (value '(3 exp 3)))
;//=> 27
(println (value '(4 b 4)))
;//=> nil
```

现在将实现一个前缀符号的 `value` 实现


```Clojure
(def value
  (fn [aexp]
    (cond
      (number_? aexp) aexp
      (= (first  aexp) '+) (+ (value (rest aexp)) (value  (rest (rest aexp))))
      (= (first (rest aexp)) '*) (* (value (first aexp)) (value (first (rest (rest aexp)))))
      (= (first (rest aexp)) 'exp) (expt (value (first aexp)) (value (first (rest (rest aexp))))))))
```

现在我们将 `提取数据` 的函数和 `提取operator` 的函数提取出来。

```Clojure
(def first-sub-exp
  (fn [aexp]
    (first (rest aexp))))

(def second-sub-exp
  (fn [aexp]
    (first (rest (rest aexp)))))

(def operator
  (fn [aexp]
    (first aexp)))

(def value
  (fn [aexp]
    (cond
      (number_? aexp) aexp
      (= (operator aexp) '+) (+ (value (first-sub-exp aexp)) (value  (second-sub-exp aexp)))
      (= (operator aexp) '*) (* (value (first-sub-exp aexp)) (value (second-sub-exp aexp)))
      (= (operator aexp) 'exp) (expt (value (first-sub-exp aexp)) (value (second-sub-exp aexp))))))

(println (value '(+ 1 1)))
;//=> 2
(println (value '(* 2 2)))
;//=> 4
(println (value '(exp 3 3)))
;//=> 27
(println (value '(b 4 4)))
;//=> nil
```

现在来呈现以下 抽象的威力 － 回到中缀表达式


```Clojure
(def first-sub-exp
  (fn [aexp]
    (first  aexp)))

(def operator
  (fn [aexp]
    (first (rest aexp))))

(println (value '(1 + 1)))
;//=> 2
(println (value '(2 * 2)))
;//=> 4
(println (value '(3 exp 3)))
;//=> 27
(println (value '(4 b 4)))
```

是时候将下一个戒律了 － `使用帮助函数抽象一些表达式`。 可以理解为提取方法。它有助于你为未来进行大范围的重构。

后文将重定义一个数学模型。

现在要写一个 `null?` 函数，用于测试，之所以重新定义，是为了让我们的api抽象到最简单。

```Clojure
;This is what we had
(def null?
  (fn [a]
    (or
      (nil? a)
      (= () a))))

;This is what they're proposing 
(def null?
  (fn [s]
    (and
      (atom? s)
      (= s '()))))

```

我们将脱离原本的数学符建立自己的数学计算语法。

从 `zero?` 开始

```Clojure
(def zero_?
  (fn [n]
    (null? n)))

(println (zero_? '()))
;//=>true
(println (zero_? 0))
;=>false
```

也许没弄明白现在在干什么

举个例子。 现在 0 是 () , 1 是 (()), 2 是 (() ())

现在定义 `add1`。

```Clojure
(def add1
  (fn [n]
    (cons '() n)))

(println (add1 '()))
;//=>(()) ie 1
(println (add1 '(())))
;//=> (() ()) ie 2
```

`sub1`

```Clojure
(def sub1
  (fn [n]
    (rest n)))

(println (sub1 '(()())))
;//=>(()) ie 1
(println (sub1 '(())))
;//=>() ie 0
(println (sub1 '()))
;//=> () unfortunately with our implementation sub1 of zero returns zero
```

定义加法

```Clojure
(def +_
  (fn [n m]
    (cond
      (zero_? m) n
      true (add1 (+_ n (sub1 m))))))

(println (+_ '() '()))
;//=> ()
;//  zero plus zero is zero
(println (+_ '(()) '(())))
;//=> (()()) ie 1 plus 1 is 2
```

检验是否是数字

```Clojure
(def number_?
  (fn [n]
    (cond
      (null? n) true
      true (and
             (null? (first n))
             (number_? (rest n))))))

(println (number_? '() ))
;//=>true
(println (number_? '(()) ))
;//=>true
(println (number_? '((())) ))
;//=>false
(println (number_? '(()()) ))
;//=>true
```

#### 总结

为什么这章叫做shadows？ 我也没弄明白 。。。（我觉得应该是后半部分的抽象吧，一样的接口不一样的实现）

这里我们以数学表达为基础，开始写简单的解释器了。我们相比之前仅仅增加了一个基础函数 `expt` (我也没看懂expt是啥， 我理解应该就是次方的实现，但不懂为什么又造了个名字)









