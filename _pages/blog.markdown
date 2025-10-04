---
layout: page
title: Blog
permalink: /blog/
includelink: true
comment: false
---

<div class="row content">
    {% for post in site.posts %}
    <div class="col-md-3">
        <div class="date">{{ post.date | date: "%b %d, %Y" }}</div>
    </div>
    <div class="col-md-9">
        <div><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></div>
    </div>
    
    {% endfor %}
</div>
