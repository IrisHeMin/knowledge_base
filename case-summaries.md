---
layout: default
title: Case Studies
permalink: /case-summaries/
---

<h2>🔧 Case Studies</h2>
<p class="page-description">Structured summaries of real support cases covering DNS, DHCP, SMB, SDN, 802.1X, and other networking & storage troubleshooting scenarios.</p>

<h3 class="topic-heading">🌐 DNS</h3>
<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% if post.path contains "dns-" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>

<h3 class="topic-heading">📡 DHCP</h3>
<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% if post.path contains "dhcp-" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>

<h3 class="topic-heading">📂 SMB & File Sharing</h3>
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
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>

<h3 class="topic-heading">🔌 Networking & TCP</h3>
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
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>

<h3 class="topic-heading">📶 802.1X & Wi-Fi</h3>
<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
      {% if post.path contains "802.1x-" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>

<h3 class="topic-heading">📦 Other</h3>
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
      </li>
      {% endunless %}
    {% endif %}
  {% endfor %}
</ul>
