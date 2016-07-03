---
layout: archive
title: All Posts
excerpt: "A List of Posts"
author: True
image:
  feature: background.jpg
  credit: kleinarl
  creditlink: http://www.stockhambauer.at/
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>