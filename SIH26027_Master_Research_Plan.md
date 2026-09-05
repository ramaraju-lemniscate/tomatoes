# SIH26027 — MASTER RESEARCH PLAN
### A structured tree of every direction to investigate before we commit to a build

**Purpose:** decide *what to build* through evidence, not assumption.
**Method:** every branch gets an owner, a time-box, named sources, and a written verdict — **why it's good AND why it's not**.
**Rule:** no branch is closed until someone has written the verdict down.

---

## HOW TO READ THIS

```
BRANCH  →  the question we're answering
  ├── sub-branch
  │     SOURCES: where to look
  │     VERDICT NEEDED: what we must be able to say when done
  │     OWNER: ______   TIME-BOX: ___
```

Every branch produces one page maximum. If a branch takes longer than its time-box, stop and write down what you know.

---

## ⚠ CORRECTION BEFORE WE START

**This is not primarily a machine learning project.** The core is **constrained optimisation**. ML is a supporting layer that estimates parameters (how long a job takes, how urgent it is) which then feed the optimiser.

Why this matters: a learned model that outputs schedules will produce plans that violate hard safety rules — two closures on one track, a block over a live train path. A railway cannot run a schedule that is *probably* conflict-free.

**The architecture we are testing:**
```
data → [ML: estimate duration & criticality] → [OPTIMISER: decide schedule] → plan
```
Every branch below either supports, challenges, or replaces some part of that.

---

# BRANCH A — DO WE EVEN HAVE THE PROBLEM RIGHT?
*Owner: ______  ·  Time-box: 4 days  ·  Do this first, everything depends on it*

```
A ── Problem understanding
  │
  ├── A1  Indian Railways domain reality
  │     What is a block, corridor block, possession, disconnection?
  │     How does a block request actually flow from request to grant?
  │     What do TMS / SMMS / TDMS / COA / BDMS each hold?
  │     SOURCES:
  │       • IRPWM — indianrailways.gov.in/railwayboard/uploads/codesmanual/IRPWM/
  │         (Para 226 = block duration norms, Para 202 = route classes, Para 213 = gang strength)
  │       • IRTMM 2019 — indianrailways.gov.in/railwayboard/uploads/codesmanual/IRTMM-RDSO/TMM_Ch/
  │         (Ch 5.2 = block norms, Ch 2 = machine output rates)
  │       • RDSO USFD Manual Rev 2012 — iricen.gov.in/iricen/Track_Manuals/USFD/usfd_new.pdf
  │       • Signal Engineering Manual — indianrailways.gov.in (SEM Part I & II)
  │       • AC Traction Manual Vol II (ACTM) — RDSO
  │       • IRICEN publications — iricen.gov.in
  │     VERDICT NEEDED: a one-page glossary we can all speak fluently
  │
  ├── A2  The audited baseline — our "before" number
  │     SOURCES:
  │       • CAG Report No. 45 of 2017, Chapter 3 (cag.gov.in)
  │         → Table 21: 58,342 hrs demanded vs 29,411 granted = 49.6% shortfall
  │         → 33% of granted block time unutilised; 46% of loss = machine travelling
  │       • CAG Report No. 22 of 2022 (Derailments)
  │         → idling: 32% blocks not given, 30% blocks not planned, 19% operational
  │       • CAG Report No. 22 of 2021 (Railways — timetabling/punctuality)
  │       • Parliament: sansad.in, eparlib.nic.in, prsindia.org — search "corridor block",
  │         "track machine utilisation", "maintenance block"
  │       • PIB — pib.gov.in (PRID 1863836 = the corridor block policy release)
  │     VERDICT NEEDED: a parameters file with every citable number and its source
  │
  ├── A3  The policy timeline — GET THIS RIGHT
  │     The PS says "2018". Research says the 3-hour corridor was announced 2020
  │     (Railway Board Chairman V K Yadav) and implemented in the All-India Time Table
  │     from 1 October 2022 (PIB PRID 1863836). The ≥4hr-or-2×2.5hr engineering norm
  │     is separate and older — IRTMM Ch 5.2 / IRPWM Para 226.
  │     VERDICT NEEDED: a correct, citable timeline. A judge will notice if we get this wrong.
  │
  └── A4  Who judges this and what do they reward?
        SOURCES: SIH past winner writeups, SIH evaluation guidelines,
                 our SPOC, seniors who've competed
        VERDICT NEEDED: what to lead the pitch with
```

