# FieldLens — Updated Solution Proposal for SIH26122

## 1. Executive Summary

**FieldLens is a planning-to-execution reconciliation layer, not a replacement for DPRs, supervisors, Primavera, or OIL's existing project analytics.**

Its job is to answer one operational question reliably:

> **What execution event actually happened, which L5/L6 schedule activity does it belong to, and how trustworthy is that conclusion?**

Field evidence can come from:

- daily progress reports;
- discipline spreadsheets;
- site diaries;
- supervisor text/voice updates;
- selected photographs/video or targeted camera feeds where visual evidence has sufficient value.

The system extracts schedule-relevant execution events, finds candidate L5/L6 activities, scores the match using multiple signals, checks evidence consistency, and either produces a high-confidence actual or routes the case to a planner for review.

After acceptance, the actualized schedule can feed OIL's existing project-control and analytics stack. CPM is used **after** reliable actuals exist to determine whether a deviation has project-level consequence.

---

# 2. Why Would OIL Need This?

The case is **not** simply "real-time information is useful."

OIL already has people, DPR processes, Primavera/project-management tooling, and downstream project analytics. The value proposition must therefore be narrower.

## The proposed business problem

There can be a gap between:

```text
FIELD REALITY
     ↓
reported / observed execution
     ↓
manual interpretation
     ↓
L5/L6 schedule update
     ↓
project analytics
```

The PS states that this reconciliation can be fragmented and delayed by days or weeks.

FieldLens aims to reduce that **actualization latency** and reduce the manual effort required to turn heterogeneous field evidence into trusted schedule actuals.

## What it does not claim

FieldLens does **not** claim that:

- supervisors are unnecessary;
- DPRs should disappear;
- every construction activity can be detected by computer vision;
- every event should be pushed into the schedule instantly;
- CPM itself is novel;
- OIL needs another project-management dashboard.

Instead:

> **FieldLens improves the quality, structure, traceability, and timeliness of schedule-relevant field actuals.**

---

# 3. Product Positioning

## The product

> **FieldLens: Field Execution Evidence Reconciliation for Project Controls**

### Core promise

```text
MESSY FIELD EVIDENCE
        ↓
WHAT HAPPENED?
        ↓
WHICH L5/L6 ACTIVITY?
        ↓
IS THE EVIDENCE TRUSTWORTHY?
        ↓
VERIFIED ACTUAL
        ↓
WHAT DOES IT DO TO THE PROJECT?
```

The final question is where schedule logic/CPM enters.

---

# 4. Architecture

```text
                         ┌────────────────────────────┐
                         │     EXISTING OIL STACK     │
                         │                            │
                         │ Primavera / PMIS          │
                         │ Project analytics         │
                         │ AI performance monitoring │
                         └─────────────▲──────────────┘
                                       │
                                       │ verified actuals
                                       │
                         ┌─────────────┴──────────────┐
                         │      FIELDLENS CORE        │
                         │                            │
                         │ Event extraction           │
                         │ L5/L6 candidate retrieval  │
                         │ Multi-signal matching      │
                         │ Evidence fusion            │
                         │ Confidence / validation    │
                         │ Human-review workflow      │
                         │ Audit trail                │
                         └─────────────▲──────────────┘
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 │                     │                     │
                 ▼                     ▼                     ▼
          ┌─────────────┐       ┌─────────────┐      ┌─────────────┐
          │     DPR     │       │ Supervisor  │      │ Optional CV │
          │ PDF / Excel │       │ Voice/Text  │      │ Photo/Video │
          │ Site diary  │       │             │      │ Targeted    │
          └─────────────┘       └─────────────┘      └─────────────┘

                         ┌────────────────────────────┐
                         │    SCHEDULE KNOWLEDGE      │
                         │                            │
                         │ L1-L6 hierarchy            │
                         │ Activity IDs               │
                         │ Descriptions               │
                         │ Discipline                 │
                         │ Location                   │
                         │ Planned dates              │
                         │ Dependencies               │
                         └────────────────────────────┘
```

---

# 5. End-to-End Pipeline

## Stage 0 — Import the baseline schedule

Import a schedule extract containing, where available:

- Activity ID;
- activity description;
- WBS path;
- discipline;
- location;
- planned start/finish;
- duration;
- predecessor/successor relationships;
- other useful tags/attributes.

