����---
����layout: default
����title:  
����---
����<h2>{{ page.title }}</h2>
����<p>��������</p>
��<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>