---
layout: home
author_profile: true
head-extra: head-custom.html
---

Welcome to **NowGlide** — my ServiceNow learning blog!
Update 18.

{% assign restaurants = site.breweries | where: "sub-type", "restaurant" %}
{% for brewery in restaurants %}
  <h1>{{ brewery.title }}</h1>
  <p>{{ brewery.location }}</p>
{% endfor %}
