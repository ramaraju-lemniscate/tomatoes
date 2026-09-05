# SIH26027 — TEAM RESEARCH BRIEFS (FINAL, v3)
### Everything you need, even if you know nothing about this project yet

**Problem:** SIH26027 · Ministry of Railways · Software · Idea deadline 20 September 2026

---

# PART 0 — READ THIS FIRST (everyone, 10 minutes)

## What we are building, in plain English

Indian Railways must close sections of track to do maintenance. A planned closure is called a **block**.

Three separate departments each need blocks:

| Department | What they maintain |
|---|---|
| **Engineering (P.Way)** | The track — rails, sleepers, ballast |
| **Signal & Telecom (S&T)** | Signals, points, interlocking |
| **Traction Distribution** | Overhead electric wires (OHE) |

Right now **each department requests its closure separately and by hand.** Nobody coordinates them. So they clash, or three separate closures happen where one shared closure would have done all three jobs. Track sits unavailable longer than needed, trains get delayed, and maintenance still falls behind.

**We are building a system that combines all three departments' requests into optimised shared closure windows that respect the train timetable.**

## Concepts you need before reading your package

**Optimisation** — finding the best answer out of millions of possibilities, without checking them all. Not machine learning. Maths that guarantees quality.

**Constraint programming** — a way of describing a problem as *rules that must hold* ("these two jobs can't be on the same track at once") and letting a solver find an answer that satisfies all of them. The tool we use is **Google OR-Tools CP-SAT**, which is free and world-class.

**Optimality gap** — when the solver finishes it gives two numbers: the best schedule it found, and a mathematical proof that nothing can be better than some bound. The difference is the gap. *"Within 2% of proven optimal"* means our answer is at most 2% worse than the theoretical best. **This is our single strongest selling point.**

**Baseline** — the thing we compare against to prove we improved something. We have two: a simple greedy rule (what competing teams will build), and a simulation of Indian Railways' current messy process.

## The one thing everyone must understand

**This is NOT a machine learning project.**

```
data → [ML: estimate how long a job takes and how urgent it is]
     → [OPTIMISER: decide the actual schedule]
     → plan
```

**ML estimates. The optimiser decides.**

If we train a neural network to output schedules, it will produce plans that break hard safety rules — two closures on one track, a block sitting on a live train path. **A railway cannot run a schedule that is only "probably" safe.**

## The government's own evidence (memorise these)

India's official auditor, the CAG, has already documented this problem.

**CAG Report No. 45 of 2017:**
- Engineering asked for **58,342 block hours**, got **29,411** — a **49.6% shortfall**
- **33%** of granted block time went unused
- **46%** of that lost time was machines just travelling to site

**CAG Report No. 22 of 2022 (Derailments)** — why track machines sat idle:
- **32%** blocks not given by Operating
- **30%** blocks not planned by Divisions

**62% of machine idle time comes from block-planning failure alone.**

## One factual correction — get this right

The problem statement says the 3-hour corridor policy is from **2018**. It is not.

- Announced by the Railway Board Chairman in **2020**
- Implemented in the All-India Time Table from **1 October 2022** (PIB release PRID 1863836)
- The separate, older ≥4-hour-or-two-2½-hour block norm comes from **IRTMM Chapter 5.2 / IRPWM Para 226**

A railway judge will notice if we get this wrong.

## Our goal

**Finish something that works and can be proven good.** We are not inventing a new algorithm. We use proven tools, model the problem correctly, prove our answer is near-optimal, and show improvement against the CAG's audited baseline. **That wins without novelty.**

---

# PART 1 — WORK ALLOCATION

## Judgment markers

| Mark | Meaning | How to use AI |
|---|---|---|
| 🔴 **HIGH JUDGMENT** | Your decisions shape the whole project. A wrong call costs weeks. | AI explains and suggests. **You decide.** Never accept an AI answer without sanity-checking it against railway reality. |
| 🟡 **MEDIUM** | AI can do most of it. You verify every fact against the real source. | Ask freely, but **open the actual PDF and confirm every number**. |
| 🟢 **MECHANICAL** | Mostly finding and downloading. Low risk. | Use AI heavily. Just don't let it invent facts. |

## Assignment table — fill this in

| # | Package | Mark | Days | Owner | Done? |
|---|---|---|---|---|---|
| **WP1** | Railway domain grounding | 🟡 | 3 | ________ | ☐ |
| **WP2** | Tooling — can a library do this? | 🟡 | 2 | ________ | ☐ |
| **WP3** | Model design | 🔴 | 2 | ________ | ☐ |
| **WP4** | Model verification | 🟡 | 1 | ________ | ☐ |
| **WP5** | Disruption & rescheduling | 🔴 | 2 | ________ | ☐ |
| **WP6** | Cost in rupees | 🟢 | 1 | ________ | ☐ |
| **WP7a** | Downloads | 🟢 | 1 | ________ | ☐ |
| **WP7b** | Synthetic data generator | 🔴 | 2 | ________ | ☐ |
| **WP8** | Visualisation & operator workflow | 🟡 | 1.5 | ________ | ☐ |
| **WP9** | Borrowed principles, rejections, past SIH | 🟢 | 1.5 | ________ | ☐ |
| **WP10** | Baselines — what we compare against | 🟡 | 1.5 | ________ | ☐ |

## What depends on what — READ BEFORE STARTING

```
DAY 1 ──┬── WP7a Downloads ────────────► finishes day 1
        ├── WP1 Domain grounding ──────► finishes day 3
        ├── WP6 Costs ─────────────────► finishes day 2
        └── WP9 Borrowed/rejections ───► finishes day 2

DAY 2 ──┬── WP2 Tooling ───────────────► finishes day 3  [needs WP7a done]
        └── WP8 Visualisation ─────────► finishes day 3

DAY 4 ──┬── WP3 Model design ──────────► finishes day 5  [needs WP1 + WP2 + WP6]
        └── WP7b Synthetic data ───────► finishes day 5  [needs WP1's numbers]

DAY 6 ──┬── WP4 Verification ──────────► finishes day 6  [needs WP3's spec]
        ├── WP5 Disruption ────────────► finishes day 7  [needs WP3's spec]
        └── WP10 Baselines ────────────► finishes day 7  [needs WP7b's data]

DAY 8 ──► EVERYTHING STOPS. Build starts.
```

