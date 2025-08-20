# Priorigraph – Product & System Design v1

> A productivity and time‑management app built on hierarchical, cross‑linked priority trees that encode a person’s values → initiatives → goals → tasks, and turns them into an adaptive schedule.

---

## 1) Vision & Principles

* **North Star:** Help people spend time in alignment with their core values, not just urgent demands.
* **Core Idea:** Users model their life as multiple **priority trees** whose roots are **Anchor** values (e.g., Family, Health). Below each anchor, users add **Initiatives** (open‑ended themes), **Goals** (clear end states), and **Tasks** (atomic, boolean‑done work). Nodes can cross‑link across trees, forming a web/graph.
* **Operating Principles**

  1. **Value alignment first:** Every scheduling nudge is traceable up to one or more Anchors.
  2. **Clarity of outcome:** Goals always have defined end states; tasks are binary done/not‑done.
  3. **Adaptive planning:** Plans reflow as deadlines, energy, and availability change.
  4. **Low‑friction capture:** Quick add for tasks and goals, later classify/attach to the graph.
  5. **APIs & openness:** A first‑class API for integrations (calendar, notes, trackers, teams).

---

## 2) Core Concepts & Taxonomy

### Node Ranks (highest → lowest)

* **Anchor** (root‑only): Core values; cannot have parent nodes; can have many children.
* **Initiative:** Open‑ended theme without a defined end state (e.g., “Express myself creatively”).
* **Goal:** Has a **defined end state**; may have **deadline** and success criteria.
* **Task:** Atomic, binary completion; may have deadline/estimate; can be scheduled.

### Parent/Child Rules

* **Anchor** → may parent **Initiatives, Goals, Tasks**.
* **Initiative** → may parent **Initiatives, Goals, Tasks** of **same or lower** rank.
* **Goal** → may parent **Goals, Tasks** of **same or lower** rank.
* **Task** → **no children**.

### Cross‑links

* Any non‑Anchor node can have **cross‑links** to nodes in other trees (e.g., a creative initiative under both *Self‑Improvement* and *Mental Health*). Cross‑links affect priority scoring by aggregating value weights.

---

## 3) Data Model (Graph‑first)

### Entities

* **User**
* **Node** (polymorphic by rank)
* **Edge** (parent→child; rank‑validated)
* **CrossLink** (undirected or directed tags between nodes in different trees)
* **ScheduleBlock** (time allocations on calendar)
* **Context** (energy, location, device, tags)
* **Integration** (connected services)

### Node Schema (draft JSON)

```json
{
  "id": "node_123",
  "rank": "anchor|initiative|goal|task",
  "title": "string",
  "description": "string",
  "anchorWeights": {"Health": 0.6, "Self-Improvement": 0.4},
  "status": "active|paused|archived|done",
  "successCriteria": ["Goal-only: measurable outcomes"],
  "deadline": "2025-10-01T17:00:00Z",
  "estimateMinutes": 90,
  "effort": 3,        
  "energy": "low|medium|high",
  "urgency": 0.0,     
  "importance": 0.0,  
  "parentIds": ["node_parent"],
  "crossLinkIds": ["node_x"],
  "tags": ["writing", "deep-work"],
  "createdAt": "...",
  "updatedAt": "..."
}
```

### Edge Rules (validation)

* **No cycles** in parent→child edges.
* **Rank guardrails:** parent.rank ≥ child.rank (Anchor > Initiative > Goal > Task).
* **Anchor cannot have a parent.**
* **Task cannot have children.**

### Derived Fields

* **Path to root:** list of anchors affecting a node.
* **Aggregate Anchor Weight (AAW):** sum of anchorWeights via parents and cross‑links (normalized).
* **Readiness:** boolean based on blockers, context, and required time window.

---

## 4) Priority Scoring & Scheduling

### Priority Score (PS)

A weighted blend that balances values, urgency, and practicality:

