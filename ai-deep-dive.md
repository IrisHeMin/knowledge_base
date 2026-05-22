---
layout: default
title: AI Deep Dive
permalink: /ai-deep-dive/
---

<h2>🔬 AI 深度解析专栏</h2>
<p class="page-description">每期聚焦一个 AI 主题，深度分析不追热点，只追本质。短段落、强观点、有数据，3-5 分钟读完一个领域。</p>

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
