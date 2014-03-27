---
layout: page
title: lelouchcr's home
tagline: xx xx
---
{% include JB/setup %}
  
This is a blog created by jekyllbootstrap and host on github  
        
About the author:   

name : caorong    

tags : programmer, google粉, 二次元

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<ul>
  {% assign pages_list = site.pages %}
  {% assign group = 'project' %}
  {% include JB/pages_list %}
</ul>
