---
layout: default
title: "Deep Dive: AI"
permalink: /deep-dive/ai/
---

<h2>🤖 AI</h2>
<p class="page-description">AI fundamentals, machine learning, LLMs, prompt engineering, AI agents, and Azure AI-102 certification.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "ai-learning/" or post.path contains "ai-102" %}
    <li class="post-list-item">
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <div class="post-list-meta">
        📅 {{ post.date | date: "%Y-%m-%d" }}
        {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
      </div>
      {% if post.excerpt %}<p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
    </li>
    {% endif %}
  {% endfor %}
</ul>
