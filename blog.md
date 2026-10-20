---
layout: default
title: Blog
permalink: /blog/
---
<section class="section"><p class="eyebrow">NOTES & ARTICLES</p><h1>Blog</h1><p class="muted">Practical writing on Databricks, Snowflake, Microsoft Fabric, and data architecture.</p>{% for post in site.posts %}<a class="post" href="{{ post.url | relative_url }}"><div><h2>{{ post.title }}</h2><p>{{ post.excerpt | strip_html | truncate: 180 }}</p></div><span class="date">{{ post.date | date: "%b %-d, %Y" }}</span></a>{% endfor %}</section>
