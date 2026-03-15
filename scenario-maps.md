---
layout: default
title: 场景导航
permalink: /scenario-maps/
---

<h2>🗺️ 场景导航</h2>
<p class="page-description">可视化排查导航图，使用 Mermaid 流程图呈现故障排查路径，帮助快速定位问题。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
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
