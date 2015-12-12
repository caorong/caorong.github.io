---
layout: post
title: "from vim to emacs"
description: ""
category: "tools"
tags: [tools, emacs, vim, linux]
published: false
---


之前很久都在折腾python的项目，clojure 放下很久了，然后意外发现以前连载了一半的 clojure 教程完结了 ＝ ＝， 开始看起来。

里面有一章 [emacs 的入门篇](http://www.braveclojure.com/basic-emacs/)，于是顺便学习下传说中强大的 emacs 后面是本片的笔记。

C = Ctrl 
M = alt/option(Mac)

emacs C-G = vim esc

#### file/buffer

C-x k   kill buffer
C-x b   change buffer
C-x C-f open/new file
C-x C-s save file

#### misc

q       close win

C-h k [key-binding]     describe the func bound to the key-binding 
C-h f   describe func

M-x     execute entered func
major-mode              change file's major mode
package-list-packages   emacs plugin repo
package-refresh-contents
package-install

replace-string

cider-jack-in           open lein repl

paredit-mode            enable/disable paredit mode


#### edit/movement

C-a     begin of line
C-e     end of line
M-m     Move to first non-whitespace character on the line
C-f     forward one character
C-b     back
M-f     forward one world
M-b     back
C-s     Regex search (C-s again to move to next match)
C-r     C-s in reverse
M-<     begin of buffer
M->     end
M-g g   go to line xxx

Tab     indent
C-j     vim o
M-/     show auto complete panel
M-\     delete all ' ' '\t' around point

C-k     kill line after point
C-/     undo
C-g C-_ redo


#### selection/regions

C-spc   start mark

M-w     vim y
M-d     vim x
C-y     vim p
M-y     Cycle through kill ring after yanking(remove one history)

#### windows

C-x o   vim Ctrl ww
C-x 1   close all win except current
C-x 2   :sp 
C-x 3   :vsp
C-x 0   close current


#### emacs config
(setq initial-frame-alist '((top . 0) (left . 0) (width . 80) (height . 20)))


#### cider

C-x C-e     cider-eval-last-expression
C-u C-x C-e cider-eval-last-expression and prints result after point

C-↑         REPL history
C-↓
C-←         (+ 1 (* 2 3 4)) ->  (+ 1 (* 2 3) 4)
C-→         (+ 1 (* 2 3) 4) -> (+ 1 (* 2 3 4))
C-enter     Close parentheses and evaluate

C-c M-n     Switch to namespace of current buffer
C-c C-k     load file and compile according to major mode
C-c C-d C-d Display documentation for symbol under point
M-. and M-, Navigate to source code for symbol under point and return to your original buffer
C-c C-d C-a clojure func fuzz search

M-(         surround expression after point in parentheses(because paredit-mode can't add paretheses)


## TODO

[GNU Emacs Reference Card](http://www.ic.unicamp.br/~helio/disciplinas/MC102/Emacs_Reference_Card.pdf)

[Mastering Emacs](http://www.masteringemacs.org/reading-guide/)

[Offical manual](http://www.gnu.org/software/emacs/manual/html_node/emacs/index.html#Top)

[great pic](http://sachachua.com/blog/wp-content/uploads/2013/05/How-to-Learn-Emacs8.png)

