---
layout: default
title: AI Deep Dive
permalink: /ai-deep-dive/
---

<h2>🔬 AI深度解析播客</h2>
<p class="page-description">每期聚焦一个AI主题，做20-30分钟的深度分析。不追热点，追本质。像跟一个懂行的朋友深聊——有数据、有观点、有预判。</p>

<ul class="post-list">
  {% assign deepdive_posts = site.posts | where_exp: "post", "post.path contains 'ai-deep-dive/'" | sort: "date" | reverse %}
  {% for post in deepdive_posts %}
  <li class="post-list-item">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <div class="post-list-meta">
      {% if post.tags.size > 0 %}
        🏷️ {{ post.tags | join: ", " }}
      {% endif %}
    </div>
  </li>
  {% endfor %}
</ul>
