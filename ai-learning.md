---
layout: default
title: AI Learning
permalink: /ai-learning/
---

<h2>🤖 AI Learning</h2>
<p class="page-description">AI learning series from scratch, covering machine learning fundamentals, deep learning, large language models, Prompt Engineering, AI Agent development, and more.</p>

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
