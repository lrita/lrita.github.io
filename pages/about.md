---
layout: page
title: About
description: 一个码农
keywords: lrita, Neal
comments: true
menu: 关于
permalink: /about/
---

## 信仰

* 追根溯源
* 不断求知

## 联系

* GitHub：[@lrita](https://github.com/lrita)
* 博客：[{{ site.title }}]({{ site.url }})
* 微博: [@苟12之](http://weibo.com/u/3444130812)

## Skill Keywords

#### Software Engineer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Research Keywords
<div class="btn-inline">
    {% for keyword in site.skill_research %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>
