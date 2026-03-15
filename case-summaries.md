---
layout: default
title: Case Studies
permalink: /case-summaries/
---

<h2>🔧 Case Studies</h2>
<p class="page-description">Structured summaries of real support cases. Select a topic to explore.</p>

{% assign dns_count = 0 %}
{% assign dhcp_count = 0 %}
{% assign smb_count = 0 %}
{% assign net_count = 0 %}
{% assign wifi_count = 0 %}
{% assign other_count = 0 %}
{% for post in site.posts %}
  {% if post.path contains "case-summaries/" %}
    {% if post.path contains "dns-" %}{% assign dns_count = dns_count | plus: 1 %}
    {% elsif post.path contains "dhcp-" %}{% assign dhcp_count = dhcp_count | plus: 1 %}
    {% elsif post.path contains "smb-" or post.path contains "sysvol-" or post.path contains "pdf-open-failure" %}{% assign smb_count = smb_count | plus: 1 %}
    {% elsif post.path contains "sdn-" or post.path contains "azshci-" or post.path contains "tcp-" or post.path contains "ncsi-" %}{% assign net_count = net_count | plus: 1 %}
    {% elsif post.path contains "802.1x-" %}{% assign wifi_count = wifi_count | plus: 1 %}
    {% else %}{% assign other_count = other_count | plus: 1 %}
    {% endif %}
  {% endif %}
{% endfor %}

<section class="category-cards">
  <a href="{{ '/case-summaries/dns/' | relative_url }}" class="category-card">
    <div class="card-icon">🌐</div>
    <h3>DNS</h3>
    <p>DNS resolution failures, glue records, CNAME issues, and local zone hijacking</p>
    <span class="card-count">{{ dns_count }} posts</span>
  </a>
  <a href="{{ '/case-summaries/dhcp/' | relative_url }}" class="category-card">
    <div class="card-icon">📡</div>
    <h3>DHCP</h3>
    <p>IP allocation conflicts, Option 82 subnet mismatches, and service resource exhaustion</p>
    <span class="card-count">{{ dhcp_count }} posts</span>
  </a>
  <a href="{{ '/case-summaries/smb/' | relative_url }}" class="category-card">
    <div class="card-icon">📂</div>
    <h3>SMB & File Sharing</h3>
    <p>SYSVOL/DFS access, mapped drive disconnects, and Azure Files path length issues</p>
    <span class="card-count">{{ smb_count }} posts</span>
  </a>
  <a href="{{ '/case-summaries/networking/' | relative_url }}" class="category-card">
    <div class="card-icon">🔌</div>
    <h3>Networking & TCP</h3>
    <p>SDN policy, Azure Stack HCI NIC config, TCP PMTU black holes, and NCSI hijacking</p>
    <span class="card-count">{{ net_count }} posts</span>
  </a>
  <a href="{{ '/case-summaries/wifi/' | relative_url }}" class="category-card">
    <div class="card-icon">📶</div>
    <h3>802.1X & Wi-Fi</h3>
    <p>Wi-Fi authentication failures, PEAP identity misconfiguration, and EAP troubleshooting</p>
    <span class="card-count">{{ wifi_count }} posts</span>
  </a>
  <a href="{{ '/case-summaries/other/' | relative_url }}" class="category-card">
    <div class="card-icon">📦</div>
    <h3>Other</h3>
    <p>NFS cluster inheritance, ADB server thread blocking, and other miscellaneous cases</p>
    <span class="card-count">{{ other_count }} posts</span>
  </a>
</section>
