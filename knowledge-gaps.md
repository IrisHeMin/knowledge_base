---
layout: default
title: Knowledge Gaps
permalink: /knowledge-gaps/
---

<h2>🧠 Knowledge Gaps · 知识盲点追踪</h2>
<p class="page-description">Track what you don't know yet. Discover blind spots through conversations, prioritize them, learn them, and watch the dashboard light up green.</p>

<style>
  .kg-stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 12px; margin: 20px 0; }
  .kg-stat { background: #f5f7fa; border-radius: 10px; padding: 18px; text-align: center; border-left: 4px solid #5b8def; }
  .kg-stat .num { font-size: 2.2em; font-weight: 700; color: #2c3e50; line-height: 1; }
  .kg-stat .lbl { color: #7a8794; margin-top: 6px; font-size: 0.92em; }
  .kg-stat.unknown   { border-left-color: #e74c3c; }
  .kg-stat.learning  { border-left-color: #f39c12; }
  .kg-stat.mastered  { border-left-color: #27ae60; }
  .kg-charts  { display: grid; grid-template-columns: repeat(auto-fit, minmax(320px, 1fr)); gap: 20px; margin: 20px 0; }
  .kg-chart   { background: #fff; border: 1px solid #e2e8f0; border-radius: 10px; padding: 16px; }
  .kg-chart h3 { margin: 0 0 12px; font-size: 1.05em; color: #2c3e50; }
  .kg-chart canvas { max-height: 260px; }
  .kg-filters { display: flex; flex-wrap: wrap; gap: 8px; margin: 20px 0 12px; align-items: center; }
  .kg-filters select, .kg-filters input { padding: 6px 10px; border: 1px solid #cbd5e0; border-radius: 6px; font-size: 0.95em; }
  .kg-filters label { font-size: 0.9em; color: #4a5568; }
  .kg-table { width: 100%; border-collapse: collapse; margin-top: 8px; font-size: 0.95em; }
  .kg-table th { background: #f5f7fa; text-align: left; padding: 10px; border-bottom: 2px solid #e2e8f0; color: #2c3e50; }
  .kg-table td { padding: 10px; border-bottom: 1px solid #edf2f7; vertical-align: top; }
  .kg-table tr:hover { background: #fafcff; }
  .kg-badge { display: inline-block; padding: 2px 8px; border-radius: 10px; font-size: 0.8em; font-weight: 600; }
  .kg-badge.unknown   { background: #fde2e2; color: #c0392b; }
  .kg-badge.learning  { background: #fef3d6; color: #b7791f; }
  .kg-badge.mastered  { background: #d4f0dc; color: #1e7e3e; }
  .kg-badge.archived  { background: #e2e8f0; color: #4a5568; }
  .kg-prio { display: inline-block; min-width: 24px; text-align: center; padding: 2px 6px; border-radius: 4px; font-weight: 700; }
  .kg-prio.p1 { background: #e74c3c; color: #fff; }
  .kg-prio.p2 { background: #ec7063; color: #fff; }
  .kg-prio.p3 { background: #f39c12; color: #fff; }
  .kg-prio.p4 { background: #5dade2; color: #fff; }
  .kg-prio.p5 { background: #95a5a6; color: #fff; }
  .kg-empty { text-align: center; padding: 40px 20px; color: #7a8794; background: #fafbfc; border-radius: 10px; border: 2px dashed #cbd5e0; margin-top: 16px; }
  .kg-empty h3 { margin-top: 0; color: #4a5568; }
  .kg-meta { font-size: 0.85em; color: #7a8794; margin-top: 8px; }
  .kg-notes { color: #4a5568; font-size: 0.9em; margin-top: 4px; }
  .kg-resource-link { color: #5b8def; text-decoration: none; }
  .kg-resource-link:hover { text-decoration: underline; }
</style>

<section class="kg-stats" id="kg-stats">
  <div class="kg-stat"><div class="num" id="kg-total">0</div><div class="lbl">📌 Total</div></div>
  <div class="kg-stat unknown"><div class="num" id="kg-unknown">0</div><div class="lbl">❓ 未学</div></div>
  <div class="kg-stat learning"><div class="num" id="kg-learning">0</div><div class="lbl">📖 学习中</div></div>
  <div class="kg-stat mastered"><div class="num" id="kg-mastered">0</div><div class="lbl">✅ 已掌握</div></div>
  <div class="kg-stat"><div class="num" id="kg-progress">0%</div><div class="lbl">🎯 掌握率</div></div>
</section>

<section class="kg-charts">
  <div class="kg-chart">
    <h3>📊 By Category</h3>
    <canvas id="kg-chart-category"></canvas>
  </div>
  <div class="kg-chart">
    <h3>🎚️ By Priority</h3>
    <canvas id="kg-chart-priority"></canvas>
  </div>
</section>

<section>
  <h3>📋 Blind Spots</h3>
  <div class="kg-filters">
    <label>Status:
      <select id="kg-filter-status">
        <option value="">All</option>
        <option value="unknown">未学</option>
        <option value="learning">学习中</option>
        <option value="mastered">已掌握</option>
        <option value="archived">已归档</option>
      </select>
    </label>
    <label>Category: <select id="kg-filter-category"><option value="">All</option></select></label>
    <label>Priority: <select id="kg-filter-priority"><option value="">All</option><option>1</option><option>2</option><option>3</option><option>4</option><option>5</option></select></label>
    <input type="text" id="kg-filter-search" placeholder="🔍 search topic / notes ..." style="flex:1; min-width: 200px;">
  </div>
  <div id="kg-table-wrap"></div>
</section>

<section style="margin-top: 30px;">
  <h3>💡 How this works</h3>
  <ol style="line-height: 1.8;">
    <li>Through conversations with the assistant, blind spots are identified and stored in a SQL database.</li>
    <li>Each entry has a topic, category, priority (1=highest, 5=lowest), status, source of discovery, learning resources, and notes.</li>
    <li>Data is exported to <code>assets/data/blind-spots.json</code>, which powers this dashboard.</li>
    <li>Mark items as <strong>learning</strong> when you start, <strong>mastered</strong> when you can teach it (Feynman test).</li>
  </ol>
  <p class="kg-meta">Data source: <code>/assets/data/blind-spots.json</code> · Last updated: <span id="kg-last-updated">—</span></p>
</section>

<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script>
(function () {
  var dataUrl = '{{ "/assets/data/blind-spots.json" | relative_url }}';
  var state = { data: null, charts: {} };

  fetch(dataUrl)
    .then(function (r) { return r.json(); })
    .then(function (d) { state.data = d; render(); })
    .catch(function (e) {
      document.getElementById('kg-table-wrap').innerHTML =
        '<div class="kg-empty"><h3>⚠️ Failed to load data</h3><p>' + e + '</p></div>';
    });

  function render() {
    var d = state.data;
    document.getElementById('kg-last-updated').textContent = d.lastUpdated || '—';

    var items = Array.isArray(d.items) ? d.items : [];
    populateCategoryFilter(d.categories || {}, items);

    document.getElementById('kg-filter-status').addEventListener('change', refresh);
    document.getElementById('kg-filter-category').addEventListener('change', refresh);
    document.getElementById('kg-filter-priority').addEventListener('change', refresh);
    document.getElementById('kg-filter-search').addEventListener('input', refresh);

    refresh();
  }

  function populateCategoryFilter(cats, items) {
    var sel = document.getElementById('kg-filter-category');
    var present = {};
    items.forEach(function (it) { if (it.category) present[it.category] = true; });
    Object.keys(present).forEach(function (k) {
      var opt = document.createElement('option');
      opt.value = k;
      opt.textContent = cats[k] || k;
      sel.appendChild(opt);
    });
  }

  function refresh() {
    var d = state.data;
    var items = Array.isArray(d.items) ? d.items : [];
    var f = {
      status: document.getElementById('kg-filter-status').value,
      category: document.getElementById('kg-filter-category').value,
      priority: document.getElementById('kg-filter-priority').value,
      search: document.getElementById('kg-filter-search').value.toLowerCase().trim()
    };

    var filtered = items.filter(function (it) {
      if (f.status && it.status !== f.status) return false;
      if (f.category && it.category !== f.category) return false;
      if (f.priority && String(it.priority) !== f.priority) return false;
      if (f.search) {
        var hay = (it.topic + ' ' + (it.notes || '') + ' ' + (it.source || '')).toLowerCase();
        if (hay.indexOf(f.search) === -1) return false;
      }
      return true;
    });

    renderStats(items);
    renderCharts(items, d.categories || {});
    renderTable(filtered, d.categories || {});
  }

  function renderStats(items) {
    var total = items.length;
    var unknown = count(items, 'status', 'unknown');
    var learning = count(items, 'status', 'learning');
    var mastered = count(items, 'status', 'mastered');
    var pct = total ? Math.round((mastered / total) * 100) : 0;
    document.getElementById('kg-total').textContent = total;
    document.getElementById('kg-unknown').textContent = unknown;
    document.getElementById('kg-learning').textContent = learning;
    document.getElementById('kg-mastered').textContent = mastered;
    document.getElementById('kg-progress').textContent = pct + '%';
  }

  function count(arr, key, val) { return arr.filter(function (x) { return x[key] === val; }).length; }

  function renderCharts(items, cats) {
    var byCat = {};
    items.forEach(function (it) {
      var k = it.category || 'other';
      byCat[k] = (byCat[k] || 0) + 1;
    });
    var catLabels = Object.keys(byCat).map(function (k) { return cats[k] || k; });
    var catData = Object.keys(byCat).map(function (k) { return byCat[k]; });

    drawChart('kg-chart-category', 'doughnut', {
      labels: catLabels,
      datasets: [{ data: catData, backgroundColor: ['#5b8def','#27ae60','#e74c3c','#f39c12','#9b59b6','#1abc9c','#34495e','#95a5a6'] }]
    });

    var prio = [0,0,0,0,0];
    items.forEach(function (it) {
      var p = parseInt(it.priority, 10);
      if (p >= 1 && p <= 5) prio[p-1]++;
    });
    drawChart('kg-chart-priority', 'bar', {
      labels: ['P1 (highest)','P2','P3','P4','P5 (lowest)'],
      datasets: [{ data: prio, backgroundColor: ['#e74c3c','#ec7063','#f39c12','#5dade2','#95a5a6'] }]
    }, { plugins:{legend:{display:false}}, scales:{y:{beginAtZero:true,ticks:{precision:0}}} });
  }

  function drawChart(canvasId, type, data, opts) {
    if (state.charts[canvasId]) state.charts[canvasId].destroy();
    var ctx = document.getElementById(canvasId).getContext('2d');
    state.charts[canvasId] = new Chart(ctx, {
      type: type, data: data,
      options: Object.assign({ responsive: true, maintainAspectRatio: false }, opts || {})
    });
  }

  function renderTable(items, cats) {
    var wrap = document.getElementById('kg-table-wrap');
    if (!items.length) {
      wrap.innerHTML = '<div class="kg-empty"><h3>🌱 No blind spots tracked yet</h3>' +
        '<p>Start a conversation to identify and add your first one.</p></div>';
      return;
    }
    items.sort(function (a, b) {
      var sa = statusOrder(a.status), sb = statusOrder(b.status);
      if (sa !== sb) return sa - sb;
      return (a.priority || 99) - (b.priority || 99);
    });
    var rows = items.map(function (it) {
      var cat = cats[it.category] || it.category || 'other';
      var resHtml = '';
      if (it.resources) {
        var isUrl = /^https?:\/\//.test(it.resources);
        resHtml = isUrl
          ? '<a class="kg-resource-link" href="' + esc(it.resources) + '" target="_blank">link ↗</a>'
          : esc(it.resources);
      }
      return '<tr>' +
        '<td><span class="kg-prio p' + (it.priority || 3) + '">' + (it.priority || '-') + '</span></td>' +
        '<td><span class="kg-badge ' + esc(it.status || 'unknown') + '">' + statusLabel(it.status) + '</span></td>' +
        '<td>' + esc(cat) + '</td>' +
        '<td><strong>' + esc(it.topic || '') + '</strong>' +
          (it.notes ? '<div class="kg-notes">' + esc(it.notes) + '</div>' : '') +
          (it.source ? '<div class="kg-meta">📍 ' + esc(it.source) + '</div>' : '') +
        '</td>' +
        '<td>' + resHtml + '</td>' +
      '</tr>';
    }).join('');
    wrap.innerHTML = '<table class="kg-table"><thead><tr>' +
      '<th>P</th><th>Status</th><th>Category</th><th>Topic</th><th>Resource</th>' +
      '</tr></thead><tbody>' + rows + '</tbody></table>';
  }

  function statusOrder(s) { return { 'learning': 0, 'unknown': 1, 'mastered': 2, 'archived': 3 }[s] || 9; }
  function statusLabel(s) { return { 'unknown':'未学','learning':'学习中','mastered':'已掌握','archived':'已归档' }[s] || s; }
  function esc(s) { var d = document.createElement('div'); d.textContent = s == null ? '' : String(s); return d.innerHTML; }
})();
</script>
