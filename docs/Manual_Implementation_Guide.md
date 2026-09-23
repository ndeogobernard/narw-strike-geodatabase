# North Atlantic Right Whale Vessel-Strike Risk — Manual Implementation Guide

**A complete handoff specification for a GIS Analyst to execute the project end to end in ArcGIS Pro, ArcPy, and Python.**

| | |
|---|---|
| **Project** | NARW Vessel-Strike Risk — Decision-Support Geodatabase and Analysis |
| **Owner** | Bernard Issifu |
| **Executed by** | GIS Analyst / Specialist ("the Analyst") |
| **Repository** | https://github.com/ndeogobernard/narw-strike-geodatabase |
| **Portfolio** | https://ndeogobernard.github.io/ndeogo/ |
| **Document version** | 1.0 — 23 September 2026 |
| **Companion documents** | `docs/Project_Overview.md` (plain-language summary) |
| **Status** | Ready for execution |

---

## Table of contents

- [0. How to use this document](#0-how-to-use-this-document)
- [1. Project summary](#1-project-summary)
- [2. Prerequisites: software, licenses, hardware, accounts, skills](#2-prerequisites-software-licenses-hardware-accounts-skills)
- [3. Study area, time scope, coordinate systems, grids](#3-study-area-time-scope-coordinate-systems-grids)
- [4. Stage 1 — Project setup](#4-stage-1--project-setup)
- [5. Stage 2 — Data acquisition (S01–S16), with links](#5-stage-2--data-acquisition-s01s16-with-links)
- [6. Stage 3 — Build the geodatabase](#6-stage-3--build-the-geodatabase)
- [7. Stage 4 — Ingest data into the geodatabase](#7-stage-4--ingest-data-into-the-geodatabase)
- [8. Stage 5 — QA/QC and Fitness-for-Use Assessment](#8-stage-5--qaqc-and-fitness-for-use-assessment)
- [9. Stage 6 — Hexagon analysis grids](#9-stage-6--hexagon-analysis-grids)
- [10. Stage 7 — Phase A: Whale presence surfaces](#10-stage-7--phase-a-whale-presence-surfaces)
- [11. Stage 8 — Phase B: AIS vessel traffic processing](#11-stage-8--phase-b-ais-vessel-traffic-processing)
- [12. Stage 9 — Phase C: Speed-rule compliance](#12-stage-9--phase-c-speed-rule-compliance)
- [13. Stage 10 — Phase D: Co-occurrence risk index](#13-stage-10--phase-d-co-occurrence-risk-index)
- [14. Stage 11 — Phase E: Management scenarios and sensitivity](#14-stage-11--phase-e-management-scenarios-and-sensitivity)
- [15. Stage 12 — Metadata (ISO 19115 / FGDC CSDGM)](#15-stage-12--metadata-iso-19115--fgdc-csdgm)
- [16. Stage 13 — Automation: ModelBuilder and ArcPy toolbox](#16-stage-13--automation-modelbuilder-and-arcpy-toolbox)
- [17. Stage 14 — Cartography: static maps and map series](#17-stage-14--cartography-static-maps-and-map-series)
- [18. Stage 15 — ArcGIS Online web map and Dashboard](#18-stage-15--arcgis-online-web-map-and-dashboard)
- [19. Stage 16 — Decision briefs](#19-stage-16--decision-briefs)
- [20. Stage 17 — Technical report](#20-stage-17--technical-report)
- [21. Stage 18 — Versioning, archival, and release](#21-stage-18--versioning-archival-and-release)
- [22. Stage 19 — Portfolio packaging](#22-stage-19--portfolio-packaging)
- [23. Week-by-week work plan with task checklists](#23-week-by-week-work-plan-with-task-checklists)
- [24. Acceptance criteria and review checklists](#24-acceptance-criteria-and-review-checklists)
- [25. Risks, issues, and mitigations](#25-risks-issues-and-mitigations)
- [Appendix A — Master link list](#appendix-a--master-link-list)
- [Appendix B — Marine Cadastre AIS field dictionary](#appendix-b--marine-cadastre-ais-field-dictionary)
- [Appendix C — Vessel class lookup (AIS type codes)](#appendix-c--vessel-class-lookup-ais-type-codes)
- [Appendix D — Seasonal Management Area reference](#appendix-d--seasonal-management-area-reference)
- [Appendix E — Analysis parameters (`configs/analysis.yaml`)](#appendix-e--analysis-parameters-configsanalysisyaml)
- [Appendix F — Templates](#appendix-f--templates)
- [Appendix G — Literature and regulatory references](#appendix-g--literature-and-regulatory-references)
- [Appendix H — Glossary](#appendix-h--glossary)

---

## 0. How to use this document

This guide is written so a competent GIS Analyst can run the project **without further scoping and without AI assistance**. It is ordered as the work should be done. Each stage has:

- **Goal** — what the stage produces.
- **Inputs / outputs** — exact files, feature classes, and tables.
- **Steps** — numbered, with the ArcGIS Pro tool name (and toolbox), key parameters, and optional ArcPy/Python equivalents.
- **Checks** — what "done" looks like before moving on.
- **Log** — what to record in `ProcessingLog`, `QAQC_Log`, or the decision log.

### 0.1 Conventions

| Marker | Meaning |
|---|---|
| **[VERIFY]** | Confirm against the live source before use. URLs, dataset versions, regulatory dates, and download formats change. Record what you found in the decision log. |
| **[DECISION]** | The Analyst may choose. The default is given. Record the final choice and the reason in `docs/decision_log.md`. |
| `monospace` | File names, field names, tool parameters, code. |
| **Tool (Toolbox)** | ArcGIS Pro geoprocessing tool, e.g. **Generate Tessellation (Data Management)**. |

### 0.2 The decision log

Create `docs/decision_log.md` on day one. Every **[DECISION]** and every **[VERIFY]** outcome gets a row:

| ID | Date | Topic | Options considered | Decision | Rationale | Section |
|---|---|---|---|---|---|---|
| D-001 | 2026-09-24 | Storage CRS | EPSG:4326 vs 4269 | EPSG:4269 (NAD 83) | U.S. federal convention; aligns with NOAA shapefiles | §3.3 |

Pre-filled default decisions are listed in [§3.5](#35-default-decisions-to-record-in-the-decision-log).

### 0.3 Golden rules for the whole project

1. **Never edit raw data.** Raw downloads go to `02_Data/raw/` and are read-only. All changes happen in `working/` or the geodatabase.
2. **Every dataset gets a `source_id`** (S01–S16) that follows it into every derived product.
3. **Every tool run gets a `ProcessingLog` row.** Every QA check gets a `QAQC_Log` row.
4. **Restricted data never leaves the analyst's machine** in raw form. Public outputs are aggregated hexagons only.
5. **Commit code and documentation to Git at least daily.** Large data stays out of Git.
6. **All area, distance, and density work happens in EPSG:5070** (equal-area). Never compute areas or distances in geographic coordinates.

---

## 1. Project summary

### 1.1 Background

The North Atlantic right whale (*Eubalaena glacialis*) is one of the most endangered large whales. The population is roughly 370–385 animals [VERIFY latest North Atlantic Right Whale Consortium Report Card estimate]. It is listed as **endangered under the Endangered Species Act (ESA)** and **depleted under the Marine Mammal Protection Act (MMPA)**. Vessel strikes and fishing-gear entanglement are the leading causes of serious injury and death. An Unusual Mortality Event (UME) has been ongoing since 2017 [VERIFY status].

NOAA Fisheries manages vessel-strike risk through:

| Measure | What it is | Legal basis |
|---|---|---|
| **Vessel speed rule** | Vessels ≥ 65 ft (19.8 m) must travel ≤ 10 knots in **Seasonal Management Areas (SMAs)** during their active periods. Federal and law-enforcement vessels are exempt; safety exceptions apply. | 50 CFR 224.105 (final rule 2008; sunset clause removed 2013) [VERIFY current text] |
| **Dynamic Management Areas / Right Whale Slow Zones** | Voluntary 10-knot zones announced for ~15 days when ≥ 3 whales are sighted (visual trigger) or whales are detected acoustically. | Voluntary program under the speed rule [VERIFY] |
| **Critical habitat** | Unit 1 (Gulf of Maine/Georges Bank foraging) and Unit 2 (Southeast U.S. calving). | 81 FR 4837 (27 Jan 2016) [VERIFY] |
| **Section 7 consultation** | Federal agencies must consult NOAA on actions that may affect listed species (e.g., offshore wind). | ESA §7 |
| **Proposed rule amendments (2022)** | Proposed expanding the rule to vessels ≥ 35 ft and replacing SMAs with larger "Seasonal Speed Zones." | 87 FR 46921 (1 Aug 2022); reported withdrawn January 2025 [VERIFY status] |

### 1.2 Management question

> **Where and when do North Atlantic right whales and vessel traffic co-occur along the U.S. Atlantic coast; how well does vessel traffic comply with 10-knot restrictions inside Seasonal Management Areas by area, month, and vessel class; and where would modified or additional seasonal management areas provide the greatest reduction in vessel-strike risk?**

### 1.3 Objectives

1. Build an integrated protected-species geodatabase combining **biological**, **regulatory**, **vessel traffic**, **environmental**, and **project** data.
2. Apply documented QA/QC to every dataset and write a **Data Fitness-for-Use Assessment**.
3. Produce complete, validated **ISO 19115 and FGDC CSDGM metadata** for every object.
4. Build **monthly whale-presence surfaces**, **vessel-traffic density and speed surfaces**, and an **SMA speed-compliance analysis** from AIS.
5. Compute a **co-occurrence vessel-strike risk index** and evaluate **candidate management-area modifications**.
6. Deliver **decision-support products**: map series, an ArcGIS Online web map and Dashboard, and one-page decision briefs.
7. **Automate** ingest, QA, aggregation, metadata, and map export (ArcPy toolbox + ModelBuilder), under version control with an archival scheme.
8. Write a **technical report** in NOAA technical-memorandum style.
9. Package as two portfolio projects: **(A) flagship analysis** and **(B) standalone geodatabase-and-metadata project**.

### 1.4 Non-goals

- Absolute strike probability or mortality estimates (a **relative** risk index is delivered).
- Entanglement risk.
- Any use of restricted data outside its stated terms.

### 1.5 Intended users of the products

| User | Product they use |
|---|---|
| NOAA Office of Protected Resources program staff | Decision briefs, compliance tables, risk maps |
| Section 7 biologists | Wind-lease co-occurrence map (M05), risk index |
| Regulators / policy analysts | Scenario summary, decision briefs |
| Public / portfolio visitors | Web map, Dashboard, portfolio page |
| Hiring managers (portfolio) | README, report, geodatabase write-up, screen capture |

---

## 2. Prerequisites: software, licenses, hardware, accounts, skills

### 2.1 Software

| Software | Version | Needed for | Notes |
|---|---|---|---|
| **ArcGIS Pro** | 3.3 or later [VERIFY latest] | Everything GIS | **Standard** license minimum (topology, attribute rules, relationship classes, editor tracking require Standard). **Advanced** unlocks GeoAnalytics Desktop tools (e.g., Reconstruct Tracks) — optional. |
| **Spatial Analyst extension** | with Pro | Kernel Density, Zonal Statistics, raster math | **Required** |
| **ArcGIS Online organizational account** | current | Hosted layers, web map, Dashboard | A public (free) account cannot host feature layers or build Dashboards. **ArcGIS for Personal Use** (~US$100/yr) includes Pro Advanced + extensions + an AGOL org account [VERIFY price/terms]. |
| **Python** | ArcGIS Pro's bundled Python (`arcgispro-py3`) cloned to a new env | ArcPy scripts, AIS processing | Clone the default environment: *Project > Package Manager > Environment Manager > Clone*. Name it `narw-py3`. |
| Python packages (in the clone) | `duckdb` (≥ 1.1), `pyarrow`, `pandas`, `pyyaml`, `pytest`, `requests`, `tqdm` | AIS preprocessing, configs, tests | Install via Package Manager or `conda install -c conda-forge duckdb pyarrow pyyaml pytest tqdm` from the Python Command Prompt. |
| **Git** + **GitHub Desktop** (optional GUI) | current | Version control | https://git-scm.com/ , https://desktop.github.com/ |
| **7-Zip** | current | Zipping snapshots | https://www.7-zip.org/ |
| **VS Code** (recommended) | current | Editing scripts, YAML, Markdown | https://code.visualstudio.com/ |
| **R + RStudio** (optional) | R ≥ 4.3 | Optional R compliance charts | Packages: `sf`, `ggplot2`, `dplyr`, `arrow` |
| **Panoply** or **QGIS** (optional) | current | Quick look at NetCDF (SST, chlorophyll) | https://www.giss.nasa.gov/tools/panoply/ |
| **Screen recorder** | OBS Studio or Windows Snipping Tool video | 60-second geodatabase demo | https://obsproject.com/ |

### 2.2 Hardware

| Resource | Minimum | Recommended | Why |
|---|---|---|---|
| RAM | 16 GB | 32–64 GB | AIS processing, kernel density |
| Free disk | 500 GB | 1 TB SSD | Three years of daily national AIS files are very large unzipped [VERIFY sizes on the download page]. Process month-by-month and delete unzipped CSVs after conversion to Parquet. |
| CPU | 4 cores | 8+ cores | DuckDB parallelizes automatically |
| OS | Windows 10/11 64-bit | Windows 11 | ArcGIS Pro is Windows-only |

### 2.3 Accounts and access requests (start in Week 1 — they take time)

| Account / request | Where | Lead time |
|---|---|---|
| ArcGIS Online organizational account | https://www.esri.com/en-us/arcgis/products/arcgis-for-personal-use/buy (or employer/university org) | Same day |
| NASA Earthdata login (for PO.DAAC MUR SST, if used) | https://urs.earthdata.nasa.gov/ | Same day |
| **North Atlantic Right Whale Consortium sightings data request** | https://www.narwc.org/ (Data → sightings/identification database request) [VERIFY page] | **Weeks.** Submit Week 1. Project proceeds without it if denied. |
| GitHub account (exists) | https://github.com/ndeogobernard | — |
| Zenodo account (optional, for a DOI on the archived snapshot) | https://zenodo.org/ | Same day |

### 2.4 Skills assumed

ArcGIS Pro geoprocessing and editing; geodatabase design (domains, subtypes, relationship classes, topology); Spatial Analyst; ArcPy and basic SQL; ArcGIS Online publishing; metadata editing; basic Git. Where Python is required (AIS volume), complete code is provided in this guide.

---

## 3. Study area, time scope, coordinate systems, grids

### 3.1 Study area

- **Extent:** U.S. Atlantic coast from the Florida–Georgia calving grounds (~ 27°N) to the Gulf of Maine and Georges Bank (~ 45°N), from the shoreline to the U.S. **200 nm EEZ** boundary.
- **Analytical focus zone:** within **50 nm (92.6 km)** of shore, where all SMAs lie.
- **Working bounding box (for filtering raw data):** longitude **−82.0 to −65.0**, latitude **24.0 to 46.0** (WGS 84). Clip to the StudyArea polygon afterwards.

**How to build `StudyArea`** (Stage 6 step details in §7.2):
1. Take the U.S. EEZ polygon (S15) for the Atlantic.
2. Clip to latitude 24°N–45.5°N and west of −65°W.
3. Erase land (states polygon).
4. Result = `Reference/StudyArea`. Also build `Reference/FocusZone_50nm` = StudyArea ∩ 50 nm buffer of the shoreline.

### 3.2 Time scope

| Data | Period | Notes |
|---|---|---|
| AIS vessel traffic | **2022-01-01 to 2024-12-31** | [DECISION] extend to 2025 if full-year data is published |
| Sightings and acoustic detections | **2010–present** for presence climatology | Pooled by calendar month across years |
| Sightings and acoustic detections | **2022–2024** | Year-specific, aligned with AIS for sensitivity analysis |
| Density models | Most recent Duke/NOAA version (monthly) | Record version and years modeled |
| Regulatory boundaries | Current + Slow Zone archive **2018–present** | Record CFR/FR citation and effective date |
| Environmental | Monthly climatologies **2010–2024** | 12 layers per variable |

### 3.3 Coordinate reference systems

| Purpose | CRS | WKID | ArcGIS name |
|---|---|---|---|
| Raw data (as delivered) | Usually WGS 84 | 4326 | GCS_WGS_1984 |
| **Storage** in the geodatabase | **NAD 83** | **4269** | GCS_North_American_1983 |
| **Analysis** (area, distance, density, grids) | **NAD 1983 Contiguous USA Albers** | **5070** | NAD_1983_Contiguous_USA_Albers |
| Regional map insets (optional) | NAD 1983 UTM Zone 17N / 18N / 19N | 26917 / 26918 / 26919 | |
| Web delivery | WGS 84 Web Mercator | 3857 | WGS_1984_Web_Mercator_Auxiliary_Sphere |
| Vertical (bathymetry) | Depth in meters relative to source datum | — | Record MLLW (CRM) vs. mean sea level (GEBCO) |

**Geographic transformation rule:** whenever projecting between WGS 84 and NAD 83, set the transformation explicitly to **`WGS_1984_(ITRF00)_To_NAD_1983`** and record it in `DataSourceRegistry.transformation_used` and in the layer's metadata lineage.

**How to set it in Pro:**
- In **Project (Data Management)**: set *Geographic Transformation* = `WGS_1984_(ITRF00)_To_NAD_1983`.
- For geoprocessing environments: *Analysis > Environments > Output Coordinates > Geographic Transformations* → add `WGS_1984_(ITRF00)_To_NAD_1983`.
- For map display: *Map Properties > Transformation* → same.

### 3.4 Analysis grids

| Grid | Cell area | Hexagon side length | Coverage | Used for |
|---|---|---|---|---|
| `HexGrid_25km2` | 25 km² | ≈ 3.10 km | Entire `StudyArea` | Coast-wide whale presence, overview maps |
| `HexGrid_4km2` | 4 km² | ≈ 1.24 km | **`FocusZone_50nm` plus candidate-scenario areas** [DECISION — see note] | Vessel traffic, compliance context, risk, scenarios |

> **Note on the 4 km² grid extent.** The original scope says 4 km² cells "inside and within 10 km of management areas." But the risk analysis must report the **share of coast-wide risk inside SMAs** and evaluate scenarios that extend areas seaward to 30 nm. That requires risk values outside today's SMAs. **Default decision:** cover the whole 50-nm focus zone at 4 km², and flag hexes inside or within 10 km of management areas with `in_sma_buffer = 1`. Record this as a decision.

Hexagon side length from area: `s = sqrt( 2 × A / (3 × sqrt(3)) )`.

### 3.5 Default decisions to record in the decision log

| ID | Decision | Default |
|---|---|---|
| D-001 | Storage CRS | NAD 83 (EPSG:4269) |
| D-002 | Analysis CRS | EPSG:5070 |
| D-003 | AIS years | 2022–2024 |
| D-004 | 4 km² grid extent | 50-nm focus zone + scenario areas |
| D-005 | Geodatabase type | File geodatabase (primary) + mobile geodatabase export (SQL demo) |
| D-006 | Metadata primary style | ISO 19139 (with FGDC CSDGM export) |
| D-007 | Sightings effort handling | Presence-only index where effort unavailable; `survey_type` flag |
| D-008 | Presence weights | Sightings 0.4 / Acoustic 0.2 / Density model 0.4; re-normalize if a component is missing |
| D-009 | AIS leg speed rule | Use mean reported SOG of the two pings if within 2 kn of implied speed; otherwise implied speed |
| D-010 | Hex assignment of AIS legs | By leg midpoint (legs are ≈ 1 minute long) |
| D-011 | Rule applicability when length is missing | Applicable if class is Cargo, Tanker, or Passenger; otherwise "Indeterminate" |
| D-012 | Compliance threshold | NonCompliant if ≥ 10% of in-SMA distance > 10.5 kn |
| D-013 | Presence time basis for risk | Monthly climatology (2010–present pooled); 2022–2024-only as sensitivity |
| D-014 | Acoustic detection radius | 20 km around each recorder |
| D-015 | Kernel bandwidth | 25 km |
| D-016 | Web publication time basis | Monthly climatology (12 months) for risk/traffic layers to limit feature counts |

---

## 4. Stage 1 — Project setup

**Goal:** a clean, standard folder structure, a Git repository, an ArcGIS Pro project, and tracking documents.

### 4.1 Folder structure (on the local drive)

Create a root folder such as `C:\GIS\NARW\`. Avoid spaces and OneDrive-synced folders (sync locks geodatabase files).

```
C:\GIS\NARW\
├── 01_Admin\                  # access requests, emails, licenses, terms of use (PDF)
├── 02_Data\
│   ├── raw\                   # untouched downloads, one subfolder per source ID (S01..S16)
│   │   ├── S01_sightings\
│   │   ├── S02_acoustic\
│   │   ├── ...
│   │   └── S09_ais\2022\ 2023\ 2024\
│   ├── working\               # intermediate files, AIS Parquet, scratch.gdb
│   └── final\                 # NARW_VesselStrike.gdb, mobile .geodatabase
├── 03_Scripts\                # = Git repository clone (see §4.2)
├── 04_Metadata\               # exported ISO/FGDC XML
├── 05_Maps\                   # NARW.aprx, layouts (.pagx), styles (.stylx)
├── 06_Reports\                # report drafts, briefs, figures
└── 07_Archive\                # versioned zipped snapshots + checksums
```

### 4.2 Git repository

1. Clone the repo into `03_Scripts`:
   ```
   cd C:\GIS\NARW
   git clone https://github.com/ndeogobernard/narw-strike-geodatabase.git 03_Scripts
   ```
2. Create the repository layout inside it:
   ```
   narw-strike-geodatabase/
   ├── README.md
   ├── configs/            # analysis.yaml, metadata_templates.yaml, vessel_classes.csv
   ├── schema/             # schema.yaml
   ├── toolbox/            # NARW_Tools.pyt, models/NARW_Models.tbx, diagrams/
   ├── src/narw/           # Python modules
   ├── ais_preprocess/     # DuckDB/pandas scripts + manifest
   ├── r/                  # optional R Markdown
   ├── tests/              # pytest + small fixture data
   ├── metadata/           # ISO/FGDC XML (copied from 04_Metadata)
   ├── maps/               # .pagx layouts, .stylx styles (NOT the large .aprx)
   ├── docs/               # design docs, reviews/, status/, decision_log.md
   └── outputs/            # small final tables/figures; large files gitignored
   ```
3. Create `.gitignore`:
   ```gitignore
   # data and GIS binaries
   *.gdb/
   *.geodatabase
   *.zip
   *.parquet
   *.csv.gz
   data/
   outputs/**/*.pdf
   outputs/**/*.tif
   *.aprx.lock
   *.lock
   .backups/
   Index/
   # python
   __pycache__/
   .pytest_cache/
   *.pyc
   # os
   Thumbs.db
   .DS_Store
   ```
4. Commit convention: `type(scope): message`, e.g. `feat(schema): add Regulatory feature dataset`, `docs(fitness): S09 AIS assessment`.
5. Branching: work on `main` with short-lived feature branches, or on a single working branch merged via pull request per milestone [DECISION].
6. Create a **GitHub Projects** board (*Repository > Projects > New project > Board*) with columns *Backlog / This week / In progress / Review / Done*. Add one card per task in [§23](#23-week-by-week-work-plan-with-task-checklists).

### 4.3 ArcGIS Pro project

1. Open ArcGIS Pro → **New > Map** → name `NARW`, location `C:\GIS\NARW\05_Maps\`. Uncheck "Create a new folder for this project" (the folder exists).
2. **Project > Options > Metadata** → *Metadata style* = **ISO 19139 Metadata Implementation Specification** [DECISION D-006].
3. **Project > Options > Geoprocessing** → check **"Write geoprocessing operations to log file"** and **"Write geoprocessing operations to dataset metadata"** (this builds lineage automatically).
4. **Analysis > Environments** (project-level):
   - *Output Coordinate System*: leave blank at project level; set per tool.
   - *Geographic Transformations*: `WGS_1984_(ITRF00)_To_NAD_1983`.
   - *Scratch Workspace*: `C:\GIS\NARW\02_Data\working\scratch.gdb` (create it: **Create File Geodatabase (Data Management)**).
   - *Parallel Processing Factor*: `75%`.
5. Add folder connections in the Catalog pane: `02_Data`, `03_Scripts`, `04_Metadata`.
6. Save.

### 4.4 Tracking documents (create now, keep updated)

| File | Purpose | Template |
|---|---|---|
| `docs/decision_log.md` | Every DECISION/VERIFY outcome | §0.2 |
| `docs/status/2026-Wxx.md` | Weekly status: progress, dependencies, risks, next | Appendix F.5 |
| `docs/reviews/review_01.md` etc. | Design review records (Weeks 2, 5, 8) | Appendix F.6 |
| `ais_preprocess/manifest.csv` | Every AIS file: name, size, SHA-256, rows in/out | Appendix F.7 |
| `docs/naming_conventions.md` | Naming standard (§6.1) | — |

### 4.5 Checks

- [ ] Folder structure exists; raw folders per source.
- [ ] Repo cloned, `.gitignore` committed, first status note committed.
- [ ] `NARW.aprx` saved with metadata style and geoprocessing logging set.
- [ ] Decision log has D-001 to D-016.

---

## 5. Stage 2 — Data acquisition (S01–S16), with links

**Goal:** every source downloaded to `02_Data/raw/Sxx_*/`, terms of use saved to `01_Admin/`, and a `DataSourceRegistry` row drafted (in a spreadsheet for now; loaded into the geodatabase in Stage 4).

**For every source, record:** `source_id, provider, dataset, url, access_terms, vintage, download_date, native_crs, transformation_used, notes`. Save a PDF or screenshot of the landing page and terms of use in `01_Admin/terms/Sxx.pdf`.

> **[VERIFY] all URLs.** Links below were current as of this document's preparation. Government portals are reorganized often. If a link has moved, search the provider's site for the dataset name and record the new URL.

---

### S01 — Right whale sightings

| | |
|---|---|
| **Primary public sources** | NOAA NEFSC **Right Whale Sighting Advisory System (RWSAS)** — https://apps-nefsc.fisheries.noaa.gov/psb/surveys/ [VERIFY] ; **WhaleMap** (Dalhousie/Johnson et al.) — https://whalemap.org/ |
| **Restricted source** | **North Atlantic Right Whale Consortium (NARWC)** Sightings Database — https://www.narwc.org/ (request access) |
| **Format** | CSV (RWSAS/WhaleMap exports); NARWC delivers CSV/Access by agreement |
| **Native CRS** | WGS 84 lat/lon (decimal degrees) — confirm |
| **Target** | `Biological/Sightings` |

**Steps:**
1. **RWSAS:** open the sightings map/table page. Filter species = right whale, date range 2010-01-01 to present. Export to CSV. If the portal limits export size, export year-by-year. Save as `S01_sightings/rwsas_YYYY.csv`.
2. **WhaleMap:** open https://whalemap.org/ → data/download page [VERIFY]. Download observations (visual sightings, and acoustic if listed separately — treat acoustic under S02). Read and save the data-use policy. WhaleMap aggregates many sources and marks some as not for redistribution — respect it.
3. **NARWC request (Week 1):** use the email template in Appendix F.8. Ask for: sightings 2010–present in U.S. waters, fields for date/time, position, number of animals, calves, platform, survey type (systematic vs. opportunistic), and **survey effort track lines** if possible. Record terms verbatim in `01_Admin/terms/S01_NARWC.pdf`.
4. Inspect each file: columns, date format, time zone, coordinate format (decimal degrees vs. DMS), null conventions. Write findings in the fitness assessment draft.

**Expected fields to map:** date, time (UTC or local — record which), latitude, longitude, number of animals, number of calves, platform (aerial/vessel/opportunistic), observer/organization, verification status.

**Gotchas:** duplicate reports of the same group from multiple platforms; positions reported to low precision; local vs. UTC times; opportunistic sightings cluster near ports and whale-watch areas.

---

### S02 — Passive acoustic detections

| | |
|---|---|
| **Source** | NOAA NEFSC **Passive Acoustic Cetacean Map (PACM)** — https://apps-nefsc.fisheries.noaa.gov/pacm/ |
| **Archive** | NOAA NCEI Passive Acoustic Data archive — https://www.ncei.noaa.gov/products/passive-acoustic-data [VERIFY] |
| **Format** | CSV download from PACM (daily presence per recorder/platform) |
| **Target** | `Biological/AcousticDetections`, `Biological/Recorders` |

**Steps:**
1. Open PACM → select species **North Atlantic right whale** → set date range 2010–present → region U.S. Atlantic.
2. Use the download/export function (CSV). PACM data are typically **daily detection results per deployment**: `Detected`, `Possibly detected`, `Not detected`, with platform type (bottom-mounted, buoy, glider, towed) and location.
3. Download both **stationary** (moored recorders — one location per deployment) and **mobile** platforms (gliders — track positions per day).
4. Save the citation text PACM requests for publications.

**Important:** "Not detected" days are **effort**, not absence of data — keep them. They are needed to compute detection rates.

**Domain note [DECISION]:** the original scope's `dm_DetectionType` (Upcall, Other) does not match PACM's output. Use `dm_DetectionResult` = **Detected / Possibly detected / Not detected** and keep call type as a text field if provided.

---

### S03 — Right whale density models (monthly)

| | |
|---|---|
| **Source** | Duke University Marine Geospatial Ecology Lab (Roberts et al.), U.S. East Coast models — https://seamap.env.duke.edu/models/Duke/EC/ |
| **Species page** | North Atlantic right whale model page on the same site (look for "North Atlantic right whale" in the species list) [VERIFY latest version, e.g., v12+] |
| **Format** | GeoTIFF / Esri grid rasters of mean density per month, plus uncertainty (CV) rasters |
| **Units** | Typically **individuals per 100 km²** [VERIFY on model page]; convert to animals/km² by dividing by 100 |
| **Native CRS** | Custom projected CRS (usually an Albers or Mercator variant) — read from the .prj |
| **Target** | `Biological/DensityModel_Monthly` (mosaic dataset) |

**Steps:**
1. Open the right whale model page. Read the model documentation (report PDF) — note the version, survey years, and any "era" splits (models since ~2020 have distinct eras reflecting the post-2010 distribution shift). **Use the most recent era.**
2. Download the 12 monthly **mean density** rasters and the 12 **CV** rasters.
3. Save the model citation exactly as the page requests (Roberts et al. + year + version).

**Gotcha:** the right whale distribution shifted after ~2010 (less time in the Gulf of Maine in summer, more in Southern New England and Gulf of St. Lawrence). Using a pre-2010-era model would misrepresent current presence.

---

### S04 — OBIS-SEAMAP observations

| | |
|---|---|
| **Source** | OBIS-SEAMAP — https://seamap.env.duke.edu/ ; species profile for *Eubalaena glacialis* (search "right whale") |
| **Alternative** | OBIS — https://obis.org/taxon/159023 [VERIFY AphiaID] |
| **Format** | CSV |
| **Target** | Used for cross-checking; loaded into `Sightings` only if not duplicative, with `source_id = S04` |

**Steps:** Search species → filter to U.S. Atlantic, 2010–present → download observation records (point) as CSV. Record the dataset-level citations (OBIS-SEAMAP aggregates many providers; each has its own terms).

---

### S05 — Seasonal Management Areas (vessel speed rule)

| | |
|---|---|
| **Program page** | https://www.fisheries.noaa.gov/national/endangered-species-conservation/reducing-vessel-strikes-north-atlantic-right-whales |
| **GIS data** | NOAA Fisheries "North Atlantic Right Whale Seasonal Management Areas" GIS/shapefile resource — search the NOAA Fisheries resource library for "Seasonal Management Areas shapefile" [VERIFY link] ; also on Marine Cadastre hub — https://hub.marinecadastre.gov/ |
| **Legal text** | 50 CFR 224.105 — https://www.ecfr.gov/current/title-50/chapter-II/subchapter-C/part-224/section-224.105 |
| **Format** | Shapefile / GeoJSON |
| **Target** | `Regulatory/SMA` |

**Steps:**
1. Download the SMA shapefile.
2. Download or print to PDF the **current eCFR text** of 50 CFR 224.105. It lists every SMA's boundary coordinates and active dates.
3. **Cross-check** every SMA polygon against the CFR coordinates (spot-check at least 5 vertices per region) and active dates. Record in `QAQC_Log`.
4. If no shapefile is available, **digitize from CFR coordinates**: enter vertices in a CSV → **XY To Line (Data Management)** or **Points To Line** → **Feature To Polygon (Data Management)**. For port SMAs described as a 20-nm radius around a center point, use **Buffer (Analysis)** with *Method = Geodesic*, distance `20 NauticalMiles`, then clip to the seaward side of the coastline.

See Appendix D for the SMA list and active periods.

---

### S06 — Dynamic Management Areas / Right Whale Slow Zones (archive)

| | |
|---|---|
| **Current/archived notices** | NOAA Fisheries Slow Zone notices on the vessel-strike program page (above) and NOAA Fisheries news/bulletins [VERIFY archive location] |
| **RWSAS** | Historic DMAs are often listed in the RWSAS system [VERIFY] |
| **Format** | Notices (text with corner coordinates); shapefiles if provided |
| **Target** | `Regulatory/DMA_SlowZones` |

**Steps:**
1. Search for downloadable Slow Zone GIS data first. If none, build the archive from notices.
2. For each notice 2018–present, record in `DMA_archive.csv`: `dma_id, name, trigger_type (Visual/Acoustic), start_datetime, end_datetime, extended_flag, notice_url, north_lat, south_lat, east_lon, west_lon`.
3. Slow Zones are usually lat/lon rectangles. Build polygons with a short ArcPy snippet:
   ```python
   import arcpy, csv
   sr = arcpy.SpatialReference(4269)
   fc = r"C:\GIS\NARW\02_Data\working\working.gdb\DMA_build"
   with arcpy.da.InsertCursor(fc, ["SHAPE@", "dma_id", "name", "trigger_type",
                                   "start_datetime", "end_datetime", "notice_url"]) as cur, \
        open(r"C:\GIS\NARW\02_Data\raw\S06_dma\DMA_archive.csv") as f:
       for r in csv.DictReader(f):
           n, s, e, w = map(float, (r["north_lat"], r["south_lat"], r["east_lon"], r["west_lon"]))
           ring = arcpy.Array([arcpy.Point(w, n), arcpy.Point(e, n), arcpy.Point(e, s),
                               arcpy.Point(w, s), arcpy.Point(w, n)])
           cur.insertRow([arcpy.Polygon(ring, sr), r["dma_id"], r["name"], r["trigger_type"],
                          r["start_datetime"], r["end_datetime"], r["notice_url"]])
   ```
4. Densify edges before projecting (**Densify (Editing)**, `10 Kilometers`) so the rectangles stay correct in Albers.

**Gotcha:** the archive may be incomplete. Record coverage gaps (years/months without notices found) in the fitness assessment.

---

### S07 — Right whale critical habitat

| | |
|---|---|
| **Source** | NOAA Fisheries critical habitat GIS data — https://www.fisheries.noaa.gov/resource/map/north-atlantic-right-whale-critical-habitat-map-and-gis-data [VERIFY] ; national critical habitat mapper — https://www.fisheries.noaa.gov/resource/map/national-esa-critical-habitat-mapper |
| **Legal** | Final rule 81 FR 4837 (27 Jan 2016); regulatory text 50 CFR 226.203 [VERIFY] |
| **Format** | Shapefile / file geodatabase |
| **Target** | `Regulatory/CriticalHabitat` (Unit 1, Unit 2) |

---

### S08 — Traffic Separation Schemes and shipping lanes

| | |
|---|---|
| **Source** | Marine Cadastre — dataset "Shipping Fairways, Lanes, and Zones" (and "Traffic Separation Schemes") — https://hub.marinecadastre.gov/ (search "shipping lanes") [VERIFY dataset name] |
| **Authoritative origin** | NOAA Office of Coast Survey charts / 33 CFR Part 167 |
| **Format** | Shapefile / file geodatabase / feature service |
| **Target** | `Regulatory/TrafficSeparationSchemes` |

---

### S09 — AIS vessel traffic (the largest dataset)

| | |
|---|---|
| **Landing page** | Marine Cadastre Vessel Traffic — https://hub.marinecadastre.gov/pages/vesseltraffic |
| **Direct download index (by year)** | https://coast.noaa.gov/htdata/CMSP/AISDataHandler/2022/index.html , `/2023/index.html` , `/2024/index.html` [VERIFY — format may have changed to Parquet/GeoParquet for recent years] |
| **Cloud copy** | NOAA Open Data Dissemination on AWS — search the Registry of Open Data (https://registry.opendata.aws/) for "Marine Cadastre AIS" [VERIFY] |
| **Vessel type codes** | https://coast.noaa.gov/data/marinecadastre/ais/VesselTypeCodes2018.pdf [VERIFY] |
| **Data FAQ / dictionary** | https://coast.noaa.gov/data/marinecadastre/ais/faq.pdf [VERIFY] ; see Appendix B |
| **Format** | One zipped CSV per day, nationwide (≈ 1-minute sampled positions) — e.g., `AIS_2023_01_01.zip` |
| **Native CRS** | WGS 84 |
| **Target** | Processed outside ArcGIS (§11) → `VesselTraffic/AIS_Transits`, `VesselTraffic/AIS_Points_Sample`, `Analysis/Hex4_VesselTraffic_Monthly` |

**Steps:**
1. Check the landing page for the current distribution format, file naming, and any changes (e.g., zone-based vs. national files, CSV vs. Parquet). Record it.
2. Estimate total size before downloading (check a few file sizes × 1,096 days). Make sure disk space is available.
3. Download with a script (resumable; writes the manifest). Save as `ais_preprocess/download_ais.py`:
   ```python
   """Download Marine Cadastre daily AIS zips and record a manifest with SHA-256."""
   import csv, hashlib, os, datetime as dt, requests
   from tqdm import tqdm

   BASE = "https://coast.noaa.gov/htdata/CMSP/AISDataHandler/{y}/AIS_{y}_{m:02d}_{d:02d}.zip"  # [VERIFY]
   OUT = r"C:\GIS\NARW\02_Data\raw\S09_ais"
   MANIFEST = r"C:\GIS\NARW\03_Scripts\ais_preprocess\manifest.csv"

   def sha256(path, chunk=1 << 20):
       h = hashlib.sha256()
       with open(path, "rb") as f:
           for b in iter(lambda: f.read(chunk), b""):
               h.update(b)
       return h.hexdigest()

   def main(start=dt.date(2022, 1, 1), end=dt.date(2024, 12, 31)):
       new = not os.path.exists(MANIFEST)
       with open(MANIFEST, "a", newline="") as mf:
           w = csv.writer(mf)
           if new:
               w.writerow(["file", "url", "bytes", "sha256", "download_utc"])
           day = start
           while day <= end:
               url = BASE.format(y=day.year, m=day.month, d=day.day)
               folder = os.path.join(OUT, str(day.year)); os.makedirs(folder, exist_ok=True)
               path = os.path.join(folder, os.path.basename(url))
               if not os.path.exists(path):
                   with requests.get(url, stream=True, timeout=120) as r:
                       r.raise_for_status()
                       with open(path + ".part", "wb") as f:
                           for b in tqdm(r.iter_content(1 << 20), desc=os.path.basename(path)):
                               f.write(b)
                   os.replace(path + ".part", path)
                   w.writerow([os.path.basename(path), url, os.path.getsize(path), sha256(path),
                               dt.datetime.utcnow().isoformat()])
                   mf.flush()
               day += dt.timedelta(days=1)

   if __name__ == "__main__":
       main()
   ```
   Run from the Python Command Prompt in the `narw-py3` environment: `python download_ais.py`.
4. Do **not** unzip everything at once. The preprocessing (§11) reads one month at a time, filters to the study bounding box, writes Parquet, and discards the CSV.

---

### S10 — BOEM offshore wind lease and planning areas

| | |
|---|---|
| **Source** | BOEM Renewable Energy GIS Data — https://www.boem.gov/renewable-energy/mapping-and-data/renewable-energy-gis-data |
| **Format** | Shapefile / file geodatabase (lease areas, planning areas, wind energy areas) |
| **Target** | `Project/WindLeaseAreas` |

**Note:** record the download date — lease status changes (leases relinquished, cancelled, or paused) [VERIFY current status of Atlantic leases at download].

---

### S11 — Ports and port approaches

| | |
|---|---|
| **Sources** | USDOT Bureau of Transportation Statistics NTAD "Principal Ports" — https://geodata.bts.gov/ (search "Principal Ports") [VERIFY] ; USACE Waterborne Commerce Statistics Center — https://www.iwr.usace.army.mil/About/Technical-Centers/WCSC-Waterborne-Commerce-Statistics-Center/ ; Marine Cadastre hub |
| **Format** | Shapefile / CSV points |
| **Target** | `Reference/Ports` (with `unlocode` and the related `sma_id`) |

UN/LOCODEs: https://unece.org/trade/cefact/unlocode-code-list-country-and-territory (U.S. list).

---

### S12 — Bathymetry

| | |
|---|---|
| **Sources** | NOAA NCEI **Coastal Relief Model** (CRM, ~3 arc-second) — https://www.ncei.noaa.gov/products/coastal-relief-model ; **GEBCO** global grid (15 arc-second; use latest year) — https://www.gebco.net/data_and_products/gridded_bathymetry_data/ |
| **Format** | NetCDF / GeoTIFF |
| **Vertical datum** | CRM: mixed, generally MLLW nearshore / MSL offshore [VERIFY]; GEBCO: MSL |
| **Target** | `Environmental/Bathymetry` |

**Steps:** Download CRM Volumes 1–3 (Northeast Atlantic, Southeast Atlantic, Florida/East Gulf) [VERIFY volume numbers]; download the GEBCO subset tile for the bounding box (GEBCO download app lets you draw a box). Use CRM nearshore and GEBCO offshore beyond CRM coverage; mosaic in Stage 4. Record datum differences.

---

### S13 — Sea surface temperature (monthly)

| | |
|---|---|
| **Primary** | NOAA **OISST v2.1** (0.25°, daily; aggregate to monthly) — https://www.ncei.noaa.gov/products/optimum-interpolation-sst |
| **ERDDAP (easiest)** | NOAA CoastWatch ERDDAP — https://coastwatch.pfeg.noaa.gov/erddap/ (search "OISST" — e.g., dataset `ncdcOisst21Agg_LonPM180`) [VERIFY dataset ID] |
| **Alternative (high-res)** | NASA JPL **MUR SST** (0.01°) via PO.DAAC — https://podaac.jpl.nasa.gov/dataset/MUR-JPL-L4-GLOB-v4.1 (Earthdata login required) |
| **Format** | NetCDF / GeoTIFF |
| **Target** | `Environmental/SST_Monthly` (mosaic dataset, 12 climatology layers) |

**Steps (ERDDAP):**
1. Open the dataset's *griddap* form. Set time range 2010-01-01 to 2024-12-31, latitude 24–46, longitude −82 to −65. Choose file type `.nc`. Download (split by year if large).
2. Build monthly climatologies (12 rasters: mean of all Januaries, etc.) in Pro:
   - **Make Multidimensional Raster Layer (Multidimension)** on the NetCDF.
   - **Aggregate Multidimensional Raster (Spatial Analyst / Image Analyst)** with *Aggregation Method* = `Mean`, *Aggregation Definition* = `Interval Keyword`, *Keyword Interval* = `Recurring Monthly` [VERIFY option names in your Pro version]. Output: a multidimensional raster with 12 slices (one per calendar month).
   - Export each slice to GeoTIFF with **Subset Multidimensional Raster (Data Management)** followed by **Copy Raster (Data Management)**, naming them `SST_clim_01.tif` … `SST_clim_12.tif`.
   - Fallback without multidimensional tools: download ERDDAP's monthly product directly (if offered) as GeoTIFF and average same-month files with **Cell Statistics (Spatial Analyst)**, *Statistics type* = `Mean`.

---

### S14 — Chlorophyll-a (monthly)

| | |
|---|---|
| **Source** | NASA Ocean Color (MODIS-Aqua / VIIRS) via ERDDAP — e.g., CoastWatch West Coast ERDDAP `erdMH1chlamday` (MODIS-Aqua monthly, 4 km) [VERIFY dataset ID/availability — MODIS-Aqua is aging; VIIRS SNPP/NOAA-20 monthly products are alternatives on https://coastwatch.noaa.gov/erddap/ ] ; NASA Ocean Color — https://oceancolor.gsfc.nasa.gov/ |
| **Format** | NetCDF |
| **Target** | `Environmental/Chl_Monthly` |

Same steps as S13. Chlorophyll is log-normally distributed: store raw mg m⁻³, and symbolize on a log scale.

---

### S15 — Reference boundaries

| Layer | Source | Link |
|---|---|---|
| U.S. EEZ / maritime limits | NOAA Office of Coast Survey — U.S. Maritime Limits and Boundaries | https://nauticalcharts.noaa.gov/data/us-maritime-limits-and-boundaries.html |
| Shoreline | NOAA Medium Resolution Shoreline (1:70,000) | https://shoreline.noaa.gov/data/datasheets/medres.html [VERIFY] |
| State boundaries | U.S. Census Cartographic Boundary Files (states, 1:500k) | https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html |
| NOAA regions | NOAA Fisheries regional boundaries (GARFO / SERO) — derive from regional office definitions or Marine Cadastre | https://hub.marinecadastre.gov/ [VERIFY] |
| Federal/state waters | Marine Cadastre "Federal and State Waters" | https://hub.marinecadastre.gov/ |

### S16 — Distance-to-shore raster (derived)

Built in Stage 4 from the shoreline (see §7.2). No download.

---

### 5.17 Stage checks

- [ ] Every source S01–S16 downloaded (or requested) with terms saved.
- [ ] NARWC request sent; date and contact recorded.
- [ ] Draft `DataSourceRegistry.xlsx` has 16 rows, all fields filled.
- [ ] AIS manifest has one row per day with checksum.
- [ ] Any [VERIFY] link changes recorded in the decision log.

---

## 6. Stage 3 — Build the geodatabase

**Goal:** `02_Data/final/NARW_VesselStrike.gdb` with every feature dataset, feature class, table, domain, subtype, relationship class, topology, attribute rule, and editor-tracking setting defined, and empty and ready to load. Also an ERD and data dictionary.

### 6.1 Naming conventions (write these into `docs/naming_conventions.md`)

| Object | Convention | Example |
|---|---|---|
| Feature dataset | PascalCase noun | `Regulatory` |
| Feature class / table | PascalCase; suffix for grain | `SMA`, `Hex4_Risk_Monthly` |
| Field | snake_case; units as suffix | `mean_sog_kn`, `area_km2`, `depth_m` |
| Domain | `dm_` + PascalCase | `dm_VesselClass` |
| Relationship class | `rc_Origin_Destination` | `rc_SMA_AIS_Transits` |
| Topology | `<Dataset>_Topology` | `Regulatory_Topology` |
| Raster / mosaic | PascalCase with variable and grain | `SST_Monthly` |
| IDs | Prefix + zero-padded number | `SMA_NE_01`, `QA-S09-004`, `RUN-2026-0001` |
| Files | `lowercase_with_underscores`; dates `YYYYMMDD` | `ais_2023_01.parquet` |
| Snapshots | `NARW_VesselStrike_vYYYY.MM.gdb.zip` | `NARW_VesselStrike_v2026.11.gdb.zip` |

**Standard fields on every feature class:** `source_id` (Text 10), `load_date` (Date), `qa_status` (Text 12, domain `dm_QAStatus`), `version` (Text 12).

### 6.2 Important geodatabase rules that shape the design

1. **Rasters and mosaic datasets cannot be inside feature datasets.** They live at the geodatabase root. Keep the conceptual grouping with a prefix, e.g. `Bio_DensityModel_Monthly`, `Env_SST_Monthly`, `Env_Chl_Monthly`, `Env_Bathymetry`, `Ref_DistanceToShore`.
2. **Standalone tables also live at the root.**
3. **A topology can only include feature classes from its own feature dataset.** The "Regulatory areas must be covered by StudyArea" rule therefore needs a copy of the study-area boundary inside `Regulatory` (`Regulatory/StudyArea_Reg`).
4. **Subtype fields must be Short or Long integers.** So `platform` (Sightings) and `vessel_class` (AIS_Transits) use **integer-coded** domains.
5. **Attribute rules require Global IDs** on the dataset. Run **Add Global IDs** first.
6. **Every feature class in a feature dataset shares its CRS.**

**CRS by feature dataset [DECISION D-017]:**

| Feature dataset | CRS | Reason |
|---|---|---|
| `Reference`, `Biological`, `Regulatory`, `VesselTraffic`, `Project` | NAD 83 (4269) | Storage standard |
| `Grids`, `Analysis`, `Results` | NAD 1983 Contiguous USA Albers (5070) | Built in equal-area space; avoids reprojecting hexagons |
| Rasters (root) | 5070 | Analysis-ready; record native CRS in `DataSourceRegistry` |

### 6.3 Step 1 — Create the geodatabase

**Create File Geodatabase (Data Management)**: *Folder* `C:\GIS\NARW\02_Data\final`, *Name* `NARW_VesselStrike`, *Version* `Current`.

### 6.4 Step 2 — Create domains

Tool: **Create Domain (Data Management)**, then **Add Coded Value To Domain** for each code (or **Table To Domain** from a CSV — faster). Range domains use **Set Value For Range Domain**.

Put all coded values in `configs/domains.csv` (`domain,code,description`) and load each with **Table To Domain (Data Management)**.

| Domain | Field type | Type | Codes (code = description) |
|---|---|---|---|
| `dm_Platform` | Short | Coded | 1 = Aerial, 2 = Shipboard, 3 = Opportunistic, 4 = Acoustic-Buoy, 5 = Acoustic-Glider, 6 = Acoustic-Bottom, 9 = Unknown |
| `dm_SurveyType` | Text | Coded | Systematic, Non-systematic, Unknown |
| `dm_Reliability` | Text | Coded | Confirmed, Probable, Unverified |
| `dm_DetectionResult` | Text | Coded | Detected, Possibly detected, Not detected |
| `dm_Behavior` | Text | Coded | Feeding, Traveling, Socializing (SAG), Resting, Mother-calf, Unknown |
| `dm_Region` | Text | Coded | Northeast, Mid-Atlantic, Southeast |
| `dm_TriggerType` | Text | Coded | Visual, Acoustic |
| `dm_VesselClass` | Short | Coded | 1 = Cargo, 2 = Tanker, 3 = Passenger, 4 = Fishing, 5 = Tug/Tow, 6 = Pleasure/Sailing, 7 = Military/LawEnforcement, 8 = Other, 9 = Unknown |
| `dm_ComplianceStatus` | Text | Coded | Compliant, NonCompliant, NotApplicable, Indeterminate |
| `dm_RiskClass` | Short | Coded | 0 = No co-occurrence, 1 = Very low, 2 = Low, 3 = Moderate, 4 = High, 5 = Very high |
| `dm_LeaseStatus` | Text | Coded | Active, Planning, Withdrawn |
| `dm_QAStatus` | Text | Coded | Raw, Reviewed, Approved, Deprecated |
| `dm_YesNo` | Short | Coded | 0 = No, 1 = Yes |
| `dm_SOG` | Double | Range | 0 – 60 |
| `dm_LengthM` | Double | Range | 0 – 450 |
| `dm_Index01` | Double | Range | 0 – 1 |
| `dm_Index100` | Double | Range | 0 – 100 |
| `dm_Month` | Short | Range | 1 – 12 |

ArcPy equivalent (repeat per domain):
```python
import arcpy
gdb = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb"
arcpy.management.CreateDomain(gdb, "dm_VesselClass", "AIS vessel class", "SHORT", "CODED")
for code, desc in [(1,"Cargo"),(2,"Tanker"),(3,"Passenger"),(4,"Fishing"),(5,"Tug/Tow"),
                   (6,"Pleasure/Sailing"),(7,"Military/LawEnforcement"),(8,"Other"),(9,"Unknown")]:
    arcpy.management.AddCodedValueToDomain(gdb, "dm_VesselClass", code, desc)
arcpy.management.CreateDomain(gdb, "dm_SOG", "Speed over ground (kn)", "DOUBLE", "RANGE")
arcpy.management.SetValueForRangeDomain(gdb, "dm_SOG", 0, 60)
```

### 6.5 Step 3 — Create feature datasets

**Create Feature Dataset (Data Management)** eight times:

| Name | Spatial reference |
|---|---|
| `Reference` | 4269 |
| `Biological` | 4269 |
| `Regulatory` | 4269 |
| `VesselTraffic` | 4269 |
| `Project` | 4269 |
| `Grids` | 5070 |
| `Analysis` | 5070 |
| `Results` | 5070 |

Set XY tolerance/resolution to the defaults.

### 6.6 Step 4 — Create feature classes and fields

Use **Create Feature Class (Data Management)**, then **Add Fields (multiple) (Data Management)** with a field table — much faster than one field at a time. Keep field definitions in `schema/schema.yaml` (see §6.13) so the build is repeatable.

Field types: `TEXT(n)`, `SHORT`, `LONG`, `DOUBLE`, `DATE`, `GUID`. "+ std" = the four standard fields in §6.1.

#### Reference (4269)

| Feature class | Geometry | Fields |
|---|---|---|
| `StudyArea` | Polygon | `name` TEXT(50), `area_km2` DOUBLE + std |
| `FocusZone_50nm` | Polygon | `name` TEXT(50), `area_km2` DOUBLE + std |
| `Shoreline` | Polyline | `name` TEXT(50) + std |
| `Land` | Polygon | `name` TEXT(50) + std (dissolved states, for on-land QA) |
| `EEZ` | Polygon | `name` TEXT(50) + std |
| `StateBoundaries` | Polygon | `state_name` TEXT(50), `stusps` TEXT(2) + std |
| `NOAARegions` | Polygon | `region` TEXT(20) `dm_Region` + std |
| `Ports` | Point | `port_name` TEXT(80), `unlocode` TEXT(5), `sma_id` TEXT(20) + std |

#### Biological (4269)

**`Sightings`** (Point)

| Field | Type | Domain | Definition |
|---|---|---|---|
| `sighting_id` | TEXT(30) | | Unique ID: `<source>-<native id>` |
| `obs_datetime` | DATE | | Observation date/time in **UTC** |
| `obs_date` | DATE | | Date only (UTC) |
| `month` | SHORT | `dm_Month` | Calculated by attribute rule |
| `year` | SHORT | | Calculated by attribute rule |
| `lat`, `lon` | DOUBLE | | Decimal degrees, NAD 83 |
| `platform` | SHORT | `dm_Platform` | **Subtype field** |
| `survey_type` | TEXT(20) | `dm_SurveyType` | |
| `n_animals` | SHORT | | Best estimate of group size |
| `n_calves` | SHORT | | |
| `behavior` | TEXT(30) | `dm_Behavior` | |
| `positional_uncertainty_m` | DOUBLE | | Estimated or by platform default |
| `reliability` | TEXT(12) | `dm_Reliability` | |
| `restricted_flag` | SHORT | `dm_YesNo` | 1 = restricted source (never publish raw) |
| `hex_id_25`, `hex_id_4` | TEXT(20) | | Filled in Stage 6 |
| `qa_flags` | TEXT(100) | | Semicolon list of QA flags |
| + std | | | |

**`Recorders`** (Point): `recorder_id` TEXT(40), `deployment_start` DATE, `deployment_end` DATE, `depth_m` DOUBLE, `platform` SHORT `dm_Platform`, `project` TEXT(80) + std.

**`AcousticDetections`** (Point): `detection_id` TEXT(40), `recorder_id` TEXT(40), `detection_date` DATE, `month` SHORT, `year` SHORT, `lat` DOUBLE, `lon` DOUBLE, `detection_result` TEXT(20) `dm_DetectionResult`, `call_type` TEXT(30), `platform` SHORT `dm_Platform`, `effort_hours` DOUBLE, `hex_id_25` TEXT(20), `hex_id_4` TEXT(20), `qa_flags` TEXT(100) + std.

#### Regulatory (4269)

| Feature class | Geometry | Fields |
|---|---|---|
| `SMA` | Polygon | `sma_id` TEXT(20), `name` TEXT(80), `region` TEXT(20) `dm_Region`, `active_start_mmdd` SHORT (e.g. 1101), `active_end_mmdd` SHORT (e.g. 430), `speed_limit_kn` DOUBLE, `vessel_length_min_ft` DOUBLE, `rule_citation` TEXT(80), `effective_date` DATE + std |
| `DMA_SlowZones` | Polygon | `dma_id` TEXT(30), `name` TEXT(80), `trigger_type` TEXT(10) `dm_TriggerType`, `start_datetime` DATE, `end_datetime` DATE, `extended_flag` SHORT `dm_YesNo`, `notice_url` TEXT(255) + std |
| `CriticalHabitat` | Polygon | `unit` TEXT(10), `description` TEXT(255), `federal_register_citation` TEXT(80), `effective_date` DATE + std |
| `TrafficSeparationSchemes` | Polygon | `tss_id` TEXT(30), `name` TEXT(100), `type` TEXT(40) + std |
| `CandidateSMA` | Polygon | `cand_id` TEXT(20), `scenario_name` TEXT(60), `rationale` TEXT(255), `active_start_mmdd` SHORT, `active_end_mmdd` SHORT, `vessel_length_min_ft` DOUBLE, `author` TEXT(50) + std |
| `StudyArea_Reg` | Polygon | copy of `StudyArea` for topology |

#### VesselTraffic (4269)

**`AIS_Transits`** (Polyline) — one vessel, one UTC day, one SMA (or DMA)

| Field | Type | Domain |
|---|---|---|
| `transit_id` | TEXT(40) | `<mmsi>_<yyyymmdd>_<sma_id>` |
| `mmsi` | LONG | |
| `vessel_type` | SHORT | AIS type code |
| `vessel_class` | SHORT | `dm_VesselClass` — **subtype field** |
| `length_m` | DOUBLE | `dm_LengthM` |
| `rule_applicable` | SHORT | `dm_YesNo` (−1 allowed for indeterminate; see D-011) |
| `transit_date` | DATE | |
| `start_datetime`, `end_datetime` | DATE | |
| `n_points` | SHORT | pings inside the area |
| `length_nm` | DOUBLE | |
| `mean_sog_kn` | DOUBLE | distance-weighted |
| `max_sog_kn` | DOUBLE | `dm_SOG` |
| `nm_over_10kn` | DOUBLE | distance at > 10.5 kn |
| `pct_dist_over_10kn` | DOUBLE | |
| `area_type` | TEXT(4) | `SMA` or `DMA` |
| `sma_id` | TEXT(30) | SMA or DMA id |
| `sma_active_flag` | SHORT | `dm_YesNo` |
| `compliance_status` | TEXT(15) | `dm_ComplianceStatus` |
| + std | | |

**`AIS_Points_Sample`** (Point): `mmsi` LONG, `base_datetime` DATE, `sog` DOUBLE `dm_SOG`, `cog` DOUBLE, `heading` DOUBLE, `vessel_type` SHORT, `vessel_class` SHORT `dm_VesselClass`, `length_m` DOUBLE + std.

#### Project (4269)

**`WindLeaseAreas`** (Polygon): `lease_id` TEXT(30), `lessee` TEXT(120), `status` TEXT(12) `dm_LeaseStatus`, `area_km2` DOUBLE, `effective_date` DATE + std.

#### Grids (5070)

**`HexGrid_25km2`**, **`HexGrid_4km2`** (Polygon): `hex_id` TEXT(20), `area_km2` DOUBLE, `water_km2` DOUBLE, `centroid_lat` DOUBLE, `centroid_lon` DOUBLE, `region` TEXT(20) `dm_Region`, `in_sma_buffer` SHORT `dm_YesNo`, `in_focus_zone` SHORT `dm_YesNo`, `dist_shore_nm` DOUBLE + std.

#### Analysis and Results — stored as **tables** (root) joined to the grids by `hex_id`

Storing monthly results as tables avoids duplicating hexagon geometry 12–36 times. Polygon copies for mapping and publishing are made in Stage 14/15 with **Add Join + Export Features**.

| Table | Fields |
|---|---|
| `Hex25_WhalePresence_Monthly` | `hex_id`, `period` TEXT(10) (`clim` or `YYYY`), `month` SHORT, `sightings_n` LONG, `animals_n` LONG, `acoustic_days_effort` LONG, `acoustic_days_det` LONG, `sightings_kde_mean` DOUBLE, `acoustic_rate` DOUBLE, `density_model_mean` DOUBLE, `idx_sightings` DOUBLE, `idx_acoustic` DOUBLE, `idx_density` DOUBLE, `presence_index` DOUBLE `dm_Index01`, `method` TEXT(10) (e.g. `S+A+D`), `version` |
| `Hex4_WhalePresence_Monthly` | same as above |
| `Hex4_VesselTraffic_Monthly` | `hex_id`, `year` SHORT, `month` SHORT, `vessel_class` SHORT, `rule_applicable` SHORT, `length_band` TEXT(8) (`lt35`, `35to65`, `ge65`, `unk` — needed for the 35-ft scenario), `transit_count` LONG, `transit_nm` DOUBLE, `mean_sog_kn` DOUBLE, `nm_over_10kn` DOUBLE, `pct_nm_over_10kn` DOUBLE, `unique_vessels` LONG, `version` |
| `SMA_Compliance_Monthly` | `sma_id`, `area_type`, `year`, `month`, `vessel_class`, `transits` LONG, `transits_noncompliant` LONG, `transits_indeterminate` LONG, `pct_noncompliant` DOUBLE, `nm_total` DOUBLE, `nm_over_10kn` DOUBLE, `pct_nm_over_10kn` DOUBLE, `version` |
| `Hex4_Risk_Monthly` | `hex_id`, `year` SHORT, `month` SHORT, `presence_index` DOUBLE, `presence_norm` DOUBLE, `traffic_nm` DOUBLE, `traffic_norm` DOUBLE, `mean_sog_kn` DOUBLE, `speed_factor` DOUBLE, `risk_index` DOUBLE `dm_Index100`, `risk_class` SHORT `dm_RiskClass`, `inside_sma` SHORT, `inside_sma_active` SHORT, `inside_candidate` TEXT(100), `version` |
| `Scenario_Summary` | `scenario_name`, `cand_id`, `months_active` TEXT(40), `risk_covered` DOUBLE, `pct_coast_risk_covered` DOUBLE, `baseline_pct` DOUBLE, `delta_vs_baseline_pct` DOUBLE, `added_area_km2` DOUBLE, `added_transits` LONG, `rank` SHORT, `version` |
| `Sensitivity_Results` | `run_id`, `parameter`, `perturbation_pct`, `scenario_name`, `pct_coast_risk_covered`, `rank`, `kendall_tau_vs_base` |

### 6.7 Step 5 — Standalone tables (root)

**Create Table (Data Management)** + **Add Fields (multiple)**.

| Table | Fields |
|---|---|
| `DataSourceRegistry` | `source_id` TEXT(10), `category` TEXT(20), `provider` TEXT(120), `dataset` TEXT(200), `url` TEXT(500), `access_terms` TEXT(1000), `vintage` TEXT(50), `download_date` DATE, `native_crs` TEXT(80), `transformation_used` TEXT(80), `fitness_decision` TEXT(30), `notes` TEXT(1000) |
| `QAQC_Log` | `check_id` TEXT(20), `run_id` TEXT(20), `layer` TEXT(60), `check_name` TEXT(100), `records_checked` LONG, `records_flagged` LONG, `threshold` TEXT(50), `passed` SHORT `dm_YesNo`, `action` TEXT(200), `reviewer` TEXT(50), `review_date` DATE, `notes` TEXT(500) |
| `ProcessingLog` | `run_id` TEXT(20), `tool` TEXT(80), `parameters_json` TEXT(4000), `inputs` TEXT(1000), `outputs` TEXT(1000), `started` DATE, `finished` DATE, `status` TEXT(12), `git_commit` TEXT(40), `operator` TEXT(50) |
| `VesselClassLookup` | `ais_type_code` SHORT, `vessel_type` TEXT(80), `vessel_class` SHORT `dm_VesselClass`, `rule_applicable_flag` SHORT `dm_YesNo`, `notes` TEXT(200) |
| `MetadataStatus` | `object_name` TEXT(80), `object_type` TEXT(20), `fgdc_complete` SHORT, `iso_complete` SHORT, `validated` SHORT, `validator` TEXT(50), `last_updated` DATE, `notes` TEXT(255) |
| `VersionHistory` | `version` TEXT(12), `date` DATE, `description` TEXT(500), `archive_path` TEXT(255), `checksum_sha256` TEXT(64), `git_tag` TEXT(40) |

### 6.8 Step 6 — Subtypes

1. `Biological/Sightings`: **Set Subtype Field (Data Management)** → `platform`. Then **Add Subtype** for codes 1, 2, 3, 9 (Aerial, Shipboard, Opportunistic, Unknown). Set per-subtype defaults and domains (e.g., `positional_uncertainty_m` default 500 for Aerial, 1000 for Shipboard, 2000 for Opportunistic [DECISION]).
2. `VesselTraffic/AIS_Transits`: subtype field `vessel_class`, subtypes 1–9.
3. (Optional) `Biological/AcousticDetections`: subtype field `platform`, codes 4, 5, 6.

In Pro you can also do this interactively: right-click the feature class → **Data Design > Subtypes**.

### 6.9 Step 7 — Relationship classes

**Create Relationship Class (Data Management)**; *Relationship type* `Simple`; *Cardinality* `ONE_TO_MANY`; *Message direction* `NONE`.

| Name | Origin (primary key) | Destination (foreign key) |
|---|---|---|
| `rc_SMA_AIS_Transits` | `SMA` (`sma_id`) | `AIS_Transits` (`sma_id`) |
| `rc_SMA_Compliance` | `SMA` (`sma_id`) | `SMA_Compliance_Monthly` (`sma_id`) |
| `rc_Recorders_Detections` | `Recorders` (`recorder_id`) | `AcousticDetections` (`recorder_id`) |
| `rc_Hex4_Risk` | `HexGrid_4km2` (`hex_id`) | `Hex4_Risk_Monthly` (`hex_id`) |
| `rc_Hex4_Traffic` | `HexGrid_4km2` (`hex_id`) | `Hex4_VesselTraffic_Monthly` (`hex_id`) |
| `rc_Hex25_Presence` | `HexGrid_25km2` (`hex_id`) | `Hex25_WhalePresence_Monthly` (`hex_id`) |
| `rc_Source_<FC>` (one per source-derived class) | `DataSourceRegistry` (`source_id`) | e.g. `Sightings`, `AcousticDetections`, `SMA`, `DMA_SlowZones`, `CriticalHabitat`, `TrafficSeparationSchemes`, `WindLeaseAreas`, `Ports`, `AIS_Transits` (`source_id`) |

Relationship classes are independent of CRS: origin and destination may sit in feature datasets with different coordinate systems, as long as both are in the same geodatabase.

### 6.10 Step 8 — Topology

1. Right-click `Regulatory` → **New > Topology**, or **Create Topology (Data Management)** → `Regulatory_Topology`, cluster tolerance default.
2. **Add Feature Class To Topology**: `SMA`, `CriticalHabitat`, `StudyArea_Reg` (and `DMA_SlowZones`, `CandidateSMA` only for the coverage rule).
3. **Add Rule To Topology**:

| Rule | Classes | Note |
|---|---|---|
| Must Not Overlap | `SMA` | SMAs in the current rule do not overlap |
| Must Not Overlap | `CriticalHabitat` | Units 1 and 2 are separate |
| Must Be Covered By Feature Class Of | `SMA` by `StudyArea_Reg` | |
| Must Be Covered By Feature Class Of | `CriticalHabitat` by `StudyArea_Reg` | Unit 1 extends toward Canada — mark exceptions if StudyArea is clipped |
| Must Be Covered By Feature Class Of | `DMA_SlowZones` by `StudyArea_Reg` | |
| Must Be Covered By Feature Class Of | `CandidateSMA` by `StudyArea_Reg` | |

> **Do not** apply "Must Not Overlap" to `DMA_SlowZones` or `CandidateSMA`. Slow Zones from different dates legitimately overlap, and different scenarios overlap by design. Document this deviation from the original scope in the decision log.

4. Grid topology: **Create Topology** `Grids_Topology` in `Grids`; add `HexGrid_25km2` and `HexGrid_4km2`; rules **Must Not Overlap** and **Must Not Have Gaps** for each. The outer boundary always produces a "gap" error — mark those as **exceptions**.
5. **Validate Topology (Data Management)** after loading data (Stage 4 / Stage 6). Review errors in the **Error Inspector** (*Edit > Manage Edits > Error Inspector*), fix or mark exceptions, and log the counts in `QAQC_Log`.

### 6.11 Step 9 — Global IDs, editor tracking, attribute rules

1. **Add Global IDs (Data Management)** on: `Sightings`, `AcousticDetections`, `Recorders`, `SMA`, `DMA_SlowZones`, `CriticalHabitat`, `CandidateSMA`, `WindLeaseAreas`, `Ports`, `AIS_Transits`.
2. **Enable Editor Tracking (Data Management)** on the same classes with *Add Fields* checked (`created_user`, `created_date`, `last_edited_user`, `last_edited_date`), *Record Dates in* `UTC`.
3. **Add Attribute Rule (Data Management)** — Arcade expressions:

| Class | Rule name | Type | Trigger | Expression |
|---|---|---|---|---|
| `Sightings` | `calc_month` | Calculation, field `month` | Insert, Update | `ISOMonth($feature.obs_datetime)` |
| `Sightings` | `calc_year` | Calculation, field `year` | Insert, Update | `Year($feature.obs_datetime)` |
| `Sightings` | `calc_obs_date` | Calculation, field `obs_date` | Insert, Update | `Date(Year($feature.obs_datetime), Month($feature.obs_datetime), Day($feature.obs_datetime))` |
| `AcousticDetections` | `calc_month` / `calc_year` | Calculation | Insert, Update | as above on `detection_date` |
| `AIS_Points_Sample` | `con_sog_range` | Constraint | Insert, Update | `IIf(IsEmpty($feature.sog), true, $feature.sog >= 0 && $feature.sog <= 60)` |
| `AIS_Transits` | `con_max_sog` | Constraint | Insert, Update | `IIf(IsEmpty($feature.max_sog_kn), true, $feature.max_sog_kn >= 0 && $feature.max_sog_kn <= 60)` |
| `SMA` | `con_active_dates` | Constraint | Insert, Update | `var s=$feature.active_start_mmdd; var e=$feature.active_end_mmdd; (s>=101 && s<=1231 && e>=101 && e<=1231)` |

> Arcade's `Month()` returns **0–11**; use `ISOMonth()` for 1–12. In `Date(y, m, d)`, `m` is 0-based, which is why the `calc_obs_date` expression passes `Month()` directly.

> Attribute rules on the `Hex*_Monthly` **tables** (e.g. `presence_index BETWEEN 0 AND 1`) are enforced by the range domains `dm_Index01` / `dm_Index100` instead, because bulk loads with `Append` are simpler without rules. Run **Validate** via the domain check in QA (§8).

4. **Loading with attribute rules:** calculation rules fire on `Append`. For very large loads, temporarily **Disable Attribute Rules (Data Management)**, load, then **Enable Attribute Rules** and run **Calculate Field** for the rule fields.

### 6.12 Step 10 — ERD and data dictionary

1. **ERD:** ArcGIS Pro does not draw a full entity-relationship diagram on its own. Use one of these options:
   - Use **Export XML Workspace Document (Data Management)** (*Schema only*) and draw the ERD in **draw.io / diagrams.net** (https://app.diagrams.net/). Save `docs/ERD.drawio` (editable) and export `docs/ERD.png`.
   - Or use the **"Generate Schema Report" tool (Data Management)** in Pro 3.x, which exports a JSON/HTML/XLSX/PDF schema report of all objects, domains, subtypes, and relationships [VERIFY availability in your version]. Use its output to build the data dictionary and ERD.
2. **Data dictionary** `docs/DataDictionary.md`: one section per object — description, geometry, CRS, source_id, and a field table (name, alias, type, length, domain, units, definition, nullable, example). Generate the skeleton from the schema report, then write definitions by hand.
3. **Design rationale** `docs/GDB_Design.md`: goals, why feature datasets are split this way, CRS choices, why results are tables, domain/subtype rationale for biological data, restricted vs. public separation (`restricted_flag`, aggregation-only publishing), relationship classes, topology and exceptions, attribute rules, editor tracking, versioning/archival.

### 6.13 Step 11 — Make the build repeatable (`schema.yaml` + `BuildSchema`)

Record the full design in `schema/schema.yaml`, e.g.:

```yaml
gdb: NARW_VesselStrike.gdb
domains:
  dm_VesselClass: {type: SHORT, kind: CODED, values: {1: Cargo, 2: Tanker, 3: Passenger, 4: Fishing,
                   5: Tug/Tow, 6: Pleasure/Sailing, 7: Military/LawEnforcement, 8: Other, 9: Unknown}}
  dm_SOG: {type: DOUBLE, kind: RANGE, min: 0, max: 60}
feature_datasets:
  Regulatory: {wkid: 4269}
  Grids: {wkid: 5070}
feature_classes:
  Regulatory/SMA:
    geometry: POLYGON
    fields:
      - {name: sma_id, type: TEXT, length: 20}
      - {name: region, type: TEXT, length: 20, domain: dm_Region}
      - {name: active_start_mmdd, type: SHORT}
      - {name: active_end_mmdd, type: SHORT}
      - {name: speed_limit_kn, type: DOUBLE}
    global_ids: true
    editor_tracking: true
relationship_classes:
  - {name: rc_SMA_AIS_Transits, origin: SMA, destination: AIS_Transits, pk: sma_id, fk: sma_id, card: ONE_TO_MANY}
topologies:
  - name: Regulatory_Topology
    dataset: Regulatory
    rules:
      - {rule: "Must Not Overlap (Area)", fc: SMA}
```

A `BuildSchema` tool (Python toolbox, §16) reads this file and calls the same tools listed above. As a manual fallback, export the finished schema with **Export XML Workspace Document** (*Schema only*) to `schema/NARW_VesselStrike_schema.xml`; **Import XML Workspace Document** into an empty geodatabase recreates it on any machine.

### 6.14 Checks

- [ ] All domains, feature datasets, feature classes, tables exist with correct fields.
- [ ] Subtypes, relationship classes, topologies, Global IDs, editor tracking, attribute rules in place.
- [ ] Schema XML exported; ERD and data dictionary v1 committed.
- [ ] **Design Review 1** (Week 2) held and recorded (`docs/reviews/review_01.md`).

---

## 7. Stage 4 — Ingest data into the geodatabase

**Goal:** every source loaded into its target, projected to the storage CRS with the documented transformation, with standard fields filled and a `ProcessingLog` row per load.

### 7.1 Standard ingest recipe (point CSVs)

1. **Inspect:** add the CSV to the map as a standalone table; check columns, types, nulls. Fix obvious format issues in a *copy* in `working/`, never in `raw/`.
2. **XY Table To Point (Data Management):** *X* = longitude, *Y* = latitude, *Coordinate System* = `GCS_WGS_1984` (4326) → `working.gdb\Sxx_points`.
3. **Convert date/time:** if dates are text, **Convert Time Field (Data Management)** with the input format (e.g. `yyyy-MM-dd HH:mm:ss`) → DATE. If times are local, **Convert Time Zone (Data Management)** to UTC. Record which.
4. **Project (Data Management):** output CRS 4269, *Geographic Transformation* `WGS_1984_(ITRF00)_To_NAD_1983` → `working.gdb\Sxx_nad83`.
5. **Append (Data Management)** into the target class with *Field Matching Type* = `Use the field map to reconcile field differences`, mapping each source field to the target. Save the field map (right-click *Field Map > Save*) in `configs/fieldmaps/Sxx.fmx` [or copy the Python snippet from the geoprocessing history].
6. **Calculate Fields (multiple) (Data Management)** on the newly loaded records (select by `source_id IS NULL`): `source_id = 'Sxx'`, `load_date = today`, `qa_status = 'Raw'`, `version = 'v0.1'`, `lat`/`lon` via **Calculate Geometry Attributes** (`POINT_Y`, `POINT_X` in 4269).
7. **Log:** add a `ProcessingLog` row (use the helper below) and update the `DataSourceRegistry` row.

**ProcessingLog helper** (save as `src/narw/log.py`; call from the Python window after each step):
```python
import arcpy, json, subprocess, datetime as dt, getpass
GDB = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb"

def git_commit(repo=r"C:\GIS\NARW\03_Scripts"):
    try:
        return subprocess.check_output(["git", "-C", repo, "rev-parse", "--short", "HEAD"], text=True).strip()
    except Exception:
        return "n/a"

def log_run(tool, params, inputs, outputs, started, status="Success"):
    run_id = "RUN-" + dt.datetime.now().strftime("%Y%m%d-%H%M%S")
    with arcpy.da.InsertCursor(GDB + r"\ProcessingLog",
            ["run_id", "tool", "parameters_json", "inputs", "outputs", "started", "finished",
             "status", "git_commit", "operator"]) as cur:
        cur.insertRow([run_id, tool, json.dumps(params)[:4000], inputs, outputs, started,
                       dt.datetime.now(), status, git_commit(), getpass.getuser()])
    return run_id
```

### 7.2 Per-source instructions

**Reference layers (S15, S16) — load first.**
1. States (Census): **Project** to 4269 (source is NAD 83 already — no transformation needed) → `Reference/StateBoundaries`. Select Atlantic coastal states (ME–FL) plus neighbors.
2. `Reference/Land` = **Dissolve** of states.
3. Shoreline (NOAA medium-resolution): Project → `Reference/Shoreline`.
4. EEZ: Project → `Reference/EEZ` (Atlantic portion).
5. `StudyArea`: **Clip** EEZ to a rectangle (−82, 24, −65, 45.5) created with **Create Fishnet** (1 row × 1 column) or by drawing, then **Erase (Analysis)** `Land` → `Reference/StudyArea`. Calculate `area_km2` with **Calculate Geometry Attributes** (*Area (geodesic)*, square kilometers). Copy to `Regulatory/StudyArea_Reg`.
6. `FocusZone_50nm`: **Buffer (Analysis)** `Shoreline`, `50 NauticalMiles`, *Method* `Geodesic`, dissolve all → **Pairwise Intersect** with `StudyArea` → `Reference/FocusZone_50nm`.
7. `NOAARegions`: build three polygons (Northeast = Maine–New York/Block Island; Mid-Atlantic = New Jersey–North Carolina; Southeast = South Carolina–Florida) by splitting `StudyArea` with lines drawn at the regional boundaries [DECISION — record the latitude/longitude cut lines used; align with GARFO/SERO jurisdiction].
8. **S16 distance-to-shore:** **Distance Accumulation (Spatial Analyst)** or **Euclidean Distance** (older) from `Land` (projected to 5070), cell size `1000`, *Environment > Mask* = `StudyArea` → multiply by `0.000539957` (m → nm) with **Raster Calculator** → root raster `Ref_DistanceToShore_nm`.

**Sightings (S01, S04).**
- Follow §7.1. Map source codes to domains in a lookup (e.g., RWSAS "AERIAL" → 1). Put the mapping in `configs/sightings_codes.csv` and apply with **Add Join + Calculate Field**, or the field calculator with a Python dictionary.
- Set `restricted_flag = 1` for NARWC records, `0` otherwise.
- `sighting_id` = `source_id + "-" + native id` (or a hash of datetime+lat+lon if no native id).
- `survey_type`: `Systematic` for dedicated aerial/shipboard surveys; `Non-systematic` for opportunistic; `Unknown` otherwise.
- Load each source separately so each gets its own `ProcessingLog` row.

**Acoustic (S02).**
1. From the PACM CSV, build the **deployment list**: **Summary Statistics (Analysis)** grouped by deployment/recorder ID with MIN(date), MAX(date), FIRST(lat), FIRST(lon), FIRST(platform) → `XY Table To Point` → `Recorders`.
2. Load every daily record (including "Not detected") as a point in `AcousticDetections`. For gliders, use the daily position. `effort_hours` = 24 unless the source gives partial-day effort.

**Density model (S03).**
1. For each monthly raster: **Project Raster (Data Management)** to 5070, *Resampling* `BILINEAR`, cell size = native (≈ 5–10 km) [record]. Divide by 100 if units are per 100 km² (**Raster Calculator**: `"in" / 100`) → animals/km².
2. **Create Mosaic Dataset (Data Management)** at the gdb root: `Bio_DensityModel_Monthly`, CRS 5070.
3. **Add Rasters To Mosaic Dataset**.
4. Add fields `month` SHORT, `model_version` TEXT, `source_id` TEXT, `units` TEXT to the mosaic footprint table; populate with **Calculate Field** (parse month from the raster name).
5. Repeat for CV rasters (`Bio_DensityModelCV_Monthly`).

**Regulatory (S05–S08).**
- SMA: Project → append to `Regulatory/SMA`. Fill `sma_id` (e.g. `SMA_NE_CCB`, `SMA_NE_ORP`, `SMA_NE_GSC`, `SMA_MA_BIS`, `SMA_MA_NYNJ`, `SMA_MA_DEL`, `SMA_MA_CHES`, `SMA_MA_MHCB`, `SMA_MA_WILM_BRUN`, `SMA_SE`), `region`, `active_start_mmdd`, `active_end_mmdd`, `speed_limit_kn = 10`, `vessel_length_min_ft = 65`, `rule_citation = '50 CFR 224.105'`, `effective_date` — from Appendix D and the eCFR text.
- DMA/Slow Zones: from §5 S06 → project → append.
- Critical habitat: Project → append; `unit` = `Unit 1` / `Unit 2`.
- TSS: Project → append. If lanes are lines, buffer to polygons only if needed for display — keep original geometry type [DECISION].
- **Validate Topology** `Regulatory_Topology` → log results.

**AIS (S09).** Processed in Stage 8 (§11); results are loaded there.

**Wind leases (S10).** Project → append to `Project/WindLeaseAreas`; map status to `dm_LeaseStatus`; compute `area_km2` (geodesic).

**Ports (S11).** Project → append to `Reference/Ports`; filter to Atlantic ports; add `unlocode`; assign `sma_id` with **Spatial Join (Analysis)** (*Match option* `CLOSEST`, *Search radius* `25 NauticalMiles`) to `SMA`.

**Bathymetry (S12).**
1. **Mosaic To New Raster (Data Management)** of CRM tiles + GEBCO subset, *Mosaic Operator* `FIRST` with CRM listed first (so CRM wins where both exist), pixel type 32-bit float.
2. **Project Raster** to 5070, cell size `500` m, `BILINEAR`.
3. Convert elevation to depth: **Raster Calculator** `SetNull("elev" >= 0, -1 * "elev")` → root raster `Env_Bathymetry` (depth_m positive down; land = NoData).
4. Record the vertical datum mix in metadata.

**SST and chlorophyll (S13, S14).** 12 climatology GeoTIFFs each (§5) → **Project Raster** to 5070 (`BILINEAR`) → mosaic datasets `Env_SST_Monthly`, `Env_Chl_Monthly` with `month`, `years` (`2010-2024`), `source_id` fields.

**DataSourceRegistry.** **Excel To Table (Conversion)** the draft spreadsheet → **Append** to `DataSourceRegistry`.

**VesselClassLookup.** Build `configs/vessel_classes.csv` from Appendix C → **Append**.

### 7.3 Checks

- [ ] Row counts in each target equal source rows minus documented exclusions (log both).
- [ ] Every feature has `source_id`, `load_date`, `qa_status = 'Raw'`, `version`.
- [ ] Every load has a `ProcessingLog` row.
- [ ] Topology validated; errors logged.

---

## 8. Stage 5 — QA/QC and Fitness-for-Use Assessment

**Goal:** every check below run, results in `QAQC_Log`, flagged records marked (`qa_flags`), reviewer sign-off, and a one-page Fitness-for-Use Assessment per source.

### 8.1 QAQC_Log conventions

- `check_id` = `QA-<source>-<nnn>` (e.g. `QA-S01-003`).
- `run_id` = the QA run (e.g. `QA-RUN-1`, `QA-RUN-2`). Re-run all checks after fixes.
- `passed` = 1 if `records_flagged` ≤ threshold.
- After review: set `qa_status` of records to `Reviewed` then `Approved`; flagged-and-excluded records to `Deprecated` (don't delete — keep the audit trail).

### 8.2 Checks for all layers

| # | Check | How (tool / method) | Threshold |
|---|---|---|---|
| A1 | CRS equals storage CRS | Catalog > Properties > Source > Spatial Reference; or `arcpy.Describe(fc).spatialReference.factoryCode` | 100% |
| A2 | Transformation recorded | `DataSourceRegistry.transformation_used` not null where native CRS ≠ 4269 | 100% |
| A3 | Geometry validity | **Check Geometry (Data Management)** → output table; then **Repair Geometry** (*Delete features with null geometry* = off first — inspect, then decide) | 0 errors after repair |
| A4 | Null geometry / empty parts | Included in A3 output | 0 |
| A5 | Domain conformance | Select features where the field value is not in the domain, e.g. `platform NOT IN (1,2,3,4,5,6,9)`; or **Validate** in the Edit tab (*Edit > Validate > Validate Features* for attribute rules/domains) | Report; **block** if a key field fails |
| A6 | Required-field nulls | Select By Attributes `field IS NULL` for key fields (`sighting_id`, `obs_datetime`, `sma_id`, `mmsi`...) | 0 on key fields |
| A7 | Duplicate IDs | **Summary Statistics** COUNT grouped by ID field → select COUNT > 1 | 0 |

**Script for A1, A3, A6 across all classes** (`src/narw/qa_generic.py`):
```python
import arcpy, datetime as dt
GDB = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb"
arcpy.env.workspace = GDB
KEYS = {"Sightings": ["sighting_id", "obs_datetime"], "SMA": ["sma_id"], "AIS_Transits": ["transit_id", "mmsi"]}

def log(check_id, layer, name, n, flagged, threshold, passed, notes=""):
    with arcpy.da.InsertCursor("QAQC_Log", ["check_id","run_id","layer","check_name","records_checked",
            "records_flagged","threshold","passed","reviewer","review_date","notes"]) as c:
        c.insertRow([check_id, "QA-RUN-1", layer, name, n, flagged, threshold, int(passed), None, None, notes])

for fds in arcpy.ListDatasets(feature_type="Feature") + [""]:
    for fc in arcpy.ListFeatureClasses(feature_dataset=fds):
        path = f"{GDB}\\{fds}\\{fc}" if fds else f"{GDB}\\{fc}"
        n = int(arcpy.management.GetCount(path)[0])
        wkid = arcpy.Describe(path).spatialReference.factoryCode
        expected = 5070 if fds in ("Grids", "Analysis", "Results") else 4269
        log(f"QA-ALL-{fc}-CRS", fc, "CRS matches storage CRS", n, 0 if wkid == expected else n, "100%", wkid == expected)
        out = arcpy.management.CheckGeometry(path, f"memory\\cg_{fc}")
        bad = int(arcpy.management.GetCount(out)[0])
        log(f"QA-ALL-{fc}-GEOM", fc, "Geometry validity", n, bad, "0", bad == 0)
        for k in KEYS.get(fc, []):
            nulls = sum(1 for _ in arcpy.da.SearchCursor(path, [k], f"{k} IS NULL"))
            log(f"QA-ALL-{fc}-NULL-{k}", fc, f"Null {k}", n, nulls, "0", nulls == 0)
```

### 8.3 Sightings checks

| # | Check | Method | Action |
|---|---|---|---|
| B1 | **Duplicates** — same source, within ±5 min and ±500 m | Script below (Generate Near Table + time comparison) | Flag `DUP`; reviewer merges (keep the record with more attributes; set the other `Deprecated`) |
| B2 | **Cross-source duplicates** (e.g. RWSAS vs. WhaleMap same event) | Same script without the same-source condition, ±15 min, ±2 km [DECISION] | Flag `XDUP`; keep one for analysis (priority NARWC > RWSAS > WhaleMap > OBIS) |
| B3 | **On land** | Project `Land` to 5070, **Pairwise Buffer** `-1 Kilometers` (inland by 1 km); **Select Layer By Location** Sightings `INTERSECT` → flag | Flag `LAND`; exclude |
| B4 | **> 300 nm offshore** | **Near (Analysis)** to `Shoreline` (geodesic) → `NEAR_DIST > 555600` m | Flag `FAR`; review |
| B5 | **Depth ≤ 0 / NoData** | **Extract Multi Values To Points (Spatial Analyst)** from `Env_Bathymetry` → NoData | Flag `DEPTH`; overlaps B3 |
| B6 | **Future dates** | `obs_datetime > CURRENT_DATE` | Flag `DATE`; exclude |
| B7 | **Unusual month for region** | e.g. Southeast region, months 6–10 (calving season is Nov–Apr) | Flag `SEASON`; **review, do not exclude** (could be real) |
| B8 | **n_animals plausibility** | `n_animals <= 0 OR n_animals > 100`; `n_calves > n_animals` | Flag `COUNT` |
| B9 | **Positional precision** | Count decimals of lat/lon; < 2 decimals = ±1 km+ | Set `positional_uncertainty_m` accordingly |

**Duplicate script** (B1/B2):
```python
import arcpy, datetime as dt
arcpy.env.overwriteOutput = True
fc = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb\Biological\Sightings"
arcpy.analysis.GenerateNearTable(fc, fc, r"memory\near", "500 Meters", closest="ALL", method="GEODESIC")
attrs = {r[0]: (r[1], r[2]) for r in arcpy.da.SearchCursor(fc, ["OID@", "obs_datetime", "source_id"])}
pairs = []
for in_id, near_id in arcpy.da.SearchCursor(r"memory\near", ["IN_FID", "NEAR_FID"]):
    if in_id < near_id:
        (t1, s1), (t2, s2) = attrs[in_id], attrs[near_id]
        if s1 == s2 and abs((t1 - t2).total_seconds()) <= 300:
            pairs.append((in_id, near_id))
flag = {oid for p in pairs for oid in p}
with arcpy.da.UpdateCursor(fc, ["OID@", "qa_flags"]) as cur:
    for oid, q in cur:
        if oid in flag:
            cur.updateRow([oid, ";".join(filter(None, [q, "DUP"]))])
print(len(pairs), "duplicate pairs")
```

### 8.4 Acoustic checks

| # | Check | Method |
|---|---|---|
| C1 | Detection outside its recorder's deployment window | **Add Join** `AcousticDetections` → `Recorders` on `recorder_id`; select `detection_date < deployment_start OR detection_date > deployment_end` → flag `WINDOW` |
| C2 | Detection with no recorder | Join + select `Recorders.recorder_id IS NULL` → flag `ORPHAN` |
| C3 | Duplicate recorder-day | Summary Statistics COUNT by `recorder_id, detection_date` > 1 |
| C4 | Recorder on land / depth | as B3/B5 on `Recorders` |

### 8.5 AIS checks (run inside the AIS pipeline, §11.4; log counts here)

| # | Check | Threshold / action |
|---|---|---|
| D1 | MMSI validity: 9 digits, first digit 2–7 (ship MIDs), not in {0, 111111111, 123456789, 999999999, 1234567xx} | Drop; log count |
| D2 | Exact duplicate rows (MMSI + timestamp) | Drop; log |
| D3 | SOG spike: SOG > 40 kn for non-passenger classes (high-speed ferries can exceed 30 kn) | Drop ping; log |
| D4 | Position jump: implied speed between consecutive pings > 60 kn | Start a new segment; drop the isolated ping if both neighbors jump; log |
| D5 | Missing or zero length | Keep; class by type; applicability per D-011; **report share** |
| D6 | Out-of-bbox / on land (ports and rivers) | Drop points on `Land` for transit analysis; log |
| D7 | Reported SOG vs. implied speed disagreement > 2 kn | Use implied speed (D-009); report share |

### 8.6 Regulatory checks

| # | Check | Method | Threshold |
|---|---|---|---|
| E1 | `active_start_mmdd`, `active_end_mmdd` parse to valid dates | Constraint rule + selection | 100% |
| E2 | No overlaps within `SMA`, `CriticalHabitat` | Topology | 0 errors |
| E3 | Boundaries match published notices | For 5 SMAs, compare 3+ vertices with eCFR coordinates (**Measure** tool or `Calculate Geometry`) | ≤ 100 m difference |
| E4 | Active dates match eCFR | Manual table comparison | 100% |
| E5 | Slow Zone archive completeness | Count notices by year vs. NOAA announcements | Report gaps |

### 8.7 Raster checks

| # | Check | Method | Threshold |
|---|---|---|---|
| F1 | NoData set correctly | Raster Properties > NoData; **Get Raster Properties** | 100% |
| F2 | Extent covers StudyArea | Compare extents | 100% |
| F3 | Cell size as documented | Get Raster Properties `CELLSIZEX` | 100% |
| F4 | Value ranges | **Calculate Statistics** then MIN/MAX: SST −2 to 35 °C; chlorophyll > 0 and < 100 mg m⁻³; density ≥ 0; depth ≥ 0 | 100% |

### 8.8 Derived-product checks (run after Stages 8–10)

| # | Check | Threshold |
|---|---|---|
| G1 | Transits per SMA vs. AIS points inside the SMA: `SUM(n_points)` equals the point count from a direct point-in-polygon count | ±1% |
| G2 | `Hex4_VesselTraffic_Monthly` total nm equals the sum of all leg lengths inside the focus zone | ±1% |
| G3 | `presence_index`, `risk_index` within range | 100% |
| G4 | Every hex-month present (no missing rows) | 100% |
| G5 | Re-run reproducibility: row counts identical on re-run | 100% |

### 8.9 Fitness-for-Use Assessment

Write `docs/Fitness_for_Use_Assessment.md` with **one page per source** using template F.2 (Appendix F). For each: description and purpose; completeness (spatial, temporal, effort); currency and update frequency; spatial reference and transformation; lineage; attribute quality (nulls, domains, consistency — with numbers from `QAQC_Log`); positional accuracy; known limitations and biases; appropriate and inappropriate uses; **decision: Use / Use with caveats / Do not use**. Put the decision in `DataSourceRegistry.fitness_decision`.

**Standard caveats to evaluate:**
- Sightings are **effort-biased**: concentrated where and when surveys fly; opportunistic records near ports and whale-watch areas.
- Acoustic detections indicate **presence, not abundance**; detection range varies with noise and conditions.
- Density models: modeled, with CV uncertainty; era-specific.
- AIS: Class B and non-broadcasting vessels under-represented; vessels < 65 ft often not required to carry AIS; 1-minute sampling; some positions spoofed or erroneous.
- Regulatory: must match current Federal Register / eCFR text.

### 8.10 Checks

- [ ] Every check in §8.2–8.7 run and logged; reviewer sign-off recorded (`reviewer`, `review_date`).
- [ ] `qa_status` updated on all records.
- [ ] 16 fitness assessments written; decisions in the registry.
- [ ] M03 data-fitness map inputs identified (coverage gaps).

---

## 9. Stage 6 — Hexagon analysis grids

**Goal:** `Grids/HexGrid_25km2` and `Grids/HexGrid_4km2` with all attributes, and every sighting/detection tagged with its hex IDs.

### 9.1 Steps

1. Project `StudyArea` and `FocusZone_50nm` to 5070 in `working.gdb`.
2. **Generate Tessellation (Data Management)**:
   - 25 km²: *Extent* = StudyArea (5070); *Shape Type* `HEXAGON`; *Size* `25 SquareKilometers`; *Spatial Reference* 5070 → `working.gdb\hex25_raw`.
   - 4 km²: *Extent* = FocusZone_50nm (5070); `HEXAGON`; `4 SquareKilometers` → `hex4_raw`. Add candidate-scenario extents if they extend beyond 50 nm.
3. Keep only hexes that touch water: **Select Layer By Location** (`INTERSECT` StudyArea / FocusZone) → **Export Features** into `Grids/HexGrid_25km2` / `Grids/HexGrid_4km2` (via **Append** to keep the schema).
4. `hex_id`: **Calculate Field** `"H25_" + str(!GRID_ID!)` (and `H4_` for the 4 km² grid). If GRID_ID is not carried over, use `!OBJECTID!`.
5. `area_km2`: Calculate Geometry Attributes (*Area*, square km, planar in 5070).
6. `water_km2`: **Tabulate Intersection (Analysis)** hex × StudyArea → join `AREA` → convert to km².
7. `centroid_lat`, `centroid_lon`: Calculate Geometry Attributes, *Centroid y/x*, coordinate system **4269**.
8. `region`: **Spatial Join** (hex centroid within `NOAARegions`) — or **Feature To Point** (inside) then spatial join, then join back on `hex_id`.
9. `in_sma_buffer`: **Select Layer By Location** `WITHIN_A_DISTANCE` 10 km of `SMA` ∪ `DMA_SlowZones` ∪ `CriticalHabitat` → Calculate `1`; else `0`.
10. `in_focus_zone` (25 km² grid): intersect with FocusZone → 1/0.
11. `dist_shore_nm`: **Zonal Statistics as Table (Spatial Analyst)** MEAN of `Ref_DistanceToShore_nm` by `hex_id` → join → calculate.
12. **Validate Topology** `Grids_Topology`; mark outer-boundary gap errors as exceptions; log.
13. Tag points: **Spatial Join** `Sightings` → `HexGrid_25km2` (`hex_id` → `hex_id_25`) and to `HexGrid_4km2` (→ `hex_id_4`); same for `AcousticDetections`. (Spatial Join outputs a new class — join back on `sighting_id` and **Calculate Field**, or use **Add Spatial Join** in Pro 3.x which joins in place [VERIFY].)

### 9.2 Checks

- [ ] Hex counts recorded (expect roughly StudyArea km² / 25 and FocusZone km² / 4).
- [ ] No duplicate `hex_id`; no overlaps or internal gaps.
- [ ] Every sighting inside the StudyArea has `hex_id_25`.

---

## 10. Stage 7 — Phase A: Whale presence surfaces

**Goal:** a monthly **presence index (0–1)** for every hexagon, at 25 km² coast-wide and 4 km² in the focus zone, combining three evidence streams.

**Outputs:** `Hex25_WhalePresence_Monthly`, `Hex4_WhalePresence_Monthly` (tables); monthly presence rasters for cartography.

### 10.1 Method overview

| Component | Weight | Source | Raw metric per hex-month | Index |
|---|---|---|---|---|
| Sightings | 0.4 | `Sightings` (approved, de-duplicated) | Mean kernel density of animals (animals/km²/yr) — or sightings per unit effort (SPUE) where effort exists | `idx_sightings` |
| Acoustic | 0.2 | `AcousticDetections` | Detection rate = detected days ÷ monitored days, within 20 km of a recorder | `idx_acoustic` |
| Density model | 0.4 | `Bio_DensityModel_Monthly` | Mean modeled density (animals/km²) | `idx_density` |

**Index rule (applied separately to each component, within each month):**
- `idx = 0` where the raw metric is 0.
- Otherwise `idx` = **percentile rank** of the value among all **non-zero** values that month, scaled to (0, 1].
- `idx = NULL` where the component has **no coverage** (e.g. no recorder within 20 km; outside the density model extent).

**Combination:** `presence_index = Σ(wᵢ × idxᵢ) / Σ(wᵢ)` over the components that are **not NULL** (weights re-normalized). `method` records which were used: e.g. `S+A+D`, `S+D`, `S`.

**Time basis:**
- **Climatology** (`period = 'clim'`): all years 2010–present pooled by calendar month. This is the primary input to the risk index (D-013).
- **Aligned years** (`period = '2022'`, `'2023'`, `'2024'`): same method with only that year's records. Used in sensitivity analysis.

### 10.2 Step A1 — Prepare inputs

1. Make a layer of analysis-ready sightings: **Make Feature Layer** on `Sightings` with the definition query
   `qa_status = 'Approved' AND (qa_flags IS NULL OR (qa_flags NOT LIKE '%DUP%' AND qa_flags NOT LIKE '%LAND%' AND qa_flags NOT LIKE '%DATE%')) AND year >= 2010`.
2. **Project** that layer to 5070 → `working.gdb\sight_5070`.
3. Same for `AcousticDetections` → `acou_5070` (keep all detection results, including "Not detected").
4. Environment settings for this stage: *Output Coordinate System* 5070; *Mask* = `StudyArea` (5070); *Snap Raster* = first kernel output; *Extent* = StudyArea.

### 10.3 Step A2 — Sightings component

For each month m = 1…12 (climatology):

1. **Select Layer By Attribute** `month = m` on `sight_5070`.
2. **Kernel Density (Spatial Analyst)**:
   - *Population field*: `n_animals`
   - *Output cell size*: `5000` (for the 25 km² grid) and, in a second run, `2000` (for the 4 km² grid)
   - *Search radius*: `25000` (25 km, D-015)
   - *Area units*: `SQUARE_KILOMETERS`
   - *Output values*: `DENSITIES`
   - *Method*: `PLANAR` (data are in an equal-area projection)
   - Output: `working.gdb\kde_sight_m01_5k` … `_m12_5k` and `_2k`.
3. Convert to per-year density: **Raster Calculator** `"kde_sight_m01_5k" / N_years` where `N_years` = number of years in the period (e.g. 2010–2025 → 16).
4. **Zonal Statistics as Table (Spatial Analyst)**: *Zone data* `HexGrid_25km2`, *Zone field* `hex_id`, *Value raster* the 5k KDE, *Statistics type* `MEAN` → `zs_sight_m01_h25`. Repeat with `HexGrid_4km2` and the 2k KDE → `zs_sight_m01_h4`.
5. Counts: **Summary Statistics (Analysis)** on `sight_5070` with statistics `COUNT sighting_id`, `SUM n_animals`, case fields `hex_id_25; month` → `cnt_sight_h25` (and `hex_id_4; month` → `cnt_sight_h4`).

**Automate the 12-month loop in ModelBuilder** (M1, §16.1) with the **Iterate Field Values** iterator on `month`, or run this ArcPy loop:
```python
import arcpy
from arcpy.sa import KernelDensity, ZonalStatisticsAsTable, Raster
arcpy.CheckOutExtension("Spatial")
arcpy.env.workspace = r"C:\GIS\NARW\02_Data\working\working.gdb"
arcpy.env.outputCoordinateSystem = arcpy.SpatialReference(5070)
arcpy.env.mask = "StudyArea_5070"
arcpy.env.overwriteOutput = True
GDB = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb"
N_YEARS = 16
for cell, grid, tag in [(5000, GDB + r"\Grids\HexGrid_25km2", "h25"), (2000, GDB + r"\Grids\HexGrid_4km2", "h4")]:
    arcpy.env.cellSize = cell
    for m in range(1, 13):
        lyr = arcpy.management.MakeFeatureLayer("sight_5070", "s_lyr", f"month = {m}")
        if int(arcpy.management.GetCount(lyr)[0]) == 0:
            continue
        kde = KernelDensity(lyr, "n_animals", cell, 25000, "SQUARE_KILOMETERS", "DENSITIES", "PLANAR")
        kde_py = kde / N_YEARS
        kde_py.save(f"kde_sight_m{m:02d}_{tag}")
        ZonalStatisticsAsTable(grid, "hex_id", kde_py, f"zs_sight_m{m:02d}_{tag}", "DATA", "MEAN")
```

**Effort adjustment (only if effort track lines are available — e.g. NARWC).** For each month:
1. **Line Density (Spatial Analyst)** of survey track lines (km of trackline per km²), same cell size and 25 km radius → `eff_m01`.
2. **Raster Calculator**: `Con("eff_m01" > 0.01, "kde_sight_m01" / "eff_m01")` → SPUE raster (animals per km of effort). Use SPUE as the sightings metric where effort exists and the presence-only KDE elsewhere; record which in `method` (e.g. `Se+A+D` for effort-adjusted).
3. If no effort data is obtained, record D-007: sightings are **presence-only** and the limitation is stated on every product.

### 10.4 Step A3 — Acoustic component

1. Code detections: add field `det_score` DOUBLE — `Detected` = 1, `Possibly detected` = 0.5 [DECISION], `Not detected` = 0.
2. **Pairwise Buffer (Analysis)** `acou_5070`, `20 Kilometers` (D-014), *Method* `GEODESIC` → `acou_buf`. (Each recorder-day becomes a 20 km disk.)
3. **Pairwise Intersect (Analysis)** `acou_buf` × `HexGrid_25km2` → `acou_x_h25` (one row per recorder-day per hex it overlaps). Repeat for `HexGrid_4km2`.
4. **Summary Statistics**: case fields `hex_id; month`, statistics `COUNT detection_id` (monitored recorder-days, `acoustic_days_effort`) and `SUM det_score` (`acoustic_days_det`). Output: `acou_sum_h25` and `acou_sum_h4` (the names the combine script in §10.6 expects).
5. `acoustic_rate = acoustic_days_det / acoustic_days_effort`. Hexes with no rows → `NULL` (no coverage).

> Performance tip: if the buffer-intersect is slow, first **Summary Statistics** detections by `recorder_id, month` (effort days, detection score sum), then buffer the **recorder** points once and intersect once.

### 10.5 Step A4 — Density-model component

For each month: **Zonal Statistics as Table** (*Zone* hex grid, *Value* the month's density raster from `Bio_DensityModel_Monthly` — select by `month` in the mosaic or use the projected GeoTIFF directly, *Statistic* `MEAN`) → `zs_dens_m01_h25`, `zs_dens_m01_h4`. Hexes outside the model extent → `NULL`.

### 10.6 Step A5 — Combine into the presence index

Assemble all month tables into one long table and compute the indices with pandas (run in the `narw-py3` environment). Save as `src/narw/presence.py`:

```python
import arcpy, pandas as pd, numpy as np

W = {"sightings": 0.4, "acoustic": 0.2, "density": 0.4}   # configs/analysis.yaml

def table_to_df(path, fields):
    return pd.DataFrame(arcpy.da.TableToNumPyArray(path, fields, null_value=np.nan))

def pct_rank_nonzero(s):
    """0 stays 0; non-zero values get percentile rank in (0,1]; NaN stays NaN."""
    out = s.copy()
    nz = s > 0
    out[nz] = s[nz].rank(method="average", pct=True)
    return out

def build(grid_tag, hex_fc, period="clim"):
    ws = r"C:\GIS\NARW\02_Data\working\working.gdb"
    hexes = table_to_df(hex_fc, ["hex_id"])["hex_id"]
    frames = []
    for m in range(1, 13):
        df = pd.DataFrame({"hex_id": hexes, "month": m})
        for comp, tbl in [("sightings_kde_mean", f"zs_sight_m{m:02d}_{grid_tag}"),
                          ("density_model_mean", f"zs_dens_m{m:02d}_{grid_tag}")]:
            if arcpy.Exists(f"{ws}\\{tbl}"):
                z = table_to_df(f"{ws}\\{tbl}", ["hex_id", "MEAN"]).rename(columns={"MEAN": comp})
                df = df.merge(z, on="hex_id", how="left")
            else:
                df[comp] = np.nan
        frames.append(df)
    df = pd.concat(frames)
    acou = table_to_df(f"{ws}\\acou_sum_{grid_tag}", ["hex_id", "month", "COUNT_detection_id", "SUM_det_score"])
    acou["acoustic_rate"] = acou["SUM_det_score"] / acou["COUNT_detection_id"]
    df = df.merge(acou.rename(columns={"COUNT_detection_id": "acoustic_days_effort",
                                       "SUM_det_score": "acoustic_days_det"}), on=["hex_id", "month"], how="left")
    df["sightings_kde_mean"] = df["sightings_kde_mean"].fillna(0)   # sightings layer covers the whole area
    g = df.groupby("month")
    df["idx_sightings"] = g["sightings_kde_mean"].transform(pct_rank_nonzero)
    df["idx_acoustic"] = g["acoustic_rate"].transform(pct_rank_nonzero)
    df["idx_density"] = g["density_model_mean"].transform(pct_rank_nonzero)
    num = sum(W[k] * df[f"idx_{k}"].fillna(0) for k in W)
    den = sum(W[k] * df[f"idx_{k}"].notna() for k in W)
    df["presence_index"] = (num / den).clip(0, 1)
    df["method"] = (np.where(df.idx_sightings.notna(), "S", "") +
                    np.where(df.idx_acoustic.notna(), "+A", "") +
                    np.where(df.idx_density.notna(), "+D", ""))
    df["period"] = period
    df["version"] = "v1.0"
    return df
```
Write the DataFrame to CSV, then **Export Table** / **Append** into `Hex25_WhalePresence_Monthly` and `Hex4_WhalePresence_Monthly`. (Or write directly with `arcpy.da.InsertCursor`.)

### 10.7 Step A6 — Cartographic rasters

For each month: **Add Join** the month's rows to the hex grid → **Polygon To Raster (Conversion)** on `presence_index`, cell size 2000 → `presence_m01.tif`… (optional; the map series can symbolize hexes directly).

### 10.8 Checks

- [ ] 12 months × every hex present in both tables (G4); `presence_index` within 0–1 (G3).
- [ ] Share of hexes by `method` reported (coverage of acoustic and density components).
- [ ] Visual sanity check: Cape Cod Bay high in Feb–Apr; Southeast calving grounds high in Dec–Mar; Southern New England high in winter–spring (post-2010 pattern). Record observations.
- [ ] ProcessingLog rows for every KDE/Zonal/combine run.

---

## 11. Stage 8 — Phase B: AIS vessel traffic processing

**Goal:** turn ~3 years of national 1-minute AIS pings into (1) clean **legs** (consecutive-ping line segments), (2) per-vessel-per-day **transits** through each SMA and Slow Zone with speed statistics, (3) **hex-month traffic** summaries, and (4) a 1% point sample.

**Why outside ArcGIS:** the volume (billions of pings) is too large for geoprocessing tools. **DuckDB** (an in-process SQL engine with a spatial extension) processes it on a laptop, one month at a time.

**Outputs:** `VesselTraffic/AIS_Transits`, `VesselTraffic/AIS_Points_Sample`, `Hex4_VesselTraffic_Monthly`, QA counts.

### 11.1 Inputs exported from the geodatabase (once)

Export these to `02_Data/working/ais_inputs/` as **GeoPackage in EPSG:5070** (**Export Features** with *Environment > Output Coordinate System* = 5070):

| File | From | Key fields |
|---|---|---|
| `sma_5070.gpkg` | `Regulatory/SMA` | `sma_id`, `active_start_mmdd`, `active_end_mmdd` |
| `dma_5070.gpkg` | `Regulatory/DMA_SlowZones` | `dma_id`, `start_datetime`, `end_datetime` |
| `cand_5070.gpkg` | `Regulatory/CandidateSMA` (Stage 11) | `cand_id`, `active_start_mmdd`, `active_end_mmdd` |
| `hex4_5070.gpkg` | `Grids/HexGrid_4km2` | `hex_id` |
| `land_5070.gpkg` | `Reference/Land` | — |
| `focus_5070.gpkg` | `Reference/FocusZone_50nm` | — |
| `vessel_classes.csv` | `configs/` (Appendix C) | `ais_type_code, vessel_class, rule_applicable_flag` |

### 11.2 Step B1 — Zip → filtered Parquet (one month at a time)

`ais_preprocess/01_zip_to_parquet.py`:
```python
"""Unzip each daily AIS file, keep study-area rows, write monthly Parquet, delete the CSV."""
import duckdb, zipfile, tempfile, os, glob, csv

RAW = r"C:\GIS\NARW\02_Data\raw\S09_ais"
OUT = r"C:\GIS\NARW\02_Data\working\ais_parquet"
COUNTS = r"C:\GIS\NARW\03_Scripts\ais_preprocess\qa_counts_ingest.csv"
BBOX = dict(xmin=-82.0, xmax=-65.0, ymin=24.0, ymax=46.0)

con = duckdb.connect()
con.execute("SET memory_limit='12GB'; SET threads=8;")

def month_files(year, month):
    return sorted(glob.glob(os.path.join(RAW, str(year), f"AIS_{year}_{month:02d}_*.zip")))

def process(year, month):
    out_file = os.path.join(OUT, f"ais_{year}_{month:02d}.parquet")
    if os.path.exists(out_file):
        return
    os.makedirs(OUT, exist_ok=True)
    rows_in = rows_out = 0
    with tempfile.TemporaryDirectory() as tmp:
        for z in month_files(year, month):
            with zipfile.ZipFile(z) as zf:
                zf.extractall(tmp)
        csvs = os.path.join(tmp, "**", "*.csv").replace("\\", "/")
        rows_in = con.execute(f"SELECT count(*) FROM read_csv('{csvs}', header=true, union_by_name=true)").fetchone()[0]
        con.execute(f"""
            COPY (
              SELECT CAST(MMSI AS BIGINT) AS mmsi,
                     CAST(BaseDateTime AS TIMESTAMP) AS ts,
                     CAST(LAT AS DOUBLE) AS lat, CAST(LON AS DOUBLE) AS lon,
                     CAST(SOG AS DOUBLE) AS sog, CAST(COG AS DOUBLE) AS cog,
                     CAST(Heading AS DOUBLE) AS heading,
                     TRY_CAST(VesselType AS INTEGER) AS vessel_type,
                     TRY_CAST(Status AS INTEGER) AS nav_status,
                     TRY_CAST(Length AS DOUBLE) AS length_m,
                     TransceiverClass AS tclass
              FROM read_csv('{csvs}', header=true, union_by_name=true)
              WHERE LON BETWEEN {BBOX['xmin']} AND {BBOX['xmax']}
                AND LAT BETWEEN {BBOX['ymin']} AND {BBOX['ymax']}
            ) TO '{out_file.replace(os.sep, "/")}' (FORMAT PARQUET, COMPRESSION ZSTD)
        """)
        rows_out = con.execute(f"SELECT count(*) FROM read_parquet('{out_file.replace(os.sep, '/')}')").fetchone()[0]
    with open(COUNTS, "a", newline="") as f:
        csv.writer(f).writerow([year, month, rows_in, rows_out])

if __name__ == "__main__":
    for y in (2022, 2023, 2024):
        for m in range(1, 13):
            process(y, m)
            print("done", y, m)
```

> [VERIFY] column names against a sample file (Appendix B). If the download is already Parquet/GeoParquet, skip the unzip and read it directly with `read_parquet`.

### 11.3 Step B2 — Clean, classify, and build legs (per month)

`ais_preprocess/02_legs.sql` — run from Python with `con.execute(open(...).read().replace('{Y}', ...))`, or in the DuckDB CLI. `{Y}` and `{M}` are placeholders for year and month.

```sql
INSTALL spatial; LOAD spatial;

-- Lookup: AIS type code -> class and rule applicability (Appendix C)
CREATE OR REPLACE TABLE vclass AS
SELECT * FROM read_csv('C:/GIS/NARW/03_Scripts/configs/vessel_classes.csv', header=true);

-- Load one month. (The first leg of each vessel on the 1st of the month is lost at the month boundary;
-- this is negligible, but record it as a known limitation.)
CREATE OR REPLACE TABLE raw AS
SELECT * FROM read_parquet('C:/GIS/NARW/02_Data/working/ais_parquet/ais_{Y}_{M}.parquet');

-- D1/D2: MMSI validity and exact duplicates
CREATE OR REPLACE TABLE pings AS
SELECT DISTINCT ON (mmsi, ts) *
FROM raw
WHERE mmsi BETWEEN 200000000 AND 799999999          -- 9-digit ship MMSIs (MID 2xx-7xx)
  AND mmsi NOT IN (111111111, 123456789, 222222222, 333333333, 444444444, 555555555,
                   666666666, 777777777, 999999999)
  AND sog IS NOT NULL AND sog >= 0 AND sog < 102.2    -- 102.3 = 'not available'
ORDER BY mmsi, ts;

-- Static vessel attributes per MMSI for the month (most common type, median positive length)
CREATE OR REPLACE TABLE vessels AS
SELECT p.mmsi,
       mode(p.vessel_type) FILTER (WHERE p.vessel_type > 0) AS vessel_type,
       median(p.length_m)  FILTER (WHERE p.length_m > 0)   AS length_m
FROM pings p GROUP BY p.mmsi;

CREATE OR REPLACE TABLE vessels_c AS
SELECT v.*,
       coalesce(c.vessel_class, 9) AS vessel_class,
       CASE
         WHEN c.rule_applicable_flag = 0 THEN 0                          -- exempt (military / law enforcement)
         WHEN v.length_m >= 19.812 THEN 1                                -- 65 ft
         WHEN v.length_m IS NULL AND coalesce(c.vessel_class, 9) IN (1, 2, 3) THEN 1   -- D-011
         WHEN v.length_m IS NULL THEN -1                                 -- indeterminate
         ELSE 0
       END AS rule_applicable,
       CASE
         WHEN v.length_m IS NULL THEN 'unk'
         WHEN v.length_m >= 19.812 THEN 'ge65'
         WHEN v.length_m >= 10.668 THEN '35to65'                         -- 35 ft
         ELSE 'lt35'
       END AS length_band
FROM vessels v LEFT JOIN vclass c ON v.vessel_type = c.ais_type_code;

-- D3: SOG spikes (> 40 kn) for non-passenger vessels
CREATE OR REPLACE TABLE pings2 AS
SELECT p.* FROM pings p JOIN vessels_c v USING (mmsi)
WHERE NOT (p.sog > 40 AND v.vessel_class <> 3);

-- Legs between consecutive pings; D4 segmentation (gap > 30 min or implied speed > 60 kn)
CREATE OR REPLACE TABLE legs AS
WITH s AS (
  SELECT *,
         lag(ts)  OVER w AS prev_ts,
         lag(lat) OVER w AS prev_lat,
         lag(lon) OVER w AS prev_lon,
         lag(sog) OVER w AS prev_sog
  FROM pings2 WINDOW w AS (PARTITION BY mmsi ORDER BY ts)
), d AS (
  SELECT *,
         epoch(ts - prev_ts) / 3600.0 AS dt_hr,
         3440.065 * 2 * asin(sqrt(
            pow(sin(radians(lat - prev_lat) / 2), 2) +
            cos(radians(prev_lat)) * cos(radians(lat)) * pow(sin(radians(lon - prev_lon) / 2), 2)
         )) AS dist_nm                                   -- haversine, nautical miles
  FROM s WHERE prev_ts IS NOT NULL
)
SELECT mmsi, prev_ts, ts, prev_lat, prev_lon, lat, lon, dist_nm, dt_hr,
       dist_nm / dt_hr AS implied_kn,
       (prev_sog + sog) / 2.0 AS mean_rep_sog,
       CASE WHEN abs((prev_sog + sog) / 2.0 - dist_nm / dt_hr) <= 2
            THEN (prev_sog + sog) / 2.0 ELSE dist_nm / dt_hr END AS leg_speed_kn,   -- D-009
       CASE WHEN abs((prev_sog + sog) / 2.0 - dist_nm / dt_hr) <= 2
            THEN 'reported' ELSE 'implied' END AS speed_source,
       ST_Transform(ST_MakeLine(ST_Point(prev_lon, prev_lat), ST_Point(lon, lat)),
                    'EPSG:4326', 'EPSG:5070', always_xy := true) AS geom,
       ST_Transform(ST_Point((prev_lon + lon) / 2, (prev_lat + lat) / 2),
                    'EPSG:4326', 'EPSG:5070', always_xy := true) AS mid
FROM d
WHERE dt_hr > 0 AND dt_hr <= 0.5                      -- gap split: 30 min
  AND dist_nm / dt_hr <= 60;                          -- jump split: 60 kn

-- QA counts for the log
SELECT '{Y}-{M}' AS ym,
       (SELECT count(*) FROM raw)    AS raw_rows,
       (SELECT count(*) FROM pings)  AS after_mmsi_dupes,
       (SELECT count(*) FROM pings2) AS after_sog_spikes,
       (SELECT count(*) FROM legs)   AS legs,
       (SELECT avg((speed_source = 'implied')::INT) FROM legs) AS share_implied_speed,
       (SELECT avg((length_m IS NULL)::INT) FROM vessels_c) AS share_missing_length;
```

Notes:
- **Segmentation is implicit:** a leg is only created between two pings ≤ 30 min apart and ≤ 60 kn implied speed. Longer gaps or jumps simply break the track (D4).
- **Datum note:** DuckDB/PROJ transforms WGS 84 → NAD 83 Albers with its default transformation. The difference from `WGS_1984_(ITRF00)_To_NAD_1983` is ~1–2 m — negligible against AIS positional accuracy. Record this in metadata lineage.
- **On-land legs (D6):** optionally drop legs whose midpoint is on land: `DELETE FROM legs WHERE EXISTS (SELECT 1 FROM land WHERE ST_Contains(land.geom, legs.mid));` (load `land_5070.gpkg` with `ST_Read`).

### 11.4 Step B3 — Intersect legs with SMAs and Slow Zones

`ais_preprocess/03_areas.sql`:
```sql
CREATE OR REPLACE TABLE sma AS SELECT * FROM ST_Read('C:/GIS/NARW/02_Data/working/ais_inputs/sma_5070.gpkg');
CREATE OR REPLACE TABLE dma AS SELECT * FROM ST_Read('C:/GIS/NARW/02_Data/working/ais_inputs/dma_5070.gpkg');

-- SMA: active by calendar date (mmdd), handling periods that wrap the year (e.g. 1101-0430)
CREATE OR REPLACE TABLE leg_area AS
SELECT l.mmsi, l.prev_ts, l.ts, l.leg_speed_kn, 'SMA' AS area_type, s.sma_id AS area_id,
       ST_Length(ST_Intersection(l.geom, s.geom)) / 1852.0 AS nm_in,
       CASE WHEN s.active_start_mmdd <= s.active_end_mmdd
            THEN (month(l.ts) * 100 + day(l.ts)) BETWEEN s.active_start_mmdd AND s.active_end_mmdd
            ELSE (month(l.ts) * 100 + day(l.ts)) >= s.active_start_mmdd
              OR (month(l.ts) * 100 + day(l.ts)) <= s.active_end_mmdd
       END AS active
FROM legs l JOIN sma s ON ST_Intersects(l.geom, s.geom)
UNION ALL
-- Slow Zones: active by date-time window
SELECT l.mmsi, l.prev_ts, l.ts, l.leg_speed_kn, 'DMA', d.dma_id,
       ST_Length(ST_Intersection(l.geom, d.geom)) / 1852.0,
       l.ts BETWEEN d.start_datetime AND d.end_datetime
FROM legs l JOIN dma d ON ST_Intersects(l.geom, d.geom)
     AND l.ts BETWEEN d.start_datetime - INTERVAL 1 DAY AND d.end_datetime + INTERVAL 1 DAY;

-- Pings inside each area (for n_points and reconciliation check G1)
CREATE OR REPLACE TABLE pings_in AS
SELECT p.mmsi, CAST(p.ts AS DATE) AS d, s.sma_id AS area_id, count(*) AS n
FROM pings2 p JOIN sma s
  ON ST_Contains(s.geom, ST_Transform(ST_Point(p.lon, p.lat), 'EPSG:4326', 'EPSG:5070', always_xy := true))
GROUP BY ALL;
```

> **Performance:** use DuckDB **≥ 1.3**, whose spatial extension automatically optimizes `ST_Intersects`/`ST_Contains` joins [VERIFY]. With older versions, pre-filter legs with a bounding-box condition (`ST_XMin`/`ST_XMax` etc.) against each SMA's envelope, or process one SMA at a time.

### 11.5 Step B4 — Build transits

```sql
CREATE OR REPLACE TABLE transits AS
SELECT a.mmsi, CAST(a.ts AS DATE) AS transit_date, a.area_type, a.area_id AS sma_id,
       min(a.prev_ts) AS start_datetime, max(a.ts) AS end_datetime,
       sum(a.nm_in) AS length_nm,
       sum(a.nm_in * a.leg_speed_kn) / nullif(sum(a.nm_in), 0) AS mean_sog_kn,
       max(a.leg_speed_kn) AS max_sog_kn,
       bool_or(a.active)::INT AS sma_active_flag,
       sum(a.nm_in) FILTER (WHERE a.active) AS nm_active,
       sum(a.nm_in) FILTER (WHERE a.active AND a.leg_speed_kn > 10.5) AS nm_over_10kn   -- 10 kn + 0.5 tolerance
FROM leg_area a
GROUP BY a.mmsi, CAST(a.ts AS DATE), a.area_type, a.area_id;

CREATE OR REPLACE TABLE transits_c AS
SELECT t.*, v.vessel_type, v.vessel_class, v.length_m, v.rule_applicable,
       coalesce(pi.n, 0) AS n_points,
       100.0 * coalesce(t.nm_over_10kn, 0) / nullif(t.nm_active, 0) AS pct_dist_over_10kn,
       CASE
         WHEN t.sma_active_flag = 0 OR v.rule_applicable = 0 THEN 'NotApplicable'
         WHEN v.rule_applicable = -1 OR coalesce(pi.n, 0) < 3 THEN 'Indeterminate'
         WHEN 100.0 * coalesce(t.nm_over_10kn, 0) / nullif(t.nm_active, 0) >= 10 THEN 'NonCompliant'
         ELSE 'Compliant'
       END AS compliance_status,
       t.mmsi::VARCHAR || '_' || strftime(t.transit_date, '%Y%m%d') || '_' || t.sma_id AS transit_id
FROM transits t
JOIN vessels_c v USING (mmsi)
LEFT JOIN pings_in pi ON pi.mmsi = t.mmsi AND pi.d = t.transit_date AND pi.area_id = t.sma_id;
```

> For Slow Zones (`area_type = 'DMA'`), `n_points` is not computed above — add a matching `pings_in` block for `dma` or treat DMA status as "voluntary" and use `nm_active > 0` in place of the 3-point rule [DECISION].

**Transit geometry** (for `AIS_Transits` polylines):
```sql
COPY (
  SELECT c.*, ST_Union_Agg(ST_Intersection(l.geom, s.geom)) AS geom
  FROM transits_c c
  JOIN legs l ON l.mmsi = c.mmsi AND CAST(l.ts AS DATE) = c.transit_date
  JOIN sma s ON s.sma_id = c.sma_id AND ST_Intersects(l.geom, s.geom)
  WHERE c.area_type = 'SMA'
  GROUP BY ALL
) TO 'C:/GIS/NARW/02_Data/working/ais_out/transits_{Y}_{M}.gpkg'
  WITH (FORMAT GDAL, DRIVER 'GPKG', SRS 'EPSG:5070');
```

### 11.6 Step B5 — Hex-month traffic

Legs are assigned to the 4 km² hexagon containing their **midpoint** (D-010).
```sql
CREATE OR REPLACE TABLE hex4 AS SELECT hex_id, geom FROM ST_Read('C:/GIS/NARW/02_Data/working/ais_inputs/hex4_5070.gpkg');

CREATE OR REPLACE TABLE leg_hex AS
SELECT l.*, h.hex_id FROM legs l JOIN hex4 h ON ST_Contains(h.geom, l.mid);

COPY (
  SELECT h.hex_id, {Y} AS year, {M} AS month, v.vessel_class, v.rule_applicable, v.length_band,
         count(DISTINCT h.mmsi::VARCHAR || '_' || CAST(h.ts AS DATE)::VARCHAR) AS transit_count,
         sum(h.dist_nm) AS transit_nm,
         sum(h.dist_nm * h.leg_speed_kn) / nullif(sum(h.dist_nm), 0) AS mean_sog_kn,
         sum(h.dist_nm) FILTER (WHERE h.leg_speed_kn > 10.5) AS nm_over_10kn,
         100.0 * coalesce(sum(h.dist_nm) FILTER (WHERE h.leg_speed_kn > 10.5), 0) / nullif(sum(h.dist_nm), 0) AS pct_nm_over_10kn,
         count(DISTINCT h.mmsi) AS unique_vessels
  FROM leg_hex h JOIN vessels_c v USING (mmsi)
  GROUP BY ALL
) TO 'C:/GIS/NARW/02_Data/working/ais_out/hex4_traffic_{Y}_{M}.csv' (HEADER);
```

### 11.7 Step B6 — 1% stratified point sample

A deterministic hash sample keeps ~1% of pings in every vessel-class × month stratum:
```sql
COPY (
  SELECT p.mmsi, p.ts AS base_datetime, p.sog, p.cog, p.heading, v.vessel_type, v.vessel_class, v.length_m,
         ST_Point(p.lon, p.lat) AS geom
  FROM pings2 p JOIN vessels_c v USING (mmsi)
  WHERE hash(p.mmsi, p.ts) % 100 = 0
) TO 'C:/GIS/NARW/02_Data/working/ais_out/sample_{Y}_{M}.gpkg'
  WITH (FORMAT GDAL, DRIVER 'GPKG', SRS 'EPSG:4326');
```

### 11.8 Step B7 — Run all months and load into the geodatabase

1. Driver script `ais_preprocess/run_all.py` loops years × months, runs 02–06 with `{Y}`/`{M}` substituted, and appends the QA count row to `qa_counts_legs.csv`.
2. **Merge** monthly outputs:
   - `transits_*.gpkg` → **Merge (Data Management)** → **Project** to 4269 (5070 and 4269 are both NAD 83 — no transformation) → **Append** to `VesselTraffic/AIS_Transits` (field map; `area_type` DMA rows can go to a sibling class or stay with `area_type='DMA'`).
   - `sample_*.gpkg` → Merge → Project to 4269 with `WGS_1984_(ITRF00)_To_NAD_1983` → Append to `AIS_Points_Sample`.
   - `hex4_traffic_*.csv` → **Export Table** → **Append** to `Hex4_VesselTraffic_Monthly`.
3. Fill `source_id = 'S09'`, `load_date`, `qa_status`, `version`.
4. Write QA counts (D1–D7) into `QAQC_Log` — one row per check per year.
5. **Reconciliation (G1):** for each SMA-month, `SUM(n_points)` in `AIS_Transits` must equal the `pings_in` total ±1%. For (G2), total `transit_nm` in `Hex4_VesselTraffic_Monthly` ≈ total leg nm with midpoints in the focus zone ±1%.
6. `ProcessingLog`: one row per script run with parameters (thresholds, years) and the Git commit.

### 11.9 ArcGIS-only alternative (small-scale test or if Python is not allowed)

For a single SMA and a few days: **XY Table To Point** → **Points To Line (Data Management)** (*Line Field* `MMSI`, *Sort Field* `BaseDateTime`) → **Split Line At Vertices** → **Calculate Geometry** (length, nm) → join times → compute speed. With ArcGIS Pro Advanced, **Reconstruct Tracks (GeoAnalytics Desktop)** builds tracks with time-gap splitting directly. Use this only to **validate** the DuckDB pipeline on a sample (e.g., 3 days in the Delaware Bay SMA) — compare transit counts and speeds, and log the comparison.

### 11.10 Checks

- [ ] Every month processed; `qa_counts_*` complete; shares of implied speed and missing length reported.
- [ ] G1 and G2 reconciliations within ±1%.
- [ ] Spot-check 10 random transits: draw pings from `AIS_Points_Sample` or raw Parquet over the transit polyline; confirm speeds.
- [ ] Unit tests (§16.3) pass for haversine, active-period wrap, leg speed rule, classification.

---

## 12. Stage 9 — Phase C: Speed-rule compliance

**Goal:** `SMA_Compliance_Monthly` — compliance metrics by SMA × year × month × vessel class, for all active periods 2022–2024, with the indeterminate share documented.

### 12.1 Compliance rule (per transit)

| Status | Condition |
|---|---|
| **NotApplicable** | SMA not active on that date, or vessel < 65 ft, or vessel exempt (federal / law enforcement) |
| **Indeterminate** | Vessel length unknown and class not Cargo/Tanker/Passenger, or fewer than 3 pings inside the SMA |
| **NonCompliant** | ≥ 10% of the distance travelled inside the active SMA was at > 10.5 kn |
| **Compliant** | Otherwise |

Parameters (Appendix E): `speed_limit_kn = 10`, `tolerance_kn = 0.5`, `noncompliant_pct_distance = 10`, `min_points_in_sma = 3`.

### 12.2 Steps

1. In ArcGIS Pro: **Summary Statistics (Analysis)** on `AIS_Transits` with *Case fields* `sma_id; area_type; year; month; vessel_class` (add `year`, `month` fields to transits first with **Calculate Field** from `transit_date`) and filter `sma_active_flag = 1 AND compliance_status <> 'NotApplicable'`:
   - `COUNT transit_id` → `transits`
   - `SUM length_nm` → `nm_total`
   - `SUM nm_over_10kn` → `nm_over_10kn`
2. Counts by status: add three helper fields to `AIS_Transits` — `is_nc`, `is_c`, `is_ind` (0/1) — and SUM them in the same Summary Statistics.
3. Calculate:
   - `pct_noncompliant = 100 × transits_noncompliant / (transits_noncompliant + transits_compliant)` (indeterminate excluded from the denominator)
   - `pct_indeterminate = 100 × transits_indeterminate / transits`
   - `pct_nm_over_10kn = 100 × nm_over_10kn / nm_total`
4. **Append** to `SMA_Compliance_Monthly`; `version = 'v1.0'`.
5. Also produce roll-ups (saved as tables in `outputs/tables/` and used in the report and Dashboard):
   - By SMA × vessel class (all years, active months)
   - By SMA × year
   - By region × month
   - Slow Zones (voluntary) vs. SMAs (mandatory) side by side
6. **Charts** (in Pro: right-click the table → **Create Chart**):
   - Bar chart: % non-compliant by SMA, grouped by vessel class.
   - Line chart: % non-compliant by month, per SMA (active months only).
   - Stacked bar: transit status shares (Compliant / NonCompliant / Indeterminate) per SMA.
   Export charts as SVG/PNG for the report. Follow color rules in §17.1.
7. **Per-SMA one-page summaries** (feed the decision briefs, §19): transits, % non-compliant, % nm over 10 kn, worst vessel class, trend 2022→2024, indeterminate share.

### 12.3 Checks

- [ ] Every SMA × active month × year has rows (zero-transit months recorded as 0, not missing).
- [ ] Indeterminate share reported per SMA (acceptance criterion 6).
- [ ] Hand-check 5 transits' classification against their pings.

---

## 13. Stage 10 — Phase D: Co-occurrence risk index

**Goal:** `Hex4_Risk_Monthly` — a relative vessel-strike risk index (0–100) for every 4 km² hexagon and month, and the **baseline share of coast-wide risk inside current SMAs**.

### 13.1 Formula

```
risk_index = 100 × presence_norm × traffic_norm × speed_factor
```

| Term | Definition |
|---|---|
| `presence_norm` | `presence_index` (climatology, same calendar month) ÷ the maximum `presence_index` in that month across the grid |
| `traffic_norm` | `ln(1 + traffic_nm) ÷ ln(1 + P99)`, capped at 1, where `traffic_nm` = nautical miles travelled by **rule-applicable (≥ 65 ft)** vessels in the hex-month and `P99` = 99th percentile of `traffic_nm` in that month [DECISION D-018 — log scaling because traffic is extremely skewed; original scope used simple normalization] |
| `speed_factor` | 0.5 if `mean_sog_kn` ≤ 10; 1.0 if ≥ 15; linear in between: `0.5 + 0.1 × (mean_sog_kn − 10)` — reflects the steep rise in strike lethality with speed (Vanderlaan & Taggart 2007; Conn & Silber 2013) |
| `risk_class` | 0 where `risk_index = 0`; otherwise quintiles 1–5 computed **once** over all non-zero values from all months (so breaks are consistent across months) |

### 13.2 Steps

1. Build the traffic input: **Summary Statistics** on `Hex4_VesselTraffic_Monthly` filtered to `rule_applicable = 1`, case fields `hex_id; year; month`, `SUM transit_nm`, and distance-weighted mean speed (add field `nm_x_sog = transit_nm × mean_sog_kn` first; then `mean_sog_kn = SUM(nm_x_sog) / SUM(transit_nm)`).
2. Join presence (climatology) by `hex_id` + `month`.
3. Compute the terms and index with pandas (`src/narw/risk.py`):
```python
import numpy as np, pandas as pd

def speed_factor(sog, min_kn=10, min_v=0.5, max_kn=15, max_v=1.0):
    return np.clip(min_v + (max_v - min_v) * (sog - min_kn) / (max_kn - min_kn), min_v, max_v)

def compute_risk(df):   # columns: hex_id, year, month, presence_index, traffic_nm, mean_sog_kn
    df = df.copy()
    df["traffic_nm"] = df["traffic_nm"].fillna(0)
    g = df.groupby(["year", "month"])
    df["presence_norm"] = df["presence_index"] / g["presence_index"].transform("max")
    p99 = g["traffic_nm"].transform(lambda s: np.nanpercentile(s, 99))
    df["traffic_norm"] = np.clip(np.log1p(df["traffic_nm"]) / np.log1p(p99), 0, 1)
    df["speed_factor"] = speed_factor(df["mean_sog_kn"].fillna(10))
    df["risk_index"] = 100 * df["presence_norm"].fillna(0) * df["traffic_norm"] * df["speed_factor"]
    nz = df["risk_index"] > 0
    breaks = np.quantile(df.loc[nz, "risk_index"], [0.2, 0.4, 0.6, 0.8])
    df["risk_class"] = np.where(nz, 1 + np.searchsorted(breaks, df["risk_index"], side="right"), 0)
    return df, breaks
```
4. **Inside-SMA flags:** **Feature To Point** (inside) on `HexGrid_4km2` → **Spatial Join** with `SMA` → table `hex_sma` (`hex_id`, `sma_id`, `active_start_mmdd`, `active_end_mmdd`). `inside_sma = 1` if a match; `inside_sma_active = 1` if the month overlaps the active period (a month counts as active if **any** day of it is active [DECISION] — alternatively weight by the fraction of active days, which is more precise for partial months such as Apr 16–30 in the Southeast).
5. **Append** results to `Hex4_Risk_Monthly`. Record the quintile breaks in the decision log and metadata.
6. **Climatology version for maps/web:** average `risk_index` across 2022–2024 by `hex_id, month` → `Hex4_Risk_Clim` (12 months).

### 13.3 Baseline metric

```
baseline_share(month) = Σ risk_index (inside_sma_active = 1) ÷ Σ risk_index (all hexes)
annual_baseline_share = Σ over all year-months (inside_sma_active = 1) ÷ Σ over all year-months
```
Report by month, by region, and annually. This is headline finding #2.

### 13.4 Checks

- [ ] `risk_index` in 0–100, `risk_class` 0–5 (G3); every hex-month present (G4).
- [ ] Sanity: high-risk hexes appear where both presence and traffic are high (e.g., approaches to New York/New Jersey, Chesapeake, and Southern New England shipping lanes in winter–spring) — record observations.
- [ ] Baseline share computed and recorded.

---

## 14. Stage 11 — Phase E: Management scenarios and sensitivity

**Goal:** `CandidateSMA` polygons, `Scenario_Summary`, `Sensitivity_Results`, and a clear statement of which scenario performs best and how stable that ranking is.

### 14.1 Candidate scenarios (build each in `Regulatory/CandidateSMA`)

Each scenario = **current SMAs + the modification** (so results are directly comparable with the baseline).

| ID | Scenario | How to build the polygon | Active period | Vessel length |
|---|---|---|---|---|
| `SC1` | **Mid-Atlantic port SMAs extended seaward to 30 nm** | Take the center points of each Mid-Atlantic port SMA from 50 CFR 224.105 (the 20-nm arcs); **Buffer** `30 NauticalMiles`, *Geodesic*; **Erase** `Land`; **Merge** with the continuous Wilmington–Brunswick SMA extended from 20 to 30 nm offshore (buffer the shoreline 30 nm, clip to that SMA's alongshore limits). | Nov 1 – Apr 30 | ≥ 65 ft |
| `SC2a` / `SC2b` | **Northeast active periods shifted −1 / +1 month** | Copy the three Northeast SMAs unchanged. | Shift start and end by −1 month (SC2a) and +1 month (SC2b) | ≥ 65 ft |
| `SC3` | **New Southern New England SMA around wind lease areas** | Select the Rhode Island/Massachusetts and Massachusetts lease areas from `WindLeaseAreas`; **Pairwise Buffer** 10 km; **Minimum Bounding Geometry** (`CONVEX_HULL`, group all) → polygon | Dec 1 – Mar 31 | ≥ 65 ft |
| `SC4` | **Rule applied to vessels ≥ 35 ft** (mirrors the 2022 proposed amendment's size threshold) | Current SMA polygons | Current | ≥ 35 ft |
| `SC5` (optional) | **2022 proposed "Seasonal Speed Zones"** | Digitize from the proposed rule's maps/coordinates (87 FR 46921) if GIS is available [VERIFY] | As proposed | ≥ 35 ft |

Fill `cand_id`, `scenario_name`, `rationale` (1–2 sentences each, with citations), `active_start_mmdd`, `active_end_mmdd`, `vessel_length_min_ft`, `author`, `version`.

### 14.2 Metrics per scenario

| Metric | Definition | How |
|---|---|---|
| `months_active` | Months with any active day | From dates |
| `risk_covered` | Σ `risk_index` of hex-months inside the **scenario footprint** (baseline SMAs ∪ candidate) during their active months, 2022–2024 | Flag hexes inside candidate (centroid, as §13.2 step 4) → sum in pandas |
| `pct_coast_risk_covered` | `risk_covered ÷ Σ risk_index (all hex-months)` | |
| `baseline_pct` | Same for current SMAs only | §13.3 |
| `delta_vs_baseline_pct` | `pct_coast_risk_covered − baseline_pct` (percentage points) | |
| `added_area_km2` | Area(scenario footprint) − Area(baseline footprint), in 5070 | **Pairwise Dissolve** + Calculate Geometry |
| `added_transits` | Transits by applicable vessels in the scenario footprint during its active months that were **not** already in an active baseline SMA | Re-run §11.4–11.5 with `cand_5070.gpkg` in place of `sma_5070.gpkg`; subtract baseline |
| `rank` | Rank by `delta_vs_baseline_pct` (ties broken by smaller `added_area_km2`) | |

**SC4 special handling:** the footprint is unchanged, so it covers no new risk under the ≥ 65 ft risk surface. Evaluate it with:
- a **≥ 35 ft risk surface** (repeat §13 using traffic where `length_band IN ('ge65','35to65')`), reporting the risk covered by the newly regulated 35–65 ft traffic; and
- `added_transits` = transits of 35–65 ft vessels in active SMAs.
- Caveat: many 35–65 ft vessels carry Class B AIS or none, so this is a **minimum**.

Optional efficiency metric: `delta_vs_baseline_pct ÷ added_area_km2 × 1000` (risk gained per 1,000 km²) — useful in decision briefs.

### 14.3 Sensitivity analysis

Perturb each parameter by **±25%** (Appendix E `sensitivity.perturbation_pct`) one at a time, recompute risk and scenario metrics, and compare rankings.

| Parameter | Base | −25% | +25% | Affects |
|---|---|---|---|---|
| Presence weight: sightings | 0.4 | 0.3 | 0.5 | Risk, scenarios (re-normalize weights to sum 1) |
| Presence weight: acoustic | 0.2 | 0.15 | 0.25 | Risk, scenarios |
| Presence weight: density model | 0.4 | 0.3 | 0.5 | Risk, scenarios |
| Speed factor minimum value | 0.5 | 0.375 | 0.625 | Risk, scenarios |
| Speed factor upper speed | 15 kn | 11.25 kn | 18.75 kn | Risk, scenarios (must stay above the 10-kn lower bound) |
| Compliance distance threshold | 10% | 7.5% | 12.5% | Compliance rates |
| Speed tolerance | 0.5 kn | 0.375 kn | 0.625 kn | Compliance rates |
| Presence time basis | Climatology | — | 2022–2024 aligned | Risk, scenarios |

**For each run:** record `pct_coast_risk_covered` and `rank` per scenario in `Sensitivity_Results`, and compute **Kendall's τ** between the run's ranking and the base ranking (`scipy.stats.kendalltau`, or `pandas.Series.corr(method='kendall')`).

**Report:**
- A tornado chart of `delta_vs_baseline_pct` for the top scenario under each perturbation.
- A table of ranks per run, with τ.
- A plain statement: "The top-ranked scenario remained first in N of M runs" (acceptance criterion 7).
- For compliance: how % non-compliant per SMA changes under the threshold/tolerance perturbations.

### 14.4 Checks

- [ ] `CandidateSMA` passes topology coverage rule; each polygon documented with rationale.
- [ ] `Scenario_Summary` complete for all scenarios; baseline row included.
- [ ] `Sensitivity_Results` complete; ranking stability stated explicitly.
- [ ] **Design Review 2** (Week 5) record filed covering Phases A–C; interim review of D–E in the Week 6 status note.

---

## 15. Stage 12 — Metadata (ISO 19115 / FGDC CSDGM)

**Goal:** complete, validated metadata for **every** feature class, table, raster, and mosaic dataset, exported as ISO 19139 and FGDC CSDGM XML to `04_Metadata/` and `metadata/` (Git), tracked to 100% in `MetadataStatus`, with a catalog page.

### 15.1 Required elements (every object)

| Element | ISO 19115 section | FGDC section | Content guidance |
|---|---|---|---|
| Title | Citation | Identification > Citation | "North Atlantic Right Whale Seasonal Management Areas (NAD 83), NARW Vessel-Strike GDB v1.0" |
| Abstract | Identification | Description > Abstract | What it is, how made, period, grain — 3–6 sentences |
| Purpose | Identification | Description > Purpose | Why it exists in this project |
| Keywords | Descriptive keywords | Keywords | **ISO topic category** (e.g. `biota`, `oceans`, `transportation`, `boundaries`); **GCMD Science Keywords** (e.g. `EARTH SCIENCE > BIOSPHERE > ... > MARINE MAMMALS`); **place** keywords (`U.S. Atlantic`, `Gulf of Maine`, …); theme keywords (`right whale`, `vessel strike`, `AIS`) |
| Credit | Identification | Data set credit | Providers from `DataSourceRegistry` |
| Use constraints / access constraints | Resource constraints | Access / Use constraints | Public-domain statement or provider terms; **restricted-data terms for S01 NARWC**; "Relative risk index — not a strike probability" |
| Status and maintenance | Maintenance | Status | `completed` / `asNeeded` |
| Extent | Extent (geographic + temporal) | Spatial domain + Time period | Bounding box (from the data), temporal range |
| Spatial reference | Reference system | Spatial reference | WKID and name; vertical datum for bathymetry |
| Lineage | Data quality > Lineage | Data quality > Lineage | **Source citations + every process step** (tool, parameters, date, operator) — from the geoprocessing history and `ProcessingLog` |
| Entity and attribute | Content information (feature catalog) | Entity_and_Attribute_Information | **Every field**: label, definition, definition source, domain (codes and meanings or range), units |
| Positional accuracy | Data quality | Positional accuracy report | e.g. AIS ± 10 m nominal; sightings `positional_uncertainty_m` |
| Attribute accuracy | Data quality | Attribute accuracy report | Summarize QA results (counts from `QAQC_Log`) |
| Completeness | Data quality | Completeness report | Coverage gaps, excluded records |
| Contact | Point of contact | Point of contact | Bernard Issifu, role, email |
| Metadata date, standard | Metadata info | Metadata reference | Date, standard name and version |

### 15.2 Step-by-step in ArcGIS Pro

1. Confirm *Project > Options > Metadata* style = **ISO 19139 Metadata Implementation Specification** (D-006). Check *Show metadata errors* (the editor flags missing required elements in red).
2. In the **Catalog pane**, right-click an object → **Edit Metadata**. Work through the pages: *Overview* (Item Description, Topics & Keywords, Citation, Citation Contacts, Locales), *Metadata* (Details, Contacts, Maintenance, Constraints), *Resource* (Details, Extents, Points of Contact, Maintenance, Constraints, Spatial Reference, Spatial Representation, **Quality**, **Lineage**, Distribution, **Fields**, Feature Catalog).
3. **Fields page:** for each field, enter *Label*, *Definition*, *Definition source*, and *Domain* (enumerated values with definitions, or range with units). Pro auto-lists fields; you add definitions from the data dictionary.
4. **Lineage:** Pro adds each geoprocessing tool run to *Geoprocessing history* if the option in §4.3 is on. Add **Data sources** (citation for each raw source) and **Process steps** for work done outside Pro (e.g. the DuckDB AIS pipeline, with script names and Git commit).
5. **Update extents:** on the *Resource > Extents* page, use *Update* to calculate the bounding box from the data.
6. **Save.**
7. **Export:** *Catalog > Metadata tab > Export* → choose **ISO 19139** → `04_Metadata/iso/<object>.xml`; again with **FGDC CSDGM Metadata** → `04_Metadata/fgdc/<object>.xml`. Pro's export validates against the selected standard's schema and reports problems.
8. Update `MetadataStatus` (`fgdc_complete`, `iso_complete`, `validated`, `last_updated`).

**Tip — build once, copy many:** complete one feature class fully, then use *Metadata tab > Save As Template*, and apply it to similar objects with *Import > From template* (e.g. all monthly tables) before customizing.

### 15.3 Automating static elements (`GenerateMetadata`)

Keep reusable text in `configs/metadata_templates.yaml`:
```yaml
defaults:
  credits: "NOAA Fisheries; NOAA NEFSC; Duke MGEL; BOEM; NOAA Office for Coastal Management (Marine Cadastre)."
  contact: {name: Bernard Issifu, email: <your email>, role: pointOfContact}
  use_limitation: >-
    For research and portfolio demonstration. Relative risk index; not a strike probability.
    Check boundaries and rules against current Federal Register notices before regulatory use.
objects:
  SMA:
    title: "North Atlantic Right Whale Seasonal Management Areas (NAD 83)"
    summary: "Vessel speed rule SMAs (50 CFR 224.105) with active periods."
    description: "Polygons for Northeast, Mid-Atlantic, and Southeast SMAs, digitized/verified against eCFR ..."
    tags: [right whale, vessel speed rule, seasonal management area, biota, transportation]
```
ArcPy (`src/narw/metadata.py`):
```python
import arcpy, yaml
from arcpy import metadata as md
GDB = r"C:\GIS\NARW\02_Data\final\NARW_VesselStrike.gdb"
cfg = yaml.safe_load(open(r"C:\GIS\NARW\03_Scripts\configs\metadata_templates.yaml"))
paths = {"SMA": GDB + r"\Regulatory\SMA"}   # extend for every object
for name, path in paths.items():
    o = cfg["objects"][name]
    m = md.Metadata(path)
    m.title = o["title"]; m.summary = o["summary"]; m.description = o["description"]
    m.tags = ", ".join(o["tags"]); m.credits = cfg["defaults"]["credits"]
    m.accessConstraints = cfg["defaults"]["use_limitation"]
    m.save()
    m.exportMetadata(rf"C:\GIS\NARW\04_Metadata\iso\{name}.xml", "ISO19139_GML32")
    m.exportMetadata(rf"C:\GIS\NARW\04_Metadata\fgdc\{name}.xml", "FGDC_CSDGM")
```
[VERIFY the exact export-standard keywords for `exportMetadata` in your Pro version's documentation.]

### 15.4 Validation

- **ISO:** Pro's export validation; optionally validate the XML against ISO 19139 schemas with `xmllint --schema` or an online validator, and review against NOAA/NCEI ISO guidance [VERIFY current NOAA metadata guidance/rubric].
- **FGDC:** USGS metadata parser **`mp`** (https://geology.usgs.gov/tools/metadata/ [VERIFY availability]) — run `mp -e errors.txt <file>.xml`; fix all errors.
- Record validator and result in `MetadataStatus.validator` and `notes`.
- **NOAA InPort** (https://www.fisheries.noaa.gov/inport/): NOAA Fisheries' metadata catalog. You can't publish there without NOAA credentials, but structure the content to match InPort's required sections (Item Identification, Keywords, Physical Location, Data Set Information, Support Roles, Extents, Access Information, Distribution, Lineage, Entity/Attribute) and say so in the write-up.

### 15.5 Hosted item metadata (ArcGIS Online)

Every hosted layer, web map, and Dashboard gets: title, summary, description, tags, terms of use (license and restricted-data statement), credits, and a thumbnail. Pro carries item metadata over when sharing if the source layer's metadata is complete.

### 15.6 Metadata catalog

`docs/metadata_catalog.md`: one table row per object — name, type, abstract (1–2 sentences), CRS, source_ids, ISO/FGDC status, link to XML in `metadata/`.

### 15.7 Checks

- [ ] `MetadataStatus` = 100% complete and validated (acceptance criterion 4).
- [ ] Lineage lists every process step, including AIS scripts with Git commits.
- [ ] XML committed to `metadata/`.

---

## 16. Stage 13 — Automation: ModelBuilder and ArcPy toolbox

### 16.1 ModelBuilder (`toolbox/models/NARW_Models.tbx`)

Create the toolbox: *Catalog > Toolboxes > right-click > New Toolbox (.atbx)*; save as `NARW_Models.atbx` (Pro 3.x default) or `.tbx` [DECISION].

**M1_PresenceSurfaces**
1. *Analysis > ModelBuilder*. Add **Iterate Field Values** (Insert > Iterators): *Input* `sight_5070`, *Field* `month`.
2. **Make Feature Layer** with expression `month = %Value%`.
3. **Kernel Density** (parameters as §10.3); output `kde_sight_m%Value%`.
4. **Raster Calculator** (÷ N years).
5. **Zonal Statistics as Table** with `HexGrid_25km2`; output `zs_sight_m%Value%_h25`.
6. **Zonal Statistics as Table** with the month's density raster → `zs_dens_m%Value%_h25`.
7. Expose inputs (sightings layer, hex grid, bandwidth, cell size) as **model parameters** (right-click > Parameter).
8. Validate, run, and **export the diagram**: *ModelBuilder tab > Export > Export To Graphic* → `toolbox/diagrams/M1_PresenceSurfaces.svg` (and `.png`).
9. Also *Export > Export To Python File* for reference.

**M2_ComplianceAndRisk**
1. Inputs: `AIS_Transits`, `SMA`, `Hex4_VesselTraffic_Monthly`, `Hex4_WhalePresence_Monthly`.
2. **Select** transits (`sma_active_flag = 1 AND compliance_status <> 'NotApplicable'`) → **Summary Statistics** (by SMA, year, month, vessel class) → **Calculate Field** (percentages) → **Append** to `SMA_Compliance_Monthly`.
3. **Summary Statistics** on traffic (rule-applicable) → **Add Join** presence by `hex_id` + month (use a concatenated key field `hex_month`) → **Calculate Fields (multiple)** for `presence_norm`, `traffic_norm`, `speed_factor`, `risk_index` using Python expressions (the normalizing maxima/P99 are pre-computed with **Summary Statistics** and joined) → **Append** to `Hex4_Risk_Monthly`.
4. Export the diagram as above.

### 16.2 Python toolbox (`toolbox/NARW_Tools.pyt`)

A `.pyt` is a Python file that defines tools with parameters. Skeleton with one tool fully written; the other 11 follow the same pattern and call functions in `src/narw/`:

```python
# -*- coding: utf-8 -*-
import arcpy, os, sys, json, datetime as dt
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

class Toolbox:
    def __init__(self):
        self.label = "NARW Tools"
        self.alias = "narw"
        self.tools = [BuildSchema, IngestSource, RunQAQC, GenerateHexGrids, BuildPresenceSurfaces,
                      ProcessAIS, ComputeCompliance, ComputeRisk, EvaluateScenarios,
                      GenerateMetadata, ExportMapSeries, ArchiveSnapshot]

class GenerateHexGrids:
    def __init__(self):
        self.label = "Generate Hex Grids"
        self.description = "Create hexagon grids of the given sizes (km2) over the study area in EPSG:5070."
    def getParameterInfo(self):
        p0 = arcpy.Parameter(displayName="Study area", name="study_area", datatype="GPFeatureLayer",
                             parameterType="Required", direction="Input")
        p1 = arcpy.Parameter(displayName="Sizes (km2)", name="sizes", datatype="GPDouble",
                             parameterType="Required", direction="Input", multiValue=True)
        p1.value = [25, 4]
        p2 = arcpy.Parameter(displayName="Output geodatabase", name="out_gdb", datatype="DEWorkspace",
                             parameterType="Required", direction="Input")
        return [p0, p1, p2]
    def execute(self, parameters, messages):
        study, sizes, gdb = parameters[0].valueAsText, parameters[1].values, parameters[2].valueAsText
        sr = arcpy.SpatialReference(5070)
        for s in sizes:
            tag = f"{int(s)}km2"
            raw = f"memory\\hex_{tag}"
            arcpy.management.GenerateTessellation(raw, arcpy.Describe(study).extent, "HEXAGON",
                                                  f"{s} SquareKilometers", sr)
            lyr = arcpy.management.SelectLayerByLocation(raw, "INTERSECT", study)
            out = os.path.join(gdb, "Grids", f"HexGrid_{tag}")
            arcpy.management.Append(lyr, out, "NO_TEST")
            prefix = f"H{int(s)}_"
            arcpy.management.CalculateField(out, "hex_id", f"'{prefix}' + str(!OBJECTID!)", "PYTHON3")
            messages.addMessage(f"{out}: {arcpy.management.GetCount(out)[0]} hexes")

# class BuildSchema, IngestSource, RunQAQC, ... follow the same structure.
```

| # | Tool | Wraps |
|---|---|---|
| 1 | `BuildSchema` | Reads `schema/schema.yaml`; §6 tool calls |
| 2 | `IngestSource` | §7.1 recipe + registry + ProcessingLog |
| 3 | `RunQAQC` | §8 checks → `QAQC_Log` |
| 4 | `GenerateHexGrids` | §9 |
| 5 | `BuildPresenceSurfaces` | §10 (M1 logic + `presence.py`) |
| 6 | `ProcessAIS` | Runs `ais_preprocess/run_all.py` via `subprocess`, then loads outputs (§11.8) |
| 7 | `ComputeCompliance` | §12 |
| 8 | `ComputeRisk` | §13 (`risk.py`) |
| 9 | `EvaluateScenarios` | §14 |
| 10 | `GenerateMetadata` | §15.3 |
| 11 | `ExportMapSeries` | §17.5 (`arcpy.mp`) |
| 12 | `ArchiveSnapshot` | §21.2 |

**Standards:** every tool reads thresholds from `configs/analysis.yaml`; logs to a file (`logs/narw_YYYYMMDD.log`) and to `ProcessingLog`; uses `arcpy.da` cursors; has docstrings. Document parameters in `docs/tool_reference.md` and in the toolbox's item description (`NARW_Tools.pyt.xml`, created by editing tool metadata in Pro).

### 16.3 Tests (`tests/`)

Use `pytest` for logic that doesn't need ArcPy:
```python
# tests/test_rules.py
from narw.rules import haversine_nm, sma_active, leg_speed, classify

def test_haversine_one_degree_lat():
    assert abs(haversine_nm(40, -70, 41, -70) - 60.0) < 0.1          # 1° latitude ≈ 60 nm

def test_active_period_wraps_year():
    assert sma_active(1101, 430, month=1, day=15)
    assert not sma_active(1101, 430, month=7, day=1)

def test_leg_speed_prefers_reported_when_consistent():
    assert leg_speed(mean_reported=12.0, implied=12.8) == 12.0
    assert leg_speed(mean_reported=12.0, implied=16.0) == 16.0

def test_classification():
    assert classify(active=True, applicable=1, n_points=10, pct_over=15) == "NonCompliant"
    assert classify(active=True, applicable=1, n_points=2, pct_over=50) == "Indeterminate"
    assert classify(active=False, applicable=1, n_points=10, pct_over=50) == "NotApplicable"
```
Put the pure-Python versions of these rules in `src/narw/rules.py` (mirroring the SQL). Add an ArcPy **smoke test** that builds the schema in a temporary geodatabase and runs `GenerateHexGrids` on a small polygon (`tests/fixtures/`). Also keep a **small sample AIS file** (one day, one SMA, a few MB) in `tests/fixtures/` and assert the pipeline's transit count.

Run: `python -m pytest -q` from the Python Command Prompt.

### 16.4 Pipeline runner

`src/run_pipeline.py` — command-line entry point:
```
python src/run_pipeline.py --step all --years 2022-2024
python src/run_pipeline.py --step compliance --years 2023
```
Steps: `schema, ingest, qaqc, grids, presence, ais, compliance, risk, scenarios, metadata, maps, archive`. Each step calls the same functions as the toolbox and writes `ProcessingLog`. Acceptance criterion 5: re-running `--step all` reproduces the analysis tables with identical row counts.

### 16.5 Optional R notebook (`r/compliance_charts.Rmd`)

Read `SMA_Compliance_Monthly` exported to CSV (or the geodatabase with `sf::st_read` — tables need GDAL's OpenFileGDB driver); reproduce the compliance bar and line charts with `ggplot2`; knit to HTML. Demonstrates R alongside ArcPy.

---

## 17. Stage 14 — Cartography: static maps and map series

### 17.1 Cartographic standards

| Item | Standard |
|---|---|
| Page | Letter landscape (11 × 8.5 in) for map series; tabloid (17 × 11 in) optional for M01 |
| Map frame projection | EPSG:5070 for coast-wide; UTM 18N/19N or state-plane-like custom Albers for insets [DECISION] |
| Basemap | Esri **Ocean** basemap (muted) or **Light Gray Canvas**; NOAA ENC-style base for nautical feel (optional) |
| Bathymetric contours | 50, 100, 200, 1000 m from `Env_Bathymetry` (**Contour List (Spatial Analyst)**), thin gray |
| Fonts | One sans-serif family (e.g., *Segoe UI* or *Tahoma* in Pro; *Source Sans* if installed); titles 16–20 pt, body ≥ 7 pt print |
| Presence colors | Sequential single-hue blue–green (ColorBrewer **YlGnBu**, 5 classes) |
| Traffic colors | Sequential purple (**BuPu** or **Purples**) |
| Speed colors | Sequential (**YlOrBr**) |
| Risk colors | Sequential red-orange (**OrRd**, 5 classes; class 0 transparent or light gray) |
| Compliance colors | Categorical, color-vision-safe (**Okabe–Ito**: Compliant `#0072B2`, NonCompliant `#D55E00`, Indeterminate `#999999`, NotApplicable `#E6E6E6`) |
| Regulatory outlines | SMA: dark outline 1.5 pt, no fill (or 10% fill); Slow Zones: dashed; Critical habitat: hatched; wind leases: thin black outline |
| Class breaks | **Fixed across months** (same breaks on all 12 pages); document breaks in the legend |
| Check colors | https://colorbrewer2.org/ ; test with a color-vision-deficiency simulator (e.g., Coblis) |

### 17.2 Layout template (`NARW_Template.pagx`)

Build once in Pro (*Insert > New Layout > Letter Landscape*) and save with *Share > Layout File (.pagx)* to `maps/`:

| Element | Position | Content |
|---|---|---|
| Title | Top left | Dynamic text: `<dyn type="page" property="name"/>` (map series) or fixed |
| Subtitle | Below title | Month / SMA name, data period |
| Main map frame | Left 70% | Graticule (2° or 1°) with labels for offshore maps |
| Inset / locator | Top right | U.S. East Coast with extent indicator |
| Legend | Right column | Fixed breaks; layer names in plain language |
| Scale bar | Bottom of map | Nautical miles **and** kilometers (two scale bars) |
| North arrow | Map corner | Simple |
| Data sources + vintages | Bottom right, 7 pt | "Sightings: NOAA RWSAS 2010–2025; AIS: Marine Cadastre 2022–2024; …" |
| CRS note | Bottom right | "NAD 1983 Contiguous USA Albers (EPSG:5070)" |
| Methods note | Bottom right | One sentence + "see Technical Report §x" |
| Limitations note | Bottom | "Relative index; sightings effort-biased; AIS excludes non-broadcasting vessels." |
| Version and date | Bottom left | `v1.0 — <dyn type="date" format="yyyy-MM-dd"/>` |
| Page number | Bottom center | `<dyn type="page" property="index"/> of <dyn type="page" property="count"/>` |
| Author | Bottom left | "Bernard Issifu" |

### 17.3 Map products

| # | Product | Layers | Notes |
|---|---|---|---|
| **M01** | Study area and management framework | StudyArea, SMAs (by region), Critical Habitat Units 1–2, TSS/lanes, wind leases, ports, EEZ, 50-nm focus line, bathymetric contours | Tabloid; labels for each SMA |
| **M02** | Data coverage | Sightings by year (graduated or small multiples 2010–2025), recorder locations (sized by monitored days), density-model extent | 2 × 2 small multiples |
| **M03** | Data fitness summary | Hexes symbolized by `method` (S / S+D / S+A+D), effort gaps, AIS coverage caveat zones | Direct link to fitness assessment |
| **MS01** | Whale presence series (12 pages) | `Hex25_WhalePresence_Monthly` (clim) joined to grid; 4 km² insets near SMAs | Monthly map series (§17.4) |
| **MS02** | Vessel traffic series (12 pages) | Rule-applicable nm (color) and mean speed (inset or second frame) | 2022–2024 monthly mean |
| **MS03** | SMA compliance series (1 page per SMA) | SMA with transits colored by status (sample) + chart of % non-compliant by class and month | Spatial map series by `SMA` |
| **MS04** | Risk series (12 pages) | `Hex4_Risk_Clim` classes with current SMAs overlaid (active SMAs bold, inactive dashed) | |
| **M04** | Scenario comparison | Small multiples: baseline + SC1–SC4 footprints over annual risk | Label each with Δ % risk covered |
| **M05** | Wind-lease co-occurrence | Presence and risk within 20 km of leases, Dec–Apr | Section 7 context |

### 17.4 How to build the monthly map series (non-spatial pages)

Pro's map series is spatial (one page per index feature). For **one page per month over the same extent**:
1. Create an index layer `MonthIndex`: 12 copies of the study-area envelope polygon, each with a `month` (1–12) and `month_name` field (**Copy Features** then Append 11 times, or a small ArcPy loop).
2. Create a polygon feature class for the data layer: `Hex4_Risk_Clim` joined to `HexGrid_4km2` → **Export Features** (so each hex appears 12 times with a `month` field).
3. Layout > **Map Series > Spatial**: *Layer* `MonthIndex`, *Name field* `month_name`, *Sort field* `month`, *Extent* = "Best fit".
4. On the data layer: *Layer Properties > Page Query* → *Field* `month`, *Match* → only features whose `month` equals the current page's are drawn.
5. Title dynamic text uses the page name.

For **MS03** (per SMA): a normal spatial map series with `SMA` as the index layer, *Extent* = "Best fit" with 20% margin; chart via a **chart frame** (Pro 3.x) or an exported chart image per SMA.

### 17.5 Exporting

- Layout > **Share > Export Layout** → PDF, 300 dpi, *Pages: All*, *Export as multiple files: No* (one multipage PDF per series), *Embed fonts*, *Output as image* off. Save to `outputs/map_series/MS01_presence.pdf` etc.
- Also PNG (150 dpi) of key pages for the portfolio and report.
- ArcPy (`ExportMapSeries` tool):
```python
import arcpy
aprx = arcpy.mp.ArcGISProject(r"C:\GIS\NARW\05_Maps\NARW.aprx")
for name, out in [("MS01_Presence", "MS01_presence.pdf"), ("MS04_Risk", "MS04_risk.pdf")]:
    lyt = aprx.listLayouts(name)[0]
    lyt.mapSeries.exportToPDF(rf"C:\GIS\NARW\outputs\map_series\{out}", "ALL", resolution=300)
```

### 17.6 Cartographic checklist (run on every product; record in `docs/reviews/review_03.md`)

- [ ] Title and subtitle state what, where, when.
- [ ] Legend matches symbology; class breaks stated and fixed across pages.
- [ ] Scale bars (nm and km), north arrow, graticule (offshore maps).
- [ ] CRS note, data sources with vintages, methods note, limitations note, version/date, author, page numbers.
- [ ] Color-vision-safe palettes; checked in a simulator.
- [ ] Minimum 7 pt text; no overlapping labels.
- [ ] No restricted raw points shown (NARWC): only aggregated hexes.
- [ ] Spelling and SMA names match eCFR.
- [ ] Alt text written for each figure (used in the report).

---

## 18. Stage 15 — ArcGIS Online web map and Dashboard

### 18.1 Prepare portfolio-safe layers (in a separate `publish.gdb`)

| Layer | Content | Time field |
|---|---|---|
| `Presence_Hex25_Clim` | 25 km² hex polygons × 12 months with `presence_index`, `method` | `month_date` (2000-MM-01, a nominal year for climatology) |
| `Risk_Hex4_Clim` | 4 km² hex × 12 months with `risk_index`, `risk_class`, `inside_sma` | `month_date` |
| `Traffic_Hex4_Clim` | 4 km² hex × 12 months, rule-applicable nm and mean speed | `month_date` |
| `SMA`, `DMA_SlowZones`, `CriticalHabitat`, `WindLeaseAreas`, `CandidateSMA` | Polygons | Slow Zones: `start_datetime`/`end_datetime` |
| `SMA_Compliance_Monthly` | Hosted **table** | `month_date` (actual year-month) |
| `Scenario_Summary` | Hosted table | — |

- **Do not publish** `Sightings` (contains restricted records) or `AIS_Points_Sample` raw pings — or publish only public-source sightings aggregated to hexes.
- Keep feature counts manageable: 12 months × ~60,000 hexes ≈ 720,000 features for the 4 km² grid. If publishing is slow, publish only hexes with `risk_index > 0`, or use **Generalize** + a **vector tile layer** for display with a feature layer for pop-ups [DECISION].
- Set **field aliases** in plain language (e.g. `risk_index` → "Relative strike risk (0–100)").

### 18.2 Publish

1. In Pro, add the layers to a map named `Web_NARW`; set symbology per §17.1.
2. **Enable time** on the time-series layers: *Layer Properties > Time > Filter layer content based on attribute values*, *Time field* `month_date`, *Time step interval* 1 month.
3. Configure pop-ups (*Configure Pop-ups*): plain-language fields, rounded numbers, a line of caveats.
4. **Share > Web Layer > Publish Web Layer**: *Layer type* `Feature`; *Location*: folder `NARW`; *Share with*: Owner only (for now); fill item *Summary*, *Tags*, *Description*. Analyze → fix errors → Publish.
5. Upload tables the same way (they become hosted tables).

### 18.3 Web map (Map Viewer)

1. ArcGIS Online > **Map** → add the hosted layers.
2. Group layers: *Whale presence*, *Vessel traffic*, *Risk*, *Management areas*, *Context*.
3. Basemap: Oceans or Light Gray.
4. **Time slider** (*Map properties > Time slider*): monthly steps, loop.
5. Pop-ups: review in Map Viewer; add Arcade expressions for friendly text, e.g. `Text(Round($feature.risk_index, 1)) + " / 100"`.
6. Bookmarks: Southeast calving grounds, Chesapeake, NY/NJ, Southern New England, Cape Cod Bay.
7. Save as **"NARW Vessel-Strike Risk — Web Map"** with complete item details.

### 18.4 Dashboard (ArcGIS Dashboards)

Create *Dashboards > Create dashboard* → name "NARW Vessel-Strike Risk Dashboard".

| Element | Type | Configuration |
|---|---|---|
| Header | Header | Title, subtitle, link to report and GitHub |
| Month selector | **Category selector** (header) | Values from `month` (1–12, labeled); **actions** filter the risk/presence/traffic layers and compliance table |
| Scenario toggle | Category selector | Values from `CandidateSMA.scenario_name` + "Baseline"; filters `CandidateSMA` layer and the scenario table |
| Main map | Map | The web map above |
| Indicator 1 | Indicator | "% of coast-wide risk inside active SMAs" — from a small hosted table of monthly baseline shares, filtered by month |
| Indicator 2 | Indicator | "% transits non-compliant" — `SMA_Compliance_Monthly`, statistic = Sum(noncompliant) / Sum(compliant + noncompliant) (use an Arcade **data expression** if a ratio of sums is needed) |
| Serial chart | Serial chart | % non-compliant by month, split by vessel class |
| Bar chart | Serial chart | % non-compliant by SMA |
| List | List | SMAs with name, active period, % non-compliant; **action**: zoom map and filter charts |
| Scenario table | Table | `Scenario_Summary` — Δ % risk covered, added area, added transits |
| Caveats | Rich text | Limitations and "appropriate uses" |

Test with each month and scenario; check the mobile layout (*Dashboard > Layout*).

### 18.5 Sharing review (before going public)

- [ ] No restricted raw points in any layer or table (acceptance criterion 8).
- [ ] Item metadata complete on every item (§15.5).
- [ ] Terms of use state data sources and "relative index — not for navigation or regulatory enforcement."
- [ ] Share items with **Everyone (public)**; copy links to README and portfolio.

---

## 19. Stage 16 — Decision briefs

**One page per SMA** (≈ 10) **and per scenario** (≈ 5). Audience: program staff and regulators.

**Layout (Letter portrait):**

| Block | Content |
|---|---|
| Header | "Decision Brief — <SMA name> Seasonal Management Area" · region · active period · v1.0 · date |
| Map (40% of page) | SMA with risk (active months), traffic lanes, Slow Zones history |
| Key numbers (4–6 tiles) | Transits (active periods 2022–2024) · % non-compliant (≥ 65 ft) · % distance > 10 kn · worst vessel class · share of coast-wide risk in this SMA · indeterminate share |
| Trend | Small chart 2022 → 2024, % non-compliant |
| What this shows | 3 bullets in plain language |
| Caveats | AIS coverage, effort bias, relative index, rule text verification |
| Can be used for / cannot be used for | e.g. *Can*: prioritizing outreach, comparing SMAs. *Cannot*: enforcement against individual vessels; absolute strike estimates |
| Contact / source | Report section reference, repository link |

**How:** build a Pro layout with a **spatial map series on `SMA`**, dynamic text for the numbers (use **Attribute** dynamic text from a table joined to the SMA layer), a chart frame, and text boxes; export one PDF per SMA. Scenario briefs use `CandidateSMA` as the index. Save to `outputs/briefs/`.

---

## 20. Stage 17 — Technical report

**File:** `docs/NARW_VesselStrike_TechReport.pdf` (25–40 pages + appendices), NOAA technical-memorandum style: plain title page (title, author, affiliation "Independent portfolio project", date, series-style number e.g. "NARW-TM-01"), suggested citation, abstract, table of contents, numbered sections, figures and tables numbered with captions, references in a consistent author–year style.

| # | Section | Pages | Content |
|---|---|---|---|
| — | Executive summary | 1–2 | Question, 3–5 key findings with numbers, recommendations, caveats |
| 1 | Introduction and management context | 2–3 | Species status, ESA/MMPA, speed rule, Slow Zones, critical habitat, Section 7, 2022 proposed amendments |
| 2 | Study area | 1 | M01 map |
| 3 | Data sources and fitness for use | 3–4 | Registry summary table; decisions; M02/M03 |
| 4 | Geodatabase design | 2 | ERD, design choices |
| 5 | QA/QC | 2 | Checks, results summary |
| 6 | Methods | 6–8 | Presence, AIS processing, compliance, risk, scenarios, sensitivity — with equations and parameter table |
| 7 | Results | 6–8 | Presence patterns; traffic; compliance by SMA/month/class; risk and baseline share; scenarios; sensitivity |
| 8 | Discussion | 2–3 | Interpretation; comparison with published compliance studies (e.g., Silber et al. 2014; NOAA 2020 speed-rule assessment) |
| 9 | Assumptions and limitations | 1–2 | §14 and the standard caveats |
| 10 | Appropriate uses | 1 | Per product |
| 11 | Reproducibility | 1 | Repo, environment, `run_pipeline.py`, tool reference, archive DOI |
| — | References | 2 | Appendix G |
| App. A–G | Data source registry; data dictionary; QA/QC summary; metadata catalog; map gallery; tool parameter reference; scenario tables | — | |

**Tools:** Word (with styles and auto-numbered captions) or LaTeX/Quarto → PDF. Include **alt text** for every figure. Proofread; have one peer read it.

---

## 21. Stage 18 — Versioning, archival, and release

### 21.1 Git practice

- Commit daily; tag milestones: `v0.1-schema` (Week 2), `v0.5-analysis` (Week 6), `v1.0` (Week 10). `git tag -a v1.0 -m "Release 1.0"` then `git push --tags`.
- `CHANGELOG.md`: per release, what changed in schema, data, methods, and products.

### 21.2 Geodatabase snapshot procedure (every milestone)

1. Close ArcGIS Pro (release locks).
2. **Compact (Data Management)** the geodatabase.
3. Zip: right-click `NARW_VesselStrike.gdb` → 7-Zip → *Add to archive* → `NARW_VesselStrike_v2026.11.gdb.zip` in `07_Archive/`.
4. Checksum (PowerShell):
   ```powershell
   Get-FileHash C:\GIS\NARW\07_Archive\NARW_VesselStrike_v2026.11.gdb.zip -Algorithm SHA256
   ```
5. Add a `VersionHistory` row: version, date, description, archive path, checksum, Git tag.
6. Write `07_Archive/README_v2026.11.md`: contents, schema version, data vintages, checksum, how to restore.
7. Copy the zip + README to cloud storage (e.g., OneDrive/Google Drive/S3) — and for v1.0, deposit on **Zenodo** to get a DOI (upload only public, portfolio-safe data; exclude restricted records).

### 21.3 Mobile geodatabase export and SQL demonstration

1. **Create Mobile Geodatabase (Data Management)** → `NARW_VesselStrike.geodatabase`.
2. **Copy** the vector feature datasets and tables into it (Catalog: copy/paste, or **Feature Class To Geodatabase** + **Table To Geodatabase**). Mosaic datasets and rasters are not supported in mobile geodatabases — note this in the write-up.
3. SQL demo — **Create Database View (Data Management)** in the mobile geodatabase, e.g.:
   ```sql
   SELECT sma_id, vessel_class,
          SUM(transits_noncompliant) AS nc,
          SUM(transits - transits_indeterminate) AS judged,
          ROUND(100.0 * SUM(transits_noncompliant) / SUM(transits - transits_indeterminate), 1) AS pct_nc
   FROM SMA_Compliance_Monthly
   GROUP BY sma_id, vessel_class
   ```
   ```sql
   SELECT month, ROUND(100.0 * SUM(CASE WHEN inside_sma_active = 1 THEN risk_index ELSE 0 END) / SUM(risk_index), 1) AS pct_risk_in_sma
   FROM Hex4_Risk_Monthly GROUP BY month ORDER BY month
   ```
4. Mobile geodatabases are SQLite files — also open read-only in **DB Browser for SQLite** (https://sqlitebrowser.org/) for a screenshot showing plain SQL access.

---

## 22. Stage 19 — Portfolio packaging

### 22.1 Repository README

Sections: title + one-sentence summary; hero image (MS04 page); **key findings (3–5 numbers)**; links (Dashboard, web map, report PDF, map series, geodatabase write-up); methods in brief with the pipeline diagram; data sources table; repository layout; how to reproduce (`run_pipeline.py`); limitations; license (code MIT; documentation CC BY 4.0 [DECISION]); citation (Zenodo DOI).

### 22.2 Portfolio pages (https://ndeogobernard.github.io/ndeogo/)

**A. Flagship page:** problem → question → data → methods (graphic) → findings (3–5 with charts/maps) → decision-support products (embedded Dashboard link, map series thumbnails) → limitations → skills demonstrated (map to the NOAA GIS Analyst JD, original scope Appendix A) → links.

**B. Geodatabase & metadata page:** design goals; ERD image; domains/subtypes rationale; restricted vs. public separation; relationship classes and topology; attribute rules; metadata completeness table; versioning/archival with checksum; **60-second screen capture**.

### 22.3 60-second screen capture — shot list

| Seconds | Show |
|---|---|
| 0–8 | Catalog pane: geodatabase tree (feature datasets, tables, mosaic datasets) |
| 8–18 | Domains view (`dm_VesselClass`, `dm_ComplianceStatus`) and Sightings subtypes |
| 18–28 | Select an SMA → attribute table → related `AIS_Transits` via the relationship class |
| 28–38 | Topology: Validate → Error Inspector (show an exception) |
| 38–46 | Attribute rule firing: edit a sighting's datetime → `month` updates |
| 46–56 | Metadata view: lineage process steps and field definitions |
| 56–60 | `VersionHistory` table with checksum |

Record with OBS; export MP4 (1080p) and a GIF preview.

---

## 23. Week-by-week work plan with task checklists

(10 weeks, part-time ≈ 12–15 h/week. Weekly status note every Friday.)

**Week 1 — Setup and acquisition**
- [ ] §4 folders, Git, `.gitignore`, Pro project, decision log (D-001…D-018)
- [ ] Accounts: AGOL org, Earthdata; **send NARWC request**
- [ ] Download S02–S08, S10–S15; start S09 downloads (background)
- [ ] Draft `DataSourceRegistry` spreadsheet (16 rows); save terms PDFs
- [ ] Status note W1

**Week 2 — Geodatabase**
- [ ] §6 domains, feature datasets, classes, tables, subtypes, relationships, topology, attribute rules, editor tracking
- [ ] Export schema XML; `schema.yaml`; ERD; data dictionary v1; `GDB_Design.md` draft
- [ ] **Design Review 1** record
- [ ] Status note W2

**Week 3 — Ingest and QA run 1**
- [ ] §7 load S01–S08, S10–S16 (not AIS)
- [ ] §8 QA run 1 → `QAQC_Log`; fixes; QA run 2
- [ ] Fitness-for-Use Assessment draft (all 16)
- [ ] §9 hex grids; tag points
- [ ] Status note W3

**Week 4 — AIS**
- [ ] §11 B1 Parquet conversion (all months)
- [ ] B2–B6 pipeline on one month; validate vs. ArcGIS-only sample (§11.9)
- [ ] Run all 36 months; load outputs; G1/G2 reconciliation
- [ ] Status note W4

**Week 5 — Presence and compliance**
- [ ] §10 presence surfaces (clim + 2022–2024)
- [ ] §12 compliance tables, charts, per-SMA summaries
- [ ] **Design Review 2** record
- [ ] Status note W5

**Week 6 — Risk and scenarios**
- [ ] §13 risk index; baseline share
- [ ] §14 CandidateSMA; Scenario_Summary; sensitivity; tornado chart
- [ ] Tag `v0.5-analysis`; snapshot
- [ ] Status note W6

**Week 7 — Metadata**
- [ ] §15 metadata for every object; exports; validation; `MetadataStatus` 100%; catalog
- [ ] ModelBuilder M1/M2 diagrams; toolbox tools wired; tests passing (§16)
- [ ] Status note W7

**Week 8 — Maps**
- [ ] §17 template; M01–M05; MS01–MS04; exports
- [ ] Cartographic checklist; **Design Review 3** record
- [ ] Status note W8

**Week 9 — Web and briefs**
- [ ] §18 publish layers; web map; Dashboard; sharing review
- [ ] §19 decision briefs (SMAs + scenarios)
- [ ] Status note W9

**Week 10 — Report and release**
- [ ] §20 technical report PDF
- [ ] `GDB_Design.md` final; screen capture
- [ ] §21 v1.0 snapshot, checksum, README, Zenodo DOI; tag `v1.0`
- [ ] §22 README and two portfolio pages live
- [ ] Final status note; acceptance checklist (§24) signed

**Critical path if time runs short:** geodatabase → QA → AIS → compliance → risk → metadata → maps. Defer: scenarios beyond SC1/SC3, sensitivity breadth, R notebook, mobile geodatabase demo.

---

## 24. Acceptance criteria and review checklists

### 24.1 Acceptance criteria (sign off each with evidence)

| # | Criterion | Evidence |
|---|---|---|
| 1 | Geodatabase can be recreated from `schema.yaml` / schema XML on a clean machine; schema matches the documented design | Rebuild log; schema report diff |
| 2 | Every source has a fitness assessment and registry row; every load/analysis step has a `ProcessingLog` row with a Git commit | Tables |
| 3 | All QA checks run, with results and reviewer sign-off in `QAQC_Log` | Table + summary appendix |
| 4 | 100% of objects have complete, validated ISO 19115 and FGDC metadata; lineage lists every step | `MetadataStatus`; validator outputs |
| 5 | `run_pipeline.py --step all` reproduces `SMA_Compliance_Monthly`, `Hex4_Risk_Monthly`, `Scenario_Summary` with identical row counts | Two run logs |
| 6 | Compliance reported by SMA, month, vessel class for all active periods 2022–2024; indeterminate share documented | Table; report section |
| 7 | Sensitivity shows scenario ranking stability explicitly | `Sensitivity_Results`; report statement |
| 8 | Map products pass the checklist; web map and Dashboard work (time slider, scenario toggle); no restricted raw points published | Review 3 record; sharing review |
| 9 | Report states methods, assumptions, limitations, appropriate uses for every product | Report |
| 10 | v1.0 snapshot archived with checksum and README | `VersionHistory`; archive |

### 24.2 Design review checklists

**Review 1 (Week 2 — design):** schema matches scope; CRS decisions; domains cover source values; restricted-data separation; naming conventions; topology rules and exceptions justified.

**Review 2 (Week 5 — data and methods):** QA results acceptable; fitness decisions; AIS pipeline validated against a sample; presence components' coverage; compliance rule parameters; early results plausible.

**Review 3 (Week 8 — products):** risk and scenario results plausible; sensitivity done; cartographic checklist; metadata completeness; web-publishing plan and restricted-data check.

Each record (`docs/reviews/review_0N.md`, template F.6): date, reviewer(s), materials reviewed, findings, decisions, action items with owners and due dates, sign-off.

---

## 25. Risks, issues, and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| NARWC access delayed or denied | Medium | Medium | Proceed with RWSAS, WhaleMap, OBIS, PACM, density models; document the gap and presence-only limitation |
| AIS volume exceeds disk/RAM | Medium | High | Month-by-month processing; Parquet + ZSTD; delete CSVs; DuckDB memory limit; external SSD |
| Data portal changes (URLs/formats) | High | Low–Medium | [VERIFY] steps; record in decision log; keep raw copies |
| Speed-rule amendments or boundary changes | Low–Medium | Medium | Version regulatory layers with citations and effective dates; parameterize scenarios |
| Effort bias in sightings | Certain | Medium | Effort adjustment where possible; density model weight; coverage map (M03); caveats |
| Density model era/version mismatch | Low | Medium | Use most recent era; record version |
| Metadata validators unavailable/aging | Medium | Low | ISO primary; document validator used; Pro export validation |
| Web layer too large for AGOL | Medium | Low | Climatology only; publish non-zero hexes; vector tiles |
| Sensitive locations exposed | Low | High | Aggregated hexes only; sharing review checklist |
| Schedule slip (part-time) | Medium | Medium | Critical path (§23); defer optional items |
| ArcGIS license lacks Spatial Analyst / Standard | Low | High | Confirm in Week 1 (*Project > Licensing*) |

---

## Appendix A — Master link list

> All links **[VERIFY]** at use; record changes in the decision log. These links were compiled from known agency URL patterns but could not be live-tested when this guide was written, so check each one in Week 1 (Stage 2).

| ID | Resource | URL |
|---|---|---|
| — | NOAA Fisheries — NARW species page | https://www.fisheries.noaa.gov/species/north-atlantic-right-whale |
| — | NOAA Fisheries — Reducing vessel strikes (speed rule, Slow Zones) | https://www.fisheries.noaa.gov/national/endangered-species-conservation/reducing-vessel-strikes-north-atlantic-right-whales |
| — | eCFR 50 CFR 224.105 (speed rule text) | https://www.ecfr.gov/current/title-50/chapter-II/subchapter-C/part-224/section-224.105 |
| — | Federal Register search | https://www.federalregister.gov/ |
| — | North Atlantic Right Whale Consortium | https://www.narwc.org/ |
| S01 | NOAA NEFSC Right Whale Sighting Advisory System | https://apps-nefsc.fisheries.noaa.gov/psb/surveys/ |
| S01 | WhaleMap | https://whalemap.org/ |
| S02 | NOAA Passive Acoustic Cetacean Map | https://apps-nefsc.fisheries.noaa.gov/pacm/ |
| S02 | NOAA NCEI Passive Acoustic Data | https://www.ncei.noaa.gov/products/passive-acoustic-data |
| S03 | Duke MGEL U.S. East Coast density models | https://seamap.env.duke.edu/models/Duke/EC/ |
| S04 | OBIS-SEAMAP | https://seamap.env.duke.edu/ |
| S04 | OBIS | https://obis.org/ |
| S05–S08, S11, S15 | Marine Cadastre data hub | https://hub.marinecadastre.gov/ |
| S07 | NARW critical habitat map and GIS data | https://www.fisheries.noaa.gov/resource/map/north-atlantic-right-whale-critical-habitat-map-and-gis-data |
| S07 | National ESA Critical Habitat Mapper | https://www.fisheries.noaa.gov/resource/map/national-esa-critical-habitat-mapper |
| S09 | Marine Cadastre Vessel Traffic | https://hub.marinecadastre.gov/pages/vesseltraffic |
| S09 | AIS daily file index (example year) | https://coast.noaa.gov/htdata/CMSP/AISDataHandler/2024/index.html |
| S09 | AWS Registry of Open Data | https://registry.opendata.aws/ |
| S10 | BOEM Renewable Energy GIS Data | https://www.boem.gov/renewable-energy/mapping-and-data/renewable-energy-gis-data |
| S11 | BTS geospatial data (NTAD) | https://geodata.bts.gov/ |
| S11 | USACE Waterborne Commerce Statistics Center | https://www.iwr.usace.army.mil/About/Technical-Centers/WCSC-Waterborne-Commerce-Statistics-Center/ |
| S11 | UN/LOCODE | https://unece.org/trade/cefact/unlocode-code-list-country-and-territory |
| S12 | NOAA Coastal Relief Model | https://www.ncei.noaa.gov/products/coastal-relief-model |
| S12 | GEBCO gridded bathymetry | https://www.gebco.net/data_and_products/gridded_bathymetry_data/ |
| S13 | NOAA OISST | https://www.ncei.noaa.gov/products/optimum-interpolation-sst |
| S13 | MUR SST (PO.DAAC) | https://podaac.jpl.nasa.gov/dataset/MUR-JPL-L4-GLOB-v4.1 |
| S13/S14 | NOAA CoastWatch ERDDAP (West Coast node) | https://coastwatch.pfeg.noaa.gov/erddap/ |
| S13/S14 | NOAA CoastWatch ERDDAP (central) | https://coastwatch.noaa.gov/erddap/ |
| S14 | NASA Ocean Color | https://oceancolor.gsfc.nasa.gov/ |
| S15 | NOAA U.S. Maritime Limits and Boundaries | https://nauticalcharts.noaa.gov/data/us-maritime-limits-and-boundaries.html |
| S15 | NOAA Shoreline Website | https://shoreline.noaa.gov/ |
| S15 | Census cartographic boundary files | https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html |
| — | NOAA InPort metadata catalog | https://www.fisheries.noaa.gov/inport/ |
| — | USGS metadata tools (mp) | https://geology.usgs.gov/tools/metadata/ |
| — | ArcGIS Pro documentation | https://pro.arcgis.com/en/pro-app/latest/help/main/welcome-to-the-arcgis-pro-app-help.htm |
| — | ArcGIS Pro tool reference | https://pro.arcgis.com/en/pro-app/latest/tool-reference/main/arcgis-pro-tool-reference.htm |
| — | DuckDB spatial extension | https://duckdb.org/docs/extensions/spatial/overview |
| — | ColorBrewer | https://colorbrewer2.org/ |
| — | Zenodo | https://zenodo.org/ |

---

## Appendix B — Marine Cadastre AIS field dictionary

[VERIFY against the current Marine Cadastre AIS FAQ / data dictionary.]

| Field | Type | Description | Notes |
|---|---|---|---|
| `MMSI` | Integer | Maritime Mobile Service Identity (9 digits) | First 3 digits = Maritime Identification Digits (country) |
| `BaseDateTime` | Timestamp | Position time, UTC (`YYYY-MM-DDTHH:MM:SS`) | |
| `LAT`, `LON` | Double | WGS 84 decimal degrees | |
| `SOG` | Double | Speed over ground, knots | 102.3 = not available |
| `COG` | Double | Course over ground, degrees | 360 = not available |
| `Heading` | Double | True heading, degrees | 511 = not available |
| `VesselName` | Text | Name | |
| `IMO` | Text | IMO number | |
| `CallSign` | Text | Radio call sign | |
| `VesselType` | Integer | AIS ship type code (Appendix C) | Includes U.S.-specific 1001–1025 codes |
| `Status` | Integer | Navigational status (0 = under way using engine, 1 = at anchor, 5 = moored, 7 = fishing, 8 = under way sailing, 15 = undefined) | |
| `Length` | Double | Length overall, meters | 0/blank = unknown |
| `Width` | Double | Beam, meters | |
| `Draft` | Double | Draft, meters | |
| `Cargo` | Integer | Cargo type code | |
| `TransceiverClass` | Text | `A` or `B` | Class B under-represented |

---

## Appendix C — Vessel class lookup (AIS type codes)

Save as `configs/vessel_classes.csv` (`ais_type_code,vessel_type,vessel_class,rule_applicable_flag,notes`). `rule_applicable_flag = 0` marks classes **exempt** from the speed rule regardless of length; `1` means applicability is decided by length (≥ 65 ft).

| AIS code(s) | Vessel type | `vessel_class` | `rule_applicable_flag` |
|---|---|---|---|
| 0 | Not available | 9 Unknown | 1 |
| 20–29 | Wing in ground | 8 Other | 1 |
| 30 | Fishing | 4 Fishing | 1 |
| 31, 32 | Towing / towing (large) | 5 Tug/Tow | 1 |
| 33 | Dredging / underwater operations | 8 Other | 1 |
| 34 | Diving operations | 8 Other | 1 |
| 35 | Military operations | 7 Military/LawEnforcement | **0** (exempt) |
| 36 | Sailing | 6 Pleasure/Sailing | 1 |
| 37 | Pleasure craft | 6 Pleasure/Sailing | 1 |
| 40–49 | High-speed craft | 3 Passenger [DECISION] | 1 |
| 50 | Pilot vessel | 8 Other | 1 |
| 51 | Search and rescue | 7 Military/LawEnforcement | **0** (government; verify) |
| 52 | Tug | 5 Tug/Tow | 1 |
| 53 | Port tender | 8 Other | 1 |
| 54 | Anti-pollution equipment | 8 Other | 1 |
| 55 | Law enforcement | 7 Military/LawEnforcement | **0** (exempt) |
| 56–57 | Spare (local use) | 8 Other | 1 |
| 58 | Medical transport | 8 Other | 1 |
| 59 | Noncombatant ship | 8 Other | 1 |
| 60–69 | Passenger | 3 Passenger | 1 |
| 70–79 | Cargo | 1 Cargo | 1 |
| 80–89 | Tanker | 2 Tanker | 1 |
| 90–99 | Other | 8 Other | 1 |
| 1001–1025 | U.S.-specific codes (e.g., commercial fishing, freight barge, tank ship, public vessel, recreational) | Map each individually | Map each individually (public/government vessels → 0) |

> Map the 1001–1025 codes **one by one** from the official Marine Cadastre vessel-type-code document (Appendix A, S09). Exemptions follow 50 CFR 224.105 (U.S. vessels owned or operated by, or under contract to, federal agencies, and law enforcement vessels of a state or political subdivision engaged in enforcement or search and rescue) [VERIFY exact wording]. AIS cannot identify all contracted federal vessels — state this limitation.

---

## Appendix D — Seasonal Management Area reference

**[VERIFY every name, boundary, and date against the current eCFR text of 50 CFR 224.105 and the NOAA SMA shapefile before use.]**

| Region | SMA | Active period (inclusive) | Geometry notes |
|---|---|---|---|
| Northeast | Cape Cod Bay | Jan 1 – May 15 | Polygon in Cape Cod Bay |
| Northeast | Off Race Point | Mar 1 – Apr 30 | Polygon north/east of Race Point |
| Northeast | Great South Channel | Apr 1 – Jul 31 | Polygon east of Cape Cod |
| Mid-Atlantic | Block Island Sound | Nov 1 – Apr 30 | Polygon across Block Island Sound approaches |
| Mid-Atlantic | Ports of New York / New Jersey | Nov 1 – Apr 30 | 20-nm radius arc around port entrance |
| Mid-Atlantic | Entrance to Delaware Bay (Philadelphia, Wilmington) | Nov 1 – Apr 30 | 20-nm radius arc |
| Mid-Atlantic | Entrance to Chesapeake Bay (Hampton Roads, Baltimore) | Nov 1 – Apr 30 | 20-nm radius arc |
| Mid-Atlantic | Ports of Morehead City and Beaufort, NC | Nov 1 – Apr 30 | 20-nm radius arc |
| Mid-Atlantic | Wilmington, NC to Brunswick, GA | Nov 1 – Apr 30 | Continuous band ~20 nm from shore |
| Southeast | Southeast U.S. calving and nursery area (GA / NE FL) | Nov 15 – Apr 15 | Polygon off Georgia and northeast Florida |

Rule parameters: vessels **≥ 65 ft (19.8 m)** overall length; **≤ 10 knots** over ground; exemptions and safety deviation provisions per the regulation.

---

## Appendix E — Analysis parameters (`configs/analysis.yaml`)

```yaml
crs:
  storage: 4269
  analysis: 5070
  web: 3857
  transformation: WGS_1984_(ITRF00)_To_NAD_1983
bbox_wgs84: {xmin: -82.0, xmax: -65.0, ymin: 24.0, ymax: 46.0}
hex:
  coastwide_km2: 25
  detail_km2: 4
  detail_extent: focus_zone_50nm      # D-004
  sma_buffer_km: 10
ais:
  years: [2022, 2023, 2024]
  rule_min_length_m: 19.812           # 65 ft
  scenario_min_length_m: 10.668       # 35 ft
  gap_split_min: 30
  max_implied_speed_kn: 60
  sog_spike_kn_nonpassenger: 40
  reported_vs_implied_tolerance_kn: 2 # D-009
  sample_fraction: 0.01
  assume_applicable_if_length_missing: [1, 2, 3]   # Cargo, Tanker, Passenger (D-011)
compliance:
  speed_limit_kn: 10
  tolerance_kn: 0.5
  noncompliant_pct_distance: 10
  min_points_in_sma: 3
presence:
  kernel_bandwidth_km: 25
  kernel_cell_m: {hex25: 5000, hex4: 2000}
  acoustic_radius_km: 20
  possibly_detected_score: 0.5
  weights: {sightings: 0.4, acoustic: 0.2, density_model: 0.4}
  time_basis: climatology             # D-013
risk:
  speed_factor: {min_kn: 10, min_value: 0.5, max_kn: 15, max_value: 1.0}
  traffic_scaling: log1p_p99          # D-018
  class_method: pooled_quintiles_nonzero
  month_active_rule: any_day          # §13.2 step 4
scenarios:
  SC1: {name: "Mid-Atlantic SMAs to 30 nm", radius_nm: 30, active: [1101, 430], min_len_ft: 65}
  SC2a: {name: "Northeast SMAs shifted -1 month", shift_months: -1}
  SC2b: {name: "Northeast SMAs shifted +1 month", shift_months: 1}
  SC3: {name: "Southern New England wind-lease SMA", buffer_km: 10, active: [1201, 331], min_len_ft: 65}
  SC4: {name: "Rule applied to vessels >= 35 ft", min_len_ft: 35}
sensitivity:
  perturbation_pct: 25
```

---

## Appendix F — Templates

### F.1 DataSourceRegistry row (example)

| Field | Example |
|---|---|
| `source_id` | S05 |
| `category` | Regulatory |
| `provider` | NOAA Fisheries |
| `dataset` | North Atlantic Right Whale Seasonal Management Areas |
| `url` | (download URL) |
| `access_terms` | Public domain (U.S. Government work) |
| `vintage` | Effective 2008-12-09; sunset removed 2013 |
| `download_date` | 2026-09-28 |
| `native_crs` | GCS WGS 1984 (4326) |
| `transformation_used` | WGS_1984_(ITRF00)_To_NAD_1983 |
| `fitness_decision` | Use |
| `notes` | Verified against eCFR 50 CFR 224.105 on 2026-09-28; 5 SMAs spot-checked |

### F.2 Fitness-for-Use Assessment (one page per source)

```markdown
## S0X — <Dataset name>
**Provider / URL / vintage / downloaded:** …
**Description and purpose in this project:** …
**Completeness:** spatial coverage …; temporal coverage …; effort information …
**Currency and update frequency:** …
**Spatial reference and transformation:** native …; stored …; transformation …
**Lineage:** how the provider produced it; our processing steps …
**Attribute quality:** nulls (n, %) …; domain conformance …; consistency … (QAQC_Log IDs …)
**Positional accuracy:** …
**Known limitations and biases:** …
**Appropriate uses:** … **Inappropriate uses:** …
**Decision:** Use / Use with caveats / Do not use — because …
```

### F.3 QAQC_Log row (example)

`QA-S01-003 | QA-RUN-1 | Sightings | Duplicates ±5 min ±500 m same source | 18,432 | 211 | report + review | 1 | B. Issifu | 2026-10-09 | 211 pairs merged; 0 unresolved`

### F.4 ProcessingLog row (example)

`RUN-20261015-142233 | KernelDensity | {"month":3,"radius_m":25000,"cell_m":5000,"pop":"n_animals"} | sight_5070 | kde_sight_m03_h25 | 2026-10-15 14:22 | 2026-10-15 14:23 | Success | a1b2c3d | bissifu`

### F.5 Weekly status note (`docs/status/2026-W40.md`)

```markdown
# Status — Week 40 (2026-09-28 → 2026-10-02)
**Progress:** …
**Completed tasks:** …
**Dependencies / waiting on:** NARWC response (requested 2026-09-24)
**Risks / issues:** …
**Decisions made:** D-0xx …
**Next week:** …
**Hours:** …
```

### F.6 Design review record (`docs/reviews/review_01.md`)

```markdown
# Design Review 1 — Geodatabase schema
**Date:** … **Reviewer(s):** … **Materials:** ERD v1, DataDictionary v1, schema XML
| # | Finding | Severity | Decision / action | Owner | Due |
|---|---|---|---|---|---|
**Sign-off:** ☐ Approved ☐ Approved with changes ☐ Re-review
```

### F.7 AIS manifest (`ais_preprocess/manifest.csv`)

`file,url,bytes,sha256,download_utc` — plus `qa_counts_ingest.csv` (`year,month,rows_in,rows_out`) and `qa_counts_legs.csv` (per-month QA counts from §11.3).

### F.8 NARWC data request email

```
Subject: Data request — North Atlantic Right Whale Consortium sightings (independent research/portfolio project)

Dear NARWC Data Manager,

I am a GIS analyst conducting an independent analysis of right whale–vessel co-occurrence and
compliance with the vessel speed rule along the U.S. Atlantic coast (2010–present). I request
access to sightings records for U.S. waters (date/time, position, group size, calves, platform,
survey type) and, if available, survey effort track lines for the same period.

Use: aggregated analysis only (monthly hexagon summaries of 25 km² or larger). No raw records
will be published or redistributed. I will follow the Consortium's data-use terms and
acknowledgment requirements, and will share the final report with the Consortium.

Project summary and data management plan attached.

Thank you,
Bernard Issifu
<email> · <portfolio URL>
```

### F.9 Metadata catalog row

`| SMA | Feature class (polygon) | Vessel speed rule SMAs with active periods … | 4269 | S05 | ISO ✅ FGDC ✅ Validated ✅ | metadata/iso/SMA.xml |`

---

## Appendix G — Literature and regulatory references

[VERIFY citations and use the latest editions.]

**Regulatory**
- NOAA. 2008. Endangered fish and wildlife; final rule to implement speed restrictions to reduce the threat of ship collisions with North Atlantic right whales. *Federal Register* 73 FR 60173 (10 October 2008).
- NOAA. 2013. Endangered fish and wildlife; final rule to remove the sunset provision of the final rule implementing vessel speed restrictions. *Federal Register* 78 FR 73726 (9 December 2013).
- NOAA. 2016. Endangered and threatened species; critical habitat for endangered North Atlantic right whale. *Federal Register* 81 FR 4837 (27 January 2016).
- NOAA. 2022. Amendments to the North Atlantic right whale vessel strike reduction rule (proposed rule). *Federal Register* 87 FR 46921 (1 August 2022).
- 50 CFR 224.105 — Speed restrictions to protect North Atlantic right whales.
- NOAA Fisheries. 2020. North Atlantic Right Whale Vessel Speed Rule Assessment (published January 2021).

**Science**
- Conn, P.B., and G.K. Silber. 2013. Vessel speed restrictions reduce risk of collision-related mortality for North Atlantic right whales. *Ecosphere* 4(4): art43.
- Davis, G.E., et al. 2017. Long-term passive acoustic recordings track the changing distribution of North Atlantic right whales (*Eubalaena glacialis*) from 2004 to 2014. *Scientific Reports* 7: 13460.
- Laist, D.W., A.R. Knowlton, J.G. Mead, A.S. Collet, and M. Podesta. 2001. Collisions between ships and whales. *Marine Mammal Science* 17(1): 35–75.
- Laist, D.W., A.R. Knowlton, and D. Pendleton. 2014. Effectiveness of mandatory vessel speed limits for protecting North Atlantic right whales. *Endangered Species Research* 23: 133–147.
- Pace, R.M., P.J. Corkeron, and S.D. Kraus. 2017. State–space mark–recapture estimates reveal a recent decline in abundance of North Atlantic right whales. *Ecology and Evolution* 7: 8730–8741.
- Pettis, H.M., R.M. Pace III, and P.K. Hamilton. (Annual). North Atlantic Right Whale Consortium Annual Report Card. NARWC.
- Record, N.R., et al. 2019. Rapid climate-driven circulation changes threaten conservation of endangered North Atlantic right whales. *Oceanography* 32(2): 162–169.
- Roberts, J.J., et al. 2016. Habitat-based cetacean density models for the U.S. Atlantic and Gulf of Mexico. *Scientific Reports* 6: 22615.
- Roberts, J.J., et al. 2024. North Atlantic right whale density surface model for the US Atlantic evaluated with passive acoustic monitoring. *Marine Ecology Progress Series* [VERIFY volume/pages].
- Silber, G.K., J.D. Adams, and C.J. Fonnesbeck. 2014. Compliance with vessel speed restrictions to protect North Atlantic right whales. *PeerJ* 2: e399.
- Vanderlaan, A.S.M., and C.T. Taggart. 2007. Vessel collisions with whales: the probability of lethal injury based on vessel speed. *Marine Mammal Science* 23(1): 144–156.

**Standards**
- ISO 19115-1:2014 Geographic information — Metadata — Part 1: Fundamentals.
- ISO/TS 19139:2007 Geographic information — Metadata — XML schema implementation.
- FGDC-STD-001-1998 Content Standard for Digital Geospatial Metadata.

---

## Appendix H — Glossary

| Term | Meaning |
|---|---|
| **AIS** | Automatic Identification System — ship transponders broadcasting position, speed, and identity |
| **Class A / Class B AIS** | Class A: mandatory on large commercial vessels; Class B: lower-power units on smaller vessels, reported less often |
| **CRS** | Coordinate reference system |
| **DMA / Slow Zone** | Dynamic Management Area — voluntary temporary 10-kn zone triggered by whale sightings or acoustic detections |
| **Effort** | Survey time or distance spent looking; needed to turn sightings into rates |
| **EEZ** | Exclusive Economic Zone (to 200 nm) |
| **ESA / MMPA** | Endangered Species Act / Marine Mammal Protection Act |
| **Hex-month** | One hexagon in one month — the unit of the analysis tables |
| **Kernel density** | Smoothed surface of point intensity within a search radius |
| **Leg** | Straight segment between two consecutive AIS pings of the same vessel |
| **MMSI** | Maritime Mobile Service Identity — a vessel's AIS ID |
| **nm / kn** | Nautical mile (1,852 m) / knot (1 nm per hour) |
| **PACM** | NOAA Passive Acoustic Cetacean Map |
| **Presence index** | 0–1 combined score of whale presence evidence |
| **Risk index** | 0–100 relative co-occurrence score (presence × traffic × speed) |
| **Section 7** | ESA consultation required for federal actions affecting listed species |
| **SMA** | Seasonal Management Area under the vessel speed rule |
| **SOG** | Speed over ground |
| **SPUE** | Sightings per unit effort |
| **Transit** | One vessel's passage through one SMA on one UTC day |
| **UME** | Unusual Mortality Event |

---

*End of document.*
