---
layout: page
title: Recherche
permalink: /search/
lang: fr
---

<div class="search-container">
  <input type="text" id="search-input" placeholder="Rechercher des articles..." autocomplete="off" class="search-input">
  <ul id="results-container" class="posts-list"></ul>
</div>

<script src="{{ '/assets/js/simple-jekyll-search.min.js' | relative_url }}"></script>

<script>
  window.simpleJekyllSearch = SimpleJekyllSearch({
    searchInput: document.getElementById('search-input'),
    resultsContainer: document.getElementById('results-container'),
    json: '{{ "/search.json" | relative_url }}',
    searchResultTemplate: `
      <article class="post-card">
        <h3 class="post-card-title">
          <a href="{url}">{title}</a>
        </h3>
        <div class="post-card-tags">{tags}</div>
        <div class="post-card-excerpt">{excerpt}</div>
        <div class="post-card-meta">{date}</div>
      </article>
    `,
    noResultsText: '<p>Aucun résultat trouvé.</p>',
    limit: 10,
    fuzzy: false,
    templateMiddleware: function(prop, value, template) {
      if (prop === 'tags' && value) {
        return value.split(', ').map(t => `<span class="tag">#${t}</span>`).join(' ');
      }
      return value;
    }
  });
</script>
