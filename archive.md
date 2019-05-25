---
layout: page
title: Archive
permalink: /archive/
---

<div class="container">
  {%- if site.posts.size > 0 -%}
    <ul class="post-list">
      {%- for post in site.posts -%}
      <li>
        {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
        <span class="post-meta">{{ post.date | date: date_format }}</span>
        <a class="post-link" href="{{ post.url | relative_url }}" title="{{ post.title | escape }}">
          {{ post.title | escape }}
        </a>
      </li>
      {%- endfor -%}
    </ul>
  {%- endif -%}
</div>
