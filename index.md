---
layout: default
title: Home
---

<div class="home-intro" markdown="1">

Hi, I'm Yuzhu. This is a record of ongoing thought — an evolving notebook of ideas spanning computer science, mathematics, biology, and whatever else happens to be on my mind.

I write here mostly to think out loud and keep track of things I don't want to forget. Have a look around.

</div>

<div class="home-recent">
  <h2>Recent posts</h2>
  <ul class="post-list">
    {% for post in site.posts limit:5 %}
    <li>
      <span class="post-date">{{ post.date | date: "%B %-d, %Y" }}</span>
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 160 }}</p>
    </li>
    {% endfor %}
  </ul>
</div>
