# this is my github.io site

## reference
 - https://ansohxxn.github.io/blog/category/
 - https://eona1301.github.io/github_blog/GithubBlog-Posting/
 - https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/


## 새로운 리스트 만들기(메모)
 - origin: https://ansohxxn.github.io/blog/category/

 - pages/categories/category-000.md 만들기

 ```h
 ---
title: "C++ 프로그래밍"
layout: archive
permalink: categories/cpp
author_profile: true
sidebar_main: true
---


{% assign posts = site.categories.Cpp %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}
 ```

 - nav_list_main 수정

```h
{% assign sum = site.posts | size %}

<nav class="nav__list">
  <input id="ac-toc" name="accordion-toc" type="checkbox" />
  <label for="ac-toc">{{ site.data.ui-text[site.locale].menu_label }}</label>
  <ul class="nav__items" id="category_tag_menu">
      <!--전체 글 수-->
      <li>
            📂 <span style="font-family:'Cafe24Oneprettynight';">전체 글 수</style> <span style="font-family:'Coming Soon';">{{sum}}</style> <span style="font-family:'Cafe24Oneprettynight';">개</style> 
      </li>
      <li>
        <!--span 태그로 카테고리들을 크게 분류 ex) C/C++/C#-->
        <span class="nav__sub-title">C/C++/C#</span>
            <!--ul 태그로 같은 카테고리들 모아둔 페이지들 나열-->
            <ul>
                <!--Cpp 카테고리 글들을 모아둔 페이지인 /categories/cpp 주소의 글로 링크 연결-->
                <!--category[1].size 로 해당 카테고리를 가진 글의 개수 표시--> 
                {% for category in site.categories %}
                    {% if category[0] == "Cpp" %}
                        <li><a href="/categories/cpp" class="">C ++ ({{category[1].size}})</a></li>
                    {% endif %}
                {% endfor %}
            </ul>
            <ul>
                {% for category in site.categories %}
                    {% if category[0] == "STL" %}
                        <li><a href="/categories/stl" class="">C++ STL & 표준 ({{category[1].size}})</a></li>
                    {% endif %}
                {% endfor %}
            </ul>
        <span class="nav__sub-title">Coding Test</span>
            <ul>
                {% for category in site.categories %}
                    {% if category[0] == "Algorithm" %}
                        <li><a href="/categories/algorithm" class="">알고리즘 구현 (with C++) ({{category[1].size}})</a></li>
                    {% endif %}
                {% endfor %}
            </ul>
            <ul>
                {% for category in site.categories %}
                    {% if category[0] == "Programmers" %}
                        <li><a href="/categories/programmers" class="">프로그래머스 ({{category[1].size}})</a></li>
                    {% endif %}
                {% endfor %}
            </ul>
      </li>
  </ul>
</nav>

```

 - 새로운 카테고리로 글 작성
 ```
 ---
title: "Post: Modified Date"
last_modified_at: 2023-09-13T23:53:12+09:00
categories:
    - test_cat
tags:
    - test_tag

toc: true
toc_sticky: true
toc_label: "My Table of Contents"
author_profile: true
sidebar_main: true
---
# Lorem
Lorem ipsum dolor sit amet...
 ```