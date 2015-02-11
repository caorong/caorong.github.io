---
layout: post
title: "little schemer in Clojure chapter 2 - Do It, Do It Again"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 2 – Do It, Do It Again](http://juliangamble.com/blog/2012/07/29/the-little-schemer-in-clojure-chapter-2/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第二章

注，后文讲用 TLS 代替 The Little Schemer

这章主要讲函数 - lat 和 member 

`lat` 在 Scheme 里表示不内嵌 list 的 list。 Clojure 版本的 lat

```Clojure
(def lat?
  (fn [l]
    (cond
      (null? l) true
      (and (seq? l)
           (atom? (first l)))
             (lat? (rest l))
      true false)))

;; 更清楚版本

(def lat?
  (fn [l]
    (cond
      (null? l) true
      (and (seq? l)
           (atom? (first l))) (lat? (rest l)) true
      :else false)))

```

这里有2点要注意，第一条也是最主要的是 用 `def ... fn[]` 而不是 `defn` 表示函数，这是本书惯例，为了让读者习惯，函数默认都是匿名的，事实上平时用的函数只是给他加了个标签。这也为了我们以后能对匿名函数的使用更有感觉。

另一点是介绍了本书 10大戒律的应用。这10条戒律有助于你更能以递归的方式思考。第一条是，`you should always ask null?  as the first question in expressing any function` 习惯以 `null?` 开始。 这点对于 Clojure 惯用 `empty?`， 不过为了和书一致，用了第一章实现的`null?`

这个背后的问题是`神马是不带嵌套的 atoms list` 以及 `为什么这么做？` 本书第一大理论是使用`递归` 而不是 `迭代` 解决问题。 关键点是解决问题前，（在遇到内嵌list时）用递归，学会在没有内嵌 list 时用[基本情况](http://zh.wikipedia.org/wiki/%E9%80%92%E5%BD%92)解决问题。

下一个函数， member?

```Clojure
; 原版本
(def member?
  (fn [a lat]
    (cond
      (null? lat) false
      true (or
        (= (first lat) a)
        (member? a (rest lat)))) ))

; 我觉的更清晰版本
(def member?
  (fn [a lat]
    (cond
      (null? lat) false
      :else (or
              (= (first lat) a)
              (member? a (rest lat))))))
```


为什么要实现这个函数？ 因为他是本书第一个有用的函数。他是我们在后面建立复杂函数的基础函数，并在之后继续巩固递归技巧。







