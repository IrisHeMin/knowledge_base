---
layout: default
title: "Deep Dive: Other"
permalink: /deep-dive/other/
---

<h2>📦 Other Deep Dives</h2>
<p class="page-description">Deep dive topics not yet assigned to a specific category.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
      {% unless post.path contains "tcpip-" or post.path contains "dns-" or post.path contains "smb-" or post.path contains "failover-clustering" or post.path contains "performance" or post.path contains "wpa-" or post.path contains "dump-" or post.path contains "windbg-" or post.path contains "ai-102" or post.path contains "dhcp-" or post.path contains "certificate-" or post.path contains "8021x" or post.path contains "eap-" or post.path contains "kerberos-" or post.path contains "nps-" or post.path contains "auth-" or post.path contains "storage-foundations-" or post.path contains "storage-stack-" or post.path contains "storage-advanced-" or post.path contains "cluster-storage-csv-" or post.path contains "storage-spaces-direct-" %}
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
