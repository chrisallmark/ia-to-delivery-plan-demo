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

You are executing an eight-phase method that auto-discovers inputs and produces
six categories of artefact. Work through every phase in order. At each phase,
state what you are doing and flag any gaps before continuing.

---

## Phase −1 — Configure

Before doing any file discovery, resolve the delivery configuration. Check
whether the user has already provided values for each parameter (in the
invocation args or earlier in the conversation). For any parameter that is
missing or ambiguous, use `AskUserQuestion` to prompt the user — group all
missing questions into a **single** `AskUserQuestion` call (max 4 questions
per call).

### Parameters to resolve

| Parameter | Default | How to detect it was provided |
|-----------|---------|-------------------------------|
| **Start date** | Today's date (from context) | User stated a specific date |
| **Team composition** | 1 full-stack developer | User described team size or split |
| **IA delivery order** | Auto (dependency → risk) | User specified an order (e.g. "Buy → In-Life → Upgrades") |
| **SIT model** | None | User mentioned SIT, testing phases, or integration testing |

### Question designs (use these if prompting)

**Start date** — single select:
- "Today (`<today's date>`)" *(default)*
- "A different date" *(Other — user types the date)*

**Team composition** — single select:
- "1 full-stack developer (solo)" *(default)*
- "1 FE + 1 BE (2 developers, parallel tracks)"
- "2 FE + 2 BE (4 developers, parallel tracks)"
- "Custom" *(Other — user describes the team)*

**IA delivery order** — single select:
- "Auto — order by dependency relationships then descending risk" *(default)*
- "Sequential — user specifies order" *(Other — user types the order)*

**SIT model** — single select:
- "None — no SIT phases in the plan" *(default)*
- "Standard — SIT starts at 80% build complete, duration = 25% of build time"
- "Custom" *(Other — user describes their SIT model)*

### Resolving team composition

Once the team is known, derive the **capacity model** to use in Phases 2 and 7:

| Team | Tracks | Speedup factor | Phase wall-clock |
|------|--------|---------------|-----------------|
| 1 full-stack | 1 | 1.0× | Sum of all SIs |
| 1 FE + 1 BE | 2 (FE / BE) | 2^0.7 = 1.625× per track | max(FE track, BE track) |
| 2 FE + 2 BE | 2 (FE / BE) | 2^0.7 = 1.625× per track | max(FE track, BE track) |
| Custom | Derive from description | N^0.7 per track | max across tracks |

If the team includes mobile developers (iOS / Android), note that they run
in parallel with each other during mobile-specific phases.

### Output of Phase −1

Record the resolved configuration as a named block at the top of your response:

```
## Resolved configuration

- Start date: <date>
- Team: <description>
- Capacity model: <derived from team>
- IA delivery order: <auto | specified order>
- SIT model: <none | standard 80%/25% | custom description>
```

All subsequent phases must use this configuration. Do not revert to the
skill's defaults once configuration is resolved.

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
>
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
- Assign SIs to tracks (BE / FE / mobile) based on the `Component` column and
  the resolved team composition from Phase −1. If the team is solo, all SIs
  are on a single track.
- A phase contains SIs that share the same set of predecessors. Within a
  track, SIs are worked sequentially — order them by dependency first, then
  ascending size (smallest first to ship value early).
- Phase N+1 starts when all Phase N work across all tracks is merged and
  deployed.
- Foundational changes (shared interfaces, data models, auth) go in Phase 1.

State your reasoning for each phase boundary and the intra-phase SI order.

---

## Phase 3 — Write feature files

For each active IA, create a file at:

```
plan/features/<ia-slug>.md
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

| SI    | Headline | Service | Track | T-shirt (IA) | Depends On |
| ----- | -------- | ------- | ----- | ------------ | ---------- |
| SI-XX | …        | …       | BE/FE | …            | …          |

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
plan/stories/<ia-slug>/<si-no>-<component>.md
```

Each story file follows this template exactly:

```markdown
---
ia: <ia-filename>
si: <SI No>
epic: <Epic name>
phase: <Phase number assigned in Phase 2>
track: <BE | FE | iOS | Android | full-stack>
service: <Component / repo name>
size_ia: <T-shirt size from the IA table>
size_estimate: <Your independent T-shirt estimate — XXS / XS / S / M / L / XL / XXL>
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
> CLAUDE.md and source files. It is a starting point, not a specification.
> Verify against the current codebase before starting work.

<Derived from the repo CLAUDE.md: which files, packages, or contracts are
affected; what the change means in practice; any conventions, gotchas, or
known risks. If the repo context is unavailable (missing folder), state
"Implementation hint unavailable — repo not present." instead.>

## Pseudo code

<See generation rules below — present the pseudo code block here, or the
skip notice if size_estimate < L.>

## T-shirt size

**IA estimate:** <size from IA>
**Story estimate:** <your independent estimate>

<One sentence rationale for your estimate, grounded in the implementation
hint. If the estimates differ, explain why.>
```

Your `size_estimate` must be independent — do not simply copy the IA value.
Base it on the implementation hint and the complexity of the change relative
to the service's conventions.

**Developer scaling.** All story estimates represent the work content for one
developer. When computing phase durations in Phase 7, the capacity model from
Phase −1 determines how much of that content runs in parallel. Do not pre-scale
story estimates here — scaling is applied in Phase 7 only.

If the IA's `Req Devs` column is greater than 1 and the resolved team is solo,
apply a non-linear adjustment: duration increases by a factor of approximately
N^0.7 (not N). State the adjustment in your rationale when `Req Devs > 1` and
the team is solo.

**Pseudo code generation rules.**

