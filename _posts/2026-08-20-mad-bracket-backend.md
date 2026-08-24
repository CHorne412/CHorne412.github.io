---
title: "Building the backend for a March Madness bracket app"
date: 2026-08-20
tags: [python, flask, mysql, databases]
description: >-
  Five tables, a Flask API, and the JOIN that turns teams, conferences, and
  season stats into one response the frontend can render.
---

Mad Bracket was a four-person database course project: a web app where you fill
out a tournament bracket, look up team stats, and save what you built. My
teammates wrote the TypeScript frontend. I wrote the schema and the Flask API
underneath it.

The code is public — [CS440-MadBracket](https://github.com/kennethsteuart/CS440-MadBracket)
— under a teammate's account, since that's where we worked.

## Modelling the domain

Five tables, and the relationships are what make the queries interesting:

- **Conference** — the conferences
- **Team** — belongs to a conference
- **Stats** — a team's record: rank, wins, losses, conference wins and losses
- **Coach** — tied to a team
- **Bracket** — a saved bracket, with a timestamp

Foreign keys run `Team → Conference`, `Stats → Team`, and `Coach → Team`.

Splitting `Stats` out of `Team` rather than adding columns is the decision I'd
defend hardest. A team's identity and a team's season record change on
completely different schedules — the name is stable, the record changes weekly.
Keeping them in one table means every stat update rewrites identity data that
didn't change, and it makes multi-season support a schema migration instead of
another row.

## One request, three tables

The stats endpoint is the clearest example of why the schema is shaped this
way. A user types a school name; the frontend needs the team, its record, and
its conference in a single response:

```sql
SELECT t.team_id, t.team_name, s.team_rank, s.wins, s.losses,
       s.conference_wins, s.conference_losses,
       c.conference_name
FROM Team t
JOIN Conference c ON t.conference_id = c.conference_id
JOIN Stats s ON t.team_id = s.team_id
WHERE t.school_name = %s
```

Three tables, one round trip. The alternative — fetch the team, then its stats,
then its conference — is three round trips to render one card, and that cost
lands on every team the user looks at.

That `%s` matters too. It's a parameterized query, not string formatting, so a
school name containing a quote is data rather than SQL. Every query in the API
is written this way.

## Brackets get full CRUD

Saved brackets are the one thing users create, so `/stored_brackets` supports
all four operations: `POST` to save, `GET` to list newest first, `PUT` to
rename or update, `DELETE` to remove. The bracket itself is stored as a JSON
blob alongside its name.

Storing the bracket as JSON rather than modelling every matchup as rows was a
deliberate call. The bracket is only ever read and written whole, by one user,
and nothing queries *inside* it — so normalizing it would have bought
constraints we'd never use. If we'd needed questions like "how many people
picked this upset," that trade flips immediately.

## Credentials stay out of the repo

Database connection details come from environment variables loaded from a
`.env` file, not from constants in the source. It's four extra lines and it's
the difference between a public repo and a public password. Worth doing on
day one of any project that touches a database, because retrofitting it means
rewriting history.

## What working in a shared repo actually taught me

Nine commits from me, out of about forty across four people, into someone
else's repository. The API contract was the interesting part: the frontend
needed shapes I hadn't thought about, and "the backend is done" turned out to
mean "done until someone tries to use it."

The commit where I fixed the backend so it communicated with the frontend, and
the one a week later where saved brackets actually persisted, are the two
points where this stopped being my project and started being ours.
