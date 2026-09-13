# SIH26122 — Intelligent Data Capture & Schedule-Linking Layer for Infrastructure Project Management

## Problem Statement

**SIH26122** asks for a software solution that bridges the gap between **planned project schedules** and **actual field execution** for large infrastructure projects.

Infrastructure schedules are structured from high-level milestones down to executable **L5/L6 activities**, often covering civil, piping, static/rotating equipment, electrical, instrumentation, HSE, QA/QC, and contractor work in parallel. The baseline plan may be maintained in **Primavera or MS Project**, while actual execution is reported through daily progress reports, site diaries, discipline-wise spreadsheets, and verbal supervisor updates.

The central problem is that these two worlds are not reliably connected.

```text
PLANNED WORLD                         FIELD WORLD

Primavera / MS Project                DPRs
L1 → L2 → L3 → L4 → L5/L6            Site diaries
Activity IDs                          Discipline spreadsheets
Planned dates                         Supervisor updates
Dependencies                          Free text
WBS hierarchy                         Photos / other evidence
        │                                      │
        └──────────────────┬───────────────────┘
                           ▼
                MISSING RECONCILIATION LAYER
```

## What the PS identifies as the problem

The problem statement highlights several related failures:

1. **Fragmented actual-progress data**
   - Execution data arrives through multiple formats and reporting cadences.
   - Different disciplines and contractors describe the same physical work differently.

2. **Weak linkage to L5/L6 activities**
   - Field descriptions are often not expressed using the exact schedule activity ID or wording.
   - Field execution may be more granular than the WBS/schedule.

3. **Manual reconciliation**
   - Planners have to interpret reports, find the corresponding schedule activity, and update the schedule.
   - This can be slow and error-prone.

4. **Late schedule actualization**
   - The problem statement states that schedule reconciliation can lag the schedule update cycle by days or weeks.
   - This means downstream analytics may operate on stale or incomplete actuals.

5. **Weak downstream analytics**
   - Delay/risk analysis and forecasting depend on the quality and timeliness of actual execution data.

6. **Loss of institutional knowledge**
   - Historical execution information is often retained as reports rather than as a structured, queryable dataset of actual durations, bottlenecks, and delay causes.

## What the expected solution asks for

The PS calls for a system that can:

- ingest heterogeneous inputs such as free-text DPRs, spreadsheets, scanned diaries, and schedule exports;
- extract activity-level actual start/end events;
- offer a low-friction conversational or voice interface for supervisors;
- fuzzy-match field descriptions to the correct L5/L6 schedule node;
- handle terminology and granularity mismatches;
- flag unmatched/new activities instead of silently dropping them;
- update actual start/end information in near real time;
- attach confidence scores and audit trails;
- create a structured, discipline-tagged actual-progress dataset;
- support both live performance analytics and long-term institutional memory.

The problem statement specifically says that a working prototype demonstrating **2–3 varied input formats** is preferable and that production-grade OCR/ASR is not required.

## Important interpretation

The PS should **not** be interpreted as:

> "Build a replacement for DPRs."

A DPR can contain many things that are outside schedule actualization, such as manpower, weather, materials, equipment, safety observations, issues, photos, and next-day planning.

The more precise interpretation is:

> **Build a planning-to-execution bridge that converts fragmented field evidence into structured, schedule-linked actual events.**

The DPR is therefore an **input/evidence source**, not necessarily the thing being replaced.

## Core terminology

### WBS — Work Breakdown Structure

A hierarchical decomposition of project work. Conceptually:

```text
L1  Project
 └─ L2  Major package / plant
     └─ L3  Area / system
         └─ L4  Work package / discipline
             └─ L5  Activity
                 └─ L6  Executable activity / task
```

The exact semantics of each level depend on the project's schedule structure.

### L5/L6 activity

The fine-grained schedule node intended to represent executable work. These are the activities we ultimately want to connect field evidence to.

### Actual

Observed or accepted execution state, such as:

- actual start;
- actual finish;
- in progress;
- partial completion;
- blocked/on hold;
- other explicitly supported execution events.

## Why the problem is non-trivial

A field statement such as:

> "XX-107 spool erected"

may have to be interpreted against schedule entries such as:

```text
PIP-1842  Erect Line 24"-XX-107
PIP-1843  Weld Line 24"-XX-107
PIP-1844  Hydrotest Line 24"-XX-107
```

The system must distinguish **what was physically reported**, **which schedule activity it corresponds to**, and **whether that statement actually proves start, progress, or completion**.

This is the central reconciliation challenge.

## Scope boundary

The PS does **not** require:

- universal project-wide computer vision;
- production-grade OCR/ASR;
- replacing Primavera/MS Project;
- replacing supervisors;
- replacing the full DPR process;
- fully autonomous schedule control.

Those are possible extensions, but they should not be treated as the core problem.

## Proposed direction for the updated solution

The most defensible interpretation is a **field execution reconciliation layer** sitting upstream of OIL's existing project-management and analytics stack:

```text
Field evidence
(DPR / Excel / diary / voice / optional CV)
                │
                ▼
        Event extraction
                │
                ▼
        L5/L6 reconciliation
                │
                ▼
        Evidence validation
                │
                ▼
        Verified actuals
                │
                ▼
     Existing PMIS / Primavera
                │
                ▼
       CPM / project analytics
```

This keeps the scope aligned with the PS while avoiding the weak claim that the solution is simply a "better DPR app" or a universal camera-based monitoring platform.

## Data availability

The PS states that anonymized/sample DPR formats, sample L5/L6 schedule extracts, and illustrative discipline-specific site-diary/spreadsheet templates can be shared under NDA, while teams should work with synthetic/sample data rather than live project data.
