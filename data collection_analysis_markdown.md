# Data Availability Audit for SIH26027: AI-Powered Automatic Block Planning (Indian Railways)

## TL;DR
- **You can build a credible prototype today entirely from public sources plus synthetic data**: the network graph (OpenStreetMap/OpenRailwayMap + datameet GeoJSON, CC0), a passenger timetable (data.gov.in, GODL-India licence), the published maintenance norms (IRPWM, IRTMM, USFD Manual, ACTM, SEM — all downloadable), and a hard "before" baseline of block demand-vs-grant from CAG Report No. 45 of 2017. The core operational inputs your optimiser actually schedules against — pending task lists, defect records, historical BDMS block requests, gang/machine rosters — **do not exist publicly and must be synthesised**, anchored to those published norms.
- **The single most useful public fact for your problem** is CAG's finding that across selected divisions only ~50% of demanded block hours were granted (as low as 40% in NCR/Allahabad; 71.65% shortfall in SER/Chakradharpur), and that ~33% of granted block time was lost, 46% of it to machine travelling/shifting — this is your baseline pain, your objective function's justification, and your validation target all at once.
- **Ask relatives only for non-sensitive, aggregate facts, never for documents**: the Working Time Table (WTT), Master Chart, BDMS records and TMS/SMMS/TDMS extracts are internal/restricted and a student should not request them. Instead ask factual questions ("what time is the corridor block on your section", "how long does a tamper take to reach site") that yield usable numbers without any document changing hands.

## Key Findings

**1. Public data is strong for structure, weak for operations.** You can fully assemble the *static* world — track topology, electrification, gauge, stations, zones, a passenger timetable, and every relevant maintenance periodicity/output norm — from open sources. What you cannot get publicly is any *dynamic operational* data: no pending-work lists, no defect logs, no block-request history, no rosters. This is the correct dividing line for your architecture: **download the structure, synthesise the operations, validate against CAG aggregates.**

**2. The problem statement's own premise is documented and quantified in public audits.** CAG Report No. 45 of 2017 provides division-level block-demand-vs-grant tables and machine-idling breakdowns; CAG Report No. 22 of 2022 (Derailments) gives the track-machine idling reason split (blocks not given by Operating 32%, blocks not planned by Divisions 30%, operational problems 19%, staff 5%, no scope 3%). These are gold for parameterising and justifying your model.

**3. The "3-hour corridor block" is a 2020–2022 policy, not 2018 — correct this in your submission.** The daily fixed 3-hour maintenance corridor was announced by Railway Board Chairman V K Yadav in 2020 ("There will be a maintenance corridor for a fixed three-hour period as the system is practised in international railways") as part of the zero-based timetable, and implemented in the All-India Time Table from 1 October 2022. PIB release PRID 1863836 (30 Sep 2022) states verbatim: "it has been planned to ensure provision of fixed corridor blocks. The duration of these corridors blocks will be from 3 hours in each section." The older engineering norm — one block of ≥4 hours or two blocks of 2½ hours — comes from the Indian Railways Track Machine Manual Chapter 5.2 and is echoed in IRPWM Para 226. Your problem statement's "2018" framing appears to be a minor inaccuracy; anchor to the PIB 2022 statement and IRTMM 5.2 instead.

**4. No official GTFS/GTFS-RT exists for Indian Railways.** There is no official or unofficial GTFS-realtime feed for IR in the Mobility Database or Transitland. Real-time running exists only via NTES (no public API, scraping discouraged). Plan around a static timetable + synthetic real-time.

**5. Foreign railways give you exactly one thing IR does not publish: real possession data — and even they mostly don't.** UK Network Rail's possession data (PPS) is *not* open, but the Netherlands (Rijden de Treinen, CC0) and Switzerland (opentransportdata.swiss) publish open disruption/timetable data, and academic RCPSP/PSPLIB benchmarks let you validate your CP-SAT model before any real data exists.

## Details

### PART 1 — PUBLICLY AVAILABLE DATA

#### 1a. Train timetable data

