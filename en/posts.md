---
layout: page
title: Posts
permalink: /en/posts/
lang: en
---

<div class="posts-list">
  {%- assign lang = page.lang | default: "en" -%}
  {%- assign posts = site.posts | where: "lang", lang -%}
  {%- for post in posts -%}
  <article class="post-card">
    <h3 class="post-card-title">
      <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h3>
    {%- if post.excerpt -%}
    <div class="post-card-excerpt">
      {{ post.excerpt | strip_html | truncatewords: 30 }}
    </div>
    {%- endif -%}
    <div class="post-card-meta">
      {{ post.date | date: "%B %d, %Y" }} · {% include read-time.html content=post.content %} · {{ post.author | default: site.author.name | default: site.title }}
    </div>
  </article>
  {%- endfor -%}
</div>