Apply these rules after writing the implementation hint for each story:

- **Skip (size_estimate < L):** Write the following notice in the Pseudo code
  section and move on — do not read source files:
  > _Pseudo code omitted — story is {size_estimate}. Source-level detail is
  > generated for L, XL, and XXL stories only._

- **Generate (size_estimate is L, XL, or XXL):**
  1. From the CLAUDE.md, identify the most relevant source files for this SI
     (controllers, services, helpers, models — whichever are in scope).
  2. Read those files from the repo. If a file does not exist or the repo is
     unavailable, note it and work from CLAUDE.md alone.
  3. Produce pseudo code that sketches the implementation at method level:
     - Show method signatures using the real class and method names found in
       the source.
     - Use natural-language steps inside method bodies rather than compilable
       code — the goal is to communicate intent, not to be a working diff.
     - Highlight any new parameters, fields, or contracts introduced by this SI.
     - Keep it to the minimum needed to convey the shape of the change; omit
       boilerplate that does not change.
  4. Wrap the output in a fenced block labelled with the primary language:
     ````
     ```java
     // pseudo code
     …
     ```
     ````
  5. Follow the block with one sentence noting which source files were read.

---

## Phase 5 — T-shirt validation

After all story files are written, produce a file at:

```
plan/tshirt-validation.md
```

```markdown
# T-shirt Validation

| IA  | SI  | Service | IA Size | Story Estimate | Δ         |
| --- | --- | ------- | ------- | -------------- | --------- |
| …   | …   | …       | …       | …              | ↑ / = / ↓ |

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
plan/dependency-graph.md
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
plan/delivery-plan.md
```

### Capacity model

State the resolved configuration before producing any durations:

```
Start date:   <from Phase −1>
Team:         <from Phase −1>
Tracks:       <derived — e.g. BE / FE / iOS / Android>
Speedup:      <N^0.7 per track where N = developers on that track>
Phase timing: <max(track durations) | sum (solo)>
SIT model:    <from Phase −1>
```

T-shirt to days mapping:

| T-shirt | Days |
| ------- | ---- |
| XXS     | 1d   |
| XS      | 2d   |
| S       | 3d   |
| M       | 5d   |
| L       | 8d   |
| XL      | 13d  |
| XXL     | 21d  |

**Computing phase durations:**

For each phase, group story estimates by track. Within each track, sum the
estimates and apply the speedup factor for that track
(days ÷ N^0.7, where N = developers on the track). The phase wall-clock
duration is the maximum across all tracks. Round to the nearest whole day.

For solo teams, phase duration = sum of all story estimates (no parallelism).

**IA ordering:**

Use the order resolved in Phase −1. If auto, order IAs by their dependency
relationships first, then by descending risk (most-blocked IA first) to
front-load the critical path.

**SIT phases (if SIT model ≠ none):**

- Determine the build phase at which 80% of total build duration is crossed.
  SIT starts at the end of that phase.
- SIT duration = 25% of total build duration for that IA, rounded to nearest
  whole day.
- SIT runs in parallel with any remaining build phases. It does not extend
  the critical path unless the build finishes before SIT ends.
- Mark SIT bars as `crit` in the Gantt.

### Gantt rules

- One section per IA epic.
- Within each section, one bar per phase (not per SI). Label each bar with
  the phase label and the count of SIs: `Phase 1 — Foundation (2 SIs)`.
- IAs are delivered in the resolved order: each IA section starts only
  `after` the last **build** phase of the previous IA (SIT may overlap).
- Use `after <prev-id>` references, not absolute dates (except the very first
  bar which anchors to the resolved start date).
- Inter-IA dependencies must be reflected as `after <blocking-phase-id>`.
- Add a milestone for each IA at its last phase completion (build or SIT,
  whichever is later).
- Add a final milestone `All IAs complete` after the last milestone.

### Gantt template

```mermaid
gantt
    dateFormat  YYYY-MM-DD
    excludes    weekends

    section <IA 1 Epic>
    <Phase 1 label> (<n> SIs)   :ia1p1, <start date>, <wall-clock days>d
    <Phase 2 label> (<n> SIs)   :ia1p2, after ia1p1, <wall-clock days>d
    …
    SIT (<n>d)                   :crit, ia1_sit, after <80%-phase-id>, <sit days>d
    <IA 1> complete              :milestone, ia1done, after <last-bar-id>, 0d

    section <IA 2 Epic>
    <Phase 1 label> (<n> SIs)   :ia2p1, after ia1done, <wall-clock days>d
    …
    <IA 2> complete              :milestone, ia2done, after <last-bar-id>, 0d

    section All features
    All IAs complete             :milestone, alldone, after ia2done, 0d
```

Omit SIT bars entirely if SIT model = none.

### Written summary

After the Mermaid block, write:

```markdown
## Critical path

<Identify the longest path from start to final milestone. Name the phases and
SIs on the critical path and their durations.>

## Key risks

<3–5 bullet points: what could extend the plan, and which phase it would
affect.>
```

---

## Artefact index

At the end of your run, print a summary of every file written:

```
## Artefacts written

plan/
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
7. **No anchor dates in the Gantt except the start date.** Everything else is `after`.
8. **Configuration from Phase −1 overrides all skill defaults.** If the user
   provided a team, start date, IA order, or SIT model, use it — do not revert.
9. **Story estimates are raw work content.** Apply team scaling in Phase 7 only.
10. **Non-linear dev scaling.** N developers on a track gives N^0.7 speedup.
    State the factor used in the capacity model block.
11. **SIT is optional.** Only add SIT phases if the resolved SIT model ≠ none.
