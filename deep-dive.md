---
layout: default
title: Deep Dive
permalink: /deep-dive/
---

<h2>📖 Deep Dive</h2>
<p class="page-description">In-depth technical analysis covering network protocols, Windows cluster architecture, performance methodology, and systematic knowledge consolidation.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
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
