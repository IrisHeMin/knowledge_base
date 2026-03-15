---
layout: default
title: 数学教育
permalink: /math-education/
---

<h2>📐 数学教育</h2>
<p class="page-description">上海小学数学知识点整理，按年级和学期分类，适合课后复习与查漏补缺。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "math-education/" %}
    <li class="post-list-item">
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <div class="post-list-meta">
        📅 {{ post.date | date: "%Y-%m-%d" }}
        {% if post.categories.size > 0 %}
          &middot; 📁 {{ post.categories | join: ", " }}
        {% endif %}
      </div>
    </li>
    {% endif %}
  {% endfor %}
</ul>
