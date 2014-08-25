---
layout: page
title: Blog of zouliping
---
{% include JB/setup %}

<ul class="posts">
  {% for post in site.posts %}
    <h3>
      <span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a>
    </h3>
  {% endfor %}
</ul>
