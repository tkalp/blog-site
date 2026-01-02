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

## V1 Example Results

As of Week 17 of the 2025 season, the V1 pipeline produces the following results.

```
Top 30 players by total impact (EPA attribution):
rank player_name         player_id                total_epa    plays   epa/play
   1 Drake Maye            00-0039851                +107.09    540   +0.198
   2 Jordan Love           00-0036264                 +83.66    469   +0.178
   3 Matthew Stafford      00-0026498                 +83.07    579   +0.143
   4 Dak Prescott          00-0033077                 +75.74    627   +0.121
   5 Josh Allen            00-0034857                 +72.87    548   +0.133
   6 Jared Goff            00-0033106                 +67.59    573   +0.118
   7 Brock Purdy           00-0037834                 +61.93    279   +0.222
   8 Daniel Jones          00-0035710                 +51.96    423   +0.123
   9 Patrick Mahomes       00-0033873                 +51.19    540   +0.095
  10 Sam Darnold           00-0034869                 +41.06    480   +0.086
  11 Bo Nix                00-0039732                 +40.91    642   +0.064
  12 Puka Nacua            00-0039075                 +39.10    163   +0.240
  13 Joe Burrow            00-0036442                 +33.83    240   +0.141
  14 C.J. Stroud           00-0039163                 +32.52    433   +0.075
  15 Mac Jones             00-0036972                 +32.16    318   +0.101
  16 George Pickens        00-0037247                 +27.76    135   +0.206
  17 Trevor Lawrence       00-0036971                 +27.24    599   +0.045
  18 Jalen Hurts           00-0036389                 +27.19    538   +0.051
  19 Malik Willis          00-0038128                 +26.93     45   +0.598
  20 Caleb Williams        00-0039918                 +26.35    579   +0.046
  21 Jonathan Taylor       00-0036223                 +26.19    363   +0.072
  22 Jaxon Smith-Njigba    00-0038543                 +25.49    162   +0.157
  23 Trey McBride          00-0037744                 +21.35    162   +0.132
  24 Stefon Diggs          00-0031588                 +20.39     99   +0.206
  25 AJ Barner             00-0039793                 +15.83     75   +0.211
  26 DeVonta Smith         00-0036912                 +15.81    110   +0.144
  27 Amon-Ra St. Brown     00-0036963                 +15.46    160   +0.097
  28 D'Andre Swift         00-0036275                 +14.35    259   +0.055
  29 Nico Collins          00-0036554                 +13.79    123   +0.112
  30 Jameson Williams      00-0037240                 +13.74    100   +0.137
```

### Interpreting These Results

These results strongly favor quarterbacks. This is expected behavior, not a flaw.

Under V1 assumptions:

- Pass plays attribute 70% of EPA to the QB
- Rushing EPA is attributed solely to the rusher
- Offensive line, defensive pressure, coverage quality, and situational leverage are not yet modeled

Because quarterbacks touch the ball on nearly every offensive play, the system naturally concentrates impact at the QB position.

This outcome serves as a validation of the attribution rules, not a claim about true player value.

### Why This Is Important

The purpose of V1 is not to produce “correct” rankings — it is to:

- Prove the pipeline is internally consistent
- Verify that impact attribution sums correctly
- Expose systemic bias introduced by assumptions
- Create a baseline for future iterations

Seeing quarterbacks dominate the rankings confirms that the system is behaving deterministically and transparently under its declared rules.

### What This Unlocks Next

These results directly motivate future work:

- Offensive line attribution on pass and rush plays
- Position-agnostic impact normalization
- Defensive player involvement
- Snap-weighted and context-adjusted impact
- Multi-year trajectory modeling

Crucially, all of these improvements can be introduced as new impact versions without invalidating or overwriting V1.

That is the core design goal.

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
