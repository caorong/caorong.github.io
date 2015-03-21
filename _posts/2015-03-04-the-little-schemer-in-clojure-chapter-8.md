---
layout: post
title: "little schemer in Clojure chapter 8 Friends and Relations"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


翻译自 [juliangamble - The Little Schemer in Clojure – Chapter 8 Friends and Relations](http://juliangamble.com/blog/2012/09/07/the-little-schemer-in-clojure-chapter-8-friends-and-relations/) 

这是 [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/) to Clojure 的第八章

注，后文讲用 TLS 代替 The Little Schemer


这章关注sets, relations 和 functions（数学意义上的）

```Clojure
(def set_?
  "a list contains a set of unique items."
  (fn [lat]
    (cond
      (null? lat) true
      true (cond
             (member? (first lat) (rest lat)) false
             true (set_? (rest lat))))))
 
(println (set_? '(toasted banana bread with butter for breakfast)))
;//=>true
(println (set_? '(breakfast toasted banana bread with butter for breakfast)))
;//=>false

; 简化版

(def _set?
  (fn [lat]
    (cond
      (null? lat) true
      (member? (first lat) (rest lat)) false
      :else (_set? (rest lat))
      )))
```

假设他用了[chapter5](/clojure/2015/02/21/the-little-schemer-in-clojure-chapter-5.html) 实现的 `member?`


给一个非set的list 把它变成set

```Clojure
(def makeset
  (fn [lat]
    (cond
      (null? lat) '()
      (member? (first lat) (rest lat)) (makeset (rest lat))
      true (cons (first lat) (makeset (rest lat))))))
 
(println (makeset '(breakfast toasted banana bread with butter for breakfast)))
;//=> (toasted banana bread with butter for breakfast)
(println (set_? (makeset '(breakfast toasted banana bread with butter for breakfast))))
;//=>true
```

用 [chapter5](/clojure/2015/02/21/the-little-schemer-in-clojure-chapter-5.html) 的multirember 重构

```Clojure
(def makeset
  (fn [lat]
    (cond
      (null? lat) '()
      true (cons (first lat) (makeset (multirember (first lat) (rest lat)))))))
 
(println "")
(println "makeset - refactored with multirember")
(println (makeset '(breakfast toasted banana bread with butter for breakfast)))
;//=> (breakfast toasted banana bread with butter for)
; note other way around
(println (set_? (makeset '(breakfast toasted banana bread with butter for breakfast))))
;//=>true
```


```Clojure
(def subset?
  "check if one set is a subset of another"
  (fn [set1 set2]
    (cond
      (null? set1) true
      true (cond
             (member? (first set1) set2) (subset? (rest set1) set2)
             true false))))
 
(println (subset? '(banana butter) '(breakfast toasted banana bread with butter for breakfast)))
;//=>true
(println (subset? '(banana butter) '(toasted banana bread with butter for breakfast)))
;//=>true
(println (subset? '(peanut butter) '(toasted banana bread with butter for breakfast)))
;//=>false

; simple version 
(def subset?
  (fn [set1 set2]
    (cond
      (null? set1) true
      (member? (first set1) set2) (subset? (rest set1) set2)
      true false)))

; refactor with `and`

(def subset?
  (fn [set1 set2]
    (cond
      (null? set1) true
      true (and
             (member? (first set1) set2)
             (subset? (rest set1) set2)))))
 
(println (subset? '(banana butter) '(breakfast toasted banana bread with butter for breakfast)))
;//=>true
(println (subset? '(banana butter) '(toasted banana bread with butter for breakfast)))
;//=>true
(println (subset? '(peanut butter) '(toasted banana bread with butter for breakfast)))
;//=>false

```

检测是否2个 `set` 相同

```Clojure
(def eqset?
  (fn [set1 set2]
    (cond
      (subset? set1 set2) (subset? set2 set1)
      true false)))
 
(println (eqset? '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>false
(println (eqset? '(toasted banana bread) '(toasted banana bread)))
;//=>true
(println (subset? '(toasted peanut butter for breakfast) '(toasted banana bread )))
;//=>false

; simply one line
(def eqset?
  (fn [set1 set2]
      (and
        (subset? set1 set2)
        (subset? set2 set1))))

```

检测是否有交集？

```Clojure
(def intersect?
  (fn [set1 set2]
    (cond
      (null? set1) false
      true (cond
             (member? (first set1) set2) true
             true (intersect? (rest set1) set2)))))
 
(println (intersect? '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>true
(println (intersect? '(toasted banana bread) '(toasted banana bread)))
;//=>true
(println (intersect? '(toasted peanut butter for breakfast) '(toasted banana bread )))
;//=>true
(println (intersect? '(strawberry yoghurt) '(toasted banana bread )))
;//=>false

; simple

(def intersect?
  "if set1 has at least one atom in set2"
  (fn [set1 set2]
    (cond
      (null? set1) false
      (member? (first set1) set2) true
      :else (intersect? (rest set1) set2))))
```

取 `set` 的交集

```Clojure
; 找到相同的，然后cons起来
(def intersect
  (fn [set1 set2]
    (cond
      (null? set1) '()
      (member? (first set1) set2) (cons (first set1) (intersect (rest set1) set2))
      true (intersect (rest set1) set2))))
 
(println (intersect '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>(toasted banana bread)
(println (intersect '(toasted peanut butter for breakfast) '(toasted banana bread )))
;//=>(toasted)
(println (intersect '(strawberry yoghurt) '(toasted banana bread )))
;//=>()

;找到不交叉的，不cons，进入下层递归
(def intersect
  (fn [set1 set2]
    (cond
      (null? set1) '()
      (not (member? (first set1) set2)) (intersect (rest set1) set2)
      :else (cons (first set1) (intersect (rest set1) set2)))))
```

与交叉相反， 并集

```Clojure
(def union
  (fn [set1 set2]
    (cond
      (null? set1) set2
      (member? (first set1) set2) (union (rest set1) set2)
      true (cons (first set1) (union (rest set1) set2)))))
 
(println (union '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>(breakfast toasted banana bread with butter for breakfast)
;// note not a set since not given a set
(println (union '(toasted peanut butter for breakfast) '(toasted banana bread )))
;//=>(peanut butter for breakfast toasted banana bread)
(println (union '(strawberry yoghurt) '(toasted banana bread )))
;//=>(strawberry yoghurt toasted banana bread)

```

```Clojure
(def nunion
  "return all the atoms in set1 that are not in set2"
  (fn [set1 set2]
    (cond
      (null? set1) '()
      (member? (first set1) set2) (nunion (rest set1) set2)
      :else (cons (first set1) (nunion (rest set1) set2)))))

(nunion '(1 2 3) '(2 3 4))
```

对于 set1 补集 

```Clojure
(def complement_
  (fn [set1 set2]
    (cond
      (null? set1) '()
      (member? (first set1) set2) (complement_ (rest set1) set2)
      true (cons (first set1) (complement_ (rest set1) set2)))))
 
(println (complement_ '(toasted banana bread) '(breakfast toasted banana bread with butter for breakfast)))
;//=>()
(println (complement_ '(toasted peanut butter for breakfast) '(toasted banana bread )))
;//=>(peanut butter for breakfast)
(println (complement_ '(strawberry yoghurt) '(toasted banana bread )))
;//=>(strawberry yoghurt)
```

多个集合的交集 (大于2)

```Clojure
(def intersect-all
  (fn [l-set]
    (cond
      (null? (rest l-set)) (first l-set)
      true (intersect (first l-set) (intersect-all (rest l-set))))))
 
(println "")
(println "intersect-all")
(println (intersect-all
           '(
              (toasted banana bread)
              (breakfast toasted banana bread with butter for breakfast)
              (toasted peanut butter for breakfast)
              (toasted banana bread ))))
;//=>(toasted)
```

后面开始引出 `pair` 的概念， 一个值对。


```Clojure
(def a-pair?
  "different but related object"
  (fn [x]
    (cond
      (atom? x) false
      (null? x) false
      (null? (rest x)) false
      (null? (rest (rest x))) true
      :else false)))

(a-pair? '(2 'a))
(a-pair? '(2))
```

从 `first` 和 `second` 开始

```Clojure
(def first_
  (fn [p]
    (cond
      true (first p))))
 
(println (first_ '(a b)))
;//=>a
 
(def second_
  (fn [p]
    (cond
      true (first (rest p)))))
 
(println (second_ '(a b)))
;//=>b
```

```Clojure
(def build
	" build a pair"
  (fn [a b]
    (cond
      true (cons a (cons b '())))))
 
(println (build 'a 'b))
;//=>(a b)
```

扩展下，取第三个


```Clojure
(def third_
  (fn [p]
    (cond
      true (first (rest (rest p))))))
 
(println (third_ '(a b c)))
;//=>c
```

进一步观察，`set<pair>` 的话，每个pair的第一个值的集合应该是一个 `set` 



```Clojure
(def firsts
  (fn [l]
    (cond
      (empty? l) '()
      true (cons (first (first l))
        (firsts (rest l))))))
 
(println "")
(println "firsts")
(println (firsts '((8 3)(4 2)(7 6)(6 2)(3 4))))
;//=>(8 4 7 6 3)
 
(def fun?
	"if all first one in pair is set?"
  (fn [rel]
    (set? (firsts rel))))
 
(println "")
(println "fun? - refactored to use set? and firsts")
(println (fun? '((4 3)(4 2)(7 6)(6 2)(3 4))))
;//=>false
(println (fun? '((8 3)(4 2)(7 6)(6 2)(3 4))))
;//=>true
(println (fun? '((8 3)(4 2)(7 1)(6 0)(9 5))))
;//=>true
```

某种情况下需要翻转pair list 里面的所有 pair


```Clojure
(def revrel
  (fn [rel]
    (cond
      (null? rel) '()
      true (cons
             (build
               (second_ (first rel))
               (first_ (first rel)))
             (revrel (rest rel))))))
 
(println (revrel '((4 3)(4 2)(7 6)(6 2)(3 4))))
;//=>((3 4) (2 4) (6 7) (2 6) (4 3))
(println (revrel '((8 3)(4 2)(7 6)(6 2)(3 4))))
;//=>((3 8) (2 4) (6 7) (2 6) (4 3))
(println (revrel '((8 3)(4 2)(7 1)(6 0)(9 5))))
;//=>((3 8) (2 4) (1 7) (0 6) (5 9))
```

fullfun 和 fun 相同只不过 针对 pair 的value 

```Clojure
(def fullfun?
  (fn [fun]
    (set_? (seconds_ fun))))
 
(println (fullfun? '((4 3)(4 2)(7 6)(6 2)(3 4))))
;//=>false
(println (fullfun? '((8 3)(4 2)(7 6)(6 2)(3 4))))
;//=>false
(println (fullfun? '((8 3)(4 2)(7 1)(6 0)(9 5))))
;//=>true
```

fullfun? 的更简易版本，即域值(domain)是独一无二的，即使他们的pair翻转, 我们称之为 one-to-one? (没理解为什么这个名字)

```Clojure
(def one-to-one?
  (fn [fun]
    (fun? (revrel fun))))
 
(println (one-to-one? '((4 3)(4 2)(7 6)(6 2)(3 4))))
;//=>false
(println (one-to-one? '((8 3)(4 2)(7 6)(6 2)(3 4))))
;//=>false
(println (one-to-one? '((8 3)(4 2)(7 1)(6 0)(9 5))))
```


















