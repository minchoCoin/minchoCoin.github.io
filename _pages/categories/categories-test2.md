---
title: "test2 categories"
layout: archive
permalink: /categories/test2
author_profile: true
sidebar_main: true
---


{% assign posts = site.categories.test2_cat %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}