---
name: ia-to-delivery-plan
description: >
  Auto-discover Impact Assessment documents and repo CLAUDE.md files in the
  current project, then produce phased feature plans, implementation stories,
  a t-shirt validation, a consolidated dependency graph, and a single Gantt
  delivery plan spanning all IAs.
  Use when asked to "build a delivery plan", "plan from the IA", "run the
  IA-to-delivery method", or "generate a Gantt from the impact assessment".
---

# IA to Delivery Plan

You are executing a seven-phase method that auto-discovers inputs and produces
six categories of artefact. Work through every phase in order. At each phase,
state what you are doing and flag any gaps before continuing.

---

## Phase 0 — Discover inputs

### IA documents

Scan the root folder of the current working directory. Every file except
`CLAUDE.md` and `README.md` is a candidate IA document.

For each candidate:
- Read the file and check whether it contains a Service Impact table with
  columns: `SI No | Component | Headline Req | Change Description |
  Dependent On | Assumed Team | Req Devs | T-Shirt Size | Notes`
- If the required table is absent or malformed, flag the file to the user
  and skip it:
  > **Skipped:** `<filename>` — no valid Service Impact table found.
- If the table is present, record the file as an active IA and extract its
  epic name (use the document title or filename as the epic label).

Do not invent or substitute data. Only proceed with files that contain a
valid table.

### Repo folders

From the combined set of active IAs, collect the unique list of values in the
`Component` column. These are the services in scope.

For each service:

1. **Folder exists, `CLAUDE.md` exists** — read the `CLAUDE.md` and proceed.
2. **Folder exists, no `CLAUDE.md`** — run `/init` inside that folder to
   generate a `CLAUDE.md`, then read it.
3. **Folder does not exist** — add the service to a "missing repos" list.

After checking all services, if there are missing repos, present the list:

> **Missing repos — please clone before continuing:**
> - `<service-name>` (referenced in `<ia-file>` SI-XX)
> - …
>
> Continue without these repos? [yes / no]

If the user chooses to continue, mark those services as `context: unavailable`
and proceed. Stories for unavailable services will carry a warning and will
omit the implementation hint. Do not block the entire run on missing repos.

---

## Phase 1 — Extract service impacts

For each active IA, parse the Service Impact table into a flat list:

```
(IA, SI No, Component, T-shirt, Depends On, Headline Req, Change Description)
```

- Preserve `Dependent On` verbatim — this is the dependency graph edge list.
- Preserve `T-Shirt Size` verbatim — sizing is validated in Phase 4, not here.
- Flag any `Dependent On` value that references an SI No not present in the
  same table.

---

## Phase 2 — Define phases per IA

For each active IA, group its SIs into sequential delivery phases:

- Treat `Dependent On` as a directed graph. Topologically sort to determine
  the natural phase order.
- A phase contains SIs that share the same set of predecessors and can be
  worked in parallel.
- Phase N+1 starts when all Phase N work is merged and deployed.
- Foundational changes (shared interfaces, data models, auth) go in Phase 1.

Do not impose structure the dependency graph does not support. State your
reasoning for each phase boundary.

---

## Phase 3 — Write feature files

For each active IA, create a file at:

```
features/<ia-slug>.md
```

where `<ia-slug>` is the IA filename without its extension, lowercased,
with spaces replaced by hyphens.

Each feature file contains:

```markdown
# Feature Plan — <Epic name>

**Source IA:** <ia-filename>
**Services in scope:** <comma-separated list of Component values>

## Summary

<2–4 sentence plain-English summary of what this IA delivers and why,
derived from the IA document.>

## Phased delivery

### Phase 1 — <label>

**Goal:** <one sentence>

| SI | Headline | Service | T-shirt (IA) | Depends On |
|----|----------|---------|-------------|------------|
| SI-XX | … | … | … | … |

**Phase rationale:** <why these SIs are grouped here>

### Phase 2 — <label>
… (repeat for each phase)

## Risks and dependencies

<Bullet list of cross-SI and cross-service risks identified during phase
definition. Include any missing-repo warnings.>
```

---

## Phase 4 — Write story files

For each SI in each active IA, create a file at:

```
stories/<ia-slug>/<si-no>-<component>.md
```

Each story file follows this template exactly:

```markdown
---
ia: <ia-filename>
si: <SI No>
epic: <Epic name>
phase: <Phase number assigned in Phase 2>
service: <Component / repo name>
size_ia: <T-shirt size from the IA table>
size_estimate: <Your independent T-shirt estimate — XS / S / M / L / XL>
---

# <Headline requirement from IA>

## Service

**<Component name>** — <one-line description of this service's role,
from its CLAUDE.md>

## Phase

Phase <N> — <phase label>

## Description

<Change description from the IA, written as an engineering-facing statement
of what must be delivered. Include scope boundaries and dependencies on other
SIs.>

## Dependencies

- Blocked by: <SI Nos that must be complete first, or "none">
- Blocks: <SI Nos that depend on this, or "none">

## Implementation hint

> ⚠️ This implementation hint is generated by Claude Code from the service's
> CLAUDE.md. It is a starting point, not a specification. Verify against the
> current codebase before starting work.

<Derived from the repo CLAUDE.md: which files, packages, or contracts are
affected; what the change means in practice; any conventions, gotchas, or
known risks. If the repo context is unavailable (missing folder), state
"Implementation hint unavailable — repo not present." instead.>

## T-shirt size

**IA estimate:** <size from IA>
**Story estimate:** <your independent estimate>

<One sentence rationale for your estimate, grounded in the implementation
hint. If the estimates differ, explain why.>
```