```
PS = wV*ValueAlign + wU*Urgency + wD*DeadlineRisk + wI*Importance - wE*EffortFriction + wR*RecencyBoost
```

* **ValueAlign:** Derived from AAW (0–1), reflecting how strongly the item ties to anchors.
* **Urgency:** Function of time‑to‑deadline and dependency pressure.
* **DeadlineRisk:** Probability of missing deadline given remaining effort and calendar availability.
* **Importance:** User‑set or learned (pairwise choices, time spent, completions).
* **EffortFriction:** Function of estimateMinutes, required energy, and context mismatch.
* **RecencyBoost:** Small boost for newly added or recently resumed items.

Weights are user‑tunable; defaults provided with presets (e.g., *Balanced*, *Deadline‑driven*, *Value‑centric*).

### Auto‑Scheduling Engine

* **Inputs:** Priority‑ranked queue, user calendar (busy/free), preferred work hours, focus windows, energy profile, constraints (no meetings before 9am, gym Tue/Thu 6pm, etc.).
* **Algorithm (greedy + backtracking):**

  1. Build candidate set of *ready* tasks/goals (no blockers, context‑compatible).
  2. Sort by **PS** then **deadline proximity**.
  3. Place into free blocks respecting min block size, splitting long tasks when allowed.
  4. Backtrack/reflow when conflicts or new events appear.
* **Output:** A draft schedule for the next N days, annotated with anchor badges.

### Timeboxing & Focus Mode

* One‑click **“Make time”** creates schedule blocks.
* **Focus Session** shows the task, goal lineage, success criteria, and anchor badges; tracks time; offers *pause*, *defer*, *complete*.

---

## 5) Graph Editing & Visualization

* **Views:**

  * **Tree View per Anchor** (collapsible levels; badges: rank, deadline, PS).
  * **Web Graph View** (force‑directed layout for cross‑links; hover reveals lineage and AAW).
  * **Roadmap View** (timeline/Gantt of Goals with milestones).
  * **Today/This Week** (calendar + prioritized list; drag‑to‑timebox).
* **Editing:** inline add, drag to reparent (with rank validation), convert Initiative↔Goal, split/merge tasks.
* **Context Filters:** energy, location, tags, duration, anchors.

---

## 6) Example

* **Anchors:** *Family*, *Health*, *Self‑Improvement*.
* **Initiative:** *Express myself creatively* (cross‑linked under *Self‑Improvement* and *Mental Health*).
* **Goals under Initiative:**

  * *Publish 3 blog posts by Nov 30* (deadline).
  * *Take an illustration course* (end state: finish syllabus).
* **Tasks under Goal:** *Outline post #1 (45m)*, *Draft intro (25m)*, *Enroll in course (10m)* …
* **Outcome:** Scheduler proposes 45m deep‑work block Tue 9:00 for *Outline post #1*, shows badges for **Self‑Improvement (0.6)** and **Mental Health (0.4)** value alignment.

---

## 7) API (Developer‑friendly)

### Auth

* OAuth2 (Authorization Code w/ PKCE) for apps; PAT for CLI.

### Resources

* `POST /nodes` (create), `GET /nodes/:id`, `PATCH /nodes/:id`, `DELETE /nodes/:id`
* `POST /edges` (create parent→child), `DELETE /edges/:id`
* `POST /crosslinks`, `DELETE /crosslinks/:id`
* `GET /graph?rank=&anchorId=&q=` (search/filter)
* `POST /schedule/plan` (returns draft time blocks for a window)
* `POST /schedule/commit` (writes ScheduleBlocks)
* `GET /analytics/value-time` (time spent per anchor over range)
* `POST /import` (from CSV/JSON/Todoist/Notion), `POST /export`
* **Webhooks:** `task.completed`, `goal.status.changed`, `schedule.updated`

### Example Payloads