**Hard blockers:**
- **WP7b cannot start until WP1 delivers `parameters.md`.** Without real periodicity numbers the synthetic data is worthless.
- **WP3 cannot start until WP2 says which tool we're using.**
- **WP4, WP5 and WP10 cannot start until WP3 delivers `model_spec.md`.**

**If a blocker is late:** don't wait idle. Help the blocking package finish.

## Reporting rules

- **One page maximum per package.** Not five. If you can't say it in one page, you haven't finished thinking.
- **Every file goes in `notes/`** with the exact filename given in your package.
- **Every number gets a source.** Format: `value | where it came from | page or paragraph`.
- **"Could not verify" is a valid and respected answer.** Writing it is better than inventing something.
- **Daily 10-minute check-in.** Each person says: what I found, what's blocking me. Nothing longer.
- **Praveen reviews every 🔴 package before it's accepted.** Those three decisions shape everything.

## ⚠️ HARD STOP: research ends day 8

Regardless of completeness. An unfinished package with an honest *"we didn't get to this"* beats perfect research and no working prototype.

---

# PART 2 — THE UNIVERSAL AI CONTEXT BLOCK

**Paste this into any AI before asking anything about this project.** Without it you get generic, useless answers — or worse, an AI that tells you to train a neural network.

```
CONTEXT FOR MY QUESTION:

I am a B.Tech student on a team building a project for Smart India Hackathon 2026,
problem statement SIH26027, sponsored by India's Ministry of Railways.

THE PROBLEM:
Indian Railways must close sections of track to do maintenance. A planned closure
is called a "block". Three separate departments each need blocks:
  - Engineering (P.Way) — maintains the track itself
  - Signal & Telecommunications (S&T) — maintains signals and points
  - Traction Distribution — maintains overhead electric wires (OHE)

Right now each department requests its block INDEPENDENTLY and MANUALLY through a
system called BDMS. Nobody coordinates between them. So blocks clash, or three
separate closures happen where one shared closure would have done. Track sits
unavailable longer than necessary and trains get delayed.

Their maintenance data lives in three separate systems: TMS (track), SMMS
(signalling), TDMS (traction). The train timetable and corridor availability
live in a fourth system, COA.

WHAT WE ARE BUILDING:
A system that takes all three departments' pending maintenance jobs, plus the
train timetable, and produces an optimised schedule of shared maintenance blocks
that maximises how much time the track is available for trains.

CRITICAL ARCHITECTURE POINT:
This is NOT a machine learning project. The core is CONSTRAINED OPTIMISATION,
using Google OR-Tools CP-SAT. Machine learning is only a small supporting layer
that estimates two things: how long a maintenance job will take, and how urgent
it is. Those estimates feed into the optimiser. The optimiser makes the actual
scheduling decision.

We do this because a neural network that outputs schedules would produce plans
that break hard safety rules — like two closures on the same track at once, or
a block sitting on top of a scheduled train. A railway cannot run a schedule
that is only "probably" safe.

KEY EVIDENCE (from India's official auditor, the CAG):
- CAG Report 45 of 2017: Engineering demanded 58,342 block hours, got 29,411.
  A 49.6% shortfall. 33% of granted block time went unused. 46% of that lost
  time was machines travelling to site.
- CAG Report 22 of 2022: of track machine idle time, 32% was "blocks not given
  by Operating" and 30% was "blocks not planned by Divisions".

OUR GOAL: finish something that works and can be PROVEN good. We are not trying
to invent a new algorithm. We use proven tools, model the problem correctly,
and prove our answer is near-optimal using the solver's optimality gap.

NO DATASET IS PROVIDED. Public data covers the network, timetable, and all the
maintenance standards. Actual maintenance job lists and defect records are not
public anywhere in the world, so we generate realistic synthetic data anchored
to published Indian Railways norms.

MY SPECIFIC QUESTION:
[write your question here]
```

## ⚠️ THE UNIVERSAL WARNING

**AI will confidently invent Indian Railways figures, manual paragraph numbers, rupee amounts, and citations that do not exist.**

**Rule: every number you report must come from a document you personally opened.** If AI gives you a figure, find it in the real PDF, or write "could not verify."

This is the single biggest risk in this whole research phase, because a fabricated number that reaches the judges destroys our credibility on everything else.

---

# WP1 — RAILWAY DOMAIN GROUNDING
### 🟡 MEDIUM · 3 days · Start day 1 · **Blocks WP7b and WP3**

## What this is about

The judges will be actual Indian Railways engineers and operations officers. They use specific vocabulary daily. If we use the wrong word or get a maintenance interval wrong, they stop believing anything else we say.

Your job: make our team able to talk like railway people, and extract the real numbers that make our synthetic data believable.

## Part 1 — Vocabulary

A **block** is a planned period when track is closed for maintenance.

- **Traffic block** — no trains on that section. Approved by the DRM (Divisional Railway Manager).
- **Power block** — overhead electric wires switched off for OHE work. The TPC (Traction Power Controller) handles the shutdown.
- **Power-and-traffic block** — everything stops.
- **Disconnection** — signalling equipment taken out of service for S&T work.
- **Corridor block** — a pre-planned daily window built into the timetable.
- **Mega block / jumbo block** — long consolidated closures, mostly Mumbai suburban, on Sundays.

Also learn: **possession, path, section, caution order, TSR** (temporary speed restriction), **GMT** (gross million tonnes — how much traffic a section carries), **DRM, Section Controller, SSE/P.Way, JE/Signal, TPC**.

## Part 2 — The four systems

