---
layout: home
permalink: /
title: "Home"
image:
    feature: Moss_1600x800.png
header:
    h1: Erick Dransch
    h2: Writer of software, drinker of coffee.
share: False
---

<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
