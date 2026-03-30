---
layout: default
title: Deep Dive
permalink: /deep-dive/
---

<h2>📖 Deep Dive</h2>
<p class="page-description">In-depth technical analysis covering network protocols, cluster architecture, performance methodology, and debugging techniques. Select a topic to explore.</p>

{% assign tcpip_count = 0 %}
{% assign proto_count = 0 %}
{% assign cluster_count = 0 %}
{% assign perf_count = 0 %}
{% assign debug_count = 0 %}
{% assign dhcp_count = 0 %}
{% assign auth_count = 0 %}
{% assign storage_count = 0 %}
{% assign azure_count = 0 %}
{% assign virt_count = 0 %}
{% assign guides_count = 0 %}
{% assign other_count = 0 %}
{% for post in site.posts %}
  {% if post.path contains "deep-dive/" %}
    {% if post.path contains "tcpip-" %}{% assign tcpip_count = tcpip_count | plus: 1 %}
    {% elsif post.path contains "dns-" or post.path contains "smb-" or post.path contains "dfs-" %}{% assign proto_count = proto_count | plus: 1 %}
    {% elsif post.path contains "failover-clustering" %}{% assign cluster_count = cluster_count | plus: 1 %}
    {% elsif post.path contains "dump-" or post.path contains "windbg-" %}{% assign debug_count = debug_count | plus: 1 %}
    {% elsif post.path contains "dhcp-" %}{% assign dhcp_count = dhcp_count | plus: 1 %}
    {% elsif post.path contains "certificate-" or post.path contains "8021x" or post.path contains "eap-" or post.path contains "kerberos-" or post.path contains "nps-" or post.path contains "auth-" %}{% assign auth_count = auth_count | plus: 1 %}
    {% elsif post.path contains "storage-foundations-" or post.path contains "storage-stack-" or post.path contains "storage-advanced-" or post.path contains "cluster-storage-csv-" or post.path contains "storage-spaces-direct-" or post.path contains "storage-performance-" %}{% assign storage_count = storage_count | plus: 1 %}
    {% elsif post.path contains "performance" or post.path contains "wpa-" %}{% assign perf_count = perf_count | plus: 1 %}
    {% elsif post.path contains "azure-" %}{% assign azure_count = azure_count | plus: 1 %}
    {% elsif post.path contains "hyper-v" %}{% assign virt_count = virt_count | plus: 1 %}
    {% elsif post.path contains "-guide" or post.path contains "navigation-map" %}{% assign guides_count = guides_count | plus: 1 %}
    {% else %}{% assign other_count = other_count | plus: 1 %}
    {% endif %}
  {% endif %}
{% endfor %}

<section class="category-cards">
  <a href="{{ '/deep-dive/tcpip/' | relative_url }}" class="category-card">
    <div class="card-icon">🌐</div>
    <h3>TCP/IP Networking</h3>
    <p>Packet flow, routing internals, and TCP/IP stack deep dives from junior to master level</p>
    <span class="card-count">{{ tcpip_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/protocols/' | relative_url }}" class="category-card">
    <div class="card-icon">📡</div>
    <h3>DNS, SMB & DFS</h3>
    <p>DNS server architecture, SMB protocol internals, and DFS Namespace deep dives</p>
    <span class="card-count">{{ proto_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/clustering/' | relative_url }}" class="category-card">
    <div class="card-icon">🖥️</div>
    <h3>Windows Clustering</h3>
    <p>Failover clustering architecture, quorum, CSV, and cluster network internals</p>
    <span class="card-count">{{ cluster_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/performance/' | relative_url }}" class="category-card">
    <div class="card-icon">📊</div>
    <h3>Performance Analysis</h3>
    <p>CPU, memory, storage, network performance methodology, PerfMon, WPA, and toolkits</p>
    <span class="card-count">{{ perf_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/debugging/' | relative_url }}" class="category-card">
    <div class="card-icon">🔍</div>
    <h3>Debugging & Dump Analysis</h3>
    <p>WinDbg crash dump analysis, dump capture techniques, and debugging methodology</p>
    <span class="card-count">{{ debug_count }} posts</span>
  </a>

  <a href="{{ '/deep-dive/dhcp/' | relative_url }}" class="category-card">
    <div class="card-icon">🔄</div>
    <h3>DHCP</h3>
    <p>DHCP server architecture, relay agent mechanisms, failover configurations, and lease management</p>
    <span class="card-count">{{ dhcp_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/auth/' | relative_url }}" class="category-card">
    <div class="card-icon">🔐</div>
    <h3>Authentication & PKI</h3>
    <p>Certificate-based authentication, Kerberos strong mapping, 802.1X / EAP-TLS, ADCS, and NPS</p>
    <span class="card-count">{{ auth_count }} posts</span>
  </a>
  <a href="{{ '/deep-dive/storage/' | relative_url }}" class="category-card">
    <div class="card-icon">💾</div>
    <h3>Storage</h3>
    <p>Storage stack architecture, hardware types, Storage Spaces, CSV, S2D, MPIO, dedup, and replica</p>
    <span class="card-count">{{ storage_count }} posts</span>
  </a>
  {% if azure_count > 0 %}
  <a href="{{ '/deep-dive/azure/' | relative_url }}" class="category-card">
    <div class="card-icon">☁️</div>
    <h3>Azure Networking</h3>
    <p>Azure load balancing, virtual networking, NSG, and cloud network architecture</p>
    <span class="card-count">{{ azure_count }} posts</span>
  </a>
  {% endif %}
  {% if virt_count > 0 %}
  <a href="{{ '/deep-dive/virtualization/' | relative_url }}" class="category-card">
    <div class="card-icon">🖥️</div>
    <h3>Virtualization</h3>
    <p>Hyper-V architecture, virtual switch, live migration, and VM management</p>
    <span class="card-count">{{ virt_count }} posts</span>
  </a>
  {% endif %}
  {% if guides_count > 0 %}
  <a href="{{ '/deep-dive/guides/' | relative_url }}" class="category-card">
    <div class="card-icon">📚</div>
    <h3>Technology Guides</h3>
    <p>Comprehensive Windows technology navigation guides and ecosystem overviews</p>
    <span class="card-count">{{ guides_count }} posts</span>
  </a>
  {% endif %}
  {% if other_count > 0 %}
  <a href="{{ '/deep-dive/other/' | relative_url }}" class="category-card">
    <div class="card-icon">📦</div>
    <h3>Other</h3>
    <p>Other deep dive topics not yet categorized</p>
    <span class="card-count">{{ other_count }} posts</span>
  </a>
  {% endif %}
</section>