- **TMS** — Track Management System. Track defects, inspections, ultrasonic test results, rail fractures.
- **SMMS** — Signalling Maintenance & Management System. S&T assets and schedules.
- **TDMS** — Traction Distribution Management System. Overhead wire data.
- **COA** — Control Office Application. Timetable and live train running.
- **BDMS** — Block & Disconnection Management System. Where block requests are raised. It digitised the paperwork but **cross-department planning is still manual** — that gap is our problem statement.

## Part 3 — Extract the numbers ⚠ THE VALUABLE PART

**WP7b cannot start until you deliver this.**

| Document | Extract | Link |
|---|---|---|
| **IRTMM Chapter 2** ⚠ most important | Machine output rates — metres per hour | https://indianrailways.gov.in/railwayboard/uploads/codesmanual/IRTMM-RDSO/TMM_Ch/ |
| **IRTMM Chapter 5.2** | Minimum block duration norms | same |
| **IRPWM Para 226** | Block duration requirements for machines | https://iricen.gov.in/iricen/Track_Manuals/IRPWM.pdf |
| **USFD Manual 2012** | Rail testing frequency table (varies by GMT) | https://iricen.gov.in/iricen/Track_Manuals/USFD/usfd_new.pdf |
| **SEM** | Signalling maintenance schedules | https://indianrailways.gov.in/railwayboard/view_section.jsp?lang=0&id=0,1,304,366,544,664 |
| **ACTM Vol II** | OHE maintenance periodicities | Search RDSO site for "AC Traction Manual" |

**Figures already found — verify each in the actual PDF:**
- Duomatic 08-32 tamping machine ≈ **800 metres per effective hour**
- Turnout tamper ≈ **1 turnout per hour**
- Deep screening a PSC turnout ≈ **4.0–5.5 hours**
- Duomatic travels **60 km/h self-propelled, 40 km/h in train formation**
- Block norm: **one block of ≥4 hours, or two of 2½ hours**

**If you cannot find a figure, write "not found" and move on.** Don't guess and don't let AI guess.

## 🤖 Ask AI

> "I am researching Indian Railways maintenance terminology. Explain in simple terms what these mean and how they differ: traffic block, power block, corridor block, disconnection, possession, mega block. Also explain who requests each and who approves it. I will verify everything against the IRPWM and IRTMM manuals."

> "I am reading the Indian Railways Track Machine Manual. Explain what 'effective hour output' means for a tamping machine, and how I would use it to calculate how long a maintenance task takes on a 2 km section."

## ⚠️ Traps

- **AI will invent paragraph numbers and periodicities.** Report nothing you haven't seen in the actual PDF.
- **Don't confuse the corridor block policy with the machine block norm.** Different documents, different things. The 3-hour corridor is timetable policy; the ≥4-hour norm is an engineering standard for machines.
- **Get the policy date right** — 2020 announcement, October 2022 implementation. Not 2018.

## Deliverables
1. `notes/glossary.md` — one page, every term, plain English
2. `notes/parameters.md` — table: what | value | source document | page/paragraph
   **⚠ WP7b is blocked until this exists.**

---

# WP2 — TOOLING: CAN AN EXISTING LIBRARY DO THIS?
### 🟡 MEDIUM · 2 days · **Blocks WP3** · Needs WP7a done

## What this is about

We need software that solves scheduling problems. **Google OR-Tools CP-SAT** is free and world-class — it won the international constraint-solving championship repeatedly.

But there's also **PyJobShop**, a Python library built on top of it that might already handle our exact problem shape. **If it fits, we save two weeks.** Find out.

## Step 1 — Test PyJobShop (day 1, no more)

`pip install pyjobshop` · https://github.com/PyJobShop/PyJobShop

It claims to support all of these, which we need:
- **Optional task selection** — a task might be deferred ✓
- **Sequence-dependent setup times** — machine travel between sites ✓
- **Breaks** — the corridor block window ✓
- **Multiple modes** — different ways to do a task ✓
- **Release dates, deadlines, due dates** ✓

Build a toy example: 5 maintenance tasks, 2 track sections, one day, no two tasks on the same section at once. Does PyJobShop express it naturally?

## Step 2 — If it doesn't fit, raw OR-Tools CP-SAT

`pip install ortools` · Docs at `ortools/sat/docs/scheduling.md` in https://github.com/google/or-tools
Also read: https://github.com/d-krupke/cpsat-primer

| Function | What it does |
|---|---|
| `NewIntervalVar` | A task with start, duration, end |
| `NewOptionalIntervalVar` | A task that might not happen |
| `AddNoOverlap` | These tasks can't share a section |
| `AddCumulative` | Only N machines available |
| `AddHint` | Warm start — re-solve fast from a previous answer (WP5 needs this) |
| `BestObjectiveBound` | The proof of how good our answer is |

## Step 3 — Prove it works

Download PSPLIB: https://www.om-db.wi.tum.de/psplib/ · Parser: https://github.com/PyJobShop/PSPLIB

**Solve a J30 instance (30 tasks) to proven optimality.** Published answers exist, so you'll know instantly if the setup is wrong.

## Step 4 — Also install PuLP

`pip install pulp` — WP4 needs it for an independent cross-check. Just make sure it works.

## 🤖 Ask AI

> "I need to model a scheduling problem in Google OR-Tools CP-SAT. Maintenance tasks must be scheduled on track sections. Two tasks cannot be on the same section at the same time. Some tasks are optional (can be deferred). There is a fixed daily window when maintenance is allowed. Show me a minimal Python example using NewOptionalIntervalVar, AddNoOverlap, and how to read the optimality gap from the solver."

> "Compare PyJobShop and raw OR-Tools CP-SAT for a resource-constrained scheduling problem with optional tasks and sequence-dependent setup times. What does PyJobShop give me out of the box, and what would I write myself?"

## ⚠️ Traps

- **One day maximum on PyJobShop.** If it's fighting you, switch to raw CP-SAT and move on.
- **AI often writes CP-SAT code with outdated API names.** Check against the official `scheduling.md`.
- **CP-SAT needs whole numbers.** 2.5 hours → 150 minutes.
- **Never report "optimal" if the solver hit a time limit.** Report the real gap.

