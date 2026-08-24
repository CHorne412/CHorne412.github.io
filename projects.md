---
layout: default
title: Projects
description: Things I have designed, built, and shipped.
permalink: /projects/
---

<div class="wrap wrap--wide">
  <h1>Projects</h1>
  <p style="color:var(--text-muted);max-width:38rem">
    Coursework and side projects. Where the source isn't mine to publish, the
    card says so.
  </p>

  {%- if site.data.projects.size == 0 %}
  <p style="color:var(--text-muted)">Nothing listed yet — I'm adding projects as I finish them.</p>
  {%- endif %}

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
