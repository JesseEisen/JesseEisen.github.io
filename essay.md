---
layout: page
title: essay
permalink: /essay/
---

<h1>Jesse's Posts</h1>

<ul>
    {% for post in paginator.posts %}
      <li>
        <a href="{{ post.url | prepend: site.baseurl }}">
           <b>{{ post.title }}</b>
           <span>{{ post.date | date: '%B %-d, %Y — %H:%M' }}</span>
        </a>
      </li>
    {% endfor %}
</ul>