Create a searchable **schedule knowledge store**.

The schedule is treated as the source of planned structure. We do not ask an LLM to invent the schedule.

---

## Stage 1 — Ingest field evidence

Accept several evidence types:

```text
DPR PDFs
Discipline spreadsheets
Site diaries
Supervisor text
Supervisor voice
Photos / video / targeted CV
```

The prototype only needs 2–3 representative input formats, consistent with the PS.

---

## Stage 2 — Normalize each input

Examples:

```text
PDF      → text/tables
Excel    → structured rows
Voice    → transcript
Image    → visual observations
```

All sources are converted into a common internal evidence representation.

---

## Stage 3 — Extract execution events

The system converts raw evidence into structured events.

Example input:

> "XX-107 spool erection started at 10 AM and completed by 5 PM."

Example normalized event:

```json
{
  "entity": "XX-107",
  "activity_description": "spool erection",
  "event_start": "2026-09-13T10:00",
  "event_end": "2026-09-13T17:00",
  "status": "COMPLETED",
  "discipline": "PIPING",
  "source": "DPR"
}
```

Important rule:

> **Unknown information stays unknown.**

"Work ongoing" must not become a fabricated start time or finish time.

---

# 6. L5/L6 Reconciliation

This is the heart of the system.

## Candidate retrieval

Do not compare a field observation against every activity in the project.

First narrow candidates using:

- project/package;
- discipline;
- location;
- WBS context;
- exact tags/entities;
- temporal plausibility;
- known activity attributes.

Example:

```text
20,000 schedule activities
        ↓
contextual retrieval
        ↓
30 plausible candidates
```

## Multi-signal matching

Candidate activities are scored using multiple signals rather than an LLM-only guess:

```text
semantic similarity
entity/tag match
WBS consistency
discipline match
location match
temporal consistency
dependency/context consistency
```

Conceptually:

\[
Score = \sum_i w_i S_i
\]

where each \(S_i\) represents one evidence signal.

The actual weights should be learned/tuned against ground-truth pilot data rather than invented arbitrarily.

---

# 7. State and Event Reasoning

A matched activity is not enough. The system must interpret what happened.

Supported states can include:

```text
NOT_STARTED
STARTED
IN_PROGRESS
PARTIALLY_COMPLETE
COMPLETE
ON_HOLD
BLOCKED
UNKNOWN
```

Supported events can include:

```text
START
PROGRESS
FINISH
PAUSE
RESUME
BLOCK
```

A statement such as:

> "XX-107 erection ongoing"

is not equivalent to:

> "XX-107 erection completed."

Events are stored historically rather than repeatedly overwriting the evidence trail.

---

# 8. Evidence Fusion

Different sources provide different kinds of evidence.

Example:

```text
DPR:        completed
Supervisor: completed
CV:         visual evidence consistent with installation

              ↓
         CONSISTENT EVIDENCE
              ↓
         HIGH CONFIDENCE
```

Conflict example:

```text
DPR:        completed
Supervisor: completed
CV:         visual evidence inconsistent
QA record:  pending

              ↓
         CONFLICT DETECTED
              ↓
         PLANNER REVIEW
```

The system should **not automatically assume that CV, the DPR, or the supervisor always wins**. Evidence precedence must be defined with the project owner.

---

# 9. Confidence and Human Review

The system should optimize for **high precision in automatic updates**, not for forcing every record through automation.

A conceptual policy:

```text
HIGH confidence
+ no material conflicts
+ clear event
        ↓
AUTO-ACCEPT

MEDIUM confidence
or evidence conflict
        ↓
PLANNER REVIEW

LOW confidence
or unmatched activity
        ↓
UNMATCHED / NEW ACTIVITY QUEUE
```

Thresholds must be calibrated from pilot data.

### Audit trail

Every accepted or rejected event should retain:

```text
source document / media
original evidence text
extracted event
candidate activities
match score / evidence signals
reviewer decision
final schedule action
timestamp
```

This is essential because an industrial schedule should not become an opaque AI-generated artifact.

---

# 10. Selective Computer Vision

CV remains part of the solution, but **not as the universal observation mechanism**.

## CV's job

> **Provide independent visual evidence for activities that are sufficiently observable and economically important to monitor visually.**

