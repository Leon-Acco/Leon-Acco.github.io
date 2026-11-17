---
layout: default
title: 归档
permalink: /archive/
---
<section class="page-head">
  <div class="wrap">
    <h1 class="page-title">全部文章</h1>
    <p class="page-sub">共 {{ site.posts.size }} 篇 · 按时间倒序 · 来自个人知识库与一线工程实践</p>
  </div>
</section>

<div class="wrap">
{% assign years = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in years %}
<section class="archive-year">
  <div class="year-head">{{ year.name }} <small>{{ year.items.size }} 篇</small></div>
  <ul class="archive-list">
    {% for post in year.items %}
    <li>
      <time>{{ post.date | date: "%m-%d" }}</time>
      {% for cat in post.categories limit: 1 %}<span class="badge">{{ cat }}</span>{% endfor %}
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </li>
    {% endfor %}
  </ul>
</section>
{% endfor %}
</div>
