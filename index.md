---
layout: page
title: Living inside the Matrix
nav-menu: home
---

{% for post in site.posts %}

<div class="page-header">
  <h1><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a> {% if post.tagline %}<small>{{post.tagline}}</small>{% endif %}</h1>
</div>

<div class="row-fluid post-full">
  <div class="span12">
    <div class="date">
      <span>{{ post.date | date_to_long_string }}</span>
    </div>
    {% if post.content contains '<!--more-->' %}
      <div class="content">
        {{ post.content | split:'<!--more-->' | first }}
      </div>
      <div class="text-right more-button-container">
        <a href="{{ BASE_PATH }}{{ post.url }}"> <b>阅读全文</b> </a>
      </div>	
    {% else %}
      <div class="content">
        {{ post.content }}
      </div>
    {% endif %}
  </div>
</div>

{% endfor %}
