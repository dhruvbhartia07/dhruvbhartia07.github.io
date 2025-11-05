---
title: "Welcome"
layout: default
---

# Welcome to My Notes

This is a collection of my learning posts, discussions, and thoughts.  
Browse the posts below or check my latest update.

👉 [Go to Posts](./posts/)

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
