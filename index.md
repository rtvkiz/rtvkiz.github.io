---
layout: default
title: Home
---

# Welcome to Ritvik's Tech Blog

A security researcher with a passion for containers!

## Latest Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> — {{ post.date | date: "%b %-d, %Y" }}
    </li>
  {% endfor %}
</ul>
