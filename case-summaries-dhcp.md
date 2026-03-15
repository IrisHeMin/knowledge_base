---
layout: default
title: "Case Studies: DHCP"
permalink: /case-summaries/dhcp/
---

<h2>📡 DHCP Cases</h2>
<p class="page-description">IP allocation conflicts, Option 82 subnet mismatches, and service resource exhaustion.</p>

<a href="{{ '/case-summaries/' | relative_url }}" class="back-link">&larr; Back to Case Studies</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" and post.path contains "dhcp-" %}
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
