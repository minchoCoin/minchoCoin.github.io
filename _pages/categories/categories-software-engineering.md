---
title: "software engineering"
layout: archive
permalink: /categories/software-engineering
author_profile: true
sidebar_main: true
---


{% assign posts = site.categories.software-engineering %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}