---

# BRANCH B — WHAT CLASS OF PROBLEM IS THIS?
*Owner: ______  ·  Time-box: 5 days  ·  This decides the whole model*

```
B ── Problem formulation
  │
  ├── B1  Railway possession scheduling literature (the direct match)
  │     SOURCES:
  │       • Lidén, T. (2015) "Railway Infrastructure Maintenance — A Survey of
  │         Planning Problems and Conducted Research", Transportation Research Procedia 10
  │         ← START HERE. The canonical survey.
  │       • Budai, Huisman & Dekker (2006) "Scheduling preventive railway maintenance
  │         activities", JORS 57(9). Free working paper: Erasmus repository (EI 2004-41)
  │         ← The closest textbook match to our problem
  │       • Lidén & Joborn (2017), Transportation Research Part C — integrated MILP
  │       • Lidén & Joborn (2020), Public Transport — the 11–17% saving result
  │       • Sedghi et al. (2021) — newer decision-support review
  │     VERDICT NEEDED: which published formulation is closest to ours, and what it misses
  │
  ├── B2  RCPSP — the abstract form of our problem
  │     Resource-Constrained Project Scheduling. Our problem IS a variant of this.
  │     SOURCES: PSPLIB (om-db.wi.tum.de/psplib/), MMLIB (projectmanagement.ugent.be),
  │              Ghent OR&S group publications
  │     VERDICT NEEDED: which RCPSP variant fits (multi-mode? time-windowed? calendar?)
  │
  ├── B3  Job-shop / machine scheduling analogues
  │     Sections = machines, tasks = jobs, blocks = time windows
  │     SOURCES: PyJobShop docs, OR-Tools scheduling.md
  │     VERDICT NEEDED: does this framing simplify or distort our problem?
  │
  ├── B4  Integrated timetabling + maintenance
  │     SOURCES:
  │       • INFORMS RAS Problem Solving Competition 2016 papers
  │       • van Aken et al. (2017) — Train Timetable Adjustment under possessions
  │       • Vansteenwegen et al. (2016)
  │     VERDICT NEEDED: do we adjust the timetable, or treat it as fixed? (Probably fixed —
  │     but we must be able to say why)
  │
  └── B5  Multi-department bundling — is there ANY published model?
        This is our claimed novelty. We must check nobody has done it.
        SOURCES: Google Scholar "multi-disciplinary possession", "combined possession",
                 "work bundling railway maintenance"; Scopus; the Lidén survey's references
        VERDICT NEEDED: an honest statement of what exists and what doesn't
```

---

# BRANCH C — THE ALGORITHM TREE
*Owner: split across 2–3 people  ·  Time-box: 7 days  ·  The biggest branch*

**For every leaf below, write: (1) what it is, (2) why it could work, (3) why it might not, (4) can we build it in 8 weeks, (5) can we explain it to a railway engineer.**

