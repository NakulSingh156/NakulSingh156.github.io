---
layout: default
title: "Nakul's GSoC 2026 Blog"
---

# Welcome to my GSoC 2026 Journey!
Documenting my progress building the Neural Extraction Framework for DBpedia.

{% assign featured = site.posts | where: "featured", true %}
{% if featured.size > 0 %}
### Start here:

<ul>
  {% for post in featured %}
    <li>
      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      - <i>{{ post.date | date: "%B %d, %Y" }}</i>
    </li>
  {% endfor %}
</ul>
{% endif %}

### Weekly Updates:

<ul>
  {% for post in site.posts %}
    {% unless post.featured %}
    <li>
      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      - <i>{{ post.date | date: "%B %d, %Y" }}</i>
    </li>
    {% endunless %}
  {% endfor %}
</ul>
