---
layout: default
title: Troubleshooting Guides
permalink: /tsg/
---

<h2>🛠️ Troubleshooting Guides (TSG)</h2>
<p class="page-description">General troubleshooting steps and solutions for common issues.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "tsg/" %}
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
