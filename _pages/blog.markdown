---
layout: page
title: Blog
permalink: /blog/
includelink: true
comment: false
---

<div class="row content">
    {% for post in site.posts %}
    <div class="col-md-12">
        <div class="date">{{ post.date | date: "%b %-d, %Y" }}</div>
        <div class="post-title"><a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></div>
        <div class="post-excerpt">{{ post.excerpt }}</div>
    </div>
    
    {% endfor %}
</div>
