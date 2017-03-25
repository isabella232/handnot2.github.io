---
layout: default
title: Hangar
index_page: true
---
{% for post in site.posts %}
  <blog-date>{{ post.date | date_to_string }}</blog-date>
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  {{ post.description }}
  
  {{ post.excerpt }}
{% endfor %}
