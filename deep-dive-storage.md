---
layout: default
title: "Deep Dive: Storage"
permalink: /deep-dive/storage/
---

<h2>💾 Storage</h2>
<p class="page-description">Windows storage stack architecture, hardware types, Storage Spaces, CSV, S2D, MPIO, deduplication, and storage replica deep dives from junior to expert level.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
      {% if post.path contains "storage-foundations-" or post.path contains "storage-stack-" or post.path contains "storage-advanced-" or post.path contains "cluster-storage-csv-" or post.path contains "storage-spaces-direct-" or post.path contains "storage-performance-" %}
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
