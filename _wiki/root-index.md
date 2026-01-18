---
layout  : wikiindex
title   : wiki
toc     : true
public  : true
comment : false
updated : 2026-01-18 22:20:36 +0900
regenerate: true
---

* TOC
{:toc}


## [[/network]]
## [[/gpg]]


## [[how-to]]

* [[mathjax-latex]]


---

## blog posts
<div>
    <ul>
{% for post in site.posts %}
    {% if post.public == true %}
        <li>
            <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">
                {{ post.title }}
            </a>
        </li>
    {% endif %}
{% endfor %}
    </ul>
</div>

