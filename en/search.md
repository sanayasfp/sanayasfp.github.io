---
layout: page
title: Search
permalink: /en/search/
lang: en
---

<div class="search-container">
  <input type="text" id="search-input" placeholder="Search posts..." autocomplete="off" autofocus class="search-input">
  <ul id="results-container" class="posts-list"></ul>
</div>

<script src="{{ '/assets/js/simple-jekyll-search.min.js' | relative_url }}"></script>

<script>
  window.simpleJekyllSearch = SimpleJekyllSearch({
    searchInput: document.getElementById('search-input'),
    resultsContainer: document.getElementById('results-container'),
    json: '{{ "/en/search.json" | relative_url }}',
    searchResultTemplate: `
      <article class="post-card">
        <h3 class="post-card-title">
          <a href="{url}">{title}</a>
        </h3>
        {tags}
        <div class="post-card-content">{content}</div>
        {date}
      </article>
    `,
    noResultsText: '<p>No results found.</p>',
    limit: 10,
    fuzzy: false,
    templateMiddleware: function(prop, value, template) {
      if (prop === 'tags') {
        if (value) {
          const tagsHtml = value.split(', ').map(t => `<span class="tag">#${t}</span>`).join(' ');
          return `<div class="post-card-tags">${tagsHtml}</div>`;
        }
        return '';
      }
      if (prop === 'date') {
        return value ? `<div class="post-card-meta">${value}</div>` : '';
      }
      return value;
    }
  });
</script>
