---
layout: archive
title: "Blog"
permalink: /blog/
author_profile: true
---

{% for post in site.categories.blog %}
**{{ post.date | date: "%B %d, %Y" }}**
· 🕒 {{ post.content | number_of_words | divided_by:200 }} min read  
[{{ post.title }}]({{ post.url }})

---
{% endfor %}
