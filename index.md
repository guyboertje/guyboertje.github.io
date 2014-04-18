---
layout: page
title: Welcome
---
{% include JB/setup %}

<p>Hopefully some of this makes sense!</p>>

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<p>Thanks</p>

