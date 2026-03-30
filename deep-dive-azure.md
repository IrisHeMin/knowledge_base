---
layout: default
title: "Deep Dive: Azure Networking"
permalink: /deep-dive/azure/
---

<h2>☁️ Azure Networking</h2>
<p class="page-description">Azure load balancing, virtual networking, NSG, and cloud network architecture deep dives.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
      {% if post.path contains "azure-" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
        {% if post.excerpt %}<p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>
