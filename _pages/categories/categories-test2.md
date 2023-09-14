---
title: "test2 categories"
layout: archive
permalink: /categories/test2
---


{% assign posts = site.categories.test2 %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}