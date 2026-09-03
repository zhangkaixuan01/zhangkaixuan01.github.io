---
layout: default
title: 首页
---
<section class="hero">
  <p class="eyebrow">TECH NOTES</p>
  <h1>你好，我是 {{ site.author }}。</h1>
  <p>{{ site.description }}</p>
</section>

<section>
  <h2>最新文章</h2>
  <div class="post-list">
    {% for post in site.posts %}
      <article class="post-card">
        <p class="post-date">{{ post.date | date: "%Y-%m-%d" }}</p>
        <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
        {% if post.description %}<p>{{ post.description }}</p>{% else %}<p>{{ post.excerpt | strip_html | strip_newlines | truncate: 120 }}</p>{% endif %}
      </article>
    {% else %}
      <p>还没有文章，写下第一篇吧。</p>
    {% endfor %}
  </div>
</section>