## Deliverable
`notes/tooling_verdict.md` — which tool, why, plus a screenshot of a solved PSPLIB instance showing its gap. **⚠ WP3 is blocked until this exists.**

---

# WP3 — MODEL DESIGN
### 🔴 HIGH JUDGMENT · 2 days · **Blocks WP4, WP5, WP10** · Needs WP1, WP2, WP6

## What this is about

**The most important package.** Everything else supports it. You are deciding how we translate a messy railway problem into mathematics.

There is no answer to look up. AI can explain options; **you decide**, and you must be able to defend it to a judge.

**Praveen reviews and signs off before this is accepted.**

## Read exactly two things

1. **Budai, Huisman & Dekker (2006)**, *"Scheduling preventive railway maintenance activities"* — the closest published model to ours. Free working paper: search "Erasmus EI 2004-41".
2. **Lidén (2015)**, *"Railway Infrastructure Maintenance — A Survey of Planning Problems"* — **skim only, for the classification.**

## Six decisions, on paper, before any code

### 1. Decision variables
- **Time slots** — a yes/no for every (task, slot) pair. Simple, but the model gets huge.
- **Interval variables** — each task floats with a start, duration, end. Cleaner and faster.

*Recommendation: intervals. Decide consciously; be able to say why.*

### 2. Stopping blocks landing on trains
- **Simple:** compute free windows from the timetable, only allow maintenance inside them.
- **Complex:** treat every train path as a fixed block and forbid overlaps.

*Recommendation: simple. Realistic for weekly/monthly planning, far easier to solve.*

### 3. Multi-department merging — our core feature

Opening a block has fixed overhead: protecting the section, issuing caution orders, getting gangs and machines to site, withdrawing afterwards. **CAG found 46% of lost block time was machines travelling.**

If all three departments work in **one shared window**, that overhead is paid **once, not three times**.

**The catch you must model:** some jobs need the overhead wires *off* (power block); others just need *no trains* (traffic block). **You cannot merge a live-wire job into a power block.**

This is the heart of the project.

### 4. Urgency versus disruption
Doing an overdue safety-critical job now means cancelling train paths. Deferring it means risk.

**Use WP6's cost figures if they exist by the time you start.** *Fallback if WP6 hasn't delivered or found nothing:* use relative weights (safety-critical = 10, routine = 1), state clearly in the spec that they're placeholders, and swap in real costs later. Don't block on WP6.

### 5. Weekly versus monthly
Long jobs and big machine campaigns → monthly. Short routine jobs → weekly.

*Recommendation: solve monthly first to fix the big closures, then fill in weekly detail. This is how ProRail in the Netherlands actually works.*

### 6. Not enough time for everything
There won't be — that's the real case. The model must **defer low-priority jobs gracefully**, not crash with "no solution."

*Use soft penalties, not hard deadlines.*

## 🤖 Ask AI

> "In constraint programming for scheduling, explain the difference between a time-indexed formulation (binary variables for each task-timeslot pair) and an interval-variable formulation. Which scales better for about 200 tasks over a week at 30-minute granularity, and why?"

> "I need to model a shared 'possession window' where multiple maintenance tasks from different departments run together, and a fixed setup cost is paid once per window rather than once per task. How would I express this in OR-Tools CP-SAT?"

> "Explain soft constraints in CP-SAT. I want tasks to meet deadlines, but if capacity is short I want low-priority tasks deferred with a penalty rather than the model returning INFEASIBLE."

## ⚠️ Traps — these would seriously hurt us

- **AI will suggest machine learning to generate the schedule. REJECT IT.** State in your prompt that scheduling must be constraint optimisation because hard safety rules must hold exactly.
- **Don't make the model detailed too early.** Start with 30-minute slots, one corridor, one department. Add complexity only after it works.
- **Don't skip block-type compatibility** (power vs traffic). Merging incompatible jobs is a mistake a railway engineer spots instantly.
- **Don't use hard deadlines.**

## Deliverable
`notes/model_spec.md` — one page: decision variables, full constraint list, objective function. **Whole team signs off before coding starts. ⚠ WP4, WP5, WP10 are blocked until this exists.**

---

# WP4 — MODEL VERIFICATION
### 🟡 MEDIUM · 1 day · Needs WP3's spec · Set up BEFORE the full model is written

## What this is about

Not "does the code run" — **does it encode what we think.**

This is the quiet killer. A constraint model can run perfectly, produce a beautiful schedule, and be silently wrong because one constraint was written backwards or was never added. **It will not crash.** It produces confident nonsense, and we find out when a judge asks something we can't answer.

## Five techniques

**1. Hand-solve a tiny instance.** Five tasks, two sections, one day. Work out the right answer on paper. Does the solver agree? Bug found in an afternoon instead of November.

**2. Deliberately infeasible test.** Feed it a task that cannot fit anywhere. It **must** return INFEASIBLE, not a wrong schedule.

**3. Constraint ablation.** Remove one constraint, re-solve. Did the answer change? If not, that constraint is redundant or broken — find out which.

**4. Independent cross-check.** Build a small MILP in PuLP solving the same tiny instance a different way. Do they agree?

**5. Automatic property tests** — assert on every solution:
- No two tasks overlap on the same section
- No block sits on a scheduled train path
- Every scheduled task is within its deadline
- Every merged block has compatible block types (no live-wire job inside a power block)

## 🤖 Ask AI

> "I have a constraint programming model in OR-Tools CP-SAT for scheduling. How do I verify the model is CORRECT, not just that it runs? Explain constraint ablation testing, property-based testing for schedules, and how to construct a deliberately infeasible test case."

> "Write Python assertions checking a schedule output for these properties: no two tasks overlap on the same resource; every task finishes before its deadline; no task starts outside its allowed window."

## ⚠️ Traps

