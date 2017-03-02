---
layout: default
title: Hangar
---
{% for post in site.posts %}
  {{ post.date | date_to_string }}
  <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
  {{ post.description }}
  
  {{ post.excerpt }}
{% endfor %}
