---
layout: default
title: Search
permalink: /search/
---

<h2>🔍 Search</h2>

<div class="search-box-page">
  <input type="text" id="search-input" class="search-input-large" placeholder="Type keywords to search..." autofocus>
</div>

<div id="search-results">
  <p class="search-hint">Enter keywords to search across all posts by title, tags, categories, and content.</p>
</div>

<script>
(function() {
  var posts = [];
  var input = document.getElementById('search-input');
  var resultsDiv = document.getElementById('search-results');

  // Load search index
  fetch('{{ "/search.json" | relative_url }}')
    .then(function(r) { return r.json(); })
    .then(function(data) { posts = data; })
    .catch(function() {
      resultsDiv.innerHTML = '<p>Failed to load search index.</p>';
    });

  // Read query from URL
  var params = new URLSearchParams(location.search);
  if (params.get('q')) {
    input.value = params.get('q');
    // Wait for index to load then search
    var checkReady = setInterval(function() {
      if (posts.length > 0) { clearInterval(checkReady); doSearch(params.get('q')); }
    }, 100);
    setTimeout(function() { clearInterval(checkReady); }, 5000);
  }

  input.addEventListener('input', function() {
    var q = this.value.trim();
    if (q.length >= 2) {
      history.replaceState(null, '', '?q=' + encodeURIComponent(q));
      doSearch(q);
    } else {
      resultsDiv.innerHTML = '<p class="search-hint">Enter keywords to search across all posts by title, tags, categories, and content.</p>';
    }
  });

  function doSearch(query) {
    var terms = query.toLowerCase().split(/\s+/);
    var titleHits = [], contentHits = [];
    posts.forEach(function(post) {
      var titleStr = (post.title + ' ' + post.categories.join(' ') + ' ' + post.tags.join(' ') + ' ' + post.excerpt).toLowerCase();
      var contentStr = (post.content || '').toLowerCase();
      var inTitle = terms.every(function(t) { return titleStr.indexOf(t) !== -1; });
      var inContent = terms.every(function(t) { return (titleStr + ' ' + contentStr).indexOf(t) !== -1; });
      if (inTitle) titleHits.push(post);
      else if (inContent) contentHits.push(post);
    });

    if (!titleHits.length && !contentHits.length) {
      resultsDiv.innerHTML = '<p class="search-no-results">No results found for "<strong>' + escHtml(query) + '</strong>"</p>';
      return;
    }

    var total = titleHits.length + contentHits.length;
    var html = '<p class="search-count">' + total + ' result' + (total > 1 ? 's' : '') + ' for "<strong>' + escHtml(query) + '</strong>"</p>';
    if (titleHits.length) {
      html += '<h3 class="search-section">Title Matches</h3><ul class="post-list">';
      titleHits.forEach(function(p) { html += renderResult(p); });
      html += '</ul>';
    }
    if (contentHits.length) {
      html += '<h3 class="search-section">Content Matches</h3><ul class="post-list">';
      contentHits.forEach(function(p) { html += renderResult(p); });
      html += '</ul>';
    }
    resultsDiv.innerHTML = html;
  }

  function renderResult(p) {
    var h = '<li class="post-list-item">';
    h += '<h2><a href="' + p.url + '">' + escHtml(p.title) + '</a></h2>';
    h += '<div class="post-list-meta">📅 ' + p.date;
    if (p.categories.length) h += ' &middot; 📁 ' + escHtml(p.categories.join(', '));
    h += '</div>';
    if (p.excerpt) h += '<p class="post-excerpt">' + escHtml(p.excerpt) + '</p>';
    h += '</li>';
    return h;
  }

  function escHtml(s) {
    var d = document.createElement('div');
    d.textContent = s;
    return d.innerHTML;
  }
})();
</script>