- **Do this before the full model is written.** Verification written afterwards tends never to get written.
- **The hand-solved instance must be genuinely tiny** — five tasks, so you can be certain of the right answer.
- **Don't trust the model because the output looks reasonable.** A wrong model produces reasonable-looking wrong answers. That's the entire problem.

## Deliverable
Running test suite in `tests/` + `notes/verification.md` describing the hand-solved instance and its correct answer.

---

# WP5 — DISRUPTION & RESCHEDULING
### 🔴 HIGH JUDGMENT · 2 days · Needs WP3's spec

## What this is about

Our model assumes a world where nothing goes wrong. Every task takes exactly the predicted time, every machine turns up, every block runs full length.

**That world does not exist.** Tasks overrun. Machines break. A late goods train eats half the block.

**"What happens when a block overruns?"** is the first question anyone who has worked a block will ask. Right now we have no answer.

## Scope note

**In these 2 days you produce a design and a plan, not a finished implementation.** The disruption simulator can only run once the scheduler exists in the build phase. Deliver: the approach, the disruption model, and the code skeleton.

## Two separate problems

**(a) Robust planning — a schedule that survives disruption.**
Don't pack five tasks back-to-back into a 3-hour block with zero slack. Leave buffer. Put critical tasks early so an overrun doesn't kill them.

**(b) Reactive rescheduling — the plan broke, now what?**
It's 01:30. The tamper hit a fault. Block ends at 04:00. Re-run the optimiser? How fast? Re-plan tonight or the whole week?

**CP-SAT supports warm starts** (`AddHint`) — re-solve quickly from the previous solution.

## The achievable version

Design a simulation that runs our schedule against **100 random disruption scenarios** (task overruns, machine failures, block cancellations) and reports **how often the schedule survives**. That's a **robustness score** — a genuinely strong slide, and almost nobody else will have one.

**If short on time:** add a slack parameter to every block, chart "tasks completed" as slack increases. Half a day, answers the question.

## 🤖 Ask AI

> "Explain the difference between robust scheduling and reactive rescheduling in operations research. For a maintenance scheduling problem where task durations are uncertain, what are the standard approaches to building a schedule that survives overruns?"

> "How do I use warm starts (AddHint) in OR-Tools CP-SAT to re-solve a scheduling problem quickly after a disruption, starting from a previous solution?"

> "Design a Monte Carlo simulation in Python that takes a maintenance schedule, tests it against 100 random disruption scenarios (task overruns, machine failures), and reports what fraction of tasks still complete."

## ⚠️ Traps

- **Don't build a full stochastic optimisation model.** That's a research project. Simulate disruptions against a deterministic schedule — enough, and honest.
- **Make disruption rates realistic.** Ask WP1's owner or a railway relative what actually goes wrong and how often. Invented disruption rates are as bad as invented data.
- **Don't let this become the whole project.** Two days, then stop.

## Deliverable
`notes/robustness.md` — the disruption model, the approach, and a code skeleton.

---

# WP6 — COST IN RUPEES
### 🟢 MECHANICAL · 1 day · Feeds WP3 · Start day 1

## What this is about

*"Improved utilisation by 27%"* is abstract. *"Recovers 340 track-hours a year, worth approximately ₹X"* is a completely different pitch.

It also fixes a real weakness: our objective function currently weighs urgency against disruption using **weights we invented**. Ground them in real costs and the model becomes defensible.

## Find these

| What | Why |
|---|---|
| Cost of a track possession hour (machine hire, gang cost, mobilisation) | The cost of taking a block |
| Cost of a train delay minute in India — passenger and freight | The cost of disruption |
| Cost of a derailment | The cost of NOT maintaining |
| Value of freight tonnage delayed | Freight impact |
| Track machine hire or ownership rates | Expensive assets sitting idle |

## Where to look

- **CAG reports** — they quantify losses in rupees. Start here: https://cag.gov.in
- **Indian Railways Year Book** — search indianrailways.gov.in for "Year Book"
- **Railway Budget documents**
- **Parliamentary answers** — https://sansad.in · https://prsindia.org
- **National Rail Plan**

## Precedent for the pitch

The UK's **Schedule 4** regime pays train operators compensation for planned disruption — that price signal forces British planners to weigh possession length against passenger cost.

India is vertically integrated, so there's no such internal market. **We substitute an internal "shadow price" for delay in our objective function.** Explaining that substitution shows we understood *why* the foreign mechanism exists rather than copying its shape.

## 🤖 Ask AI

> "Help me find published figures on the cost of railway track possession and the cost of train delay in India. Which Indian government sources — CAG reports, Railway Year Book, parliamentary answers — would contain these? I need actual citable numbers, not estimates."

> "Explain the UK railway Schedule 4 compensation regime in simple terms — what it pays for and why it exists."

## ⚠️ Traps

- **AI WILL invent rupee figures. This is the biggest risk in this package.** Every number needs a document behind it. "Not found" is a fine answer.
- **Distinguish cost from loss.** "₹X crore of assets idle" ≠ "each block hour costs ₹Y."
- **Don't convert foreign figures.** UK delay costs don't transfer to India.

## Deliverable
`notes/costs.md` — table: parameter | value | source | confidence (high / medium / not found)

---

# WP7a — DOWNLOADS
### 🟢 MECHANICAL · 1 day · Start day 1 · **Blocks WP2**

Download everything below into the project folder. That's the whole job.

### Network & geography
- **OSM India extract** — https://download.geofabrik.de/asia/india.html → `india-latest.osm.pbf` *(ODbL)*
- **datameet/railways** — https://github.com/datameet/railways → `stations.json`, `trains.json`, `schedules.json` *(CC0. Data ~2016 — good for network layout, not current timings)*
- **OpenRailwayMap** — https://www.openrailwaymap.org *(visual checking)*

### Timetable
- **data.gov.in Train Time Table** — https://www.data.gov.in/catalog/indian-railways-train-time-table
  *Free registration for an API key. Licence: GODL-India (we must credit them).*
  Helper: `pip install datagovindia`

