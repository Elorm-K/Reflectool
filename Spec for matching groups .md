# Reflection-Group Matching Function — Technical Specification (DRAFT)

---

## Section 4 — The Matching Specification

This is the core of the document.

### 4.1 Inputs / data model

The matcher takes three inputs: the **roster** (one record per student), the **slot vocabulary** (the candidate weekly meeting times), and a small **config**.

**Slot vocabulary.** Availability is collected on a timeful-style weekly grid: **30-minute slots running 8:00am to 8:00pm, Monday through Sunday.** That is 24 half-hour slots per day × 7 days \= **168 slots** (`S = 168`), each with a stable integer index `0..167` (e.g. index 0 \= Mon 08:00–08:30, index 24 \= Tue 08:00–08:30, … index 167 \= Sun 19:30–20:00). The student paints the slots they are free on this grid exactly as in timeful; the matcher consumes the resulting length-168 boolean vector. \[ASSUMPTION: the 8am–8pm window and Mon–Sun span are the instructor's defaults; both the window and the grid resolution are config values so a course can narrow them.\]

**Student record.**

- `student_id`: stable unique identifier (string). Used for output *and* for deterministic tie-breaking. Never a name.  
- `availability`: a length-`S` (168) boolean vector (the timeful grid result), `true` \= free in that slot. Stored as an explicit bitset/boolean array so set-intersection is trivial and testable.  
- `gender`: raw value from the survey — an **open set**, e.g. `man`, `woman`, `non_binary`, `self_described`, `prefer_not_to_disclose`, `blank`. The matcher must **not assume a closed set**: new self-described values are handled gracefully (matched as distinct values, never dropped) — see 4.4. \[VERIFY: full real value set.\]  
- `disability`: raw value — e.g. `none`, `adhd`, `mental_health`, `autism`, `multiple`, `undiagnosed`, `prefer_not_to_disclose`, `blank`.

**Config.**

- `target_size = 5`, `min_size = 3`, `max_size = 5` (**hard ceiling — never 6**, even when a dropout is anticipated; the brief's earlier "6 acceptable" allowance is overridden).  
- `grid`: `{ days: Mon..Sun, start: "08:00", end: "20:00", slot_minutes: 30 }` → `S = 168`.  
- `min_overlap`: minimum number of mutually-free slots a group must share to be viable. \[ASSUMPTION: default `min_overlap = 2` — one primary 30-min meeting slot plus one backup — pending team decision; see Open Questions.\]  
- `priority`: an **ordered list** of the matchable factors that defines the objective's tiers (default `["availability", "gender", "disability"]`). The instructor controls this order; reordering it reorders the objective (see 4.2). The first entry is the highest-weight tier.  
- `tiebreak_seed`: fixed integer for reproducible tie-breaks (default `0`).

**Scope note (configurability).** v1 fixes the matchable field set to **availability, gender, and disability**; the instructor controls only their **priority order** via `config.priority`. A natural future extension — raised in review — is letting instructors also choose **which fields are collected** from students and add custom matchable attributes. The objective is built from a small **factor registry** (§4.2) so that adding a factor later is a small, local change rather than a rewrite; but custom-field collection and its UI are explicitly out of scope for v1. The **equity safeguard for women and non-binary students (§4.4) sits above `config.priority`** and is not subject to reordering.

**Concrete example input (6 students, 4 slots).** *Reduced grid for readability — the real grid has 168 slots; here the 4 columns stand for four candidate evening slots.*

{

  "config": {

    "target\_size": 5, "min\_size": 3, "max\_size": 5, "min\_overlap": 2,

    "priority": \["availability", "gender", "disability"\], "tiebreak\_seed": 0

  },

  "slots": \["Mon-18:00", "Tue-18:00", "Wed-18:00", "Thu-18:00"\],

  "students": \[

    { "id": "s01", "availability": \[1,1,0,0\], "gender": "woman",      "disability": "adhd" },

    { "id": "s02", "availability": \[1,1,1,0\], "gender": "woman",      "disability": "mental\_health" },

    { "id": "s03", "availability": \[1,1,0,1\], "gender": "non\_binary", "disability": "none" },

    { "id": "s04", "availability": \[0,0,1,1\], "gender": "man",        "disability": "none" },

    { "id": "s05", "availability": \[0,0,1,1\], "gender": "man",        "disability": "autism" },

    { "id": "s06", "availability": \[0,1,1,1\], "gender": "man",        "disability": "prefer\_not\_to\_disclose" }

  \]

}

### 4.2 Algorithm

**Choice: a weighted constraint-optimization (CSP) formulation.** The matcher declares the grouping rules as **hard constraints** and an instructor-weighted **objective**, and a solver returns an assignment that is **optimal** with respect to that objective — or, when proving optimality is too costly, the **best assignment found within a time budget together with a bounded optimality gap** ("within X% of optimal"). This is preferred over a hand-rolled greedy/heuristic approach because (a) it is guaranteed optimal when a feasible solution exists, (b) the rules live in declarative constraints rather than imperative search-and-repair code that is easy to get subtly wrong, and (c) it re-solves cleanly when a student is added or drops. The exact solver is an **open choice** — see §5 for the selection criteria.

**Objective — instructor-configurable, built from a factor registry.** Hard constraints (size bounds, `min_overlap`, every student placed-or-`unplaced`; see 4.3) are *not* objective terms — they gate which assignments are feasible at all. The objective then maximizes, in the tier order given by `config.priority`, the homogeneity of each named factor:

- The objective is assembled by iterating `config.priority` and, for each factor name, adding its weighted **homogeneity term** at the corresponding tier. Factor definitions live in a small **registry** (`{name, homogeneity_fn}`), so the matchable set is data, not hardcoded control flow — adding a factor later (the future extension in §4.1) is one registry entry, not a rewrite. v1 registers exactly three factors: `availability`, `gender`, `disability`.  
- **`availability` as an objective factor** rewards a group **comfortably clearing `min_overlap`** (a viable meeting slot plus a small buffer), *not* maximizing total shared slots. Schedule beyond the minimum is one weighted factor among the others — it does not dominate identity. (The hard `min_overlap` floor in 4.3 still guarantees every group can meet, regardless of where `availability` sits in `config.priority`.)  
- **Equity safeguard (above priority).** A built-in term — **not** part of `config.priority` and not reorderable by the instructor — preferentially groups **women and non-binary students together and penalizes isolating a lone woman or non-binary member** (see 4.4). It is weighted above every `config.priority` tier so configuration cannot switch it off.

**Lexicographic tiers.** `config.priority` defines a strict tier order (tier 1 \= first listed). A solver realizes this either by **staged optimization** (solve for tier 1, lock its value, then optimize tier 2 within that, etc. — provably correct ordering) or by a **weighted sum with sufficiently separated tier coefficients**; staged optimization is preferred because it cannot silently reorder priorities through mis-chosen weights (§6 tests this). The equity safeguard is a tier above tier 1\.

**Steps.**

1. **Validate & normalize** inputs (4.4 handles messy values). Reject malformed availability vectors; never reject a student for a blank demographic.  
2. **Schedule-decompose** the roster into independent components: students who share no mutually-free slots can never be in the same group, so the roster partitions into schedule-compatible components that are solved independently (and in parallel). This bounds problem size by the largest component rather than the whole roster (see §5 Scale).  
3. **Build the CSP model** for each component: assignment variables (student → group), the hard constraints of 4.3 (including a reified "group shares ≥ `min_overlap` common slots"), the `unplaced` escape (4.4) modeled as a penalized allowed state so an unplaceable student yields a partial solution rather than INFEASIBLE, and the weighted objective above.  
4. **Solve** under a configurable time limit. Pin the solver to a **single worker with a fixed seed** and add **symmetry-breaking constraints** (group labels are arbitrary) so the result is reproducible. Return the optimal assignment, or the best found with its optimality gap.  
5. **Remainder resolution** (4.4) for anyone the solver places on `unplaced`.  
6. **Emit a *proposal*** (4.5) — group id, members, chosen meeting slot(s). No demographics. This is a draft for instructor review, **not** a published result; nothing is shown to students until the instructor approves (see 4.6 Approval & publication workflow).

With a fixed seed, single worker, and symmetry-breaking, `match()` is a **deterministic mapping from input to output** — the same input yields the same grouping, which is what makes the validation test (§6) meaningful.

### 4.3 Constraints & invariants

**Hard constraints (a partition violating any of these is invalid):**

- Every group has `min_size (3) ≤ size ≤ max_size (5)`. **5 is a hard ceiling — never 6 however can be overridden by the instructor given the availability match.**  
- Every group shares at least `min_overlap` mutually-free slots.  
- Every student appears in exactly one group (no one ungrouped) — *or* is explicitly emitted on an `unplaced` list for manual instructor handling, never silently dropped.  
- **Privacy invariant:** no output field carries gender or disability, directly or by implication (see 4.5).


**Soft preferences (optimized, but never override a hard constraint):**

- Maximize the homogeneity of each factor in the order given by `config.priority` (default `availability`, then `gender`, then `disability`). `availability` beyond the `min_overlap` floor is just one weighted factor, not a dominant term.  
- **Equity safeguard (weighted above all `config.priority` tiers, not reorderable):** preferentially group women and non-binary students together; avoid isolating a lone woman or non-binary member.  
- Prefer group sizes at `target_size = 5`.

### 4.4 Edge cases & open-decision resolutions

**Tie-breaking when groupings score equally.** Resolve deterministically: prefer the partition whose sorted multiset of group sizes is closest to all-5s; if still tied, prefer the one with greater total schedule overlap; if still tied, pick the partition that is lexicographically smallest by its sorted list of (sorted member-id) tuples. `tiebreak_seed` only enters this last step and is fixed by default — so output is reproducible. *No randomness anywhere.*

**Uneven counts / size distribution.** Choose the size mix that (a) uses the *fewest* groups that keeps every group in `[3,5]`, and (b) among those, maximizes the count of 5s, putting any remainder into 4s before any 3s, and never producing a 3 if a valid all-4s-and-5s mix exists. Worked example: 38 students → fewest groups in range is 8 (38 \= 5×6 \+ 4×2 → **six groups of 5, two of 4**). \[ASSUMPTION: this size rule is schedule/identity-blind and only sets the *target* shape; the actual Spring 2025 run produced **9** groups (sizes incl. 4, 3, 5, …) because identity homogeneity forced smaller groups. That is expected and correct: **identity/schedule homogeneity can override the size-distribution preference, producing more, smaller groups than the arithmetic minimum.** The size rule is the lowest-priority soft term, so this is by design, not a bug.\]

**Remainder students (fit no group on schedule, or only via a constraint violation).** A student who cannot join any group meeting `min_overlap` is **never forced** into an infeasible group and **never silently dropped**. Resolution order: (1) try to grow an existing feasible group to absorb them (up to size 5, the hard ceiling) without breaking `min_overlap`; (2) try to form a new feasible group of ≥3 from other remainders; (3) if neither works, place them on an explicit `unplaced` list in the output with the reason, for manual instructor resolution. Visibility over silent failure.

**How much overlap is "enough."** Governed by `min_overlap` (default 2 — primary \+ backup). This is a hard gate: below it, the group cannot reliably meet weekly, so safety never gets a chance to form. \[ASSUMPTION: 2; flagged for the team because it directly trades off against how many identity-homogeneous groups are achievable.\]

**Identity vs. schedule when they conflict.** Strict lexicographic precedence matching the authoritative priority order: **schedule feasibility is a hard gate that is never traded away** for identity. Identity homogeneity is maximized only *within* the set of schedule-feasible groupings. Equity safeguard — women and non-binary students" (groups them together \+ penalizes isolating a lone woman/nb member; a lower-weight disability anti-isolation term retained)

**Lone-minority avoidance.** Active, as a soft penalty (objective term 4). Definition: a group containing exactly one member with a *known* minority value (e.g. one `woman` among `man`s, or one student with a known disability among `none`s) is penalized **when a feasible alternative exists** that either pairs that student with a same-identity peer or places them in a group where they are not the sole differing member. The repair pass (step 6\) attempts such swaps. It will *not* break a hard constraint to do so, and whether it may override the `target_size = 5` preference is an explicit Open Question — the brief's "isolate a lone minority" harm argues yes, but that is a values call for the team. *Note:* a single-gender or single-disability group is **fine** (that's homogeneity, the goal); the penalty targets the *lone differing* member, not homogeneous groups.

**Missing / blank / "prefer not to disclose" demographics.** Treated as a distinct `unknown`/`undisclosed` bucket that is **never penalized**. Concretely, for homogeneity scoring, unknown values are *neutral*: they neither add a same-identity bonus nor incur a mismatch penalty, and they are never counted as a "minority" for the lone-minority penalty (a student who chose not to disclose must not be singled out by the very value they declined to share). Effect: these students are grouped on the remaining factors (schedule and any disclosed values) and drift to wherever they fit without distorting others' homogeneity. \[ASSUMPTION: "neutral, never penalized" is the privacy- and dignity-preserving default; the team should confirm — see Open Questions. This also honors learner agency: opting out of disclosure costs the student nothing.\]

**Homogeneity score (definition used above).** For a group and an attribute, score \= size of the largest single *known* value class divided by the count of members with *known* values (1.0 \= all knowns agree). Unknowns excluded from both numerator and denominator. A group with all-unknown values scores neutral (excluded from the attribute's sum), never penalized.

### 4.5 Outputs

The matcher emits a **proposal** object: group assignments and meeting times **and nothing else**. Each group has an opaque `group_id` (a sequence number, not derived from any demographic), a list of member `student_id`s, and one or more chosen meeting slots drawn from the group's mutual availability. A `status` field marks the lifecycle stage (`proposed`).

**Privacy invariant (restated and enforced):** the output schema has **no** `gender` or `disability` field, and `group_id` is a plain counter so it cannot encode identity. Gender and disability are *never* shown to anyone downstream of the matcher. A test asserts the output schema contains no demographic keys (Section 6). Note the brief's "students see only a group number and a meeting time" is **refined per the instructor's decision**: once a grouping is approved, students *are* shown their own group's members and meeting time slots (4.6) — but never any gender/disability data. The demographic non-disclosure invariant is absolute and unchanged.

**Concrete example proposal (for the 4.1 input):**

{

  "status": "proposed",

  "groups": \[

    { "group\_id": 1, "members": \["s01", "s02", "s03"\], "meeting\_slots": \["Mon-18:00", "Tue-18:00"\] },

    { "group\_id": 2, "members": \["s04", "s05", "s06"\], "meeting\_slots": \["Wed-18:00", "Thu-18:00"\] }

  \],

  "unplaced": \[\]

}

(Here the two women and one non-binary student share Mon/Tue and are grouped together — the equity safeguard keeps women and non-binary students from being isolated; the three men share Wed/Thu. Both satisfy `min_overlap = 2` and `min_size = 3`. `s06`'s undisclosed gender doesn't affect scoring; they are placed by schedule. No demographic data leaves the matcher.)

### 4.6 Approval & publication workflow (human-in-the-loop)

The matcher never publishes directly to students. A proposed grouping must be reviewed and approved by the instructor before anyone is notified. The lifecycle (see the workflow diagram):

1. **Collect.** Students enter availability on the timeful-style grid (30-min slots, 8am–8pm, Mon–Sun) and complete the week-3 demographic survey. CollaborAITE stores these.  
2. **Propose.** The matcher runs and emits the `proposed` grouping (4.5).  
3. **Instructor review.** The instructor sees the full proposal — groups, members, meeting slots, and the `unplaced` list — in a review screen. The instructor *may* see demographic composition here (they collected it and need it to judge group quality); this is the one authorized place identity is visible, and it is never propagated to students. \[ASSUMPTION: the review screen surfaces composition to the instructor to support an informed approval; the team should confirm how much identity detail to show even the instructor — see Open Questions.\]  
4. **Approve or adjust.** The instructor can approve as-is, or manually override (move a student, merge/split, assign an `unplaced` student). Any manual edit is re-validated against the hard constraints (size 3–5, `min_overlap`) before it can be saved, so human edits can't produce an infeasible group silently. On approval the grouping transitions `proposed → approved`.  
5. **Publish & notify.** Only after approval does CollaborAITE alert each student of **their own group: the group number, the other members, and the meeting time slot(s)** — and nothing about gender or disability. Students see only their own group, not the whole class's assignments. Status transitions `approved → published`.

This keeps the algorithm deterministic and testable while putting a human firmly in control of the consequential, sensitive decision — matching the instructor's accountability for the groups.

### 4.7 Workflow

End-to-end flow, from collection through the human-in-the-loop approval to publication:

flowchart LR

    A \[Students paint availability timeful-style grid\] \--\> B\[Week-3 survey gender · disability\]

    B \--\> C\[Instructor config priority · group size · min\_overlap\]

    C \--\> D\[(CollaborAITE store)\]

    D \--\> E\[Schedule-decompose roster into compatible components\]

    E \--\> F\[Weighted CSP solve per component\]

    F \--\> G\[Apply hard floor min\_overlap · size 3–5\]

    G \--\> H\[Apply weighted objective config.priority \+ women/nb equity safeguard\]

    H \--\> I\[Proposed grouping groups · slots · no demographics\]

    I \--\> J\[Unplaced list \+ reason\]

    J \--\> K\[Instructor review members · slots · unplaced\]

    K \--\> L{Approve or adjust?}

    L \--\>|adjust| M\[Manual edit move · merge · assign\]

    M \--\> N{Re-validate vs hard constraints}

    N \--\>|invalid| M

    N \--\>|valid| L

    L \--\>|approve| O\[Status: proposed → approved → published\]

    O \--\> P\[Notify each student group number · members · slots never gender/disability\]

---

## Section 5 — Technical Architecture

**Stack.** Python 3.11+, standard library only for the core. Rationale: the data volumes are tiny (tens of students), the logic is set operations and sorting, and a dependency-free core is the easiest to test, audit, and drop into CollaborAITE. `dataclasses` for the records; `pytest` for tests (dev-only). **No solver, no ML, no agent framework** — there is no need and adding one would be the exact "complexity without a justifying need" the brief warns against.

**Modules.**

- `io_parse.py` — input parsing & validation. Loads roster/slots/config, normalizes messy demographic strings into the known buckets \+ `unknown`, validates availability vector lengths (= 168), and rejects structurally malformed input with clear errors. Never rejects on blank demographics.  
- `matcher.py` — the matcher core. Pure function `match(roster, slots, config) -> Result`. Implements 4.2 steps 2–7. No I/O, no global state, fully deterministic.  
- `objective.py` — the scoring functions (schedule quality, homogeneity, lone-minority penalty, size preference) used by `matcher.py`, isolated so each term is independently unit-testable.  
- `output_format.py` — builds the privacy-safe proposal structure and asserts the no-demographics invariant before emitting.  
- `review_workflow.py` — the human-in-the-loop gate (4.6). Manages the `proposed → approved → published` state machine; exposes the proposal (and, for the instructor only, optional composition detail) to the review screen; re-validates any manual instructor override against the hard constraints before accepting it; and gates publication so nothing reaches students pre-approval. The matcher core has no knowledge of this module — review/approval is layered on top of the pure function, not baked into it.  
- `notify.py` — on `published`, builds each student's personal view (their group number, their group members, their meeting slot(s); never demographics) and hands it to CollaborAITE's notification channel. A student only ever receives their own group's data.

**Complexity.** Greedy seed-and-grow is roughly `O(N² · S)` for availability intersections (N students, S \= 168 slots) plus a bounded repair pass that tries a constant-capped number of swaps. For N≈40, S=168 this is well under a millisecond — trivially acceptable. There is no scaling concern at the expected scale; if N ever reached the thousands the same algorithm still runs comfortably.

**Determinism guarantees.** (1) All sorts are stable and keyed ultimately on `student_id`. (2) No `random`/`Date`/wall-clock calls anywhere in the core. (3) Tie-breaks are resolved by the fixed rule in 4.4 with a default-`0` seed. (4) `match()` is a pure function: same input → byte-identical output. This is what makes the validation test (Section 6\) meaningful.

**Fit into CollaborAITE.** Ships as a normal importable module/service: the platform collects availability via the timeful-style grid and demographics via the week-3 survey, hands the matcher a roster, stores the returned proposal, routes it through the instructor review/approval gate (`review_workflow.py`), and only on approval notifies students (`notify.py`). The matcher is a plain, synchronous code module behind a function call — *not* an agent, not a background reasoning loop. **Library note:** only `pytest` (dev). If a graph abstraction ever clarifies the seed-and-grow step, `networkx` is the candidate — but the stdlib version should be written and shown to be insufficient first.

---

## Section 6 — Evaluation & Testing

### Success criteria (tied to the problem's root causes)

1. **Every group can actually meet.** *Root cause: no shared time → no weekly meeting → no safety.* **Indicator:** 100% of emitted groups have `|⋂ availability| ≥ min_overlap`; the `unplaced` list, if non-empty, is surfaced to the instructor rather than hidden.  
2. **Identity-sharing is maximized subject to schedule.** *Root cause: mismatched identity → no vulnerability → no growth.* **Indicator:** mean gender homogeneity, then disability homogeneity, across groups meets or beats a random-assignment baseline on the same input by a defined margin, and equals the manual Spring 2025 result on reconstructed inputs within tolerance (see validation test).  
3. **No avoidable lone-minority isolation.** *Root cause: stranding the one woman / one disabled student suppresses disclosure exactly where the group matters most.* **Indicator:** zero groups contain a lone known-minority member *for whom a feasible non-isolating alternative existed*; any unavoidable case is reported, not silent.

### Test plan

**Unit tests — constraints & invariants.**

- Every group within size `[3,5]`; assert no group ever has size 6 (hard ceiling).  
- Every group meets `min_overlap`.  
- Every student placed exactly once across `groups ∪ unplaced`.  
- Output schema contains **no** `gender`/`disability` keys, and `group_id`s are a plain sequence (privacy invariant).  
- Determinism: running `match()` twice on identical input yields identical output; permuting input student order yields the same partition (modulo group\_id labels).  
- Each `objective.py` term tested on hand-computed small inputs.

**Edge-case tests.**

- Uneven counts: 38 → expected size shape (six 5s \+ two 4s baseline; confirm identity pressure can yield more, smaller groups).  
- Missing/blank/"prefer not to disclose": such students placed on schedule, never penalized, never counted as lone minority.  
- No-overlap student: lands on `unplaced` with a reason, not force-fit.  
- All-same-demographic class: homogeneity trivially satisfied; schedule still gates.  
- Lone-minority repair: a constructed case where a single swap removes isolation is taken; a case where no feasible swap exists is reported, not forced.

**Approval-workflow tests (4.6).**

- A freshly matched grouping has `status = "proposed"`; no notification is emitted in this state.  
- Students are notified **only** after `status = "published"`; assert nothing is sent in `proposed`/`approved` states (no pre-approval leak).  
- The state machine only allows `proposed → approved → published`; illegal transitions rejected.  
- A manual instructor override that breaks a hard constraint (size \<3 or \>5, or below `min_overlap`) is rejected at save time, not silently accepted.  
- The published per-student view contains the student's group number, their group members, and meeting slots — and **no** gender/disability field, and **no** other group's data.

**Validation test — reproduce a Spring-2025-comparable run.** Reconstruct the Spring 2025 inputs (38 students with their availability \+ demographic values) from the paper/run data and assert the matcher produces a **comparable** partition. \[VERIFY: obtain the actual per-student Spring 2025 availability \+ demographic data — it is *not* in this repo and must be sourced from the study; the published paper gives only aggregate compositions.\] "Comparable" is defined as: (a) group count within ±1 of 9; (b) the *characteristic groups* the manual method found are reproduced when feasible — specifically an all-disabled-women group, an all-men all-disabled group, and at least one evening MS-cohort group; (c) per-group gender and disability homogeneity ≥ the manual run's, group-for-group, allowing label permutation. Measure by matching emitted groups to the documented ones via best-overlap assignment and comparing size \+ homogeneity profiles. This test is meaningful *only because* the matcher is deterministic.

---

## OPEN QUESTIONS FOR YOUR TEAM

1. **Minimum schedule overlap (`min_overlap`).** Is one shared 30-min slot enough, or do we require a primary \+ backup (default proposed: 2)? This directly bounds how many identity-homogeneous groups are even achievable — a higher bar means more mixed groups.  
2. **Does lone-minority avoidance override the target group size of 5?** The "don't strand the one woman / one disabled student" harm is real; honoring it may force a group of 3 or 4 (never 6 — that ceiling is fixed). Which wins when they conflict — and is there a hard rule ("never a lone minority, ever") or only a strong preference?  
3. **How is "prefer not to disclose" / blank treated in matching?** We propose *neutral, never penalized, never counted as a minority*. Confirm — or decide whether undisclosed students should instead be actively grouped together, which would have its own implications.  
4. **Exact precedence when gender and disability homogeneity trade off against schedule *quality* (not feasibility).** Feasibility is a hard gate, but when two feasible partitions differ — one with more shared time, one with better identity match — how much extra schedule robustness justifies a worse identity match (or vice versa)?  
5. **Sourcing the Spring 2025 per-student data for the validation test.** The repo has only aggregate compositions; the validation test needs the actual inputs. Who can provide the de-identified availability \+ demographic records, and under what IRB/consent terms?  
6. **How much identity does the instructor review screen (4.6) show?** Approval is human-in-the- loop, so the instructor needs *enough* to judge group quality — but should they see per-student gender/disability, only aggregate per-group composition (e.g. "homogeneous on gender"), or only the homogeneity scores? More visibility aids judgment; less protects students even from the instructor. Also: can the instructor's manual overrides be re-run through the matcher, or are they final once saved?  
7. **Grid window.** We default the availability grid to 8am–8pm, Mon–Sun (168 slots). Confirm that window — should weekends be included, and is 8am–8pm the right daily span for this cohort?

