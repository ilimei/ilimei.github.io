---
title:测试
---
{% if page.title %}
<title>{{ page.title }}</title>
{% else %}
<title>{{ site.name }}</title>
{% endif %}