Example:

```text
Photo/video
   ↓
CV detection / tracking
   ↓
physical-state evidence
   ↓
activity context
   ↓
reconciliation
```

## What CV should not claim

It should not claim to infer every construction activity from cameras.

Many activities are not reliably visual:

```text
inspection
approval
QA clearance
material approval
instrument calibration
documentation
administrative dependencies
```

Even visually observable work can be affected by occlusion, lighting, camera position, weather, dust, and changing site geometry.

Therefore CV is a **supporting evidence channel**.

---

# 11. Camera Deployment Strategy

Avoid the "camera everywhere" model.

Use a prioritization layer:

```text
                    ALL ACTIVITIES
                          │
             ┌────────────┴────────────┐
             │                         │
       High-value / critical       Routine
             │                         │
             ▼                         ▼
        targeted CV               DPR / human
```

A work front can justify visual monitoring when it has high:

- schedule criticality;
- delay impact;
- value of independent verification;
- observability;
- expected ROI.

This means camera coverage becomes a **resource-allocation problem**, not a requirement for universal surveillance.

---

# 12. Visual Quality Gate

The CV channel must be able to say:

> **"I do not have reliable visual evidence."**

Conceptually:

```text
Image / frame
      ↓
quality assessment
      ↓
usable?
 ┌────┴────┐
 YES       NO
  │         │
  ▼         ▼
 CV      CV evidence unavailable
 inference       │
  │              │
  └──────┬───────┘
         ▼
   Evidence fusion
```

This prevents poor lighting or occlusion from becoming a false completion event.

---

# 13. Verified Actuals

Once an event is accepted:

```text
Activity: PIP-1842

Baseline
Start: 10-Sep
Finish: 12-Sep

Actual
Start: 10-Sep
Finish: 13-Sep

Variance
Finish: +1 day
```

Baseline and actuals should be preserved separately.

Do not overwrite history.

---

# 14. CPM / Schedule Impact

CPM comes **after** reconciliation.

Its role is not:

> "detect what happened."

Its role is:

> **"determine how much the deviation matters to the project."**

Example:

```text
Activity delay      +3 days
Available float      1 day
                         ↓
                float consumed
                         ↓
           possible project impact
```

The system can surface:

- criticality;
- float consumption;
- affected successors;
- schedule slippage;
- project completion implications.

CPM itself is not our novelty. Better field actuals are what make downstream schedule intelligence more useful.

---

# 15. Exception-First Operation

Do not generate a notification for every tiny change.

The system should distinguish between:

```text
ROUTINE EVENT
    ↓
record / synchronize

MATERIAL DEVIATION
    ↓
CPM impact assessment
    ↓
exception / alert if action is justified
```

This is why the system can provide near-real-time capability without making "real time" the entire business case.

The real value is **decision-relevant freshness**.

---

# 16. Execution Memory

After accepted actuals accumulate, store structured history:

```text
Activity
Planned duration
Actual duration
Variance
Discipline
Location
Contractor
Delay cause
Dependencies
Evidence
```

This can eventually support future questions such as:

> How long do comparable activities actually take?

> Which activities repeatedly exceed their baseline?

> Which delay causes recur within a discipline?

This is a long-term benefit, not the primary reason for the first deployment.

---

# 17. Why Not Just Use DPRs?

A DPR can remain the authoritative human reporting mechanism.

The problem is the work required after it exists:

```text
DPR
 ↓
find relevant statement
 ↓
identify schedule activity
 ↓
interpret event / status
 ↓
resolve ambiguity
 ↓
update actuals
 ↓
assess impact
```

FieldLens automates and structures the middle of that chain.

It is therefore more accurate to call it a **reconciliation layer** than a reporting replacement.

---

# 18. Why Not Just Hire a Software Vendor?

OIL can and should use enterprise software vendors for infrastructure, storage, integration, and system hosting.

That is not our differentiation.

The differentiated component is the **execution-evidence intelligence** that turns heterogeneous site evidence into schedule-linked actuals.

The product should integrate with, not replace, enterprise systems such as Primavera/PMIS.

---

# 19. Total Problem / Bottleneck Inventory

The following are the major technical, operational, economic, and adoption bottlenecks identified during solution design.

## A. Problem-definition bottlenecks

