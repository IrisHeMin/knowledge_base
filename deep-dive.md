---
layout: default
title: 知识深潜
permalink: /deep-dive/
---

<h2>📖 知识深潜</h2>
<p class="page-description">技术原理深度解析，包括网络协议、Windows 集群架构、性能分析方法论等系统性知识整理。</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "deep-dive/" %}
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
