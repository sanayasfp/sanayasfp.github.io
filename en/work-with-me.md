---
layout: page
title: Work with me
permalink: /en/work-with-me/
lang: en
---

{% assign t = site.data.work_with_me[page.lang] %}

<p class="availability-status">{{ t.availability }}</p>

{% include work-with-me-orgs.html lang=page.lang %}

{% if t.testimonials.size > 0 %}
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
{% endif %}
