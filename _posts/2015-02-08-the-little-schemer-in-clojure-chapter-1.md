---
layout: post
title: "little schemer in Clojure chapter 1 - Toys"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 1 – Toys](http://juliangamble.com/blog/2012/07/20/the-little-schemer-in-clojure-chapter-1/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第一章

注，后文讲用 TLS 代替 The Little Schemer

第一章的主题是`数据简化`，在 The Little Schemer 的世界里在语法层面有两种描述数据的方式 - `atoms` 和 `lists`。 注， TLS 里面的 atom 指字符串([或者说不是 list 的都是](http://hyperpolyglot.org/lisp#atom), -ie a non-list), 而在 Clojure 里 `atom` 是事务的一部分。 [doc](https://clojuredocs.org/clojure.core/atom)


测试下 atoms － 使用 atom? function:

```Clojure
(def atom?
  (fn [a]
    not (seq? a))
  )
```

注意， 我们这里没有使用 defn 而是 def ... fn。 故意这样是因为 TLS 将以此作为后文讲匿名函数的铺垫。


说一下 `null` - [原生 Scheme 里的 nil](http://hyperpolyglot.org/lisp) 和 [今天 Clojure 的 nil](http://clojure.org/lisps) 的概念并不一样, 为了使 Clojure 和书里的表现一致，我们使用自己定义的 null? 函数

```Clojure
(def null?
  (fn [a]
    (or
      (nil? a)
      (= () a))))
```

TLS 的重心也在`简化语法`上。所有的结构都以 list, 和 nested listed (list 套 list) 形式表现。Clojure 内置许多类型的数据结构，不过最接近我们想要的是 [Sequence](http://clojure.org/sequences). (不过，他功能不仅是那么单一，因为 Clojure 的 sequence 更像一个可迭代的接口)

第一章的另一个要点是 lists 比起 array 更像是一个 stack。先进后出。TLS 和 Clojure 对应的操作的函数


|   |push|pop|get he remainder|
|---    |---|---|---|
|LTS    | cons| car  | cdr  |
|Clojure| cons| first  |  rest |

注， Clojure 也有 `conj` 不过我们目前仍使用 sequences 和 cons 


第一章的最后一个要点是 nested lists. 简单总结，将一个 list push 到 list 得到一个 nested list (用 cons)。 反之，用 (car/first) 得到 nested list 里面的 list。


