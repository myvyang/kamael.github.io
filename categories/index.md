---
layout: page
title: Categories
excerpt: "An archive of posts sorted by categories."
---

{% capture site_cats %}{% for cat in site.categories %}{{ cat | first }}{% unless forloop.last %},{% endunless %}{% endfor %}{% endcapture %}
{% assign cats_list = site_cats | split:',' | sort %}

<ul class="tag-box inline">
  {% for item in (0..site.categories.size) %}{% unless forloop.last %}
    {% capture this_word %}{{ cats_list[item] | strip_newlines }}{% endcapture %}
    <li><a href="#{{ this_word }}">{{ this_word }} <span>{{ site.cats[this_word].size }}</span></a></li>
  {% endunless %}{% endfor %}
</ul>

{% for item in (0..site.categories.size) %}{% unless forloop.last %}
  {% capture this_word %}{{ cats_list[item] | strip_newlines }}{% endcapture %}
  <h2 id="{{ this_word }}">{{ this_word }}</h2>
  <ul class="post-list">
  {% for post in site.categories[this_word] %}{% if post.title != null %}
    <!-- <li><a href="{{ site.url }}/{{ post.url }}">{{ post.title }}<span class="entry-date"><time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %d, %Y" }}</time></span></a></li> -->
      <ul>
        <span class="post-date">{{ post.date | date_to_string }}</span>
        <a href="{{ site.url }}/{{ post.url }}"> {{ post.title }} </a>
      </ul>
  {% endif %}{% endfor %}
  </ul>
{% endunless %}{% endfor %}