Your `size_estimate` must be independent — do not simply copy the IA value.
Base it on the implementation hint and the complexity of the change relative
to the service's conventions.

---

## Phase 5 — T-shirt validation

After all story files are written, produce a file at:

```
tshirt-validation.md
```

```markdown
# T-shirt Validation

| IA | SI | Service | IA Size | Story Estimate | Δ |
|----|----|---------|---------|---------------|---|
| …  | …  | …       | …       | …             | ↑ / = / ↓ |

## Pattern analysis

<Cluster the table by direction and interpret:>
- **All match** — IA and story estimates are aligned.
- **Systematic story up-size** — IA underestimated complexity. Re-baseline
  before locking sprints.
- **Systematic story down-size** — IA was conservative. Capacity may free up.
- **Mixed** — inconsistent sizing methods. Identify and challenge outliers.

## Outliers

<List any individual SI where |Δ| > 1 size band and explain the cause.>
```

---

## Phase 6 — Consolidated dependency graph

Produce a file at:

```
dependency-graph.md
```

The file contains two sections:

### Dependency map (prose)

For each dependency edge across all IAs, write one line:

```
<IA> SI-XX (<service>) → <IA> SI-YY (<service>)
  Reason: <why SI-YY cannot start until SI-XX is done>
```

Group intra-IA edges first, then inter-IA edges. If there are no inter-IA
edges, state that explicitly.

### Mermaid graph

```mermaid
graph LR
  subgraph <IA epic name>
    SI01[SI-01\nService\nS]
    SI02[SI-02\nService\nM]
    SI01 --> SI02
  end
  subgraph <Second IA epic name>
    …
  end
  SI02 --> SI05["SI-05\nOther service\nS"]
```

Rules:
- One node per SI. Node label format: `SI-XX\n<service>\n<IA size>`.
- Subgraph per IA (epic name as subgraph label).
- Edges follow the `Dependent On` column plus any identified inter-IA
  dependencies.
- Do not add edges the IA does not support.

---

## Phase 7 — Delivery plan (Gantt)

Produce a file at:

```
delivery-plan.md
```

### Capacity model

State this before producing any durations:

| T-shirt | Days |
|---------|------|
| XS | 2d |
| S | 4d |
| M | 6d |
| L | 10d |
| XL | 16d |

Use **story estimates** (from Phase 4) as durations, not IA estimates.
Start date: **today** (use the current date from context).
Weekends excluded (Mermaid handles this automatically).

### Gantt rules

- One section per IA epic.
- Within each section, one bar per phase (not per SI). Label each bar with
  the phase label and the count of SIs: `Phase 1 — Foundation (2 SIs)`.
- Duration of a phase bar = the longest story estimate in that phase
  (phases run in parallel within the phase).
- Use `after <prev-id>` references, not absolute dates (except the first bar
  of each IA which anchors to today).
- Inter-IA dependencies must be reflected as `after <blocking-phase-id>`.
- Add a milestone for each IA at its last phase completion.
- Add a final milestone `All IAs complete` after the last milestone.

### Gantt template

```mermaid
gantt
    dateFormat  YYYY-MM-DD
    excludes    weekends

    section <IA 1 Epic>
    <Phase 1 label> (<n> SIs)   :ia1p1, <today>, <Nd>
    <Phase 2 label> (<n> SIs)   :ia1p2, after ia1p1, <Nd>
    …
    <IA 1> complete              :milestone, ia1done, after ia1pN, 0d

    section <IA 2 Epic>
    …
```

### Written summary

After the Mermaid block, write:

```markdown
## Critical path

<Identify the longest path from start to final milestone. Name the SIs on
the critical path and their durations.>

## Key risks

<3–5 bullet points: what could extend the plan, and which phase it would
affect.>
```

---

## Artefact index

At the end of your run, print a summary of every file written:

```
## Artefacts written

features/
  <ia-slug>.md

stories/
  <ia-slug>/
    <si-no>-<component>.md   (× N)

tshirt-validation.md
dependency-graph.md
delivery-plan.md
```

---

## Rules

1. **IA = scope.** It defines what changes, not how long.
2. **CLAUDE.md = feasibility.** It defines what the change means in code.
3. **Phases = dependency graph.** Never impose structure the IA does not support.
4. **Story estimates are independent.** Do not copy IA sizes without reasoning.
5. **Delivery plan uses story estimates.** IA sizes are for validation only.
6. **Missing repos do not block the run.** Flag them, offer to continue, proceed.
7. **No anchor dates in the Gantt except today's start.** Everything else is `after`.
