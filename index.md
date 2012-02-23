---
layout: page
title: Index
---
{% include JB/setup %}

I don't really expect to update this blog very often, however I figured that I should create it just to test out Jekyll.

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
