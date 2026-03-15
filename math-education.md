---
layout: default
title: Math Education
permalink: /math-education/
---

<h2>📐 Math Education</h2>
<p class="page-description">Shanghai elementary math curriculum (Grades 1–6) organized by grade and semester, ideal for review and gap identification.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "math-education/" %}
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