**(a) Verified public & downloadable**
- **data.gov.in — "Indian Railways Train Time Table"** catalog (`https://www.data.gov.in/catalog/indian-railways-train-time-table`). Contains train-wise arrival/departure at each station, route, distance, source/destination. Format: CSV + API (JSON/XML). This is the ~2,810-train static schedule dump (a snapshot, historically "as on 01.11.2017" for the reservation-trains resource). **Licence: Government Open Data License – India (GODL-India)** — royalty-free, worldwide, commercial and non-commercial use permitted, requires attribution (published as Extraordinary Gazette, Feb 2017). This is the cleanest legal timetable source you have.
  - API access: register free at data.gov.in for an API key; endpoint pattern `https://api.data.gov.in/resource/{resource_id}?api-key=...&format=json`. The Python wrapper `datagovindia` (PyPI, MIT licence) simplifies discovery. Caveat: field names are inconsistent across resources; the timetable snapshot is several years old.
- **datameet/railways (GitHub)** — `https://github.com/datameet/railways`. Three GeoJSON/JSON files: `stations.json` (~8,900 stations: code, name, zone, state, lat/long), `trains.json` (train paths as LineStrings with classes, distance, timings), `schedules.json` (per-stop arrival/departure). **Licence: CC0** (public domain) — the most permissive option. Caveat: gathered ~2016, so it is stale and best used for topology/geocoding rather than current timings.
- **Kaggle mirrors** (e.g. `sripaadsrinivasan/indian-railways-dataset`, `harsh16/...`) repackage the above; check each dataset's stated licence.

**(b) Public but requires registration/scraping — approach with caution**
- **NTES (National Train Enquiry System)**, `enquiry.indianrail.gov.in/ntes` — real-time running status and schedules, built by CRIS. **No official public API.** Terms of use restrict automated access; a student team should not build on scraped NTES data for a submission. Treat live running as something to *synthesise*, not scrape.
- **Trains at a Glance / All-India Railway Time Table** — published as PDF (from 1 Oct 2022 the TAG incorporates the corridor-block policy). Human-readable, not cleanly machine-readable; use for reference, not ingestion.
- **Third-party sites (erail.in, RailYatri, Trainman, ConfirmTkt, RailRadar)** — expose schedules/running derived from IR. Their ToS generally prohibit scraping/redistribution and data provenance is unclear. **Do not rely on these for a competition submission**; prefer data.gov.in + datameet.

