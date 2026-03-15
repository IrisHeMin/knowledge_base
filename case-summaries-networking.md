---
layout: default
title: "Case Studies: Networking & TCP"
permalink: /case-summaries/networking/
---

<h2>🔌 Networking & TCP Cases</h2>
<p class="page-description">SDN policy push failures, Azure Stack HCI NIC configuration, TCP PMTU black holes, and NCSI HTTP hijacking.</p>

<a href="{{ '/case-summaries/' | relative_url }}" class="back-link">&larr; Back to Case Studies</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% if post.path contains "sdn-" or post.path contains "azshci-" or post.path contains "tcp-" or post.path contains "ncsi-" %}
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
