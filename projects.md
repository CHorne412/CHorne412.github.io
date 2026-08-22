---
layout: default
title: Projects
description: Things I have designed, built, and shipped.
permalink: /projects/
---

<div class="wrap wrap--wide">
  <h1>Projects</h1>
  <p style="color:var(--text-muted);max-width:38rem">
    A few things I've built. Most are open source — the write-ups on the
    <a href="{{ '/blog/' | relative_url }}">blog</a> go deeper on the parts that were hard.
  </p>

  {%- assign by_year = site.data.projects | sort: "year" | reverse -%}
  {%- assign current_year = "" -%}

  {%- for project in by_year %}
    {%- if project.year != current_year %}
      {%- unless forloop.first %}</div>{% endunless %}
      <h2 class="year-heading">{{ project.year }}</h2>
      <div class="projects projects--grid">
      {%- assign current_year = project.year -%}
    {%- endif %}
    {% include project-card.html project=project %}
    {%- if forloop.last %}</div>{% endif %}
  {%- endfor %}
</div>
