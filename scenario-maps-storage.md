---
layout: default
title: "Scenario Maps: Storage"
permalink: /scenario-maps/storage/
---

<h2>💾 Storage Troubleshooting</h2>
<p class="page-description">Visual troubleshooting navigation maps for Windows storage stack, Cluster Shared Volumes, and Storage Spaces Direct scenarios.</p>

<a href="{{ '/scenario-maps/' | relative_url }}" class="back-link">&larr; Back to Scenario Maps</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
      {% if post.path contains "storage-" or post.path contains "cluster-storage-" %}
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
