---
layout: page
title: essay
permalink: /essay/
---

<div class="essays"> 
<h1><a href="{{ site.baseurl }}">Jesse's Posts </a></h1>
<div class="back">
<a href={{site.url}}>↩︎</a>
</div>

<ul>
    {% for post in site.posts %}
      <li>
        <a href="{{ post.url | prepend: site.baseurl }}">
           <b>{{ post.title }}</b>
           <span>{{ post.date | date: '%B %-d, %Y — %H:%M' }}</span>
        </a>
      </li>
    {% endfor %}
</ul>
</div>