### Indian Railways manuals
- **IRPWM** — https://iricen.gov.in/iricen/Track_Manuals/IRPWM.pdf
- **IRTMM** ⚠ most important — https://indianrailways.gov.in/railwayboard/uploads/codesmanual/IRTMM-RDSO/TMM_Ch/
- **USFD Manual 2012** — https://iricen.gov.in/iricen/Track_Manuals/USFD/usfd_new.pdf
- **SEM** — https://indianrailways.gov.in/railwayboard/view_section.jsp?lang=0&id=0,1,304,366,544,664
- **IRSTMM** — https://indianrailways.gov.in/railwayboard/uploads/codesmanual/IRSTMM/IRSTMM.pdf
- **ACTM Vol II (AC Traction Manual)** — search RDSO site; needed by WP1 and WP7b

### Audits — our baseline evidence
- **CAG 45/2017 Chapter 3** — https://cag.gov.in/uploads/download_audit_report/2018/Chapter_3_Utilisation_of_resources_and_infrastructure_for_track_maintenance_of_Report_No.45_of_2018_%E2%80%93_Compli_1.pdf
  *If dead: go to https://cag.gov.in and search "Report No. 45 of 2017 Railways track maintenance"*
- **CAG 22/2022 Derailments** — search https://cag.gov.in for "Derailment in Indian Railways 2022"
- **IR Year Book 2023-24** — search indianrailways.gov.in for "Year Book"

### Benchmarks
- **PSPLIB** — https://www.om-db.wi.tum.de/psplib/
- **MMLIB** — https://www.projectmanagement.ugent.be

### Code to clone
- **OR-Tools** — https://github.com/google/or-tools
- **PyJobShop** — https://github.com/PyJobShop/PyJobShop
- **CP-SAT Primer** — https://github.com/d-krupke/cpsat-primer
- **Flatland** — https://github.com/flatland-association/flatland-rl
- **Flatland MAPF winner** — https://github.com/Jiaoyang-Li/Flatland

### Python packages
```
pip install ortools pyjobshop pulp pandas matplotlib datagovindia
```

### Papers
- PyJobShop — https://arxiv.org/abs/2502.13483
- Flatland — https://arxiv.org/pdf/2012.05893
- Budai/Huisman/Dekker (2006) — search "Scheduling preventive railway maintenance activities Erasmus EI 2004-41"
- Lidén (2015) — search "Railway Infrastructure Maintenance Survey Transportation Research Procedia 10"

## ⚠️ Traps
- **Some links may be dead.** Don't give up — search the site. Government sites reorganise constantly.
- **Record the licence for each source.** datameet = CC0 (free rein), data.gov.in = GODL-India (must credit), OSM = ODbL (credit + share-alike).

## Deliverable
Populated folders + `notes/attribution.md`. **⚠ WP2 is blocked until the code and benchmarks are downloaded.**

---

# WP7b — SYNTHETIC DATA GENERATOR
### 🔴 HIGH JUDGMENT · 2 days · **Needs WP1's `parameters.md`** · Blocks WP10

## What this is about

**Our biggest credibility risk.**

There is no public dataset of railway maintenance jobs, defects, or block requests — anywhere in the world. Even the UK doesn't publish possession data. So we generate our own.

But our judges are railway engineers. **If our synthetic data looks fake to them, they stop believing our results.** Everything downstream depends on this being convincing.

## Generate five things

| Data | Anchor to |
|---|---|
| **Pending maintenance tasks** | USFD Manual (rail testing by GMT), IRPWM (tamping), SEM (signalling), ACTM (OHE) |
| **Task durations** | IRTMM Ch 2 machine output rates (800 m/hr tamping → duration = length ÷ rate + setup) |
| **Defect records** | Rail fracture and derailment rates from CAG reports |
| **Block request history** | CAG's ~50% grant ratio; refusal reasons (32% Operating, 30% not planned) |
| **Machine & gang availability** | CAG: ~33% of small track machines out of order, 9–22% staff vacancies |

## The rules

- **Every generated field traces to a published norm.** Write down which one.
- **Due date = last done + periodicity** from the manual.
- **Tag each task:** criticality, required block type (traffic / power / disconnection), section, duration.
- **Use a real section's real timetable** from the data.gov.in download, so blocks genuinely compete with actual train paths.
- **Model machine travel explicitly** — CAG's 46% figure justifies it.
- **Reproduce the CAG baseline:** ~50% grant ratio, 33% of granted time unused.

## The sanity check

For every generated row ask: *would an SSE/P.Way look at this and say "that's not right"?*

Example: a 2 km tamping task at 800 m/hour needs 2.5 hours of working time plus setup. **It cannot fit in a 90-minute block.** If your generator produces that without flagging it as a multi-block job, the data is wrong.

## 🤖 Ask AI

> "Help me design a synthetic data generator for railway maintenance tasks. Each task needs a due date derived from a maintenance periodicity, a duration derived from a machine output rate in metres per hour, a criticality level, and a required block type. What statistical distributions suit defect arrival and task-duration variation?"

> "I want to generate maintenance task lists where roughly 50% of requested block hours get granted, matching an audited real-world figure. How should I model the grant/refusal process?"

## ⚠️ Traps — these destroy credibility

- **Do not invent periodicities.** Take them from WP1's `parameters.md`. **If WP1 hasn't delivered, do not start — help WP1 finish instead.**
- **Do not generate durations that can't fit any realistic block** without flagging them as multi-block.
- **Do not use a random network.** Use a real Indian Railways section with its real timetable.
- **Document every assumption in a table.** A judge will ask where the numbers came from.
- **Get a railway relative to review the output if at all possible.** One review session is worth more than everything else here.

## Deliverable
Generator code + `data/synthetic/` + `notes/assumptions.md` — every assumption and its source. **⚠ WP10 is blocked until this exists.**

---

# WP8 — VISUALISATION & OPERATOR WORKFLOW
### 🟡 MEDIUM · 1.5 days · Demo-defining · Can start day 2

## Part A — We've been planning the wrong picture

