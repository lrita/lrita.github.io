---
layout: page
title: About
description: 一个码农
keywords: lrita, Neal
comments: false
menu: 关于
permalink: /about/
---

## 信仰

* 追根溯源
* 不断求知

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
