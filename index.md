---
layout: default
title: Mārtiņš Kalvāns 
---

# Mārtiņš Kalvāns

<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p>{{ post.excerpt | strip_html | truncatewords: 40 }}</p>
  </li>
  {% endfor %}
</ul>
