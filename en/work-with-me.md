---
layout: page
title: Work with me
permalink: /en/work-with-me/
lang: en
---

{% assign t = site.data.work_with_me[page.lang] %}

<p class="availability-status">{{ t.availability }}</p>

<div class="organizations-section">
  <p class="org-list-sentence">
    {{ t.org_intro }}
    {% for org in t.organizations %}
      <a href="{{ org.url }}">{{ org.name }}</a>{% unless forloop.last %}, {% endunless %}
    {% endfor %}
    {{ t.org_others }}
  </p>
  
  <div class="org-grid">
    {% for org in t.organizations %}
    <div class="org-item">
      {% if org.logo %}
      <img src="{{ org.logo | relative_url }}" alt="{{ org.name }}" title="{{ org.name }}" class="org-logo">
      {% else %}
      <span>{{ org.name }}</span>
      {% endif %}
    </div>
    {% endfor %}
  </div>
</div>

<div class="testimonials-section">
  <h3>{{ t.testimonial_title }}</h3>
  {% for testimonial in t.testimonials %}
  <div class="testimonial-item">
    <div class="testimonial-author">
      <img src="{{ testimonial.avatar | relative_url }}" alt="{{ testimonial.author }}" class="testimonial-avatar">
      <div class="author-info">
        <a href="{{ testimonial.company_url }}" class="author-name">{{ testimonial.author }}</a>, {{ testimonial.role }} chez <a href="{{ testimonial.company_url }}">{{ testimonial.company }}</a>
      </div>
    </div>
    <blockquote class="testimonial-quote">
      {{ testimonial.quote }}
    </blockquote>
  </div>
  {% endfor %}
</div>
