---
layout: default
---

{% for post in site.posts %}
- <span class="date">{{ post.date | date: "%Y-%m-%d" }}</span> [{{ post.title }}]({{ post.url | prepend: site.baseurl }})
{% endfor %}
