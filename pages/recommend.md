---
layout: page
title: Recommend
description: 推荐一本好书，一部电影，一首歌...
comments: false
canvas: true
menu: 推荐
permalink: /recommend/
---

<div>
  {% assign privious_type = 'none' %}
  {% for recommend in site.data.recommends %}
    {% if recommend.type != privious_type %}
      {% if privious_type != 'none' %}
        </ol>
      {% endif %}
      <h3>{{ recommend.type }}</h3>
      {% assign privious_type = recommend.type %}
      <ol class="posts-list" >
    {% endif %}
    <li class="posts-list-item">
      <a class="posts-list-name" style="color:#4169E1" href="{{ recommend.url }}" target="_blank">{{ recommend.name }}</a>
    </li>
  {% endfor %}
  </ol>
</div>