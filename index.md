---
layout: splash
author_profile: true
title: "Cybersecurity Writeups & Projects"
excerpt: "Blue teaming • Threat hunting • Incident response • Offensive security"
header:
  overlay_color: "#000"
  overlay_filter: "0.6"
  actions:
    - label: "LinkedIn"
      url: "https://www.linkedin.com/in/gabriel-barbir/"
    - label: "GitHub"
      url: "https://github.com/Zero-For683"
---


{% include feature_row id="intro" type="center" %}
{% include paginator.html %}

## Recent Posts
{% for post in site.posts limit:5 %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
