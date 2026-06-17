---
layout: page
title: Tags
permalink: /tags/
lang: fr
---

{% assign tags = "" | split: "," %}
{% for post in site.posts %}
  {% if post.lang == page.lang %}
    {% for tag in post.tags %}
      {% assign tags = tags | push: tag %}
    {% endfor %}
  {% endif %}
{% endfor %}
{% assign unique_tags = tags | uniq | sort %}

<div class="tags-archive-container">
  <!-- Tag Cloud Section -->
  <div id="tag-cloud-section">
    <div class="tag-cloud">
      {% for tag in unique_tags %}
        {% assign count = 0 %}
        {% for post in site.posts %}
          {% if post.lang == page.lang and post.tags contains tag %}
            {% assign count = count | plus: 1 %}
          {% endif %}
        {% endfor %}

        <a href="#{{ tag | slugify }}" class="tag-archive-button">
          #{{ tag }}
          <span class="tag-count">{{ count }}</span>
        </a>
      {% endfor %}
    </div>
  </div>

  <!-- Tag Results Section -->
  <div id="tag-results-section" style="display: none; margin-top: 1rem;">
    <div id="tag-posts-list">
      {% for tag in unique_tags %}
        <div class="tag-group" id="group-{{ tag | slugify }}" style="display: none;">
          <div class="posts-list">
            {% for post in site.posts %}
              {% if post.lang == page.lang and post.tags contains tag %}
                <article class="post-card">
                  <h3 class="post-card-title">
                    <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
                  </h3>
                  <div class="post-card-meta">
                    {{ post.date | date: "%B %d, %Y" }} · {% include read-time.html content=post.content %}
                  </div>
                  <div class="post-card-content">
                    {{ post.content | strip_html | strip_newlines }}
                  </div>
                </article>
              {% endif %}
            {% endfor %}
          </div>
        </div>
      {% endfor %}
    </div>
  </div>
</div>

<script>
  const originalTitle = document.querySelector('.post-title').textContent;

  function handleTagChange() {
    const hash = window.location.hash.substring(1);
    const cloudSection = document.getElementById('tag-cloud-section');
    const resultsSection = document.getElementById('tag-results-section');
    const groups = document.querySelectorAll('.tag-group');
    const mainTitle = document.querySelector('.post-title');

    if (hash) {
      cloudSection.style.display = 'none';
      resultsSection.style.display = 'block';

      groups.forEach(g => g.style.display = 'none');
      const activeGroup = document.getElementById('group-' + hash);

      if (activeGroup) {
        activeGroup.style.display = 'block';
        const displayTag = '#' + hash.replace(/-/g, ' ');
        mainTitle.textContent = displayTag;
      } else {
        window.location.hash = '';
      }
    } else {
      cloudSection.style.display = 'block';
      resultsSection.style.display = 'none';
      mainTitle.textContent = originalTitle;
    }
  }

  window.addEventListener('hashchange', handleTagChange);
  window.addEventListener('DOMContentLoaded', handleTagChange);
</script>