### 1. Unproven pain magnitude
We do not have public OIL data for planner hours/week spent on reconciliation.

**Required validation:** baseline person-hours per project/week.

### 2. Unproven error rate
We do not have OIL-specific public data showing how often field progress is linked to the wrong L5/L6 activity.

**Required validation:** historical/sample linkage accuracy.

### 3. Unproven value of "near real time"
Hours of latency are not automatically valuable. Value depends on whether earlier knowledge changes decisions.

**Required validation:** decision windows and avoided-delay impact.

### 4. Existing systems may already solve part of the problem
OIL already has project-management tooling and an AI project-performance initiative.

**Implication:** FieldLens must sit upstream and avoid duplicating downstream functionality.

---

## B. Data and semantic bottlenecks

### 5. Activity IDs may already exist
If field records contain exact IDs/tags, AI reconciliation adds little value.

**Required validation:** fraction of records that are not deterministically linkable.

### 6. Field terminology mismatch
"Spool erected" may refer to a schedule activity with very different wording.

### 7. Granularity mismatch
One L5/L6 activity may correspond to many physical sub-events, and one field statement may describe multiple schedule activities.

### 8. Non-unique entities
Several visually/semantically similar lines, spools, supports, or equipment items may exist.

### 9. Ambiguous completion semantics
Physical installation, QA completion, testing, and schedule completion are not necessarily identical.

### 10. Missing start times
"Ongoing" does not prove when an activity started.

### 11. Missing finish times
Physical visibility does not necessarily prove schedule completion.

### 12. Progress-definition ambiguity
"80% complete" may mean quantity, physical progress, activity progress, or another project-control definition.

### 13. Incomplete or inconsistent evidence
OCR/ASR/reporting errors, missing documents, duplicate reports, and contradictory statements can degrade reconciliation.

---

## C. Computer-vision bottlenecks

### 14. Limited visual coverage
Many project activities are not reliably observable from imagery.

### 15. Exact asset identity
Detecting a pipe is easier than proving it is **XX-107**.

### 16. Occlusion
Scaffolding, equipment, workers, temporary structures, and site geometry can hide work.

### 17. Lighting
Day/night differences, shadows, glare, artificial lighting, dust and weather can reduce visual reliability.

### 18. Camera placement drift
Construction changes the scene; a camera's useful field of view can change over time.

### 19. Camera scalability
Project-wide fixed camera deployment creates hardware, networking, compute, maintenance, and relocation burdens.

### 20. Video/network cost
Continuous high-resolution streaming can be expensive and difficult to operate on industrial sites.

### 21. Model generalization
A model trained on one project/site/camera may not generalize to another.

### 22. CV event limitations
A frame can show a physical state without proving the exact start/finish event.

### 23. Visual-only false positives
A physically installed component may still be awaiting QA, inspection, testing, or another schedule condition.

---

## D. Evidence-fusion bottlenecks

### 24. Source disagreement
DPR, supervisor, CV, QA and schedule evidence may conflict.

### 25. Evidence precedence
There is no purely technical answer for which source should win every conflict.

### 26. Confidence calibration
A numeric match score is not automatically a statistically calibrated probability.

### 27. False positives are costly
An incorrect automatic schedule update can propagate errors into downstream analytics.

### 28. Human-review burden
If too many events require manual review, automation loses its economic advantage.

### 29. Auditability
Every accepted schedule update needs traceable evidence and reasoning.

---

## E. Schedule / CPM bottlenecks

### 30. Integration complexity
Primavera/PMIS integration involves permissions, update semantics, calendars, baselines, status dates, and workflow controls.

### 31. Schedule ownership
FieldLens should not become an uncontrolled authority that mutates the official schedule without governance.

### 32. CPM redundancy
Primavera already provides schedule logic; our CPM layer must add useful interpretation rather than simply reproduce existing functionality.

### 33. Dependency semantics
An activity delay may not equal project delay because float and successor relationships matter.

### 34. Remaining-duration uncertainty
Actual start/finish events alone may not fully represent current remaining work.

---

## F. Business and deployment bottlenecks

### 35. ROI uncertainty
Hardware and operational costs may exceed measurable savings if the use case is poorly selected.

### 36. Contractor adoption
Contractors may resist systems perceived as surveillance or contractual verification mechanisms.

