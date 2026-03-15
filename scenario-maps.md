---
layout: default
title: Scenario Maps
permalink: /scenario-maps/
---

<h2>🗺️ Scenario Maps</h2>
<p class="page-description">Visual troubleshooting navigation maps using Mermaid flowcharts to illustrate fault diagnosis paths for quick problem identification.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
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
