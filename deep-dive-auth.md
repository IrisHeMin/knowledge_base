---
layout: default
title: "Deep Dive: Authentication & PKI"
permalink: /deep-dive/auth/
---

<h2>🔐 Authentication & PKI</h2>
<p class="page-description">Certificate-based authentication, Kerberos strong mapping, 802.1X / EAP-TLS, ADCS, NPS, and PKI infrastructure deep dives.</p>

<a href="{{ '/deep-dive/' | relative_url }}" class="back-link">&larr; Back to Deep Dive</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
      {% if post.path contains "certificate-" or post.path contains "8021x" or post.path contains "eap-" or post.path contains "kerberos-" or post.path contains "nps-" or post.path contains "auth-" %}
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
