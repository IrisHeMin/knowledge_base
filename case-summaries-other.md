---
layout: default
title: "Case Studies: Other"
permalink: /case-summaries/other/
---

<h2>📦 Other Cases</h2>
<p class="page-description">NFS cluster inheritance, ADB server thread blocking, and other miscellaneous cases.</p>

<a href="{{ '/case-summaries/' | relative_url }}" class="back-link">&larr; Back to Case Studies</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% unless post.path contains "dns-" or post.path contains "dhcp-" or post.path contains "smb-" or post.path contains "sysvol-" or post.path contains "pdf-open-failure" or post.path contains "sdn-" or post.path contains "azshci-" or post.path contains "tcp-" or post.path contains "ncsi-" or post.path contains "802.1x-" %}
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
