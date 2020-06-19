---
sitemap: true
redirect_from: 
- "/articles"
- "/posts"
---

## Blog

{% assign indexable_posts = site.posts | where: "index",true %}

{% unless indexable_posts.size > 0 %}
There aren't any posts here yet. Check back soon!
{% else %}
  {% for post in indexable_posts %}
  <div class='blogpost'>
    <div id='date'>
      <div id='day'>{{ post.date | date: "%-d" }}</div>
      <div id='month'>{{ post.date | date: "%b %Y" }}</div>
    </div>
    <div id='overview'>
      <div id='title'><a href="{{ post.url }}">{{ post.title }}</a></div>
      <div id='excerpt'>{{ post.excerpt | strip_html | truncatewords: 10 }}</div>
    </div>
  </div>
  {% endfor %}
{% endunless %}

<div style='text-align: center; margin-top: 3em;'><a href="/feed.xml">Atom/RSS</a></div>
