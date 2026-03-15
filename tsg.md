---
layout: default
title: 排查指南
permalink: /tsg/
---

<h2>🛠️ 排查指南</h2>
<p class="page-description">常见问题的通用排查步骤与解决方案 (Troubleshooting Guide)。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "tsg/" %}
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