```
C ── Solution methods
  │
  ├── C1  EXACT METHODS (guarantee optimality or a proven bound)
  │   │
  │   ├── C1.1  Mixed Integer Linear Programming (MILP)
  │   │       TOOLS: HiGHS (free, fast), CBC (free), SCIP (free, academic),
  │   │              Gurobi (free academic licence), CPLEX (free community edition,
  │   │              limited to 1000 vars — too small for us)
  │   │       PYTHON: PuLP, Pyomo, python-mip
  │   │       ✓ Mature, well-understood, provable bounds
  │   │       ✗ Time-indexed formulations explode; scheduling is awkward in pure LP
  │   │
  │   ├── C1.2  Constraint Programming — CP-SAT  ← CURRENT FAVOURITE
  │   │       TOOLS: Google OR-Tools CP-SAT (free), IBM CP Optimizer (commercial),
  │   │              MiniZinc (modelling language, multiple backends)
  │   │       DOCS: github.com/google/or-tools → ortools/sat/docs/scheduling.md
  │   │             CP-SAT Primer by Dominik Krupke (github.com/d-krupke/cpsat-primer)
  │   │       KEY FEATURES: NewIntervalVar, NewOptionalIntervalVar, AddNoOverlap,
  │   │                     AddCumulative, ObjectiveValue + BestObjectiveBound
  │   │       ✓ Built for scheduling; handles optional intervals natively;
  │   │         reports optimality gap; free; won MiniZinc challenge repeatedly
  │   │       ✗ Needs integer data; can stall on very large instances
  │   │
  │   ├── C1.3  SAT / MaxSAT
  │   │       PRECEDENT: Reisch, Großmann & Weiß (2026), Journal of Rail Transport
  │   │       Planning & Management Vol 37 art. 100569 — DB InfraGO-funded, whole-Germany
  │   │       model, 5,841 demands, 94.62% coverage, beat a MIP solver by 6 points
  │   │       TOOLS: Loandra, RC2, PySAT
  │   │       ✓ Proven at national scale on exactly our problem
  │   │       ✗ Harder to model; less accessible; already published (no novelty here)
  │   │
  │   └── C1.4  Decomposition methods
  │           Benders decomposition, column generation, Lagrangian relaxation,
  │           rolling horizon
  │           ✓ How you scale to a whole zone
  │           ✗ Significant complexity; probably beyond 8 weeks
  │
  ├── C2  HEURISTICS & METAHEURISTICS (fast, no guarantee)
  │   │
  │   ├── C2.1  Greedy / priority rules  ← WE MUST BUILD THIS ANYWAY
  │   │       This is our BASELINE. It's what every competing team will submit as
  │   │       their solution. We build it to beat it.
  │   │       ✓ Trivial to implement; essential for comparison
  │   │       ✗ No quality guarantee — and this is exactly what we differentiate against
  │   │
  │   ├── C2.2  Genetic Algorithms
  │   │       PRECEDENT: Khosravi et al., Heliyon (2024) — Trafikverket tamping,
  │   │       MILP + GA on the Swedish Main Western Line
  │   │       ✓ Handles messy objectives; well-known
  │   │       ✗ No bound; hard to prove quality; parameter tuning eats time
  │   │
  │   ├── C2.3  Simulated Annealing / Tabu Search
  │   │       ✓ Simple, robust
  │   │       ✗ Same problem: no provable bound, which is our main differentiator
  │   │
  │   └── C2.4  Large Neighbourhood Search (LNS / ALNS)
  │           NOTE: CP-SAT already uses LNS internally — worth understanding
  │           ✓ Very strong on scheduling; hybridises well with CP
  │           ✗ Extra implementation on top of CP-SAT for uncertain gain
  │
  ├── C3  LEARNING-BASED METHODS  ← RESEARCH THESE TO REJECT THEM WELL
  │   │
  │   │   Important: we research these so we can say in the pitch
  │   │   "we evaluated X and rejected it because Y". That is a strength, not a gap.
  │   │
  │   ├── C3.1  Deep Reinforcement Learning for scheduling
  │   │       KEY RESOURCE: **Flatland** — github.com/flatland-association/flatland-rl
  │   │         Built by SBB, Deutsche Bahn, SNCF and AIcrowd. A railway RL environment.
  │   │         NeurIPS 2020 challenge, 700+ participants from 51 countries.
  │   │       CRITICAL FINDING TO CITE: the winning NeurIPS 2020 entry
  │   │         (github.com/Jiaoyang-Li/Flatland) used **multi-agent path finding —
  │   │         a planning/search method — and outperformed ALL reinforcement
  │   │         learning entries in both tracks.**
  │   │       ✓ Fashionable; SBB/DB have prototyped with it
  │   │       ✗ Cannot guarantee hard constraint satisfaction; needs training data
  │   │         we don't have; not explainable to a railway engineer; and on the
  │   │         railway benchmark built for it, classical search won
  │   │       → This is our strongest "we considered and rejected" argument
  │   │
  │   ├── C3.2  Graph Neural Networks for scheduling
  │   │       SOURCES: github.com/topics/job-shop-scheduling (GNN + JSSP repos)
  │   │       ✓ Natural fit for network structure
  │   │       ✗ Same constraint-guarantee problem; heavy data need
  │   │
  │   ├── C3.3  ML-guided search (learning to branch, warm starts)
  │   │       ✓ Keeps optimality guarantees, uses ML to speed search
  │   │       ✗ Advanced; only worth it if CP-SAT is too slow
  │   │
  │   └── C3.4  Neural combinatorial optimisation
  │           ✗ Research-stage; not appropriate here
  │
  └── C4  HYBRID  ← WHERE OUR NOVELTY LIVES
      │
      ├── C4.1  ML estimates parameters → optimiser decides   ← OUR ARCHITECTURE
      │       ML predicts task duration and criticality/risk.
      │       CP-SAT makes the scheduling decision.
      │       ✓ Answers the PS's "AI/ML" wording honestly; matches how ProRail,
      │         Trafikverket and Network Rail actually operate; keeps hard
      │         constraints exact; explainable
      │       ✗ ML layer must demonstrably improve decisions or it's decoration
      │
      ├── C4.2  Matheuristics (optimiser inside a heuristic loop)
      ├── C4.3  Rolling horizon with freeze window (borrowed from Network Rail T-26/T-7)
      └── C4.4  Two-stage: monthly plan fixes big possessions → weekly fills detail
              (this is how ProRail actually works)
```

