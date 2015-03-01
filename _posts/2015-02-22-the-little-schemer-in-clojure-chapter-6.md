---
layout: post
title: "little schemer in Clojure chapter 6 oh-my-gawd-its-full-of-stars"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 6 oh-my-gawd-its-full-of-stars](http://juliangamble.com/blog/2012/08/19/the-little-schemer-in-clojure-chapter-6-oh-my-gawd-its-full-of-stars/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第五章

注，后文讲用 TLS 代替 The Little Schemer


本章将介绍如何处理嵌套的list

先看 `leftmost` 函数（高阔 `non-atom?` 和 `not` 函数）。作用是找到list里面最最左边的 S表达式包括 ()。


```Clojure
(def not_
  (fn [b]
    (cond
      b false
      true true)))
 
(println (not_ (= 'a 'b)))
 
(def non-atom?
  (fn [s]
    (not_ (atom? s))))
 
(println (non-atom? (quote thai)))
 
(def leftmost
  (fn [l]
    (println "(leftmost " l)
    (println (non-atom? l))
    (cond
      (null? l) '()
      (non-atom? (first l)) (leftmost (first l))
      true (first l))))
 
(println (leftmost (quote ((((pad))thai))chicken())))

; my version
(def leftmost
  "The function leftmost finds the leftmost
   atom in a non-empty list of S-expressions that does not contain the empty list"
  (fn [l]
    (cond
      (null? l) '()
      (atom? (first l)) (first l)
      :else (leftmost (first l)))))

(leftmost '(2 (3 1)))
```

我们现在将重写 `rember*` 他的作用是将所有等于 a 的list给去除掉。

```Clojure
(def rember*
  (fn [a l]
    (cond
      (null? l) '()
      (non-atom? (first l)) (cons (rember* a (first l)) (rember* a (rest l)))
      true (cond
             (= (first l) a) (rember* a (rest l))
             true (cons (first l) (rember* a (rest l)))))))
 
(println (rember* 'bacon '(((bbq sauce)) (with (egg and (bacon))))))

; v2
(def rember*
  "
  remove a from nested list l
  "
  (fn [a l]
    (cond
      (null? l) '()
      (atom? (first l)) (cond
                          (= a (first l)) (rember* a (rest l))
                          :else (cons (first l) (rember* a (rest l))))
      :else (cons (rember* a (first l)) (rember* a (rest l))))))

(rember* 1 '(1 (2 1 3) 3))
```

`insertR*` 

```Clojure
(def insertR*
  (fn [new old l]
    (cond
      (null? l) '()
      (non-atom? (first l)) (cons (insertR* new old (first l)) (insertR* new old (rest l)))
      true (cond
             (= (first l) old) (cons old (cons new (insertR* new old (rest l))))
             true (cons (first l) (insertR* new old (rest l)))))))
 
(println (insertR* 'chicken 'baked '(((baked)) (with roast) vegetables)))

;; v2
(def insertR*
  (fn [new old l]
    (cond
      (null? l) '()
      (atom? (first l)) (cond
                          (= old (first l))
                          (cons old (cons new (insertR* new old (rest l))))
                          :else (cons (first l) (insertR* new old (rest l))))
      :else (cons (insertR* new old (first l)) (insertR* new old (rest l)))
      )
    )
  )

(insertR* 0 1 '(3 1 1 7))
(insertR* 0 1 '(3 1 (3 2 1 3) 2 (3 1 (3 (3 0 8) 8 1)) 1 7))
```

这里要修订下之前的戒律

	When recurring on a list of atoms, lat, or a tup, tup, ask two questions about them, and use (rest lat) or (rest tup) for the natural recursion.

	When recurring on a list of S-expressions, l, ask three questions: (null? l), (atom? (first l)), and (non-atom? (first l)); and use (first l) and (rest l) for the natural recursion.

	When recurring on a number, n, ask two questions, and use (sub1 n) for the natural recursion.

主要是增加了中间一条，对于嵌套的 list 的处理方法。


 `occour*` 将 a 出现的次数打印出来

```Clojure
(def occur*
  (fn [a l]
    (cond
      (null? l) 0
      (non-atom? (first l)) (+_ (occur* a (first l)) (occur* a (rest l)))
      true (cond
             (= (first l) a) (add1 (occur* a (rest l)))
             true (occur* a (rest l))))))
 
(println (occur* 'creamy '(((creamy)) new (york (cheesecake)) with a ((creamy) latte))))

;; or
(def occur*
  (fn [a l]
    (cond
      (null? l) 0
      (atom? (first l)) (cond
                          (= a (first l)) (add1 (occur* a (rest l)))
                          :else (occur* a (rest l)))
      :else (+_ (occur* a (first l)) (occur* a (rest l)))
      )))

(occur* 1 '(1 2 3 1))
(occur* 1 '(1 2 (1 2 (1 2)) 1))
```

`subst*` mutlti replace

```Clojure
(def subst*
  (fn [new old l]
    (cond
      (null? l) '()
      (non-atom? (first l)) (cons (subst* new old (first l)) (subst* new old (rest l)))
      true (cond
             (= (first l) old) (cons new (subst* new old (rest l)))
             true (cons (first l) (subst* new old (rest l)))))))
 
(println (subst* 'baked 'creamy '(((creamy) cheesecake) (with (hot (espresso))))))

;; or
(def subst*
  (fn [new old l]
    (cond
      (null? l) '()
      (atom? (first l)) (cond
                          (= old (first l)) (cons new (subst* new old (rest l)))
                          :else (cons (first l) (subst* new old (rest l))))
      :else (cons (subst* new old (first l)) (subst* new old (rest l))))))

(subst* 0 1 '(1 2 3 4 1 3 1))
(subst* 0 1 '(1 (32 1 (33 33 2 1 2)) 3 4 1 3 1))
```

`insertL*` 将 new 插入 l 里面 old 的左边

```Clojure
(def insertL*
  (fn [new old l]
    (cond
      (null? l) '()
      (non-atom? (first l))
        (cons
          (insertL* new old (first l))
          (insertL* new old (rest l)))
      true (cond
        (= (first l) old)
          (cons new
            (cons old
              (insertL*
                new old (rest l))))
        true (cons (first l)
          (insertL*
            new old (rest l)))))))
 
(println (insertL* 'fresh 'creamy '(((creamy) cheesecake) (with (hot (espresso))))))

;; or
(def insertL*
  (fn [new old l]
    (cond
      (null? l) '()
      (atom? (first l)) (cond
                          (= old (first l)) (cons new (cons old (insertL* new old (rest l))))
                          :else (cons (first l) (insertL* new old (rest l))))
      :else (cons (insertL* new old (first l)) (insertL* new old (rest l))))))

(insertL* 0 1 '(1 2 3 4 1 3 1))
(insertL* 0 1 '(1 (32 1 (33 33 2 1 2)) 3 4 1 3 1))
```

member* 

```Clojure
(def member*
  (fn [a l]
    (cond
      (null? l) '()
      (atom? (first l))
        (or
          (= (first l) a)
          (member* a (rest l)))
      true (or
        (member* a (first l))
        (member* a (rest l))))))
 
(println (member* 'creamy '(((creamy) cheesecake) (with (hot (espresso))))))

;; or
(def member*
  (fn [a l]
    (cond
      (null? l) false
      (atom? (first l)) (or (= a (first l)) (member* a (rest l)))
      :else (or (member* a (first l)) (member* a (rest l))))))

(member* 1 '(3 2 1))
(member* 1 '(3 (3 2 (1)) 5))
```

`eqlist?` 判断list是否相同

```Clojure
(def eqlist?
  (fn [l1 l2]
    (cond
      (and (null? l1) (null? l2)) true
      (or (null? l1) (null? l2)) false
      (and (non-atom? (first l1)) (non-atom? (first l2)))
        (and (eqlist? (first l1) (first l2))
             (eqlist? (rest l1) (rest l2)))
      (or (non-atom? (first l1)) (non-atom? (first l2))) false
      true (and
        (eqan? (first l1) (first l2))
        (eqlist? (rest l1) (rest l2))))))
 
(println (eqlist? '(with (hot (espresso))) '(with (hot (espresso)))));//=>true
(println (eqlist? '(with (hot (espresso))) '((creamy) cheesecake)));//=>false


;; or
(def eqlist?
  (fn [l1 l2]
    (cond
      (and (null? l1) (null? l2)) true
      (and (atom? l1) (atom? l2)) (= l1 l2)
      ; l1 atom  l2 list or null
      (atom? l1) false
      (atom? l2) false
      ; all lists
      :else (and (eqlist? (first l1) (first l2)) (eqlist? (rest l1) (rest l2)))
      )))

(eqlist? '(1) '(2))
(eqlist? '(1) '(1))
(eqlist? '(1 2) '(1 2))
(eqlist? '(1 (1 (1))) '(1 (1 (1))))
(eqlist? '(1 (1 (1))) '(1 (1 (1 2))))
```

后文讲了些重构代码。不过感觉有些太牵强，就不写了。但有一个要点，简化前先保证函数是正确的。

#### 总结

这里我们仅仅是将以前的函数扩展了下，让他能处理嵌套的list。



