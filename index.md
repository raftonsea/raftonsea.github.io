



<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
</head>
<body>
 　　---
　　layout: default
　　title:  
　　---
　　<h2>{{ page.title }}</h2>
　　<p>恷仟猟嫗</p>
　<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
</body>
</html>