Everyone assumes we'll show a **Gantt chart**. **Railway planners do not read Gantt charts.**

They read a **time-distance diagram** — stations down one axis, hours across the other, every train a diagonal line. Blocks appear as rectangles in the white space between the train lines. Called a **control chart**, **train graph**, or **Marey diagram**.

**With a time-distance chart, a railway judge reads our output instantly and without explanation. With a Gantt, they have to translate.**

This may be the single highest-value demo decision available to us.

**Research:** search "railway time distance diagram", "train graph plotting python", "Marey chart". Look at how IRPWM and the CAG reports illustrate master charts. Matplotlib can do it.

## Part B — Who actually uses this?

We've been building an optimiser without asking who sits in front of it.

A Divisional planning officer? A Section Controller? When do they open it — once a week, or at 23:00 when something breaks?

**This changes what we build:**
- **Override must exist.** If a controller can't say "no, not that one — I know something you don't," they will never use it.
- **It must explain itself.** "Why here?" needs an answer.
- **It must be fast enough.** If it takes 10 minutes and they have 2, they go back to the spreadsheet.

**The evidence:** CAG found **30% of track machine idling was "blocks not planned by Divisions"** — a people-and-workflow problem, not an algorithm problem. Sweden reserved maintenance windows properly and still got **only 34% utilisation**, for the same reason.

**A perfect optimiser nobody uses fixes nothing.**

**Cheapest research on the whole list:** one question to a railway relative — *"If a computer gave you a maintenance plan, what would make you actually use it, and what would make you ignore it?"*

## 🤖 Ask AI

> "Explain what a railway time-distance diagram (train graph / Marey chart) is and how to plot one in Python with matplotlib. Stations on the y-axis, time on the x-axis, trains as diagonal lines. I also want to overlay maintenance block windows as shaded rectangles."

> "What design principles apply to decision-support tools used by operational controllers who need to override the system's recommendations? What makes operators trust or distrust automated scheduling advice?"

## ⚠️ Traps
- **Don't build a beautiful UI with a weak engine.** A rough chart with a real optimiser beats a polished dashboard with a greedy rule.
- **Don't skip the override function.** A judge will ask.

## Deliverable
Working time-distance chart prototype + `notes/operator_requirements.md`

---

# WP9 — BORROWED PRINCIPLES, REJECTIONS, PAST SIH
### 🟢 MECHANICAL · 1.5 days · Can start day 1

## Part A — Three borrowed principles (half a day, no more)

Other countries face the same problem. **Only three of their ideas matter. Don't research beyond these three.**

| Principle | Evidence | What it becomes in our model |
|---|---|---|
| **Bundle departments into one block** | ProRail (NL) prefers 9-hour over 4-hour windows; Germany closed the Riedbahn for 5 months instead of years of night blocks and cut infrastructure disruptions ~27%; SNCF calls it "massification" | One shared window; setup cost paid once |
| **Reserve the maintenance window before allocating train paths** | Sweden (Lidén & Joborn): **11–17% maintenance cost saving** from integrated vs sequential planning | The corridor block is a reservation to fill, not an afterthought |
| **Rolling horizon with a freeze point** | Network Rail (UK) publishes its plan 26 weeks ahead, freezes ~7 weeks out | Blocks inside the freeze window are fixed inputs |

**The cautionary finding — include it:** Sweden reserved 10% of line capacity for maintenance windows and got **only 34% utilisation; 68% of trackwork happened outside the windows anyway.**

Reserving windows fixes nothing by itself. **We must measure utilisation as a headline metric**, or we rebuild India's current problem behind a nicer interface.

## Part B — Rejection notes (half a page each)

Research these *only* enough to answer a judge asking "why didn't you use AI?"

**Why not deep reinforcement learning?**
> SBB (Swiss railways), Deutsche Bahn and SNCF built **Flatland**, an open railway RL environment, and ran a NeurIPS 2020 challenge with 700+ participants from 51 countries. **The winning entry used multi-agent path finding — a classical search method — and beat every reinforcement learning entry in both tracks.** Beyond that: RL cannot guarantee hard constraint satisfaction, we have no training data, and it can't explain itself to a railway engineer.

Sources: https://github.com/flatland-association/flatland-rl · https://github.com/Jiaoyang-Li/Flatland

**Why not a neural network that outputs schedules?**
> It cannot guarantee two closures never land on the same track at once.

**Why not genetic algorithms or simulated annealing?**
> No provable bound on solution quality — and that bound is our main differentiator.

## Part C — What did past SIH teams build? (half a day)

We've researched Dutch railways and never looked at **what actually wins SIH.**

- Previous railway problem statements — what did winners build?
- Search GitHub: "SIH 2024 railway", "smart india hackathon railway"
- Find SIH winner writeups and post-mortems
- What does a winning SIH demo look like versus what we're imagining?

**This tells us the real standard we're judged against**, which may differ from the academic bar we've been holding ourselves to.

## 🤖 Ask AI

> "Summarise how the Netherlands (ProRail), Sweden (Trafikverket), the UK (Network Rail) and Germany (DB InfraGO) plan railway maintenance possessions. Focus on: do they bundle work from different departments into one closure, and do they reserve maintenance windows before or after allocating train paths?"

> "What are the practical limitations of deep reinforcement learning for scheduling problems where hard constraints must never be violated?"

## ⚠️ Traps
- **Don't over-claim novelty.** CP-SAT for maintenance scheduling is published — Germany did a national model in 2026, ProRail in 2022.
- **Don't research a fourth country.** Three principles is enough.

## Deliverable
`notes/borrowed.md`, `notes/rejections.md`, `notes/past_sih.md`

---

# WP10 — BASELINES: WHAT WE COMPARE AGAINST
### 🟡 MEDIUM · 1.5 days · Needs WP7b's data and WP3's spec

## What this is about

**This package was missing from earlier drafts and it's essential.** Without it we have no number to put on a slide.

Our entire pitch is *"we improved things."* That sentence is meaningless without a **before**. You are building the "before."

## Two baselines

