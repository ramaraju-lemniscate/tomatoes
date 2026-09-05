# SIH26027 — Team Handbook
### AI-Powered Automatic Block Planning for Indian Railways

**Problem ID:** SIH26027 · **Sponsor:** Ministry of Railways · **Track:** Software
**Idea submission deadline:** 20 September 2026 · **Dataset provided:** None

*This document is written so anyone on the team can follow it, whether or not you've worked on scheduling or optimisation before. No prior knowledge assumed. Read it once end to end before we start building.*

---

## Part 1 — What the problem actually is

### The situation in plain English

To fix a railway track, you have to stop the trains running on it. There's no other way. That planned shutdown of a section of track is called a **block**.

Indian Railways has three separate departments that need to do maintenance:

| Department | What they maintain |
|---|---|
| **Engineering (P.Way)** | The track itself — rails, sleepers, ballast |
| **Signal & Telecom (S&T)** | Signals, points, interlocking |
| **Traction Distribution** | The overhead electric wires (OHE) |

**Here's the problem:** each of these three departments asks for their track shutdown *separately and by hand*. Nobody coordinates between them.

So what happens:
- Two departments book overlapping windows on the same stretch, and they clash.
- Or three departments each shut the same section on three different nights — when one combined shutdown would have done all three jobs.
- Track sits closed longer than it needs to be, trains get delayed, and maintenance still falls behind.

### What we have to build