**(c) Freight paths — not public.** No public source exposes freight train paths/timings. Freight runs largely "on demand" without fixed public schedules (confirmed by CAG's note that goods trains run "without any scheduled timing"). Freight paths live only in the WTT/Control charts (insider/restricted). **Synthesise freight occupancy.**

#### 1b. Network & infrastructure data

**(a) Verified public & downloadable**
- **OpenStreetMap / OpenRailwayMap** — the best network geometry source. Extract via:
  - **Geofabrik India extract**: `https://download.geofabrik.de/asia/india.html` (.osm.pbf / shapefile, updated daily, ODbL licence).
  - **Overpass API** for targeted queries (e.g. `railway=rail` within a bounding box).
  - Relevant tags: `railway=rail`, `electrified` (contact_line/rail/no), `voltage`, `frequency`, `gauge` (1676 for IR broad gauge), `usage` (main/branch), `tracks`, `maxspeed`, `service` (siding/yard/crossover). **Licence: ODbL** (attribution + share-alike). Completeness for IR trunk routes is good; branch-line electrification/maxspeed tagging is patchy — sanity-check against the Year Book aggregates.
- **datameet/maps** — Indian admin/geo shapefiles, CC BY 4.0.
- **Indian Railways Year Book** (Ministry of Railways, published by CRIS/Stat & Econ Directorate) — downloadable PDFs, e.g. 2023-24 at `rdso.indianrailways.gov.in/.../IR Year Book 2023-24-English.pdf`; 2018-19 at `ircep.gov.in/files/IR-Year Book_2018-19.pdf`. The 2023-24 Year Book reports **69,181 route km total (all gauges) — BG 66,820, MG 1,159, NG 1,202** (up from 68,584 route km in 2022-23), and **62,253 electrified route km** (up from 58,074 the prior year); it also carries running/total track km, traffic density, and loco/coach/wagon counts. Also **Annual Statistical Statements** at `indianrailways.gov.in/railwayboard/view_section.jsp?id=0,1,304,366,554`. Use these to calibrate synthetic data volumes and validate OSM extracts.

**(b) Exists but access restricted / not public at the granular level**
- **Zone/division/section boundaries and block-section lists**: zone and division identities are public (Year Book, Wikipedia), but authoritative *section definitions* and *block-section lists* (the atomic units your optimiser schedules) are not published as datasets. Approximate them from OSM (station-to-station segments between block-working stations).
- **Bhuvan (ISRO)** and **Survey of India** host geospatial layers but railway-specific, machine-usable maintenance layers are limited; OSM is more practical.

**(c) Railway Board circulars** — many are published at `indianrailways.gov.in/railwayboard` (Codes & Manuals, and directorate pages). Availability of specific block-working circulars is inconsistent; the maintenance-corridor policy is best cited via the PIB release (below) and IRTMM.

#### 1c. Maintenance standards & norms (to make synthetic data credible) — all essentially public

- **Indian Railways Permanent Way Manual (IRPWM)** — official landing page `indianrailways.gov.in/railwayboard/uploads/codesmanual/IRPWM/PermanentWayManual_index.htm`; full PDF via IRICEN `iricen.gov.in/iricen/Track_Manuals/IRPWM.pdf` (2020 reprint, ACS up to No. 149; redrafted per Railway Board letter 2013/CE-II/TK/IRPWM dated 24/09/2019, merging the LWR manual). **Para 226 "Track maintenance by machines"** is the key clause: machines should get a single block of ≥4 hours or two blocks of 2½ hours; block time should be interpolated in the master chart; it is "as much the responsibility of the Operating Department as that of the Engineering Department to ensure provision of adequate time for economical working of machines." (CAG quotes Para 226 verbatim for this.) Para 202 groups BG lines into route classes A–E by speed; Para 213 covers gang strength.
- **Indian Railways Track Machine Manual (IRTMM, 2019 2nd edn)** — official HTML chapters at `indianrailways.gov.in/railwayboard/uploads/codesmanual/IRTMM-RDSO/TMM_Ch/`. **Chapter 5.2 is the block-norm source** (as stipulated by Railway Board: on single line, one ≥4-hr block or two 2½-hr blocks, or in exceptional cases min 2 hrs; on double line, one 4-hr spell or two 2½-hr split blocks or one 2½-hr block per line daily; corridor blocks "shall be incorporated in the working time-tables"). **Chapter 2 gives machine outputs**: Duomatic 08-32 stipulated output 800 m per effective hour; Unimat/PCTM point-and-crossing tamper ~1 turnout/hour; deep screening of a PSC turnout takes 4.0 hrs (RM-80-92U) or 5.5 hrs (RM-76); self-propelled speeds Duomatic 60 km/h (40 km/h in train formation), BCM RM-76 40 km/h (30 in formation). These per-hour outputs are exactly what you need to convert a task into a block duration.
- **Small Track Machine Manual (IRSTMM)** — `indianrailways.gov.in/railwayboard/uploads/codesmanual/IRSTMM/IRSTMM.pdf`.
- **RDSO Manual for Ultrasonic Testing of Rails and Welds (Revised 2012)** — full PDF at `irot.in/pdf/engg/MANUAL FOR ULTRASONIC TESTING OF RAILS AND WELDS Revised 2012 Engg.pdf` and IRICEN `iricen.gov.in/iricen/Track_Manuals/USFD/usfd_new.pdf`. Contains the **GMT-based testing-frequency table** (e.g., for main-line BG on CC+8+2t routes: GMT >24–40 → every 2½ months; >40–60 → 1½ months; >60–80 → 1 month; >80 → 20 days; loop lines 6-monthly on 'A' route to 2-yearly by GMT). This drives synthetic USFD due-dates. IRS-T-53 (Revised 2020) is the related specification at `rdso.indianrailways.gov.in/uploads/files/IRS-T-53 SPECIFICATION.pdf`.
- **Manual of AC Traction Maintenance & Operation (ACTM), Vol II** — mirrored publicly (RDSO doc ETI/OHE base, reproduced from ETI/OHE/53 June 1988). Contains OHE maintenance periodicities (Fixed Installations). Use for OHE/TRD synthetic schedules.
- **Signal Engineering Manual (SEM), Parts I & II** — official landing `indianrailways.gov.in/railwayboard/view_section.jsp?lang=0&id=0,1,304,366,544,664`; Part I PDF at `scr.indianrailways.gov.in/.../1406180239974-Signal_Engineering_Manual_1.pdf`. Contains S&T gear maintenance schedules (fortnightly/monthly/quarterly/half-yearly/yearly). Use for signalling synthetic schedules.
- **G&SR / Block Working Manual** — General & Subsidiary Rules and the zonal Block Working Manual govern block working; the Block Manual is issued by each zone's PCOM. Portions circulate publicly (IRISET materials, zonal PDFs).

#### 1d. Audit & oversight data — the "before" baseline (verified public, high value)

- **CAG Report No. 45 of 2017, "Maintenance of track on heavy traffic sections over Indian Railways"** — full report and per-chapter PDFs at `cag.gov.in`. Chapter 3 PDF: `cag.gov.in/uploads/download_audit_report/2018/Chapter_3_Utilisation_of_resources_and_infrastructure_for_track_maintenance_of_Report_No.45_of_2018_...pdf`. **Quantitative tables you can use directly:**
  - **Table 21 — block demand vs grant by division (2016-17):** NCR/Allahabad demanded 27,648 hrs, granted 10,921 (60.50% shortfall); ECR/Danapur 4,538.65 → 1,767.80 (61.05%); ECR/Mughalsarai 5,553.60 → 3,689.78 (33.56%); SWR/Hubli 13,591 → 9,942 (26.85%); SR/Chennai 1,326.15 → 1,102.19 (16.89%); SER/Kharagpur 1,042.75 → 566.50 (45.67%); SER/Chakradharpur 4,147.50 → 1,176 (71.65%); SER/Ranchi 487.33 → 238 (51.16%). **Total 58,342 demanded → 29,411 granted = 49.60% shortfall.**
  - NCR: only 40% of demanded block given; corridor block in WTT was 90–120 min vs the ≥2.5-hr norm.
  - Machine utilisation: of 6,878 machine-days in Allahabad, 2,341 unused; against an 11,717 km target only 5,041 km (43%) achieved; ~33% of block time not utilised by 25 machines; **46% of block-time loss attributable to machine travelling/shifting.** SER TMS: 19,276 machine-days available, worked 10,031, idle 9,245.
  - Manpower: track-maintainer vacancies 9–22% across the five zones (Table 18: total sanctioned 12,291, men-on-roll 10,183, 17% vacancy); SSE jurisdictions varied 16.65 km (Santragachi) to 149 km (Tamluk); 33% of small track machines found out of order.
- **CAG Report No. 22 of 2022, "Performance Audit on Derailment in Indian Railways"** — full PDF via `cag.gov.in`. **Track-machine idling reason split (verbatim):** "Idling of Track machines was noticed due to various reasons, including blocks not given by the Operating Department (32 per cent), blocks not planned by Divisions (30 per cent), operational problems (19 per cent), non-availability of staff (five per cent), and no scope of work (three per cent)." Also TRC inspection shortfalls 30–100%; **"A total of 422 derailments were attributable to the 'Engineering Department'. The primary factor ... was related to 'maintenance of track' (171 cases), followed by 'deviation of track parameters beyond permissible limits' (156 cases)"**; Operating Dept 275, Mechanical 182, Loco Pilots 154. Analysed 2017-18 to 2020-21.
- **CAG Report No. 22 of 2021 (Railways)** — timetabling/punctuality audit; notes >1 lakh signal failures/year and the lack of a prescribed yardstick for "traffic recovery/engineering allowance." Useful for justifying the WTT allowances your model needs. (Note: the "~2.5 hours" figure in this report refers to potential *travel-time savings* on NDLS–HWH, **not** a corridor-block duration — do not conflate.)
- **Parliamentary / other:** search `sansad.in`, `eparlib.nic.in`, and PRS India (`prsindia.org`) for Lok Sabha/Rajya Sabha answers on corridor blocks, track-machine utilisation, and derailments. The Zero-Based Time Table was covered in a 2020 parliamentary reply ("ensuring adequate corridor blocks for maintenance"). These yield qualitative confirmations and occasional numbers.
- **Safety statistics:** derailment counts by cause and rail-fracture time series appear in CAG 22/2022, the Year Book (Safety section), and PIB/Parliament answers.

#### 1e. Operational performance data

- **Punctuality/delay:** no official open delay dataset. NTES has live data but no API. **Academic/Kaggle** train-running datasets exist (community-scraped, variable licence/quality) — usable for ML parameter estimation with caveats, not as ground truth.
- **Section-wise line-capacity utilisation:** not published as a dataset, but CAG 45/2017 states the selected HDN sections ran at **>100% line-capacity utilisation** (a citable qualitative fact). 
- **National Rail Plan (NRP) 2030 / Vision 2024-25:** documents via PIB (`pib.gov.in`, PRIDs 1797575, 1806617) and railjournal.com summaries. Per the NRP, "The Golden Quadrilateral and its diagonals which comprise merely 16 per cent of India's rail route length carry about 52 per cent of the nation's passengers"; the plan explicitly calls for AI tools for capacity/asset planning and demand prediction (directly relevant framing for your pitch — SCR zone was tasked with AI plans with the Indian School of Business). The full NRP draft is downloadable from the Ministry of Railways.

### PART 2 — DATA THAT MUST BE SYNTHESISED

None of the following is public; generate each anchored to published norms. General approach: build the static world from Part 1, then simulate operations. Seed a reproducible RNG; document every anchor.

- **Pending maintenance task list** (task_id, department, section/block-section, activity, due_date, criticality, est_duration, resource_needs). *Anchors:* activity periodicities from IRPWM (tamping/through-packing), USFD Manual (GMT table → USFD due dates), SEM (S&T gear schedules), ACTM (OHE). *Durations:* IRTMM Chapter 2 outputs (tamping 800 m/eff.hr → duration = section length / rate + setup; turnout deep screening 4.0/5.5 hr). *Distribution:* due-dates as a renewal process per periodicity; criticality as ordinal (IMR/OBS-style for rails; overdue-days as a driver). *Sanity check an engineer applies:* a 2 km tamping task cannot fit a 90-min block at 800 m/hr — durations must be block-feasible or explicitly flagged as multi-block.
- **Defect records** (rail fractures, geometry defects TGI/CTR-style, OHE defects, signal failures). *Anchors:* rail-fracture and derailment-cause rates from CAG 22/2022 and Year Book; signal-failure magnitude (>1 lakh/yr, CAG 22/2021). *Distribution:* Poisson arrivals scaled by GMT/traffic density; severity mixture. *Sanity check:* defect rate per km per GMT should be within the same order of magnitude as published fracture statistics.
- **Historical block requests & grants** (requested_start/duration/section/dept vs granted). *Anchors:* CAG 45/2017 Table 21 grant ratios (target ~50% overall grant, 28–83% granted by division) and the reason-for-non-grant split from CAG 22/2022 (Operating 32%, not planned 30%, operational 19%…). *Distribution:* Bernoulli grant with p ≈ division grant ratio; cancellation reasons drawn from the CAG split. *Sanity check:* aggregate granted/demanded should reproduce ~50%; corridor block often 90–120 min vs 3-hr policy.
- **Gang & machine availability rosters** (machine_id, type, base depot, daily availability, gang strength). *Anchors:* IRTMM machine fleet types; gang strength 10–15 (IRPWM Para 213); MCNTM manpower formula; CAG 45/2017 vacancy 9–22% and "33% of small track machines out of order." *Sanity check:* ~1/3 machines unavailable at any time; gang cannot exceed sanctioned strength.
- **Machine travel/mobilisation times.** *Anchor:* CAG's finding that **46% of block-time loss is travelling/shifting** and machine self-propelled speeds (Duomatic 60 km/h self-propelled, 40 km/h in train formation; BCM 40/30 km/h). *Recipe:* travel time = depot-to-site distance / self-propelled speed + fixed setup/winding overhead; make mobilisation a first-class cost in the objective (this is precisely what makes your optimiser valuable). *Sanity check:* total non-working time within a block should trend toward the CAG-observed ~33% loss.

### PART 3 — DATA REQUIRING INSIDER ACCESS (what to ask relatives for)

Be conservative. The items below are internal working documents; several are restricted. **Ask for facts, not files.**

- **Working Time Table (WTT)** — the internal timetable. Contains what the public TAG does not: **freight paths, passing times at every station (not just halts), engineering/maintenance allowance, traffic recovery time, corridor-block timings, and permanent speed restrictions.** Some zones' WTTs surface unofficially online, but the WTT is generally treated as an **internal/restricted document**. **Assessment: sensitive — a student should NOT ask for the document.** Instead ask factual questions (below).
- **Master Chart / Control Chart** — the graphical time-distance chart of all train paths on a section, into which the corridor block is interpolated (IRPWM Para 226 references this). Shows conflicts and available maintenance windows. **Internal; do not request the chart.** You may ask conceptually what a corridor window on their section looks like.
- **Section line capacity / charted capacity figures** — the number of trains a section can take vs actual. Often quotable at aggregate level (e.g. "our section runs ~140% capacity"); **a single number is generally shareable**, the underlying capacity statement is not.
- **BDMS block-request records** — the actual electronic block requests/grants. **Restricted internal system data; do NOT request extracts.**
- **TMS / SMMS / TDMS extracts** — the maintenance databases. **Restricted; do NOT request extracts.** These are the "real" version of your synthetic task lists; obtaining them is neither necessary nor appropriate for a student project.

**Ethical/practical rule:** never ask an officer to export, photograph, or forward any system data or document. Ask only for (a) general process explanations and (b) round, non-attributable numbers they are comfortable stating from experience.

**Specific factual questions that yield usable numbers without any document sharing:**
- "On your section, what time is the daily corridor/maintenance block and how long is it in practice?"
- "Roughly what fraction of the blocks your engineering team asks for actually get granted?"
- "When a tamping machine is called to a site, how long does it typically take to reach and set up before it starts working?"
- "For a routine tamping run, how many metres per hour do you actually achieve inside a block (net of setup/winding)?"
- "How often is a granted block cancelled or cut short, and what's the usual reason?"
- "How many track machines does your division have, and how many are typically available on a given day?"
- "How do Engineering, S&T and OHE currently coordinate when they all want the same block?" (this validates your problem framing directly)

### PART 4 — ALTERNATIVE & PROXY DATA SOURCES

**Foreign open railway data (for structural proxies & model validation):**
- **Netherlands — Rijden de Treinen open data** (`rijdendetreinen.nl/en/open-data`): stations (CC0), **train disruptions archive with causes** (CC0), tariff/distance matrices, and a train-services archive. NS API and ProRail geodata are also open (CC0 via openOV). **Best open proxy for disruption/cause structure.**
- **Switzerland — opentransportdata.swiss** (operated by SBB for the Federal Office of Transport): GTFS timetable, real-time, and train-formation APIs. File-based data needs no registration; service APIs need a free API key (citation of opentransportdata.swiss required). GTFS `calendar_dates.txt` encodes planned construction-site service changes — a partial proxy for possession effects. Also `opendata.swiss` SBB datasets.
- **UK — Network Rail open data** (`publicdatafeeds.networkrail.co.uk`, free registration; feeds: TRUST/movements, TD, VSTP, SCHEDULE, plus CORPUS/SMART reference data; Darwin via NRDP; RTPPM). **Crucial caveat: possession data (PPS) and NROL are NOT open data** — confirmed by the Open Rail Data community and the ORR/GHD "Possessions Efficiency Review" (April 2021), which analysed PPS extracts internally. So the UK gives you excellent *timetable/performance* feeds but **not** open possession records.
- **Germany — DB** (`data.deutschebahn.com` / DB API Marketplace): timetable and station open data.
- **Bottom line on real possession data:** **no major railway publishes real, granular track-possession/maintenance-window records as open data.** The ORR/GHD report confirms UK possession data is internal. This strengthens the case that your team must synthesise — and it means your synthetic generator is itself a contribution.

**Scheduling benchmarks (to test CP-SAT before real data):**
- **PSPLIB** (`om-db.wi.tum.de/psplib/`): the standard RCPSP benchmark library — J30/J60/J90/J120 instances (30–120 activities, 4 renewable resources), with best-known/optimal makespans. Ideal first target: solve J30 to proven optimality with OR-Tools CP-SAT to validate your solver harness. Parsers: `PyJobShop/PSPLIB` (GitHub).
- **MMLIB / OR&S Ghent** (`projectmanagement.ugent.be`): multi-mode RCPSP (MM50, MM100) — good because your problem is naturally multi-mode (a task can run in different block configurations).
- **RCPSP variants with time windows / calendars** map well to maintenance windows; frame the corridor block as a resource-availability calendar and the section as a unit-capacity renewable resource (single/double line).

**Academic anonymised maintenance-scheduling datasets:** rail maintenance possession-scheduling papers (European rail OR literature) occasionally publish anonymised instances; PSPLIB/MMLIB remain the most citable and reproducible. Treat any paper dataset as validation-only.

## Recommendations

**Stage 1 — Download the static world (Day 1–2).**
1. Clone `datameet/railways` (CC0) for stations + topology. *Fallback:* OSM Overpass station query.
2. Pull the data.gov.in "Train Time Table" catalog via API key (GODL-India) for the passenger schedule. *Fallback:* datameet `schedules.json`.
3. Download the Geofabrik India OSM extract (ODbL) and filter `railway=rail` with electrified/gauge/maxspeed tags for the network graph. *Fallback:* OpenRailwayMap tiles for visual QA.
4. Download IRPWM (Para 226, 202, 213), IRTMM (Ch 2 outputs, Ch 5.2 block norms), USFD Manual 2012 (GMT table), SEM, ACTM Vol II. These are your norm anchors.
*Benchmark to proceed:* you can draw the network of one division and list activity periodicities. If OSM tagging for your chosen division is too sparse, pick a well-mapped trunk section (e.g. part of the Golden Quadrilateral).

**Stage 2 — Lock the baseline (Day 2–3).**
5. Download CAG 45/2017 (Ch 3, Table 21) and CAG 22/2022 (idling split). Extract every number into a parameters file. These become both your synthetic-generator anchors and your validation targets.
*Benchmark:* your synthetic block-grant ratio reproduces ~50% overall and the CAG idling reason split within a few points.

**Stage 3 — Validate the solver on benchmarks (Day 3–5, parallelisable).**
6. Implement the CP-SAT model against PSPLIB J30; confirm you match known optima. Then extend to multi-mode (MMLIB) and add a resource-availability calendar (the corridor block).
*Benchmark:* J30 solved to optimality; only then feed synthetic data.

**Stage 4 — Build the synthetic generator (Day 4–8).**
7. Generate task lists, defects, block history, and rosters per Part 2 recipes, each field traceable to a cited norm. Emit CSVs matching a plausible TMS/SMMS/TDMS/BDMS schema.
*Benchmark:* a domain check (ideally sanity-checked via a Part 3 factual question to a relative) shows durations are block-feasible and machine travel dominates lost time as CAG observed.

**Stage 5 — Insider sanity-check, not data extraction (ongoing).**
8. Use the Part 3 factual questions to calibrate a handful of parameters (real corridor timing, real grant ratio, real mobilisation time). Never request documents or system extracts.

**Things to skip:** scraping NTES/erail/RailYatri (ToS risk, marginal benefit over data.gov.in); chasing UK PPS possession data (not open); requesting WTT/BDMS/TMS extracts (restricted, unnecessary given synthesis works).

**What would change these recommendations:** if the SIH organisers release a dataset or a sample TMS/BDMS schema, pivot immediately to match it and downgrade synthesis to gap-filling. If a relative can state real corridor timings and grant ratios for a specific section, weight those over CAG aggregates for that section.

## Caveats
- **Timetable snapshots are stale.** The data.gov.in dump and datameet are years old; treat them as structural, not current. State this in your submission.
- **The "2018 / 3-hour" premise is imprecise.** The 3-hour corridor is a 2020 announcement (Railway Board Chairman V K Yadav) / Oct-2022 implementation (PIB PRID 1863836); the ≥4hr-or-2×2½hr engineering norm is IRTMM 5.2 / IRPWM 226. Cite these, not "2018 policy," to avoid a factual error in judging.
- **Licences differ and matter.** datameet = CC0; data.gov.in = GODL-India (attribution); OSM = ODbL (attribution + share-alike); Rijden de Treinen = CC0; opentransportdata.swiss = its own terms (citation required). Keep an attribution file.
- **Manual copies on Scribd/third-party sites** are convenient but the authoritative versions are on indianrailways.gov.in / iricen.gov.in / rdso.indianrailways.gov.in — cite the official URL.
- **No real possession data exists openly anywhere** (confirmed for UK; none found for NL/CH/DE at possession granularity). Your model structure can be validated on PSPLIB/RCPSP and on foreign *disruption* data, but end-to-end validation against real IR block outcomes is impossible pre-competition — be transparent that CAG aggregates are your only real-world anchor.
- **Do not overstate NTES/third-party legality.** Scraping is discouraged and potentially against ToS; keep the prototype on clearly-licensed data.
- **Freight is the biggest synthetic unknown.** Because freight paths are non-public and partly unscheduled, freight occupancy is your least-anchored synthetic element; flag its uncertainty in results.