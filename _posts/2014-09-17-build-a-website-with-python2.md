---
layout: post
title: "用python写网站（二）"
description: ""
category: "python"
tags: [python, website]
---

注，由于 与 jekyll 模版冲突 后文所有的双大括号中间都多加了各空格。

===

#### 模版部分

本来模版应该没什么好谈的，flask自带Jinja2。但如果前端又配合这angular的话就冲突了。他们的模版占位符都是

这里总结了下网上找到几个方案

1. 修改 angular 不修改flask 

flask 占位符内模版里面 写angular的模版

``` html
<span>{ {'{ {book}}'} }</span>
```

2. 前后各包裹 {% raw %} 和 {% endraw %} 让 jinja 不解析中间的所有字符串

``` html
{% raw %}
<span>{ { book } }</span>
{% endraw %}
```

3. 修改 angular 的启动参数，修改 angular 的模版符号

``` python 
app.jinja_env.variable_start_string = '{ { '
app.jinja_env.variable_end_string = ' } }' 

// 这样的话 带空格的是jinja的 不待空格的是angular的

```

4. 同上，不过修改的是 angular 的模版符号

``` javascript

var app = angular.module('myApp', []);

    app.config(['$interpolateProvider', function($interpolateProvider) {
      // 将 angular 的模版符号改称 {[
      $interpolateProvider.startSymbol('{[');
      $interpolateProvider.endSymbol(']}');
    }]);
```




