---
layout: default
title: 归档
permalink: /archive/
---
<h1 class="page-title">🗂️ 全部文章</h1>
<p class="page-sub">共 {{ site.posts.size }} 篇 · 来自个人知识库的系统整理</p>

{% assign years = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in years %}
<section class="archive-year">
  <h2 class="series-h2">{{ year.name }} <small>{{ year.items.size }} 篇</small></h2>
  <ul class="archive-list">
    {% for post in year.items %}
    <li>
      <time>{{ post.date | date: "%m-%d" }}</time>
      {% for cat in post.categories %}<span class="badge badge-{{ cat }}">{{ cat }}</span>{% endfor %}
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </li>
    {% endfor %}
  </ul>
</section>
{% endfor %}
