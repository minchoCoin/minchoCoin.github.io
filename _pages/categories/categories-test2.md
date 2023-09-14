---
title: "test2 categories"
layout: archive
permalink: /categories/test2
author_profile: true
sidebar_main: true
---


{% assign posts = site.categories.test2 %}
{% for post in posts %} {% include archive-single2.html type=page.entries_layout %} {% endfor %}