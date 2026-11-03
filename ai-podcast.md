---
layout: default
title: AI Podcast
permalink: /ai-podcast/
---

<nav aria-label="播客栏目">
  <a href="#ai-news">AI播客快讯</a> &middot; <a href="#parenting">育儿栏目</a>
</nav>

<h2 id="ai-news">🎙️ AI播客快讯</h2>
<p class="page-description">每日AI行业快讯播客文稿。用口语化、轻松易懂的方式，帮你10-15分钟掌握AI圈当天最重要的大事。</p>

<ul class="post-list">
  {% assign podcast_posts = site.posts | where_exp: "post", "post.path contains 'ai-podcast/'" | sort: "date" | reverse %}
  {% for post in podcast_posts %}
  <li class="post-list-item">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <div class="post-list-meta">
      📅 {{ post.date | date: "%Y-%m-%d" }}
      {% if post.tags.size > 0 %}
        &middot; 🏷️ {{ post.tags | join: ", " }}
      {% endif %}
    </div>
  </li>
  {% endfor %}
</ul>

<h2 id="parenting">育儿栏目</h2>
<p class="page-description">我是小敏。从家庭里的小困惑出发，聊孩子成长，也聊父母成长。</p>

<ul class="post-list">
  {% assign parenting_posts = site.posts | where: "type", "parenting-podcast" | sort: "episode" | reverse %}
  {% for post in parenting_posts %}
  <li class="post-list-item">
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <div class="post-list-meta">
      📅 {{ post.date | date: "%Y-%m-%d" }}
      {% if post.tags.size > 0 %}
        &middot; 🏷️ {{ post.tags | join: ", " }}
      {% endif %}
    </div>
  </li>
  {% endfor %}
</ul>
