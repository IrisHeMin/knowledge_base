---
layout: default
title: 故障排查案例
permalink: /case-summaries/
---

<h2>🔧 故障排查案例</h2>
<p class="page-description">真实 Support Case 的结构化总结，涵盖 DNS、DHCP、SMB、SDN、802.1X 等网络与存储故障场景。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "case-summaries/" %}
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
