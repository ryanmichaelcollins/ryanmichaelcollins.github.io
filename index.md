---
layout: default
title: Ryan M. Collins's online scratchpad and stuff
---

### Home Page
This is my first attempt at setting up a website through Github pages (and in general). I just set this whole thing up recently, so this is a (rough) work in progress. The appearance may vary a lot as I play with themes and formats and whatnot. At the moment anything remotely worth reading will probably be found in the [Blog]({{ site.url }}/blog) section.

### Recent Blog Posts
{% for post in site.posts limit:5 %}
* {{ post.date | date_to_string }} [{{ post.title }}]({{ post.url }})
{% endfor %}


