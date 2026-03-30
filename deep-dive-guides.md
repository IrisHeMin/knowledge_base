---
layout: default
title: "Deep Dive: Technology Guides"
permalink: /deep-dive/guides/
---

<h2>📚 Technology Guides</h2>
<p class="page-description">Comprehensive Windows technology navigation guides and ecosystem overviews covering networking, storage, security, virtualization, and more.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
      {% if post.path contains "-guide" or post.path contains "navigation-map" %}
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