---

# BRANCH D — THE ML LAYER (scoped, not central)
*Owner: ______  ·  Time-box: 4 days*

```
D ── What ML actually does here
  │
  ├── D1  Task duration prediction
  │     Regression on tabular features (activity type, section length, machine,
  │     season, gang size)
  │     ANCHOR: IRTMM Ch 2 gives stipulated outputs (Duomatic 08-32 = 800 m/effective hr;
  │     turnout tamper ≈ 1 turnout/hr; PSC turnout deep screening = 4.0–5.5 hrs)
  │     METHODS: gradient boosting (XGBoost/LightGBM), random forest, linear baseline
  │     ✗ Honest risk: on synthetic data, ML just relearns our generator. Say so openly.
  │
  ├── D2  Criticality / risk scoring
  │     Ranking or classification. Anchor to Indian defect classes (IMR marking,
  │     USFD defect classes), overdue days, GMT, traffic density
  │
  ├── D3  Degradation modelling / Remaining Useful Life
  │     Track geometry degradation, survival analysis, Kalman filtering
  │     SOURCES: track geometry degradation ML reviews; tamping effectiveness regression
  │     ✗ Probably out of scope for 8 weeks — but worth knowing it exists
  │
  └── D4  The honest question
        Does the ML layer measurably improve scheduling decisions?
        If not → replace with a transparent rule-based score and SAY SO.
        A defensible rule beats an opaque model that adds nothing.
```

---

# BRANCH E — INTERNATIONAL PRACTICE
*Owner: ______  ·  Time-box: 5 days  ·  Partly done already*

```
E ── How other countries do this
  │
  ├── E1  ALREADY RESEARCHED (see international benchmarking doc)
  │     • ProRail (NL) — bundling, 9h vs 4h train-free periods, DONNA, cyclic 4-week grids
  │     • Trafikverket (SE) — underhållsfönster; windows allocated BEFORE train paths;
  │       11–17% saving from integrated planning; BUT only 34% window utilisation
  │     • Network Rail (UK) — T-26 CPPP, T-7 freeze, Engineering Access Statement,
  │       Schedule 4 compensation pricing disruption
  │     • DB InfraGO (DE) — Generalsanierung full-closure corridors; Riedbahn 5-month
  │       closure cut infrastructure disruptions ~27% in 3 months
  │
  ├── E2  STILL TO RESEARCH — China  ← genuinely relevant to us
  │     The "skylight" (天窗) maintenance window system.
  │     KNOWN: skylight time is generally arranged between 00:00 and 06:00, and once
  │     fixed on the train diagram the skylight cannot be moved at will.
  │     RECENT: Gao, Wang & Tie (2025), Transportation Research Interdisciplinary
  │     Perspectives Vol 34, 101741 — proposes **spatial-segmented** vs
  │     **temporal-divided** maintenance windows for overnight operation.
  │     ← This is directly useful: it's about running trains AND maintaining
  │       simultaneously, which is exactly India's problem.
  │     SOURCES: Beijing Jiaotong University publications, China Academy of Railway
  │              Sciences, DOAJ, arXiv
  │
  ├── E3  STILL TO RESEARCH — Russia (RZD)
  │     ⚠ Honest warning: Russian railway OR literature is poorly indexed in English
  │     and may not be accessible. Time-box this to 1 day. If nothing solid surfaces,
  │     write "not found" — do not fabricate.
  │
  ├── E4  Japan (JR) — structural contrast, not template
  │     Shinkansen gets a nightly traffic-free window ~00:00–06:00 by design.
  │     India does not. Use as contrast in the pitch.
  │
  └── E5  VERDICT NEEDED for the whole branch
        A table: principle | country | evidence | transfers to India? | why/why not
        Then: each transferable principle → the exact CP-SAT constraint it becomes
```

---

# BRANCH F — OPEN SOURCE LANDSCAPE
*Owner: ______  ·  Time-box: 4 days  ·  What can we stand on?*

