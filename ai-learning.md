---
layout: default
title: AI 学习
permalink: /ai-learning/
---

<h2>🤖 AI 学习</h2>
<p class="page-description">从零开始的 AI 学习系列，涵盖机器学习基础、深度学习、大语言模型、Prompt Engineering、AI Agent 开发等。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "ai-learning/" or post.path contains "ai-102" %}
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
