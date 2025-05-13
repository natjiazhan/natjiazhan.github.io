---
layout: archive
title: "Blog"
permalink: /blog/
author_profile: true
---

{% for post in site.categories.blog %}
  - **{{ post.date | date: "%B %d, %Y" }}** — [{{ post.title }}]({{ post.url }})
{% endfor %}
