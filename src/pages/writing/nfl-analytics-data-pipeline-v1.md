---
layout: ../../layouts/ArticleLayout.astro
title: Designing an NFL Player Impact Pipeline (V1)
description: Attribution-driven NFL analytics using play-by-play data, explicit assumptions, and versioned metrics.
pubDate: "2025-01-15"
tags: ["Systems Design", "Sports Analytics", "Data Pipelines", "Python", "NFL"]
---

> This post is adapted from the project README and focuses on **engineering decisions, modeling assumptions, and system design tradeoffs**, not football takes.

# NFL Impact Forecasting (V1)

## Why I built this

I love football — and I love arguing about it.

But most NFL debates collapse immediately because people skip the hardest part:

> *What exactly are we measuring, and what assumptions are we making?*

Player impact is inherently subjective. Football is interdependent, contextual, and noisy. Pretending otherwise doesn’t make models better — it just hides the assumptions.

This project is my attempt to do the opposite:
- make assumptions **explicit**
- encode them into the system
- version them
- test them
- and leave room for disagreement

The goal is not to claim truth.  
The goal is to build a system where **truth can evolve without breaking everything downstream**.

---

## The problem in one paragraph

Using NFL play-by-play data, we want to attribute value to individual players in a way that is:

- consistent  
- reproducible  
- defensible  
- extensible  

Given a play with an Expected Points Added (EPA) value, the system must decide **who gets credit**, **how much**, and **why** — while supporting reruns, metric revisions, and multi-year analysis.

Correctness and clarity matter more than sophistication.

---

## Constraints first, architecture second

Before writing any attribution logic, I locked in a few non-negotiable constraints:

- **Assumptions must be declared**, not implied  
- **Raw data must be immutable**  
- **Attribution must be deterministic**  
- **Metrics must be versioned**  
- **Re-runs must be idempotent**  

If you can’t rerun a season and get the same result, the system isn’t trustworthy — no matter how clever the model is.

---

## High-level architecture

```
Raw NFL PBP (nflverse)
↓
Parquet Artifacts (immutable)
↓
Canonical Plays (SQLite)
↓
Play-Level Impact Attribution (versioned)
↓
Season Aggregations / Reporting
```
The pipeline is intentionally linear and layered.

Each stage produces **new derived data** rather than mutating existing records, which keeps recomputation explicit and auditable.

---

## Core design principles

### 1. Immutable raw data

Play-by-play data is ingested once and stored as immutable Parquet artifacts.

This provides:
- a permanent audit trail  
- safe historical backfills  
- insulation from upstream data changes  

If attribution logic changes, the past is recomputed — not patched.

---

### 2. Canonical plays as the foundation

Before attribution, all plays are normalized into a single canonical table.

This includes:
- play type normalization (PASS / RUSH / OTHER)
- filtering invalid or non-actionable plays
- consistent identifiers for games, plays, and players

Every downstream metric depends on this table.  
If something is wrong here, everything downstream is wrong — so this layer is deliberately boring and conservative.

---

### 3. Attribution is assumption-driven, not “correct”

V1 uses a deliberately simple attribution rule:

| Play Type | Attribution |
|---------|------------|
| PASS | 70% QB, 30% Receiver |
| RUSH | 100% Rusher |
| OTHER | Ignored |

This is stored explicitly as:

```
impact_version = "epa_v1_qb70_rec30"
```

This is not a claim of truth.

It *is* a claim of:
- consistency  
- testability  
- debuggability  

Future versions can change the weights, add offensive line credit, or introduce defensive attribution — **without overwriting history**.

---

### 4. Versioned metrics are non-negotiable

Every impact row is keyed by:

```
(game_id, play_id, player_id, impact_version)
```

This guarantees:
- no silent overwrites  
- safe re-runs  
- side-by-side metric comparisons  

If V2 disagrees with V1, that’s a feature — not a bug.

---

## Example schema: play_impact

Each row represents a single attribution decision:

| Column | Description |
|------|------------|
| game_id | NFL game identifier |
| play_id | Play identifier |
| player_id | Player credited |
| role | QB / WR / RB (V1) |
| impact_metric | EPA |
| impact_value | Attributed EPA |
| impact_version | Attribution ruleset |
| batch_id | Lineage back to raw data |
| computed_at | Computation timestamp |

Nothing is aggregated prematurely.  
Season-level insights are derived later, intentionally.

---

## Testing philosophy

Tests focus on **invariants**, not exact values:

- total impact per play equals EPA  
- missing players don’t break attribution  
- re-running attribution doesn’t duplicate rows  
- metric versions don’t overwrite each other  

If those invariants hold, the system is doing its job — even if the assumptions evolve.

---

## Why this is position-agnostic (eventually)

V1 still labels QB / WR / RB roles explicitly.

But the long-term goal is **position-agnostic impact**:
- value attributed based on involvement, not title  
- aggregation driven by contribution, not roster slot  

This is harder — and more honest — than position-specific metrics.

---

## What I intentionally didn’t build

This project deliberately excludes:

- predictive models  
- fantasy scoring  
- betting logic  
- dashboards or visualizations  

Those are downstream consumers.

This project exists to answer a more fundamental question:

> *Can we build an analytics system where assumptions are first-class citizens?*

---

## What comes next

Planned evolutions include:

- offensive line attribution  
- defensive impact modeling  
- snap-weighted contributions  
- multi-season forecasting (5–10 year horizon)  
- capital-aware team projections  

Each extension builds on the same foundation rather than replacing it.

---

## Closing thoughts

Football analytics debates rarely fail because of math.

They fail because assumptions are hidden, pipelines are brittle, and history gets quietly rewritten.

This project is my attempt to build the opposite:
- explicit assumptions  
- deterministic pipelines  
- versioned disagreement  

If people argue with the results — good.  
At least now we’re arguing about something real.

---

**Source code:**  
https://github.com/tkalp/nfl-analytics