```json
POST /schedule/plan {
  "windowStart": "2025-08-21T08:00:00-06:00",
  "windowEnd": "2025-08-27T18:00:00-06:00",
  "constraints": {"workHours": ["09:00-17:00"], "minBlockMin": 25},
  "filters": {"anchors": ["Health", "Self-Improvement"], "energy": ["medium","high"]}
}
```

---

## 8) Integration Surface

* **Calendars:** Google, iCloud, Outlook (read busy/free; write blocks; RSVP influence).
* **Notes/Docs:** Notion, Obsidian, Evernote; attach note URLs to nodes.
* **Task Sources:** Todoist, GitHub issues, Jira; map into *Task* nodes with source metadata.
* **Health/Energy:** Apple Health, Oura, Fitbit (optional) to modulate **EffortFriction**.
* **Communication:** Slack/Email quick‑add via slash command or forwarding address.

---

## 9) Learning & Personalization

* **Pairwise priority calibrator:** Occasionally ask, “Which would you do first?” to learn **Importance** and weights.
* **Completion feedback:** Use on‑time vs late completion to update **DeadlineRisk** model.
* **Contextual success:** Recommend times of day for certain anchors based on past focus quality.

---

## 10) UX Flows

* **Onboarding:** choose starter anchors; import tasks; quick add 3 goals; connect calendar; preview schedule.
* **Daily Flow:** review *Today*; accept/adjust blocks; run focus; quick capture; end‑of‑day value‑time report.
* **Weekly Review:** surface stalled goals; rebalance value distribution; propose next week plan.

---

## 11) Privacy & Safety

* Local‑first option for graph data; end‑to‑end encryption of notes; fine‑grained scopes for integrations.
* Private anchors/goals by default; sharing is explicit and scoped (view/comment only, or delegate tasks).

---

## 12) Architecture (suggested)

* **Frontend:** React/Next.js; Graph rendering with Canvas/WebGL (e.g., Cytoscape.js); offline cache (IndexedDB).
* **Backend:** TypeScript (NestJS) or Go; Postgres + pgvector for search; Redis for queues; temporal.io for scheduling workflows.
* **Graph:** Store parent edges in Postgres (adjacency + path); validate ranks with triggers; optional Neo4j for analytics.
* **Sync:** CalDAV/Graph APIs; webhooks for deltas; conflict resolver for schedule blocks.

---

## 13) Validation & Invariants (pseudo‑code)

```python
# enforce parent-child rank rules and acyclicity
RANK = {"anchor": 3, "initiative": 2, "goal": 1, "task": 0}

def can_parent(parent, child):
    if child.rank == "anchor":
        return False
    if parent.rank == "task":
        return False
    if child.rank == "initiative" and parent.rank == "goal":
        return False  # goal cannot parent initiative (higher rank)
    if RANK[parent.rank] < RANK[child.rank]:
        return False
    return True

# detect cycle using DFS before committing edge
```

---

## 14) Roadmap (proposed)

**MVP (8–12 weeks)**

* Graph CRUD with rank rules; calendar read; timeboxing; Today/This Week views; priority score v0; API v1.

**Beta**

* Auto‑scheduling with backtracking; cross‑link weights; integrations (Google Calendar, Notion, Todoist); focus mode; weekly review.

**V1.0**

* Webhooks; analytics; mobile apps; import/export; pairwise learning; privacy controls; templates.

---

## 15) Success Metrics

* Weekly Active Users; % of scheduled time completed; On‑time rate for goals; Value‑time balance (variance across anchors); Net Schedule Adjustment (avg manual changes).

---

## 16) Nice‑to‑Have Ideas

* **“Value Debt”**: show anchors under‑invested this month; propose corrective blocks.
* **Snooze with intent**: defer plus reason; model learns.
* **Social accountability**: opt‑in buddy for a goal; progress snapshots.

---

## 17) Open Questions (to refine next)

* How to visualize cross‑link strength elegantly in small screens?
* What’s the best default weighting for ValueAlign vs DeadlineRisk?
* Should initiatives optionally have “north star metrics” (qualitative checkpoints)?
