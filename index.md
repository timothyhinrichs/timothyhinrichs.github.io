---
layout: base_without_sidebar
comments: False
---

{% for post in site.posts limit:4 %}

<div class="indexpost">
<h1><span class="indextitle">{{ post.date | date_to_string }}:
    <a href="{{ post.url }}">{{ post.Title }}</a></span></h1>
<div class="postexcerpt">
{{ post.excerpt }}
<!-- {{post.excerpt | truncatewords:150, "..."}} -->
</div>
<hr class="softhr" />
</div>

{% endfor %}





