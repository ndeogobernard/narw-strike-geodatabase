# North Atlantic Right Whale Vessel-Strike Risk — Project Overview

**Owner:** Bernard Issifu
**Repository:** [ndeogobernard/narw-strike-geodatabase](https://github.com/ndeogobernard/narw-strike-geodatabase)
**Portfolio:** [ndeogobernard.github.io/ndeogo](https://ndeogobernard.github.io/ndeogo/)
**Document version:** 1.0 — 23 September 2026

This is a plain-language summary of the project: what we're building, the data behind it, the analysis, and what it produces. The full technical scope is in the project specification.

---

## Contents

1. [The big picture](#1-the-big-picture)
2. [Data sources](#2-data-sources)
3. [The geodatabase](#3-the-geodatabase-standalone-portfolio-piece)
4. [Data quality checks and metadata](#4-data-quality-checks-qaqc-and-metadata)
5. [The analysis: five phases](#5-the-analysis-five-phases)
6. [Automation: the ArcPy toolbox](#6-automation-the-arcpy-toolbox)
7. [Maps and decision-support products](#7-maps-and-decision-support-products)
8. [Technical report and project management](#8-technical-report-and-project-management)
9. [Timeline](#9-timeline-10-weeks-part-time)
10. [Deliverables checklist](#10-deliverables-checklist)
11. [What ends up on the portfolio](#11-what-ends-up-on-the-portfolio)
12. [Who does what](#12-who-does-what)

---

## 1. The big picture

### The problem

About 370 North Atlantic right whales (*Eubalaena glacialis*) are left. The species is listed as endangered under the Endangered Species Act (ESA) and depleted under the Marine Mammal Protection Act (MMPA). Being hit by ships is one of the main things killing them.

NOAA Fisheries reduces this risk in four ways:

- **The vessel speed rule** (50 CFR 224.105) sets a 10-knot limit for vessels 65 ft and longer inside **Seasonal Management Areas (SMAs)** during their active months.
- **Dynamic Management Areas / Slow Zones** are voluntary slow-downs announced when whales are seen or heard.
- **Critical habitat** protects Unit 1 (feeding grounds) and Unit 2 (calving grounds).
- **Section 7 consultations** review federal actions such as offshore wind development.

### The question we're answering

> **Where and when do North Atlantic right whales and vessel traffic overlap along the U.S. Atlantic coast; how well do ships comply with the 10-knot limit inside Seasonal Management Areas by area, month, and vessel class; and where would changed or new management areas reduce vessel-strike risk the most?**

In short:

1. **Where and when** do whales and ships overlap?
2. **How well do ships obey** the 10-knot rule?
3. **Where would new or changed management areas** reduce strike risk the most?

### Why this project

The project is built to line up with a NOAA protected-resources GIS Analyst job description. Each piece demonstrates a skill in that role: geodatabase design, data evaluation, QA/QC, FGDC/ISO metadata, spatial analysis, ArcPy/ModelBuilder automation, cartography, ArcGIS Online delivery, and ESA/MMPA regulatory context.

It produces **two portfolio projects**:

- **A. Flagship:** the full vessel-strike risk analysis.
- **B. Standalone:** the geodatabase design and metadata work.

### Scope

| | |
|---|---|
| **Study area** | U.S. Atlantic coast from the Florida–Georgia calving grounds to the Gulf of Maine and Georges Bank, from the shoreline out to the 200 nm EEZ. Analysis focuses within 50 nm of shore. |
| **Analysis grid** | Hexagons: 25 km² coast-wide, and 4 km² inside and within 10 km of management areas |
| **Ship traffic period** | 2022–2024 (3 full years) |
| **Whale data period** | 2010–present for presence surfaces; 2022–2024 for comparing whales with ships |
| **Coordinate systems** | Storage: NAD 83 (EPSG:4269) · Analysis: NAD 1983 Contiguous USA Albers (EPSG:5070) · Web: Web Mercator (EPSG:3857) |

### Out of scope

- Absolute strike probabilities or death estimates. The project produces a *relative* risk index.
- Entanglement risk.
- Any use of restricted data beyond its stated terms.

---

## 2. Data sources

We gather 16 datasets. Each one is logged in the `DataSourceRegistry` table and rated before use.

| ID | Group | Dataset | Provider | What it's for |
|---|---|---|---|---|
| S01 | Whales | Right whale sightings (aerial, shipboard, opportunistic) | NOAA Right Whale Sighting Advisory System; WhaleMap; NARW Consortium (access by request) | Where whales have been seen |
| S02 | Whales | Acoustic detections (underwater recorders hearing whale calls) | NOAA Passive Acoustic Cetacean Map (PACM) | Presence where no one is watching, including at night and in winter |
| S03 | Whales | Density models (predicted whales per km², by month) | Duke University / NOAA (Roberts et al.) via OBIS-SEAMAP | Fills gaps where surveys didn't go |
| S04 | Whales | OBIS-SEAMAP observations | OBIS-SEAMAP | Cross-check and historical records |
| S05 | Regulations | Seasonal Management Areas (SMAs) | NOAA Fisheries | Compliance and risk analysis |
| S06 | Regulations | Slow Zones / Dynamic Management Areas (archive) | NOAA Fisheries | Past management actions |
| S07 | Regulations | Critical habitat (Units 1 and 2) | NOAA Fisheries | Context, Section 7 |
| S08 | Navigation | Traffic Separation Schemes and shipping lanes | Marine Cadastre | Traffic context |
| S09 | Ships | AIS vessel positions (daily CSVs) | MarineCadastre.gov / NOAA Open Data on AWS | Traffic volume, speeds, compliance |
| S10 | Projects | Offshore wind lease and planning areas | BOEM | Development context, Section 7 |
| S11 | Projects | Ports and port approaches | Marine Cadastre / USACE | Compliance by port SMA |
| S12 | Environment | Bathymetry | NOAA Coastal Relief Model; GEBCO 2024 | Habitat context |
| S13 | Environment | Sea surface temperature (monthly) | NOAA OISST / NASA MUR via ERDDAP | Habitat context |
| S14 | Environment | Chlorophyll-a (monthly) | NASA Ocean Color via ERDDAP | Habitat context |
| S15 | Reference | Shoreline, EEZ, state boundaries, NOAA regions | NOAA, Marine Cadastre, Census TIGER | Base layers |
| S16 | Reference | Distance-to-shore raster | Derived | Filtering |

### Two things to keep in mind

- **AIS is huge.** It's tens of GB per year. We clean it with Python (pandas/DuckDB) outside ArcGIS, and only load transit summaries plus a 1% sample of points into the geodatabase.
- **Some sightings data is restricted.** Consortium data is used only under its terms and never redistributed. Public outputs show aggregated hexagons, never raw whale locations.

---

## 3. The geodatabase (standalone portfolio piece)

We design and build **`NARW_VesselStrike.gdb`** as a documented, rule-based database, with a mobile geodatabase export to demonstrate SQL.

### Organization

| Feature dataset | Contents |
|---|---|
| `Reference` | Study area, shoreline, EEZ, states, NOAA regions, hex grids, ports |
| `Biological` | Sightings, acoustic detections, recorders, monthly density model rasters |
| `Regulatory` | SMAs, Slow Zones, critical habitat, shipping lanes, candidate SMAs |
| `VesselTraffic` | AIS transits, AIS point sample |
| `Environmental` | Bathymetry, monthly SST and chlorophyll rasters |
| `Project` | Wind lease areas |
| `Analysis` | Monthly whale presence, vessel traffic and SMA compliance |
| `Results` | Monthly risk index, scenario summary |

### Tracking tables

| Table | Purpose |
|---|---|
| `DataSourceRegistry` | Where each dataset came from, its terms, date and coordinate system |
| `QAQC_Log` | Result of every quality check |
| `ProcessingLog` | Every tool run, its parameters, and the git commit |
| `VesselClassLookup` | Maps AIS vessel type codes to vessel classes |
| `MetadataStatus` | Metadata completeness for every object |
| `VersionHistory` | Archived snapshots with checksums |

### Built-in data rules

- **Domains** are pick lists and allowed ranges, for example platform = Aerial / Shipboard / Acoustic, vessel class = Cargo / Tanker / Passenger…, or speed must be 0–60 knots.
- **Subtypes** split a class into groups: sightings by platform, transits by vessel class.
- **Relationship classes** link tables, for example one SMA to its many transits, or one recorder to its many detections.
- **Topology** catches geometry errors: regulatory polygons must not overlap within a class, and hex grids must have no gaps or overlaps.
- **Attribute rules** fill or check fields automatically, for example month and year are calculated from the date, and speed and presence values must fall within range.
- **Editor tracking** records who changed what, on all editable classes.

### Built from code

`schema/schema.yaml` plus a `BuildSchema` tool recreate the whole geodatabase on any machine. A schema-diff script confirms the delivered geodatabase matches the design.

### Documentation

ERD, data dictionary, design rationale, naming conventions, and versioned zip snapshots with SHA-256 checksums.

---

## 4. Data quality checks (QA/QC) and metadata

### Fitness-for-Use Assessment

A one-page review of each of the 16 sources. It covers:

- completeness (area, time period, survey effort)
- how current the data is and how often it's updated
- coordinate system and any conversion applied
- where the data came from (lineage)
- attribute quality and positional accuracy
- known biases and limitations
- appropriate and inappropriate uses
- a verdict: **Use / Use with caveats / Do not use**

### Automated checks (`RunQAQC` tool)

| Layer | What's checked |
|---|---|
| All | Correct coordinate system, valid geometry, allowed values, missing required fields |
| Sightings | Duplicates (within 5 min and 500 m), points on land or unrealistically far offshore, future or impossible dates |
| Acoustic | Detections outside the recorder's deployment dates |
| AIS | Speed spikes, position jumps (implied speed above 60 kn), invalid vessel IDs (MMSI), duplicate rows |
| Regulatory | Active-period fields are readable, no overlaps, boundaries spot-checked against published notices |
| Rasters | NoData handling, extent, cell size, realistic value ranges |
| Derived | Row counts reconcile within ±1% |

Every check writes its result to `QAQC_Log`, with reviewer sign-off.

### Metadata

Every feature class, raster and table gets complete **ISO 19115** and **FGDC CSDGM** metadata, structured to fit NOAA's InPort catalog conventions. Each record includes:

- title, abstract, purpose, keywords
- use and access limits
- extent in space and time
- coordinate system
- **lineage**, listing every processing step
- field definitions
- accuracy statements
- contact

The `GenerateMetadata` tool fills metadata from templates and the processing log. A validator confirms every record is **100% complete**. A metadata catalog page lists every object.

---

## 5. The analysis: five phases

### Phase A — Where are the whales? (whale presence surfaces)

- Combine sightings and acoustic detections after QA.
- Adjust sightings for survey effort where effort data exists. Otherwise treat them as presence-only.
- Run kernel density (25 km bandwidth) by month and summarize by hexagon.
- Calculate the average density-model prediction for each hexagon.
- Combine everything into a **presence index from 0 to 1**:

| Component | Weight |
|---|---|
| Sightings, adjusted for survey effort | 0.4 |
| Acoustic detections | 0.2 |
| Density model | 0.4 |

**Output:** `Hex25_WhalePresence_Monthly`, a 4 km² version near SMAs, and monthly rasters for maps.

### Phase B — Where are the ships? (AIS processing)

- Clean the raw AIS pings using the QA rules above.
- Assign each vessel a class and flag vessels **65 ft (19.8 m) and longer**, the ones the rule applies to.
- Link pings into tracks. Start a new segment after a gap of more than 30 minutes or a jump faster than 60 kn.
- Clip tracks to SMAs, Slow Zones and hexagons, accounting for whether each area was active at the time.

**Output:** `AIS_Transits` (one vessel, one day, one SMA), `Hex4_VesselTraffic_Monthly`, and a 1% point sample.

### Phase C — Are ships obeying the rule? (compliance)

| Status | Meaning |
|---|---|
| **NonCompliant** | At least 10% of the distance inside an active SMA was faster than 10.5 kn (10 kn plus 0.5 kn tolerance) |
| **Compliant** | Otherwise |
| **NotApplicable** | Vessel under 65 ft, or the SMA wasn't active |
| **Indeterminate** | Fewer than 3 pings inside the SMA |

**Metrics per SMA, month and vessel class:** transits, non-compliant transits, % non-compliant, total miles, miles above 10 kn, % of miles above 10 kn. Voluntary Slow Zones are included for comparison.

**Output:** `SMA_Compliance_Monthly`, charts, and one-page summaries per SMA.

### Phase D — Where is the risk? (co-occurrence risk index)

For each 4 km² hexagon and month:

```
risk_index = 100 × norm(whale presence) × norm(miles traveled by rule-applicable vessels) × speed_factor
```

- `norm()` rescales each value to 0–1 within each month.
- The **speed factor** rises from 0.5 at 10 kn or below to 1.0 at 15 kn or above, because faster strikes are more likely to be deadly.
- Risk is grouped into 5 classes by quintile.
- Key output: **the share of the coast's total risk that falls inside the current SMAs.**

**Output:** `Hex4_Risk_Monthly`.

### Phase E — What if? (management scenarios)

Candidate changes, stored in `CandidateSMA`:

- Extend Mid-Atlantic port SMAs out to 30 nm.
- Shift Northeast active periods by ±1 month.
- Add a Southern New England wind-lease SMA from December to March.
- Apply the rule to vessels 35 ft and longer.

For each scenario we report months active, risk covered, % of coast-wide risk covered, area added, extra transits affected, and change from today.

**Sensitivity test:** move the compliance threshold, speed-factor settings and presence weights by ±25%, then check whether the scenario ranking holds.

**Output:** `Scenario_Summary`, maps, and decision briefs.

### Limitations (stated on every product)

- Sightings follow where surveys went.
- Acoustic detections show presence, not how many whales.
- Density models carry their own uncertainty.
- Some vessels don't broadcast AIS, and smaller Class B vessels are under-represented.
- Speed thresholds and tolerances are analytical choices.
- **The risk index is relative, not a probability of a strike.**
- Boundaries and rules must be checked against current Federal Register notices.

---

## 6. Automation: the ArcPy toolbox

### `NARW_Tools.pyt` — 12 tools

| # | Tool | What it does |
|---|---|---|
| 1 | `BuildSchema` | Builds the whole geodatabase from `schema.yaml` |
| 2 | `IngestSource` | Loads a raw dataset and logs its source |
| 3 | `RunQAQC` | Runs all quality checks and logs the results |
| 4 | `GenerateHexGrids` | Creates the 25 km² and 4 km² hexagon grids |
| 5 | `BuildPresenceSurfaces` | Phase A: whale presence index |
| 6 | `ProcessAIS` | Phase B: runs the Python AIS preprocessor and loads its output |
| 7 | `ComputeCompliance` | Phase C: SMA compliance metrics |
| 8 | `ComputeRisk` | Phase D: risk index |
| 9 | `EvaluateScenarios` | Phase E: scenario comparison |
| 10 | `GenerateMetadata` | Writes and validates ISO/FGDC metadata |
| 11 | `ExportMapSeries` | Exports multi-page PDF map series |
| 12 | `ArchiveSnapshot` | Zips the geodatabase, records its checksum and version |

### Around the toolbox

- **Settings file:** `configs/analysis.yaml` holds all thresholds, weights and grid sizes.
- **Logging:** every run writes to a log file and to `ProcessingLog`.
- **Tests:** `pytest` unit tests for the core logic (compliance rules, vessel class mapping, speed calculations), plus ArcPy smoke tests on a small test geodatabase.
- **ModelBuilder:** `NARW_Models.tbx` holds `M1_PresenceSurfaces` and `M2_ComplianceAndRisk`, exported as diagrams.
- **One-command rerun:** `python run_pipeline.py --step all --years 2022-2024`
- **Optional:** an R Markdown notebook (`sf`, `ggplot2`) that reproduces the compliance charts.

---

## 7. Maps and decision-support products

### Map standards

A layout template (`NARW_Template.pagx`) includes:

- title, legend, scale bar, north arrow, graticule
- coordinate system note, data sources, methods note, limitations note
- version and date

Colors are colorblind-safe, and class breaks stay the same across months.

### Static maps and map series

| # | Product | Content |
|---|---|---|
| M01 | Study area and management framework | SMAs, critical habitat, shipping lanes, wind leases, ports |
| M02 | Data coverage | Sightings and acoustic effort by year; recorder locations |
| M03 | Data fitness summary | Coverage gaps and caveat zones |
| MS01 | Whale presence map series | One page per month (12 pages) |
| MS02 | Vessel traffic map series | One page per month: miles traveled and average speed |
| MS03 | SMA compliance map series | One page per SMA: % non-compliant by vessel class, with chart |
| MS04 | Risk map series | One page per month: risk classes with current SMAs overlaid |
| M04 | Scenario comparison | Today vs. each scenario (small multiples) |
| M05 | Wind leases and risk | Risk and whale presence in and around lease areas |

### ArcGIS Online

- **Hosted layers:** aggregated hexagons, SMAs, Slow Zones, critical habitat, wind leases, compliance tables. No restricted raw points.
- **Web map:** monthly time slider, plain-language pop-ups, grouped layers.
- **Dashboard:** month selector, map, indicators (% of risk inside SMAs, % non-compliant by SMA), compliance charts by vessel class and month, SMA list with drill-down, scenario toggle.
- Item metadata is completed for every hosted item.

### Decision briefs

One page per SMA and per scenario: map, key numbers, trend, caveats, and **"what this can and cannot be used for."** Written for program staff and regulators.

---

## 8. Technical report and project management

### Technical report

`NARW_VesselStrike_TechReport.pdf`: 25–40 pages in NOAA technical-memorandum style.

**Sections:**

- executive summary
- management context (ESA, MMPA, speed rule, Section 7)
- study area
- data and fitness for use
- geodatabase design
- QA/QC
- methods
- results
- discussion
- assumptions and limitations
- appropriate uses
- reproducibility
- references

**Appendices:**

- A. Data source registry
- B. Data dictionary
- C. QA/QC summary
- D. Metadata catalog
- E. Map gallery
- F. Tool reference
- G. Scenario tables

### Project management

- A GitHub Projects board and weekly status notes (`docs/status/`).
- Design reviews at weeks 2, 5 and 8, with written records (`docs/reviews/`).
- Standard project folders: `01_Admin`, `02_Data/{raw,working,final}`, `03_Scripts`, `04_Metadata`, `05_Maps`, `06_Reports`, `07_Archive`.
- Code, schema, metadata and docs are kept in Git. Large data stays out of Git and is archived as versioned zip snapshots.

---

## 9. Timeline (10 weeks, part-time)

| Week | Milestone | Done when |
|---|---|---|
| 1 | Download sources, request restricted data, `DataSourceRegistry`, folder and Git setup | All public sources downloaded; Consortium request submitted |
| 2 | Build geodatabase with `BuildSchema`; ERD; data dictionary; **design review 1** | Schema builds cleanly; review record filed |
| 3 | Load all non-AIS data; first QA run; draft fitness assessment | `QAQC_Log` populated; every source assessed |
| 4 | Process 2022–2024 AIS; build transits and hex grids | Row counts reconcile; sample points loaded |
| 5 | Whale presence surfaces; compliance analysis; **design review 2** | `SMA_Compliance_Monthly` complete |
| 6 | Risk index; scenarios; sensitivity | `Hex4_Risk_Monthly` and `Scenario_Summary` complete |
| 7 | Metadata for every object; validation; catalog | `MetadataStatus` at 100% |
| 8 | Map series and static maps; **design review 3** | PDFs exported; map checklist passed |
| 9 | ArcGIS Online web map, Dashboard, decision briefs | Items published (portfolio-safe layers only) |
| 10 | Technical report; geodatabase write-up; v1.0 archive; portfolio pages | All acceptance criteria met |

---

## 10. Deliverables checklist

### A. Flagship project

- [ ] `NARW_VesselStrike.gdb` v1.0 snapshot (zipped, with checksum) and mobile geodatabase export
- [ ] Data Fitness-for-Use Assessment (all sources)
- [ ] QA/QC log and summary
- [ ] Validated ISO 19115 and FGDC metadata for every object; metadata catalog
- [ ] `NARW_Tools.pyt` (12 tools), ModelBuilder models with diagrams, and tests
- [ ] Analysis outputs: presence, traffic, compliance, risk, scenarios
- [ ] Static maps M01–M05 and map series MS01–MS04 (PDF)
- [ ] ArcGIS Online web map and Dashboard (public links)
- [ ] Decision briefs (per SMA and per scenario)
- [ ] Technical report PDF
- [ ] Weekly status notes and three design-review records
- [ ] Portfolio page with 3–5 key findings

### B. Geodatabase and metadata project

- [ ] `docs/GDB_Design.md`, ERD, data dictionary, naming and versioning standards
- [ ] `schema.yaml` and `BuildSchema` tool
- [ ] Metadata templates and `GenerateMetadata` tool, with validation evidence
- [ ] Archival and versioning demonstration (`VersionHistory`, checksums)
- [ ] 60-second screen recording (domains, relationships, topology validation, metadata)
- [ ] Portfolio page

### Acceptance criteria (summary)

1. `BuildSchema` recreates the geodatabase on a clean machine, and a schema diff matches the design.
2. Every source has a fitness assessment and a registry row, and every step has a `ProcessingLog` row with its git commit.
3. Every QA check has been run and signed off.
4. 100% of objects have complete, validated ISO and FGDC metadata.
5. `run_pipeline.py --step all` reproduces the compliance, risk and scenario outputs with identical row counts.
6. Compliance is reported by SMA, month and vessel class for 2022–2024, with the indeterminate share documented.
7. Sensitivity results show whether the scenario ranking holds.
8. All maps pass the checklist, the web map and Dashboard work, and no restricted raw points are published.
9. The report states methods, assumptions, limitations and appropriate uses.
10. The v1.0 snapshot is archived with a checksum and README.

---

## 11. What ends up on the portfolio

1. **Flagship page:** 3–5 headline findings, such as:
   - % of non-compliant transits by SMA and vessel class
   - share of coast-wide strike risk inside current SMAs
   - the scenario that captures the most added risk

   It links to the Dashboard, report, map series and this repository.

2. **Geodatabase and metadata page:** the ERD, design rationale, schema-from-code approach, validated metadata, and a 60-second screen recording showing domains, relationships, topology and metadata in ArcGIS Pro.

---

## 12. Who does what

| Work | Claude Code (cloud session) | Bernard (ArcGIS Pro on Windows) |
|---|---|---|
| Repository structure, configs, schema | ✅ Writes and pushes | Reviews |
| ArcPy toolbox and ModelBuilder logic | ✅ Writes code and tests | Runs in ArcGIS Pro, reports errors |
| AIS preprocessing (pandas/DuckDB) | ✅ Writes, runs and tests | Runs on the full dataset if needed |
| Geodatabase build, data loading, geoprocessing | Writes the tools | ✅ Runs them (or Claude Code on the Windows PC runs them via `propy.bat`) |
| Map layouts and map series | Writes `arcpy.mp` export scripts and templates | ✅ Designs and fine-tunes in ArcGIS Pro |
| ArcGIS Online web map and Dashboard | Drafts content, pop-ups and item metadata | ✅ Publishes and configures |
| Docs, metadata templates, report text, portfolio copy | ✅ Drafts | Reviews and finalizes |

> **Tip:** Installing Claude Code on the Windows PC that has ArcGIS Pro lets Claude run ArcPy scripts directly with Pro's Python (`propy.bat`). ArcGIS Pro's interface still can't be clicked by Claude, and Pro should be closed before scripts edit the `.aprx` project file.
