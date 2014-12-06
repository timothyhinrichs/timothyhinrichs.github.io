---
layout: page
title: Blog Entries
header: Blog Entries
comments: False
---

{% for post in site.posts %}

<div class="indexpost">
    <ul>
        <li>
            {{ post.date | date_to_string }}:
                <a href="{{ post.url }}">{{ post.title }}</a>
        </li>
    </ul>
</div>

{% endfor %}