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
  <div class="tag-cloud">
    {% for tag in unique_tags %}
      {% assign count = 0 %}
      {% for post in site.posts %}
        {% if post.lang == page.lang and post.tags contains tag %}
          {% assign count = count | plus: 1 %}
        {% endif %}
      {% endfor %}
      
      <a href="{{ '/tags/' | append: tag | slugify | relative_url }}" class="tag-archive-button">
        #{{ tag }}
        <span class="tag-count">{{ count }}</span>
      </a>
    {% endfor %}
  </div>
</div>
