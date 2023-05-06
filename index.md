---
layout: default
---

{% include_relative head.html %}

{% for post in site.posts %}
<div class="post-excerpt">
  <h2 class="post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
  {{ post.excerpt }}
</div>
{% endfor %}
