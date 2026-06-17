---
layout: default
title: "Nakul's GSoC 2026 Blog"
---

# Welcome to my GSoC 2026 Journey!
Documenting my progress building the Neural Extraction Framework for DBpedia.

### Weekly Updates:

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      - <i>{{ post.date | date: "%B %d, %Y" }}</i>
    </li>
  {% endfor %}
</ul>
