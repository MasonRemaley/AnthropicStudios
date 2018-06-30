---
layout: page
title: Blog
---

<!-- <h2 class="post-list-heading">{{ page.list_title | default: "Projects" }}</h2> -->
{% assign posts = site.posts | sort: 'order' %}
<ul class="post-list">
  {% for post in posts %}
  <li>
    <h3>
      <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
    </h3>
    {% if site.show_excerpts %}
      <p class="post-meta">
        <time class="dt-published" datetime="{{ post.date | date_to_xmlschema }}" itemprop="datePublished">
          {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
          {{ post.date | date: date_format }}
        </time>
        {%- if post.author -%}
          {% assign author = site.authors[post.author] %}
          • <span itemprop="author" itemscope itemtype="http://schema.org/Person"><span class="p-author h-card" itemprop="name"><a href="{{ author.social }}">{{ author.name }}</a></span></span>
        {%- endif -%}
      </p>
      {{ post.excerpt }}
      <a href="{{ post.url | relative_url }}">Read more...</a>
    {% endif %}
  </li>
  {% endfor %}
</ul>
