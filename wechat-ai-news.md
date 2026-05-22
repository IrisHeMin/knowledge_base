---
layout: default
title: WeChat AI News
permalink: /wechat-ai-news/
---

<h2>📱 小敏说 AI · 公众号专栏</h2>
<p class="page-description">每日精选 3-5 条最值得聊的 AI 新闻，公众号风格深度点评。3-5 分钟读完，看懂 AI 圈一天。</p>

<ul class="post-list">
  {% assign wechat_posts = site.posts | where_exp: "post", "post.path contains 'wechat-ai-news/'" | sort: "date" | reverse %}
  {% for post in wechat_posts %}
  <li class="post-list-item">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <div class="post-list-meta">
      📅 {{ post.date | date: "%Y-%m-%d" }}
      {% if post.tags.size > 0 %}
        &middot; 🏷️ {{ post.tags | join: ", " }}
      {% endif %}
    </div>
  </li>
  {% endfor %}
</ul>