### Baseline 1 — The greedy scheduler
A simple first-fit rule: take tasks in priority order, put each in the first free slot that fits.

**This is what almost every competing team will submit as their entire solution.** We build it in an afternoon so we can show our optimiser beating it — and so we can answer *"why not just use a simple rule?"* with a number instead of an argument.

### Baseline 2 — The current messy process ⚠ the important one
Simulate how Indian Railways does it **today**: each of the three departments independently books its own windows, greedily, with no coordination between them.

Then tune it so the outcome roughly reproduces the CAG's audited findings:
- ~50% of demanded block hours granted
- ~33% of granted block time unused

**This is our headline.** *"Current practice achieves X% corridor utilisation. Our system achieves Y%."*

## Metrics to compute for all three (greedy, current-process, ours)

| Metric | Why |
|---|---|
| Corridor utilisation % | The Sweden warning — reserving isn't using |
| Block hours granted vs demanded | Direct CAG comparison |
| Tasks completed on time | Does maintenance actually get done? |
| Overdue tasks cleared | Safety backlog |
| Total asset downtime | The PS's stated objective |
| Optimality gap | Only ours has one — that's the point |
| Solve time | Is it usable? |

## 🤖 Ask AI

> "I need to build a greedy first-fit baseline scheduler in Python to compare against an optimised schedule. Tasks have priorities, durations, deadlines, and must be assigned to time windows on resources without overlap. Show a simple implementation."

> "How do I fairly compare an optimisation-based schedule against a greedy heuristic baseline? What metrics matter, and what are common mistakes that make such comparisons unfair or misleading?"

## ⚠️ Traps

- **Don't make the baseline artificially bad.** If we cripple the greedy rule to make ourselves look good, a judge will see it and we lose all credibility. Make it a genuinely reasonable greedy rule.
- **Use identical inputs for all three.** Same tasks, same timetable, same machines. Any difference and the comparison is worthless.
- **The current-process baseline must reproduce the CAG numbers** — otherwise it isn't a model of reality, it's just another algorithm.

## Deliverable
Baseline code + `notes/baselines.md` with the metric comparison table.

---

# PART 3 — WHAT WE SAY AND NEVER SAY

**Never claim:**
- ❌ "We invented a new algorithm"
- ❌ "First use of CP-SAT for maintenance scheduling"
- ❌ "We beat commercial solvers"
- ❌ "Optimal" when the solver timed out

**Always say:**
- ✅ "We applied proven constraint-programming methods to a problem Indian Railways currently does by hand"
- ✅ "Our schedule is provably within X% of optimal"
- ✅ "We measured against the CAG's own audited baseline"
- ✅ "We coordinate three departments in one possession, which published models mostly keep separate"
- ✅ "This is decision support. The final block grant stays with the DRM and Section Controller. Our system recommends; humans authorise."

That last line pre-empts an entire line of safety questioning — and it's another independent argument for optimisation over neural networks, because safety-related systems cannot be black boxes.

---

# PART 4 — THE GATES THAT MATTER

| Gate | When | Test |
|---|---|---|
| **1** | Day 3 | WP7a done, WP1's `parameters.md` delivered |
| **2** | Day 4 | We solve a PSPLIB benchmark and report an honest optimality gap |
| **3** | Day 8 | Model spec signed off + realistic synthetic data + baselines built. **Research ends.** |
| **4** | Build day 16 | Scheduler produces a conflict-free plan beating both baselines on a showable number |
| **5** | Before demo | It runs live on a task list a judge invents on the spot — **not** a pre-baked scenario |

---

# PART 5 — WHAT KILLS US

| Failure | Prevention |
|---|---|
| Pretty dashboard, dumb algorithm | Real CP-SAT with a reported gap |
| Hardcoded demo | Must accept a judge's made-up input |
| Fake-looking synthetic data | Anchor to published norms; get a railway relative to review |
| Silent model bug | WP4, set up before the model is finished |
| No answer on disruption | WP5, even the half-day version |
| Solving one department only | Multi-department merging is the entire point |
| Ignoring the timetable | A block over a scheduled train isn't a plan |
| Model too slow to demo | Start with 30-minute slots |
| **No baseline** | **WP10 — without it we have no number** |
| AI-invented numbers reaching judges | Every figure traced to a real document |
| Research never ends | Hard stop day 8 |

---

# PART 6 — FOLDER STRUCTURE

```
sih26027/
├── data/
│   ├── network/      osm pbf, datameet json
│   ├── timetable/    data.gov.in csv
│   ├── benchmarks/   psplib, mmlib
│   └── synthetic/    WP7b output
├── manuals/          IRPWM, IRTMM, USFD, SEM, IRSTMM, ACTM
├── audits/           CAG reports, Year Book
├── papers/           pdfs
├── code_refs/        or-tools, pyjobshop, cpsat-primer, flatland (cloned)
├── src/              our code
├── tests/            WP4 verification suite
└── notes/
    ├── glossary.md              WP1
    ├── parameters.md            WP1  ⚠ blocks WP7b
    ├── tooling_verdict.md       WP2  ⚠ blocks WP3
    ├── model_spec.md            WP3  ⚠ blocks WP4/5/10
    ├── verification.md          WP4
    ├── robustness.md            WP5
    ├── costs.md                 WP6
    ├── assumptions.md           WP7b ⚠ blocks WP10
    ├── operator_requirements.md WP8
    ├── borrowed.md              WP9
    ├── rejections.md            WP9
    ├── past_sih.md              WP9
    ├── baselines.md             WP10
    └── attribution.md           licence tracking
```

---

# FIRST ACTIONS

**Today:**
1. Everyone reads Part 0 — the whole thing. Non-negotiable shared knowledge.
2. Fill in the owner column in the assignment table.
3. **WP7a, WP1, WP6 and WP9 start immediately** (no dependencies).

**Separately, someone must:**
- Confirm with our SPOC that we can block SIH26027
- Check the idea counter on the SIH portal in early September — Ministry of Railways will draw a crowd
