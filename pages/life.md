---
layout: page
title: Life
description: 做厨如行医，烹鲜若治国。祖国尚未统一，减肥改日再议。
menu: 生活
repositories: false
share: false
comments: false
categories: true
calendar: true
canvas: true
permalink: /life/
---


<div>
  {% assign privious_type = 'none' %}
  {% for life in site.data.lifes %}
    {% if life.type != privious_type %}
      {% if privious_type != 'none' %}
        </ol>
      {% endif %}
      <h3>{{ life.type }}</h3>
      {% assign privious_type = life.type %}
      <ol class="posts-list" >
    {% endif %}
    <li class="posts-list-item">
      <a class="posts-list-name" style="color:#4169E1" href="{{ life.url }}" target="_blank">{{ life.title }}</a>
    </li>
  {% endfor %}
  </ol>
</div>