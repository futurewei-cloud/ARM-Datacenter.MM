---
layout: single
permalink: /site-stats/
title: "Post and Page Stats"
sidebar:
  nav: "site_map"
classes: wide
partial: all
---

# Posts

|---
| Counter | Title | URL  |
:--------|:------|:-----|{% for post in site.posts %}
 | ![Hits](https://hitcounter.pythonanywhere.com/nocount/tag.svg?url=https://futurewei-cloud.github.io/ARM-Datacenter{{ post.url }}) | {{ post.title }} | {{ post.url }} |{%- endfor -%}
|---


# Pages

|---------|-------|------|
| Counter | Title | URL  |
|:--------|:------|:-----|{% for page in site.pages %}{% if page.title and page.url != "/" %}
| ![Hits](https://hitcounter.pythonanywhere.com/nocount/tag.svg?url=https://futurewei-cloud.github.io/ARM-Datacenter{{ page.url }}) |{{ page.title }} | {{ page.url }}{% endif %}{% endfor %}
|--
