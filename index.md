

<style type="text/css" >
  #post_list li{
    height:30px;
    line-height:30px;
  }
</style> 

<br />
<br />
<ul id="post_list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>


<br />
xxxxx

{% assign image_files = site.static_files | where: "geek", true %}
{% for myimage in image_files %}
 
  <a href="{{ site.url }}/{{ myimage.path }}">{{ myimage.path }}</a>
{% endfor %}
 
