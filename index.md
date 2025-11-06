---
title: "Home"
layout: default
---

# Welcome to My Notes

This is a collection of my learning posts, discussions, and thoughts.  
Browse the posts below or check my latest update.

## Posts

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }})

{% endfor %}
