---
layout: default
description: A description about my blog homepage
---


<div id="posts">
  <h2>日志</h2>
  <ul>
    {% for post in site.posts %}
      <li><span class="date">{{ post.date | date_to_string }}</span> - <a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
</div>
<div id="pages">
  <h2></h2>
  <ul>
    {% for page in site.html_pages %}
      {% if page.title %}
        <li><a href="{{ page.url }}">{{ page.title }}</a></li>
      {% endif %}
    {% endfor %}
  </ul>
</div>

