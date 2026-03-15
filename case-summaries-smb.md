---
layout: default
title: "Case Studies: SMB & File Sharing"
permalink: /case-summaries/smb/
---

<h2>📂 SMB & File Sharing Cases</h2>
<p class="page-description">SYSVOL/DFS access failures, mapped drive disconnects, and Azure Files path length issues.</p>

<a href="{{ '/case-summaries/' | relative_url }}" class="back-link">&larr; Back to Case Studies</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% if post.path contains "smb-" or post.path contains "sysvol-" or post.path contains "pdf-open-failure" %}
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
