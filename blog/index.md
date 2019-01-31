---
layout: page
title: Blog
---

<ul class="post-list">
{% for post in site.categories.blog %} 
  <article>
      <span class="post-date">{{ post.date | date_to_string }}</span>
      <a href="{{ site.url }}/{{ post.url }}"> {{ post.title }} </a>
  </article>
{% endfor %}
</ul>