A system that takes:
- all the pending maintenance jobs from all three departments,
- the train timetable (which we're not allowed to disturb),
- and which stretches of track are available when,

and produces **the best possible shutdown schedule** — weekly and monthly.

That's it. That's the whole project.

---

## Part 2 — The evidence (this is our strongest weapon)

The judges will be railway professionals. So we shouldn't tell them there's a problem — we should show them the government's own audit saying so.

### The CAG audit findings

The **Comptroller and Auditor General of India (CAG)** — the government's official auditor — published *Report No. 45 of 2017* on track maintenance, laid in Parliament on 13 March 2018. It found:

> **Engineering departments asked for 58,342 hours of maintenance blocks. They got 29,411 hours. That's a 49.6% shortfall — roughly half of what was needed.**

It gets worse:

- **33% of the block time that WAS granted couldn't even be used** by the track machines.
- **46% of that lost time was just machines travelling** to and from the site.
- Some divisions were getting corridor blocks of only 90–120 minutes, when the norm requires at least 2.5 hours.

A later CAG audit (*Report No. 22 of 2022*, on derailments) found that when track machines sat idle:
- **32%** of the time it was because Operating didn't give the block
- **30%** of the time it was because the Division never planned one

**That's 62% of machine idle time caused by block-planning failure alone.**

### Why this matters for us

This gives us a **"before" number.** We can build a simulation of the current messy process, show it producing roughly the CAG's ~50% shortfall, then show our system doing better.

Almost no student team defines their baseline from a national audit. This alone will separate us.

---

## Part 3 — Railway vocabulary (learn this — judges will notice)

We will be talking to people who use these words every day. Getting them wrong makes us look like outsiders. Getting them right buys us enormous credibility.

**Block** — a planned period where a section of track is taken out of use so work can happen safely.

**Types of block:**
- **Traffic block** — no trains at all on that section. Requested by the engineering official, approved at DRM (Divisional Railway Manager) level.
- **Power block** — the overhead electric wires are switched off for OHE work. The **TPC (Traction Power Controller)** handles the shutdown.
- **Power-and-traffic block** — everything stops.
- **Disconnection** — taking signalling equipment out of service for S&T work. (This is the "D" in BDMS.)
- **Corridor block** — a pre-planned daily window built into the timetable, reserved for maintenance.
- **Mega block / jumbo block** — long consolidated shutdowns, several hours to 60+ hours, mostly on Mumbai suburban lines on Sundays.

**Other terms we must use naturally:**

| Term | Meaning |
|---|---|
| **Possession** | Maintenance staff formally "taking possession" of the track |
| **Path** | A train's scheduled slot through a section |
| **Section** | A stretch of track between two points |
| **Caution order** | Written warning issued to the last train before a block |
| **TSR** | Temporary Speed Restriction |
| **GMT** | Gross Million Tonnes — how much traffic a section carries |
| **DRM** | Divisional Railway Manager (approves blocks) |
| **Section Controller** | Decides exact block timing on busy sections |
| **SSE/P.Way** | Senior Section Engineer, Permanent Way (track) |
| **JE/Signal** | Junior Engineer, Signal |

### The four systems we're integrating

The problem statement names these — we must know what each one holds:

- **TMS (Track Management System)** — the track asset register. Inspections, track geometry recordings, ultrasonic flaw detection results, rail fractures, defect severity. Live network-wide since 2020.
- **SMMS (Signalling Maintenance & Management System)** — S&T assets with GPS tags, maintenance schedules, overdue reports, failure reports.
- **TDMS (Traction Distribution Management System)** — overhead wire and power supply data, GPS foot-patrolling, real-time defect capture, OHE power-block requests.
- **COA (Control Office Application)** — the timetable and live train running. This is where corridor availability comes from.

**BDMS (Block & Disconnection Management System)** — built by CRIS, this is the platform where block requests get raised and tracked. It digitised the paperwork, but **planning across departments is still manual and separate.** That gap is exactly our problem statement.

### The 3-hour corridor policy

In **2018**, Indian Railways decided every section would keep its tracks free for **at least three hours daily** for maintenance, built into the "zero-based timetable."

Railways Minister Ashwini Vaishnaw confirmed this in the Rajya Sabha in March 2025, calling it a "tough decision" and crediting a detailed study by **IIT Bombay**. He also stated rail fractures dropped from about 2,500 in 2013-14 to about 250 in 2024.

*(Treat that last figure as an official claim, not independently verified.)*

**We must build this 3-hour corridor into our model as a hard constraint.** It shows we know current policy.

### How often does maintenance actually happen?

We'll need this to generate realistic data. Published norms:

- **Ultrasonic rail testing (USFD):** frequency depends on traffic. Roughly one test per 8 GMT. Heavier sections get tested monthly; lighter ones yearly.
- **Tamping (track machines):** the manual says machines need **at least 4 hours in one block per day**, or two blocks of 2.5 hours.
- **Track Recording Car:** fortnightly on high-speed routes, monthly elsewhere.
- **OHE/traction:** per the AC Traction Maintenance Manual.
- **Signalling:** fortnightly, monthly, quarterly, half-yearly and yearly checks depending on the equipment.

---

## Part 4 — The single most important design decision

### The trap in the problem statement

The problem statement says:

> *"Uses AI/ML algorithms to prioritize and schedule maintenance tasks..."*

**Scheduling is not a machine learning problem.** This matters enormously, so let's be clear about why.

If we train a neural network to output a schedule, it will produce plans that break hard rules — two shutdowns on the same track at once, or a block sitting on top of a scheduled Rajdhani. A railway cannot run a schedule that is *probably* conflict-free. It has to be *definitely* conflict-free.

### The correct architecture

Split the system in two layers:

```
        MAINTENANCE DATA
               ↓
    ┌──────────────────────┐
    │   ESTIMATION LAYER   │   ← Machine Learning belongs HERE
    │                      │
    │  • How long will     │
    │    this job take?    │
    │  • How urgent/risky  │
    │    is this defect?   │
    └──────────────────────┘
               ↓
    ┌──────────────────────┐
    │    DECISION LAYER    │   ← Optimisation belongs HERE
    │                      │
    │  • Which jobs, when, │
    │    on which section? │
    │  • Merged with whom? │
    └──────────────────────┘
               ↓
         BLOCK SCHEDULE
```

**ML estimates. The optimiser decides.**

### How to say this to judges

Don't contradict them. Reframe:

> *"The problem statement asks for AI/ML to prioritise and schedule. We've split that into its two natural halves. The **prioritisation** is a learned criticality and risk score — that's the ML. The **scheduling** is a constrained optimisation, because a railway schedule has to satisfy hard safety rules exactly, not probably. This is how ProRail in the Netherlands, Trafikverket in Sweden and Network Rail in the UK actually do it."*

That answers their wording with something more sophisticated than what they asked for. It's the strongest thirty seconds in our pitch.

---

## Part 5 — The tools we'll use

### Google OR-Tools CP-SAT

This is a free, open-source solver from Google. It's genuinely world-class at scheduling problems — it won the international constraint-solving championship repeatedly.

**We are allowed to use it.** (Unlike some other SIH problems, nothing here says build your own solver.)

What it gives us, in plain terms:

| Feature | What it means |
|---|---|
| **Interval variables** | "This job takes 3 hours, starting somewhere" |
| **NoOverlap** | "These two jobs can't happen on the same track at once" |
| **Cumulative** | "We only have 4 machines, don't schedule 6 jobs" |
| **Optional intervals** | "This job might get deferred to next week" |

Read the official guide: `ortools/sat/docs/scheduling.md`. It has working Python examples for everything above.

### Why not something else?

For pure scheduling with whole-number data, CP-SAT beats the free alternatives (CBC, HiGHS) and matches commercial tools. Use a small MILP model in PuLP as a **cross-check** on tiny cases, to confirm our CP-SAT model is correct.

### Honest reporting — the "optimality gap"

This concept will win us points, so everyone should understand it.

When the solver finishes, it tells us two numbers:
- **The best schedule it found** (say, 340 hours of downtime)
- **A proven lower bound** — mathematically, nothing can be better than this (say, 333 hours)

The difference is the **gap**: about 2%. That means *"our answer is at most 2% worse than the theoretical best possible."*

**Say this out loud in the demo.** Most competing teams will use a simple greedy rule and have no answer when a judge asks "how do you know your schedule is good?" We will have a proof.

**Never claim "optimal" if the solver timed out.** Report the real gap. Judges respect honesty far more than an inflated claim they can poke a hole in.

---

## Part 6 — The hard parts we have to think about ourselves

These are the genuine design decisions. Nobody can hand us the answers — this is the actual intellectual work of the project.

### 1. What are our decision variables?

Two options:
- **Time slots:** a yes/no for every (job, time slot) pair. Simple, but the model gets huge.
- **Interval variables:** each job is an object with a start, duration and end that floats. Cleaner and faster.

**Start with intervals.** But make the choice consciously and be ready to explain it.

### 2. How do we stop blocks clashing with trains?

Two approaches:
- **Simple:** work out the free windows from the timetable first, then only allow maintenance inside them.
- **Complex:** treat every train path as a fixed block and forbid overlaps.

**Start simple.** It's realistic for weekly/monthly planning and far easier to solve.

### 3. Multi-department merging — this is our novelty

**The key insight:** opening a block has a fixed overhead. You have to protect the section, issue caution orders, get the gang and machines there, and then withdraw everything afterwards. The CAG found **46% of lost block time was just machines travelling.**

If Engineering, S&T and Traction all work in **one shared window**, that overhead is paid **once instead of three times.**

But there's a catch we have to model correctly:
- Some jobs need the **overhead wires switched off** (power block)
- Others just need **no trains** (traffic block)

You can't merge a live-wire job into a power block. Getting this right is the heart of the project.

### 4. Urgency versus disruption

Doing an overdue safety-critical job *now* means cancelling valuable train paths. Deferring it means risk.

How do we weigh those against each other? Options:
- A weighted score (and be ready to defend the weights)
- Safety always first, then everything else
- Show a range of trade-off options

### 5. Weekly versus monthly

Long jobs and big machine campaigns → monthly plan.
Short routine jobs → weekly plan.

Easiest approach: solve the monthly plan first to fix the big shutdowns, then fill in weekly detail. This mirrors how ProRail actually works.

### 6. What if there isn't enough time for everything?

There won't be — that's the real-world case. Our model must **defer the lowest-priority jobs gracefully**, not crash with "no solution possible."

Use soft penalties, not hard deadlines.

---

## Part 7 — Where our novelty comes from

Being honest: railway maintenance scheduling is a well-studied problem worldwide. We're not inventing the mathematics. But there are four places where we can make a real contribution:

1. **Multi-department block merging.** Published research overwhelmingly handles *one* department's track possessions. Coordinating Engineering + S&T + Traction together is genuinely under-explored.

2. **The Indian 3-hour corridor policy** as a built-in structural constraint. No international paper models this.

3. **Criticality scoring grounded in Indian defect classifications** — things like IMR (Imminent-failure Marked for Removal) rail marking and USFD defect classes.

4. **Evaluating against the CAG-documented baseline.** Reproducing the audited ~50% shortfall and beating it is a rigorous evaluation almost nobody does.

Worth knowing: there is essentially **no published optimisation work on Indian Railways block planning.** Plenty on Dutch, Swedish, UK and German networks. That's our opening.

---

## Part 8 — What can go wrong (read this twice)

| Failure mode | Why it kills us | How we avoid it |
|---|---|---|
| **Building a pretty dashboard with a dumb algorithm behind it** | Judge asks "how do you know this schedule is good?" and we have nothing | Real CP-SAT model first, UI last |
| **Fake AI** | Bolting on a neural net that does nothing useful, obvious to any technical judge | ML only where it genuinely estimates something |
| **Unrealistic synthetic data** | Railway engineers spot it instantly and stop believing everything else | Anchor every number to published norms |
| **Claiming "optimal" after a timeout** | One question and it collapses | Always report the real gap |
| **Ignoring the timetable** | A block plan that closes track during a scheduled train is meaningless | Build the timetable constraint in Week 4 |
| **Model too big to solve** | Nothing runs during the demo | Start with coarse time slots, refine later |
| **Solving only one department** | That misses the entire point of the problem | Multi-department merging is non-negotiable |
| **No baseline to compare against** | No number to put on a slide | Build the "current messy process" simulator |

---

## Part 9 — Eight-week build plan

Notice the weighting: **five of eight weeks on the actual problem**, not on frontend and deployment.

### Week 1 — Domain immersion
Everyone learns the vocabulary. Read the key survey papers. Skim the railway manuals and the CAG report.
**Deliverable:** one-page glossary + the problem restated in railway language.

### Week 2 — Learn the solver
Work through the official OR-Tools scheduling guide. Build a toy scheduler for one section, one department.
**Deliverable:** something that schedules 10 jobs without clashing.

### Week 3 — Data
Pull a real section's timetable. Build the synthetic maintenance-job generator anchored to real periodicity norms. Define the data schema mirroring TMS/SMMS/TDMS.
**Deliverable:** realistic dataset + a written table of every assumption we made.

### Week 4 — Core model
Single department, real constraints, real timetable, criticality-weighted objective. Report optimality gaps from day one.
**Deliverable:** a conflict-free weekly schedule.

### Week 5 — Multi-department merging
The novel core. Shared possession windows, overhead amortisation, power-vs-traffic block typing.
**Deliverable:** the actual differentiated scheduler. **Spend real time here.**

### Week 6 — Horizons and ML layer
Weekly + monthly planning. Add the estimation layer (duration prediction, criticality scoring).
**Deliverable:** estimates feeding decisions.

### Week 7 — Evaluation and visuals
Build the "current process" baseline. Measure the uplift. Build a Gantt chart of blocks by section and department, with before/after numbers.
**Deliverable:** our headline number.

### Week 8 — Harden and rehearse
Freeze scope. Write the pitch. Practise domain Q&A. Prepare the gap and scalability story.

---

## Part 10 — What each of us needs to learn

### Must learn (no way around these)

**Constraint programming for scheduling**
- Official OR-Tools guide: `ortools/sat/docs/scheduling.md`
- The **CP-SAT Primer** by Dominik Krupke — especially "Understanding the Log" and "Parameters"

**The problem's academic foundation**
- Tomas Lidén, *"Railway Infrastructure Maintenance — A Survey of Planning Problems and Conducted Research"* (2015) — the standard survey
- Budai, Huisman & Dekker, *"Scheduling preventive railway maintenance activities"* (2006) — the closest textbook match to our problem. The free 2004 working-paper version is on the Erasmus University repository.

**Basic linear/integer programming** — an intro OR text plus the PuLP docs, enough to cross-check our model.

**The Indian railway workflow** — blocks, possessions, and the four source systems.

### Nice to have (if time allows)
- Integrated timetabling + maintenance research (INFORMS RAS 2016 competition papers) — good depth for Q&A
- Track degradation and remaining-useful-life prediction
- Metaheuristics (genetic algorithms, large-neighbourhood search) as a fallback for very large instances

---

## Part 11 — Data situation

### What's genuinely public
- **Train timetables** — data.gov.in has train-wise arrival/departure per station
- **Network structure** — the `datameet/railways` GitHub repo has station and line geodata
- Several Kaggle timetable dumps
- NTES / RailRadar for live running

### What isn't
There is **no public dataset of maintenance defects, blocks or possessions.** None. That's why the problem statement supplies nothing.

### So we generate it — carefully

This is our biggest risk. Judges are railway engineers; unrealistic data destroys our credibility on everything else.

**Rules for the data generator:**
- Every job's frequency comes from a published norm (USFD by GMT, tamping by the manual, OHE by the traction manual, signalling by the standard schedule)
- Due date = last done + periodicity
- Each job tagged: criticality, required block type (traffic / power / disconnection), section, duration range
- Use a **real section's real timetable** so blocks genuinely compete with real train paths
- **Write down every assumption in a table we can show**

---

## Part 12 — Things to double-check before quoting them

Being straight about what's solid and what isn't:

**Solid — quote freely:**
- The CAG figures (Report 45 of 2017, Report 22 of 2022). These come from primary government documents.
- The existence and roles of TMS, SMMS, TDMS, COA, BDMS.
- The OR-Tools capabilities.

**Verify before quoting exact numbers:**
- The precise USFD frequency table, gang strengths, and signalling schedule intervals — some of these came from study-guide sites reproducing the manuals. Check against the primary RDSO USFD Manual (Revised 2012) and the IRPWM.

**Treat as claims, not facts:**
- The "2,500 → 250 rail fractures" figure is a ministerial statement.

**Couldn't verify:**
- The exact internal BDMS approval chain and screens (the CRIS manual isn't publicly accessible).
- The circular number for the 2016 corridor-block document.

If a judge asks about something in this last group, **say we couldn't verify it from a primary source.** That's a strong answer, not a weak one.

---

## Part 13 — The one-paragraph summary

Three railway departments each shut down track for maintenance separately and by hand, which wastes track time and delays trains — the CAG found roughly half the requested maintenance time never materialises, and a third of what is granted goes unused. We're building a system where machine learning estimates how long jobs take and how urgent they are, and a constraint optimiser then decides which jobs run when and which can share a single shutdown window across departments. Our differentiator is that we can prove how good our schedule is, we can show the improvement against the government's own audited baseline, and we handle the multi-department coordination that published research mostly ignores.

---

## First three things to do

1. **Everyone reads Parts 1–4 of this document.** The vocabulary and the ML-vs-optimisation split are non-negotiable shared knowledge.
2. **Someone starts the OR-Tools scheduling tutorial this week.** Ideally two people.
3. **Someone confirms with our SPOC** that we can block SIH26027, and checks the idea-count on the portal in early September to see how crowded it's getting.
