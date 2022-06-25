---
layout: page
title: About me
permalink: /about/
---

Sysops engineer - AWS Certified Cloud Practitioner - Hashicorp Certified Terraform Associate

<div id='tag_cloud'>
{% for tag in site.tags %}
<a href="#{{ tag[0] }}" title="{{ tag[0] }}" rel="{{ tag[1].size }}">{{ tag[0] }}</a>
{% endfor %}
</div>
