---
layout: default
title: 首页

---

<ul>
	{% for post in site.posts %}
	<li>
		<a href="{{ post.url }}">{{ post.title }}</a><br/>
		{{ post.date | date: "%F"}}<br/>
		{% if post.category %}{{ post.category }}<br/>{% endif %}
		{% if post.excerpt %}{{ post.excerpt }}{% endif %}
	</li>
	{% endfor %}
</ul>

