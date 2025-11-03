---
title: "quantum mechanics and computing"
layout: archive
permalink: categories/quantum
author_profile: true
sidebar_main: true
---


{% assign posts = site.categories.quantum %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}