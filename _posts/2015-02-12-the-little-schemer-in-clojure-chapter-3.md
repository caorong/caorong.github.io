---
layout: post
title: "little schemer in Clojure chapter 3 - Cons the Magnificent"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 1 – Cons the Magnificent](http://juliangamble.com/blog/2012/08/03/the-little-schemer-in-clojure-chapter-3/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第三章

注，后文讲用 TLS 代替 The Little Schemer


第一个函数是 `rember`

```Clojure
(def rember
  (fn [a lat]
    (cond
      (null? lat) '()
      true (cond
             (= (first lat) a) (rest lat)
             true (cons (first lat)
                        (rember
                          a (rest lat)))))))
(println (rember 'banana '(apple banana orange)))

; my version

(def rember
  (fn [a lat]
    (cond
      (null? lat) `()
      (= a (first lat)) (rest lat)
      :else (cons
              (first lat)
              (rember a (rest lat))))))

; for test

(rember '1 '(1 2 3))
(rember '2 '(1 2 3))
```

这里主要是用了第二要点，用 cons 合并多个列表


后一个函数 `firsts`

```Clojure
(def firsts
  (fn [l]
    (cond
      (null? l) '()
      true (cons (first (first l))
                 (firsts (rest l))))))

(println (firsts '((large burger) (fries coke) (chocolate sundae))))


```

这里出现了 TLS 的第三要点 `When building a list, describe the first typical element, and then cons it onto the natural recursion` 总的来说就是寻找一个可以迭代的过程， 然后利用它递归的填充list。 这回到了前文 `lat` 的讨论， 其实和前面的解决方法是一样的， 寻找递归过程，然后组合起来。


-----

补充


insertR 

```Clojure
(def insertR
  "
  It takes three arguments: the atoms new and old, and a lat.
  The function insertR builds a lat with new inserted to the right
  of the first occurrence of old
  "
  (fn [new old lat]
    (cond
      ;(null? lat) (seq [])
      (null? lat) '()
      ;(= old (first lat)) (cons (cons (first lat) new) (rest lat))
      (= old (first lat)) (cons old (cons new (rest lat)))
      :else (cons
              (first lat)
              (insertR new old (rest lat))))))

(insertR 2 1 '(1 3 4 1 3))
```

insertL

```Clojure
(def insertL
  (fn [new old lat]
    (cond
      (null? lat) (seq [])
      (= old (first lat)) (cons new lat)
      :else (cons
              (first lat)
              (insertL new old (rest lat))))))

(insertL 2 3 '(1 3 4))
```

subst

```Clojure
(def subst
  "
  replaces the first occurrence of old in the lat with new
  "
  (fn [new old lat]
    (cond
      (null? lat) (seq [])
      (= old (first lat)) (cons new (rest lat))
      :else (cons
              (first lat)
              (subst new old (rest lat))))))

(subst 10 2 '(1 2 3))
```

subst2

```Clojure
(def subst2
  "
  replaces either the first occurrence of o1 or
   the first occurrence of o2 by new
  "
  (fn [new o1 o2 lat]
    (cond
      (null? lat) (seq [])
      :else (cond
              ;(= (first lat) o1) (cons new (rest lat))
              ;(= (first lat) o2) (cons new (rest lat))
              (or (= (first lat) o1)
                  (= (first lat) o2)) (cons new (rest lat))
              :else (cons
                      (first lat)
                      (subst2 new o1 o2 (rest lat)))))))

(subst2 0 1 2 '(1 2 3 4))
```

#### 后面的函数都是前面的multi 版本

multi的是对非multi的得到的结果再进行迭代（直到 (null? lat)）

multirember

```Clojure
(def multirember
  "
  gives as its final value the lat with all occurrences of a removed.
  "
  (fn [a lat]
    (cond
      (null? lat) '()
      :else (cond
              (= a (first lat)) (multirember a (rest lat))
              :else (cons (first lat)
                          (multirember a (rest lat)))))))

(multirember 3 '(1 2 3 4 3 5))
```


```Clojure
(def multiinsertR
  "
  remove multi old
  "
  (fn [new old lat]
    (cond
      (null? lat) '()
      :else (cond
              (= old (first lat))
              (cons old (cons new (multiinsertR new old (rest lat))))
              :else (cons (first lat)
                          (multiinsertR new old (rest lat)))))))

(multiinsertR 0 3 '(1 2 3 4 5 3 6))
```

```Clojure
(def multiinsertL
  (fn [new old lat]
    (cond
      (null? lat) '()
      :else (cond
              (= old (first lat))
              ;; 这里如果写成 (cons new (multiinsertL new old lat)) stackOverflow 仔细想想
              (cons new (cons old (multiinsertL new old (rest lat))))
              :else (cons (first lat)
                          (multiinsertL new old (rest lat)))))))

(multiinsertL 0 3 '(1 2 3 4 5 3 6))
```

上文第一次我就写成了 (multiinsertL new old lat) 这就导致，一旦满足 `(= old (first lat))` 就无限递归了。因为 lat 不会因为递归的深度加深而变小

第四戒律

Always change at least one argument while recurring. It must be changed to be closer to termination. The changing argument must be tested in the termination condition:
when using cdr, test termination with null?

总之就是，递归深度增加，就必须越来越满足（靠近）结束条件

上文就是数组越来越接近 null？

