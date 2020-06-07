---
layout: default
---

Pages:

<ul>
{% for post in site.posts %}
	<li><a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>:<br> {{ post.description }}</li>
{% endfor %}
</ul>

Some of my older stuff can be found [here](http://callenblogi.blogspot.com/).
