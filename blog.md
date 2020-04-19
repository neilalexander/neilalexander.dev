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
  <ul>
  {% for post in indexable_posts %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
      </li>
  {% endfor %}
  </ul>
{% endunless %}
