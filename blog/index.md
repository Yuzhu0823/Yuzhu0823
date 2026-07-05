---
layout: default
title: Blog
permalink: /blog/
---

<h1 class="page-title">Blog</h1>

<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <span class="post-date">{{ post.date | date: "%B %-d, %Y" }}</span>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 200 }}</p>
    <a class="read-more" href="{{ post.url | relative_url }}">Read more &rarr;</a>
  </li>
  {% endfor %}
</ul>
