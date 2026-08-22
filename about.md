---
layout: page
title: About
subtitle: Who I am and what I'm looking for
description: Background, experience, and how to reach me.
permalink: /about/
---

I'm a software engineer based in **{{ site.location }}**. I work mostly on the
back end — services, data stores, and the tooling that keeps a team shipping
without fear.

Right now I'm interested in roles where the hard part is the system, not the
process: high-throughput data infrastructure, developer platforms, or anything
where correctness under load actually matters.

## Experience

**Senior Software Engineer** · Company Name · 2023 – present
: Led the migration of the billing pipeline from nightly batch to streaming,
cutting settlement latency from 18 hours to under 90 seconds. Owned the
service through two 10× traffic events.

**Software Engineer** · Previous Company · 2020 – 2023
: Built and operated the internal API gateway used by 40+ services. Introduced
contract testing, which took integration incidents from roughly one a week to
one a quarter.

## Tools I reach for

Go, Rust, TypeScript, Python · Postgres, Redis, Kafka · Kubernetes, Terraform,
AWS · OpenTelemetry and a healthy suspicion of dashboards nobody reads.

## Elsewhere

- Email — [{{ site.email }}](mailto:{{ site.email }})
{% if site.social.github %}- GitHub — [@{{ site.social.github }}](https://github.com/{{ site.social.github }}){% endif %}
{% if site.social.linkedin %}- LinkedIn — [/in/{{ site.social.linkedin }}](https://www.linkedin.com/in/{{ site.social.linkedin }}){% endif %}
{% if site.resume_url %}- [Résumé (PDF)]({{ site.resume_url | relative_url }}){% endif %}

If you're hiring, or you just want to argue about database isolation levels,
[send me a note](mailto:{{ site.email }}).
