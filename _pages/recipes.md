---
layout: default
title: Recipes
---

<section class="posts">
<ul>
{% for recipe in site.recipes %}
<li><a href="{{ site.baseurl }}{{ recipe.url }}">{{ recipe.title }}</a><time datetime="{{ recipe.date | date_to_xmlschema }}">{{ recipe.date | date: "%d-%m-%Y" }}</time></li>
{% endfor %}
</ul>
</section>

