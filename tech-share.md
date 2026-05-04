---
layout: default
title: Tech Share
permalink: /tech-share/
---

<h2>📡 Tech Share</h2>
<p class="page-description">Curated technical sharings from internal Tech Share threads. Compact, problem-driven, and battle-tested. Each post follows a 4-section format: Problem → Investigation → Solution → Takeaway.</p>

{% assign dns_count = 0 %}
{% assign smb_count = 0 %}
{% assign cluster_count = 0 %}
{% assign tool_count = 0 %}
{% assign other_count = 0 %}
{% for post in site.posts %}
  {% if post.path contains "tech-share/" %}
    {% if post.path contains "dns-" or post.path contains "spooler-dns" %}{% assign dns_count = dns_count | plus: 1 %}
    {% elsif post.path contains "smb-" %}{% assign smb_count = smb_count | plus: 1 %}
    {% elsif post.path contains "cluster-" or post.path contains "azlocal-" or post.path contains "failover-" %}{% assign cluster_count = cluster_count | plus: 1 %}
    {% elsif post.path contains "tool-" or post.path contains "agent-" %}{% assign tool_count = tool_count | plus: 1 %}
    {% else %}{% assign other_count = other_count | plus: 1 %}
    {% endif %}
  {% endif %}
{% endfor %}

<section class="category-cards">
  <a href="{{ '/tech-share/dns/' | relative_url }}" class="category-card">
    <div class="card-icon">🌐</div>
    <h3>DNS</h3>
    <p>Resolution behavior, query type quirks, conditional forwarders, and protocol edge cases</p>
    <span class="card-count">{{ dns_count }} posts</span>
  </a>
  <a href="{{ '/tech-share/smb/' | relative_url }}" class="category-card">
    <div class="card-icon">📂</div>
    <h3>SMB & File</h3>
    <p>SMB performance, dialect negotiation, share access, and DFS-related sharings</p>
    <span class="card-count">{{ smb_count }} posts</span>
  </a>
  <a href="{{ '/tech-share/cluster/' | relative_url }}" class="category-card">
    <div class="card-icon">🧱</div>
    <h3>Cluster & Azure Local</h3>
    <p>Failover Cluster, Network ATC, Azure Local, and live migration scenarios</p>
    <span class="card-count">{{ cluster_count }} posts</span>
  </a>
  <a href="{{ '/tech-share/tools/' | relative_url }}" class="category-card">
    <div class="card-icon">🛠️</div>
    <h3>Tool Sharing</h3>
    <p>Custom Copilot agents, log analyzers, and productivity tools shared by the team</p>
    <span class="card-count">{{ tool_count }} posts</span>
  </a>
  <a href="{{ '/tech-share/other/' | relative_url }}" class="category-card">
    <div class="card-icon">📦</div>
    <h3>Other</h3>
    <p>Firewall, Defender, BIOS, NIC, certificate, and miscellaneous sharings</p>
    <span class="card-count">{{ other_count }} posts</span>
  </a>
</section>

<section class="recent-posts">
  <h2>📝 Latest Tech Share</h2>
  <ul class="post-list">
  {% assign tech_posts = site.posts | where_exp: "p", "p.path contains 'tech-share/'" %}
  {% for post in tech_posts limit: 10 %}
    <li class="post-list-item">
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <div class="post-list-meta">
        📅 {{ post.date | date: "%Y-%m-%d" }}
        {% if post.shared_by %} &middot; 👤 {{ post.shared_by }}{% endif %}
        {% if post.categories.size > 0 %} &middot; 📁 {{ post.categories | join: ", " }}{% endif %}
      </div>
      {% if post.excerpt %}<p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 200 }}</p>{% endif %}
    </li>
  {% endfor %}
  </ul>
</section>
