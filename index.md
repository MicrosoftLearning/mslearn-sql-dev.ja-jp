---
title: オンラインでホストされる手順
permalink: index.html
layout: home
---

# Azure SQL 開発の演習

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %} [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) {% endfor %}


