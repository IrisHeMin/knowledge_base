---
layout: default
title: "Scenario Maps: Other"
permalink: /scenario-maps/other/
---

<h2>📦 Other Scenario Maps</h2>
<p class="page-description">Scenario maps not yet assigned to a specific category.</p>

<a href="{{ '/scenario-maps/' | relative_url }}" class="back-link">&larr; Back to Scenario Maps</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
      {% unless post.path contains "tcpip-adapter" or post.path contains "tcpip-ip-missing" or post.path contains "tcpip-default-gw" or post.path contains "tcpip-connection" or post.path contains "tcpip-packet" or post.path contains "tcpip-port" or post.path contains "cluster-" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
        {% if post.excerpt %}<p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
      </li>
      {% endunless %}
    {% endif %}
  {% endfor %}
</ul>
