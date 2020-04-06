---
layout: page
title: Categories
description: Every day to be a little better.
keywords: 分类
comments: false
share: false
repositories: false
categories: true
canvas: true
menu: 分类
permalink: /categories/
---

<div>
  {% assign sorted_categories = site.categories | sort %}
  {% for category in sorted_categories %}
    <h3 id="{{ category[0] }}" name="{{ category[0] }}">{{ category | first }}</h3>
    <ol class="posts-list">
      {% for post in category.last %}
        <li class="posts-list-item">
          <span class="posts-list-meta">{{ post.date | date:"%Y-%m-%d" }}</span>
          <a class="posts-list-name" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
        </li>
      {% endfor %}
    </ol>
  {% endfor %}
</div>