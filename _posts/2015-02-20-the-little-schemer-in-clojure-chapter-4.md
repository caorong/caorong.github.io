---
layout: post
title: "little schemer in Clojure chapter 4 - Numbers Games"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 4 – Numbers Games](http://juliangamble.com/blog/2012/08/10/the-little-schemer-in-clojure-chapter-4-numbers-games/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第四章

注，后文讲用 TLS 代替 The Little Schemer

你会发现在前面三章里面，ch1 讲了list ，ch2 讲了取list里面的数据，ch3 讲了如何组合一个list。本书最重要的是说如何用 `递归` 和 `lambdas` 用最简单的基元操作数据结构。

现在我们利用这些原则做一个自循环解释器。我们也想能够进行数学计算， 并用尽量少的基元实现。我们将用 `equals`, `add1`, `sub1` 建立起乘法，除法，次方。 让我们开始吧！

我们的第一个函数是 +_（加法）带 `_` 是为了和 Clojure 自带的 + 做区分。并定义 add1， sub1.

```Clojure
(def zero_?
  (fn [n]
    (= 0 n)))

(def add1
  (fn [n]
    (+ n 1)))

(def sub1
  (fn [n]
    (- n 1)))

(def +_
  (fn [n m]
    (cond
      (= 0 m) n
      :else (add1 (+_ n (sub1 m))))))
```

看了上面的代码可能会有点沮丧，为什么把cpu已经实现了的＋ － 重新定义？并限定了只能支持 Integer。

这是为了帮你建立数字的递归化实现的思路。  一部分的list 的计算操作是机遇自己的 `numberic tower`－ 你也能看到Clojure处理数字的方式 － 部分非常不合理。

另一个原因是为了能从底层开始一步一步构建一个自循环解释器。

下一个函数 减

```Clojure
(def -_
  (fn [n m]
    (cond
      (zero_? m) n
      :else (sub1 (-_ n (sub1 m))))))

(-_ 4 3)
```

这里我们扩展了 `lat` (a List-ATom) 的概念 － TLS里面称之为 `tup`，就是没有内嵌 list 的 Integer。


``` Clojure
(def addup
  (fn [tup]
    (cond
      (null? tup) 0
      :else (+_ (first tup) (addup (rest tup))))))

(addup '(1 2 3))
```

乘法

``` Clojure
(def *_
  (fn [n m]
    (cond
      (zero_? m) 0
      :else (+_ n (*_ n (sub1 m))))))

(*_ 3 2)
```

又造了个乘法的轮子

现在要讲一个第5戒律。 这些可能让你感觉有点死记硬背。其实不然，如果你自己写一个递归的实现的时候，这戒律可以用来当公理来用。

第五戒律： 当构建加法操作的时候，用0作为结束条件的返回值， 因为加 0 不会影响最终结果。

同理，当构建乘法的时候，用1 作为结束条件的返回值。

同理，当用 `cons` 组 `list` 的时候用 `'()`。

现在看 `tup+` 这函数可以将 2个 tup 里面的数字按位置分别加起来，并以一个 tup 返回。这在矩阵操作经常要用到。

```Clojure
(def tup+
  (fn [tup1 tup2]
    (cond
      (null? tup1) tup2
      (null? tup2) tup1
      ;(and (null? tup1) (null? tup2)) '()
      :else (cons (+_ (first tup1) (first tup2))
                  (tup+ (rest tup1) (rest tup2))))))

(tup+ '(1 2 3) '(1 2))
```

`>` 大于

```Clojure
(def >_
  (fn [n m]
    (cond
      ; false 在上边，以便 n==m
      (zero_? n) false
      (zero_? m) true
      :else (>_ (sub1 n) (sub1 m))
      )))

(>_ 3 2)
(>_ 2 3)
```

`<` 小于， 和大于非常相似

```Clojure
(def <_
  (fn [n m]
    (cond
      (zero_? m) false
      (zero_? n) true
      :else (<_ (sub1 n) (sub1 m))
      )))

(<_ 3 2)
(<_ 3 3)
```

`=` 等于

```Clojure
(def =_
  (fn [n m]
    (cond
      (or (>_ n m) (<_ n m)) false
      :else true
      )))

(=_ 3 2)
(=_ 0 2)
```

次方

```Clojure
(def exp
  (fn [n m]
    (cond
      (=_ 0 m) 1
      :else (*_ n (exp n (sub1 m))))))

(exp 2 3)
```

除法

```Clojure
(def div_
  (fn [n m]
    (cond
      (<_ n m) 0
      :else (add1 (div_ (-_ n m) m)))))

(div_ 30 3)
```

现在我们将用上面的基础函数对 atoms 和 lats 进行操作 － 这些在后面的自循环解释器用到。

长度函数

```Clojure
(def length_
  (fn [lat]
    (cond
      (null? lat) 0
      :else (add1 (length_ (rest lat))))))

(length_ '(1 2 3))
```

```Clojure
(def pick_
  "
  从lat里面取出第n个atom 
  "
  (fn [n lat]
    (cond
      (zero_? (sub1 n)) (first lat)
      :else (pick_ (sub1 n) (rest lat)))))

(pick_ 2 '(1 2 3 4))   ; (2)
```

```Clojure
(def rempick_
  "
  返回lat里面除了第n个元素外的list
  "
  (fn [n lat]
    (cond
      (zero_? (sub1 n)) (rest lat)
      :else (cons (first lat) (rempick_ (sub1 n) (rest lat))))))

(rempick_ 3 '(1 2 3 4))  ;(1 2 4)
```

这里小小作弊了下，用了Clojure内置的 `number?` 函数，判断是否是数字。

这个函数将返回lat里面的所有非数字的集合

```Clojure
(def no-nums
  "remove all numbers from lat"
  (fn [lat]
    (cond
      (null? lat) '()
      (number? (first lat)) (no-nums (rest lat))
      :else (cons (first lat) (no-nums (rest lat))))))

(no-nums '(1 "xx" 3 "bb"))   ;("xx" "bb")
```

与上面相反

```Clojure
(def all-nums
  "extracts a tup from lat"
  (fn [lat]
    (cond
      (null? lat) '()
      (number? (first lat)) (cons (first lat) (all-nums (rest lat)))
      :else (all-nums (rest lat)))))

(all-nums '(1 "xx" 3 "bb"))
```

最后一个函数是 `eqan?` 

```Clojure
(def eqan?
  "check number or atom is equal
  Remember to use =for numbers and eq? for all other atoms.
  "
  (fn [a1 a2]
    (cond
      (and (number? a1) (number? a2)) (=_ a1 a2)
      (or (number? a1) (number? a2)) false
      :else (= a1 a2))))

(eqan? 1 1)   ;true
(eqan? '1 '1) ;true
```

#### 总结

这章引入了2个新的基元，`add1` 和 `sub1` 并以它为基础建立了上面所有的函数。

我们目前接触的基础函数有: `atom?`, `null?`, `first`, `rest`, `cond`, `fn`, `def`, `empty?`, `=`, `cons`, `add1` and `sub1`。这些都是在后面实现自循环解释器的基础。

--- 
补充函数 

```Clojure
(def occur
  "counts the number of times an atom a appears in a lat"
  (fn [a lat]
    (cond
      ;(null? lat) (+_  0 (occur a (rest lat)))
      (null? lat) 0
      ;(= a (first lat)) (+_  1 (occur a (rest lat)))
      (= a (first lat)) (add1 (occur a (rest lat)))
      :else (occur a (rest lat)))))

(occur 1 '(1 3 1 3 1))  ;3
```

```Clojure
(def one?
  "(one? n) is #t if n is 1 and #f (Le., false) otherwise."
  (fn [n]
    (= n 1)))

(one? 1)  ;true
```

前面rempick的更精简版本 用了 `ones?`

```Clojure
(def rempick_
  "simplify function"
  (fn [n lat]
    (cond
      (one? n) (rest lat)
      :else (cons (first lat) (rempick_ (sub1 n) (rest lat))))))

(rempick_ 2 '(1 2 3)) ;(1 3)
```


