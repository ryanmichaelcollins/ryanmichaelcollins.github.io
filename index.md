---
layout: default
title: Ryan M. Collins's online scratchpad and stuff
---
# [](#header-1)My home page
This is an attempt at setting up a webiste through Github pages. I just set this whole thing up recently, so this is a (rough) work in progress. The appearance may vary a lot as I play with themes and formats and whatnot.

# Posts
{% for post in site.posts %}
* {{ post.date | date_to_string }} [{{ post.title }}]({{ post.url }})
{% endfor %}