### 37. Reporting burden
If FieldLens requires extra photos, voice logs, confirmations, and corrections, it may increase rather than reduce reporting effort.

### 38. Workflow adoption
Planners may reject a new dashboard if it does not integrate into their existing workflow.

### 39. Cybersecurity
Industrial/project environments require strong access control, network segmentation, audit, and security practices.

### 40. Data governance
Ownership and retention of photos, video, derived observations, corrections, and audit records must be defined.

### 41. Historical-data cold start
Institutional-memory benefits require accumulated structured history and therefore compound over time rather than appearing immediately.

---

## G. AI / research bottlenecks

### 42. LLM hallucination
The language model must not invent dates, status, quantities, or activity IDs.

### 43. Baseline comparison
A serious evaluation needs deterministic/rule-based and simpler semantic baselines.

### 44. Synthetic-data bias
Clean synthetic examples may overestimate real-world performance.

### 45. Cross-discipline generalization
Piping, civil, electrical, instrumentation, equipment, QA and HSE use different terminology and activity semantics.

### 46. Delay-cause inference
A detected delay does not automatically reveal its cause.

### 47. Quantity/progress estimation
Partial progress and quantities are harder than binary event detection.

### 48. Temporal synchronization
Event time, recording time, upload time, and report time can differ.

---

# 20. MVP Scope

To keep the SIH prototype credible, build a narrow vertical slice rather than attempting every capability.

## Recommended MVP

### Discipline
**Piping**

Reason: the PS itself gives piping terminology/identifier examples, making it a natural demonstration domain.

### Schedule

100-ish synthetic L5/L6 activities containing:

- IDs;
- descriptions;
- WBS paths;
- discipline;
- tags;
- locations;
- planned dates;
- predecessor/successor context.

### Evidence

A deliberately messy set such as:

- DPR text/PDF;
- discipline spreadsheet;
- supervisor message/voice transcript;
- a small number of visual examples.

### Core output

For every extracted event:

```text
Field statement
      ↓
Extracted event
      ↓
Top candidate L5/L6 activities
      ↓
Match explanation
      ↓
Confidence / conflict status
      ↓
Auto-accept or planner review
      ↓
Actual schedule state
      ↓
CPM impact
```

---

# 21. Evaluation Plan

The prototype should not only demonstrate a UI. It should establish whether the reconciliation method is useful.

Compare at least:

```text
Baseline A — exact/rule-based matching
Baseline B — semantic retrieval
Baseline C — direct LLM classification
FieldLens — retrieval + multi-signal scoring + evidence validation
```

Measure:

### Linking precision

How often an automatically linked event maps to the correct activity.

### Linking recall

How often the correct activity is recovered.

### Auto-resolution rate

Percentage of events that can be processed without human review.

### False-update rate

Percentage of automatic schedule updates that are incorrect.

### Review reduction

Planner effort compared with a manual baseline.

### Latency

Time from evidence arrival to verified actual.

### Schedule-impact detection

Whether the system correctly distinguishes material schedule deviations from routine progress.

---

# 22. Success Criteria

The project should not claim success merely because an LLM produces plausible text.

A credible prototype should demonstrate:

1. **Correct extraction** of supported event types.
2. **High-precision L5/L6 linkage** on the tested scope.
3. **Conservative handling of uncertainty**.
4. **Traceable evidence/audit trails**.
5. **Useful planner-review workflow**.
6. **Meaningful reduction in manual reconciliation effort**.
7. **Correct propagation of accepted actuals into schedule-impact analysis**.

The exact numerical targets should be established from the pilot dataset rather than invented beforehand.

---

# 23. One-Line Architecture

```text
Heterogeneous field evidence
        ↓
Execution-event extraction
        ↓
L5/L6 schedule reconciliation
        ↓
Multi-source evidence validation
        ↓
Confidence + human review
        ↓
Verified actuals
        ↓
Existing PMIS / Primavera
        ↓
CPM / project impact
        ↓
Execution history
```

# 24. Final Product Thesis

> **FieldLens does not replace the DPR, the supervisor, or Primavera. It replaces the manual reconciliation gap between field evidence and schedule actuals, while using selective computer vision as independent evidence and CPM as the downstream mechanism for understanding project impact.**
