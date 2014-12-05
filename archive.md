---
layout: base_without_sidebar
comments: False
---

# Archive

{% for post in site.posts %}

<div class="indexpost">
    <ul>
        <li>
            {{ post.date | date_to_string }}:
                <a href="{{ post.url }}">{{ post.Title }}</a>
        </li>
    </ul>
</div>

{% endfor %}