```
F ── Code we don't have to write
  │
  ├── F1  Solvers
  │     • OR-Tools — github.com/google/or-tools (Apache 2.0)
  │     • HiGHS — github.com/ERGO-Code/HiGHS (MIT)
  │     • SCIP — scipopt.org
  │     • CBC — github.com/coin-or/Cbc
  │     • MiniZinc — minizinc.org
  │
  ├── F2  Scheduling libraries  ← check these BEFORE writing our own model
  │     • **PyJobShop** — github.com/PyJobShop/PyJobShop
  │       Python CP scheduling library, uses CP-SAT as default backend.
  │       Supports: release dates, deadlines, due dates, multiple modes,
  │       sequence-dependent setup times, breaks, optional task selection,
  │       arbitrary precedence. Tested on 9,000+ benchmark instances.
  │       Paper: Lan & Berkhout (2025), arXiv:2502.13483
  │       ← "Optional task selection" + "setup times" + "breaks" map almost
  │         exactly onto our problem. INVESTIGATE SERIOUSLY.
  │     • JobShopLib and others — github.com/topics/job-shop-scheduling
  │     • Timefold / OptaPlanner (Java) — has a maintenance-scheduling tag
  │
  ├── F3  Railway-specific open source
  │     • **Flatland** — github.com/flatland-association/flatland-rl
  │       (SBB + DB + SNCF railway simulation environment)
  │     • Winning MAPF solution — github.com/Jiaoyang-Li/Flatland
  │     • RailML — railml.org (data exchange standard for railway data)
  │     • OpenRailwayMap — openrailwaymap.org
  │
  ├── F4  Benchmark instances  ← to validate our solver BEFORE real data
  │     • PSPLIB — om-db.wi.tum.de/psplib/ (J30/J60/J90/J120 RCPSP instances
  │       with known optima)
  │     • MMLIB — projectmanagement.ugent.be (multi-mode RCPSP)
  │     • MIPLIB — miplib.zib.de
  │     • Parser: github.com/PyJobShop/PSPLIB
  │     MILESTONE: solve PSPLIB J30 to proven optimality with our CP-SAT harness
  │     before feeding it any railway data.
  │
  └── F5  Search these places systematically
        • github.com/topics/job-shop-scheduling
        • github.com/topics/maintenance-scheduling
        • github.com/topics/constraint-programming
        • Papers With Code — paperswithcode.com
        • arXiv cs.AI / math.OC / eess.SY
        • Google Scholar citation-forward from Lidén (2015)
```

---

# BRANCH G — NOVELTY POSITIONING
*Owner: ______  ·  Time-box: 3 days  ·  Do AFTER B, C, E*

```
G ── What can we honestly claim?
  │
  ├── G1  What's already published (so we DON'T claim it)
  │     ✗ "We used CP-SAT for maintenance scheduling" — done (DB 2026, ProRail 2022)
  │     ✗ "We integrated maintenance with timetabling" — literature consensus since 2017
  │     ✗ "We built a national optimiser" — ProRail did, at scale
  │
  ├── G2  What appears genuinely open
  │     ✓ Cross-departmental bundling (Engineering + S&T + Traction in ONE possession).
  │       Published models mostly treat disciplines separately.
  │     ✓ The Indian corridor-block policy as a hard structural constraint
  │     ✓ Criticality grounded in Indian defect classification
  │     ✓ Benchmarking against a CAG-audited baseline
  │     ✓ No compensation market → internal shadow price for disruption
  │
  └── G3  VERDICT NEEDED
        One paragraph we can say out loud that is true, specific, and defensible.
```

---

# BRANCH H — EVALUATION & VALIDATION
*Owner: ______  ·  Time-box: 3 days  ·  Decide this BEFORE building*

```
H ── How will we know it works?
  │
  ├── H1  Baselines we must beat
  │     • Greedy first-fit (what competitors will submit)
  │     • Simulated "current process" reproducing the CAG ~50% shortfall
  │
  ├── H2  Metrics
  │     • Corridor utilisation % (Sweden's 34% is our cautionary benchmark)
  │     • Block hours granted vs demanded
  │     • Tasks completed on time / overdue cleared
  │     • Asset downtime
  │     • Optimality gap
  │     • Solve time
  │
  ├── H3  Solver validation
  │     PSPLIB J30 to proven optimality before touching railway data
  │
  └── H4  Domain validation
        Show synthetic data + results to a serving railway officer.
        Ask what looks wrong. This is the highest-value single hour available to us.
```

