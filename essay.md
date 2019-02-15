---
layout: essay
title: essay
permalink: /essay/
---

<div class="essays"> 
<div class="essay-title"><a href="{{ site.url }}"> Blog </a></div>


<div class="post-list">
    {% for post in site.posts %}
      <div class="post-item">
        <div class="post-specify">
          <div class="date"><span>{{ post.date | date: '%B %-d, %Y — %H:%M' }}</span></div>
          <a class="title" href="{{ post.url | prepend: site.baseurl }}"><b>{{ post.title }}</b></a>
        </div>
      </div>
    {% endfor %}
</div>
</div>
