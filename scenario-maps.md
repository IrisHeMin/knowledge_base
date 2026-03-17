---
layout: default
title: Scenario Maps
permalink: /scenario-maps/
---

<h2>🗺️ Scenario Maps</h2>
<p class="page-description">Visual troubleshooting navigation maps using Mermaid flowcharts. Select a topic to explore.</p>

{% assign config_count = 0 %}
{% assign packet_count = 0 %}
{% assign port_count = 0 %}
{% assign cluster_count = 0 %}
{% assign storage_count = 0 %}
{% assign other_count = 0 %}
{% for post in site.posts %}
  {% if post.path contains "scenario-maps/" %}
    {% if post.path contains "tcpip-adapter" or post.path contains "tcpip-ip-missing" or post.path contains "tcpip-default-gw" %}{% assign config_count = config_count | plus: 1 %}
    {% elsif post.path contains "tcpip-connection" or post.path contains "tcpip-packet" %}{% assign packet_count = packet_count | plus: 1 %}
    {% elsif post.path contains "tcpip-port" %}{% assign port_count = port_count | plus: 1 %}
    {% elsif post.path contains "cluster-" %}{% assign cluster_count = cluster_count | plus: 1 %}
    {% elsif post.path contains "storage-" or post.path contains "cluster-storage-" %}{% assign storage_count = storage_count | plus: 1 %}
    {% else %}{% assign other_count = other_count | plus: 1 %}
    {% endif %}
  {% endif %}
{% endfor %}

<section class="category-cards">
  <a href="{{ '/scenario-maps/tcpip-config/' | relative_url }}" class="category-card">
    <div class="card-icon">⚙️</div>
    <h3>TCP/IP: Configuration</h3>
    <p>Adapter missing, IP not assigned, default gateway missing or unreachable</p>
    <span class="card-count">{{ config_count }} posts</span>
  </a>
  <a href="{{ '/scenario-maps/tcpip-packet/' | relative_url }}" class="category-card">
    <div class="card-icon">📦</div>
    <h3>TCP/IP: Packet Flow</h3>
    <p>Connection resets, packet drops (middle device, WFP), and packet modification</p>
    <span class="card-count">{{ packet_count }} posts</span>
  </a>
  <a href="{{ '/scenario-maps/tcpip-port/' | relative_url }}" class="category-card">
    <div class="card-icon">🔌</div>
    <h3>TCP/IP: Port Issues</h3>
    <p>Port exhaustion and port not listening scenarios</p>
    <span class="card-count">{{ port_count }} posts</span>
  </a>
  <a href="{{ '/scenario-maps/clustering/' | relative_url }}" class="category-card">
    <div class="card-icon">🖥️</div>
    <h3>Windows Clustering</h3>
    <p>Cluster creation failures and CNO repair troubleshooting</p>
    <span class="card-count">{{ cluster_count }} posts</span>
  </a>
  <a href="{{ '/scenario-maps/storage/' | relative_url }}" class="category-card">
    <div class="card-icon">💾</div>
    <h3>Storage</h3>
    <p>Windows storage stack, CSV, and Storage Spaces Direct troubleshooting scenarios</p>
    <span class="card-count">{{ storage_count }} posts</span>
  </a>
  {% if other_count > 0 %}
  <a href="{{ '/scenario-maps/other/' | relative_url }}" class="category-card">
    <div class="card-icon">📦</div>
    <h3>Other</h3>
    <p>Other scenario maps not yet categorized</p>
    <span class="card-count">{{ other_count }} posts</span>
  </a>
  {% endif %}
</section>
