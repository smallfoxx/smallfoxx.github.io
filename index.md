---
layout: default
---
<link rel="stylesheet" type="text/css" href="{{ site.baseurl }}/style.css" />
<div class="posts">
  {% for post in site.posts %}
    <article class="post">

      <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

      <div class="entry">
        {{ post.excerpt }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
    </article>
  {% endfor %}
    <article class="post">
    <br>
          {% include svg-icons.html %}
    </article>
</div>