---

# 📁 DOWNLOAD MANIFEST
### Everything to pull into the project folder now

```
project/
├── data/
│   ├── network/
│   │   ├── india-latest.osm.pbf        ← download.geofabrik.de/asia/india.html  [ODbL]
│   │   └── datameet-railways/          ← github.com/datameet/railways           [CC0]
│   │         stations.json, trains.json, schedules.json
│   ├── timetable/
│   │   └── data.gov.in train timetable ← data.gov.in/catalog/indian-railways-train-time-table
│   │                                      [GODL-India, needs free API key]
│   └── benchmarks/
│       ├── psplib/                     ← om-db.wi.tum.de/psplib/
│       └── mmlib/                      ← projectmanagement.ugent.be
│
├── manuals/                            ← all public, all downloadable
│   ├── IRPWM.pdf                       ← iricen.gov.in/iricen/Track_Manuals/IRPWM.pdf
│   ├── IRTMM/                          ← indianrailways.gov.in (Ch 2 + Ch 5.2 essential)
│   ├── USFD_Manual_2012.pdf            ← iricen.gov.in/iricen/Track_Manuals/USFD/usfd_new.pdf
│   ├── SEM_Part1.pdf                   ← indianrailways.gov.in
│   ├── ACTM_Vol2.pdf                   ← RDSO
│   └── IRSTMM.pdf                      ← indianrailways.gov.in
│
├── audits/                             ← our baseline evidence
│   ├── CAG_45_2017_Ch3.pdf             ← cag.gov.in
│   ├── CAG_22_2022_Derailments.pdf     ← cag.gov.in
│   ├── CAG_22_2021_Railways.pdf        ← cag.gov.in
│   └── IR_Year_Book_2023-24.pdf        ← rdso.indianrailways.gov.in
│
├── papers/
│   ├── Liden_2015_survey.pdf
│   ├── Budai_Huisman_Dekker_2006.pdf   ← free via Erasmus repository
│   ├── Liden_Joborn_2020_PublicTransport.pdf
│   ├── Reisch_2026_MaxSAT_JRTPM.pdf
│   ├── Oudshoorn_2022_ProRail.pdf
│   ├── Gao_2025_China_maintenance_windows.pdf
│   ├── PyJobShop_arXiv_2502.13483.pdf
│   └── Flatland_arXiv_2012.05893.pdf
│
├── code_refs/                          ← clone, don't reinvent
│   ├── or-tools/                       ← github.com/google/or-tools
│   ├── cpsat-primer/                   ← github.com/d-krupke/cpsat-primer
│   ├── PyJobShop/                       ← github.com/PyJobShop/PyJobShop
│   └── flatland-rl/                    ← github.com/flatland-association/flatland-rl
│
└── notes/
    ├── glossary.md                     ← Branch A1 output
    ├── parameters.md                   ← every citable number + source
    ├── verdicts/                       ← one file per branch
    └── attribution.md                  ← licence tracking (ODbL, CC0, GODL all differ)
```

**Licence note:** datameet = CC0 (do anything), data.gov.in = GODL-India (attribution required), OSM = ODbL (attribution + share-alike). Keep `attribution.md` current — it takes five minutes and protects us.

---

# SUGGESTED SEQUENCING

| Phase | Days | Branches | Gate to pass |
|---|---|---|---|
| **1. Ground truth** | 1–4 | A | Glossary written, baseline numbers in a file |
| **2. Parallel deep dive** | 5–12 | B, C, E, F simultaneously | Every leaf has a written verdict |
| **3. Converge** | 13–15 | G, H | Novelty paragraph + evaluation plan agreed |
| **4. Decide** | 16 | — | Architecture locked, then build |

**Hard rule:** research stops on day 16 regardless of completeness. An unfinished branch with an honest "we didn't get to this" beats a finished research phase and no prototype.

---

# QUESTIONS I NEED ANSWERED

1. **How many people, and what can each actually do?** The tree assumes 3–5 people working in parallel. With 2, we cut Branches C3, D3 and E3 entirely.

2. **How long until the internal hackathon?** That's the real deadline, not the 20 September idea submission. It sets whether we get 16 days of research or 5.

3. **Is SIH26027 locked, or still provisional?** If provisional, Branch A comes first and we hold everything else until the choice is final.
