# VNP Land Use Compliance & Enforcement System

A land use compliance and enforcement platform for the Volcanoes National
Park (VNP) region of Rwanda, built on ArcGIS Enterprise. It carries a case
from satellite detection, through field inspection and supervisory
decision, to enforcement and closure — with every status change written to
an audit table by server-side rules.

Built at Esri Rwanda for the Rwanda Development Board.

---

## Contents

- [How a case moves](#how-a-case-moves)
- [Architecture](#architecture)
- [Data model](#data-model)
- [Attribute rules](#attribute-rules)
- [Case ID](#case-id)
- [Applications](#applications)
- [Repository layout](#repository-layout)
- [Deployment](#deployment)
- [Known limitations](#known-limitations)
- [Conventions](#conventions)

---

## How a case moves

```
Satellite detection
        │
        ▼
  Detection point            status = New
        │
        │  inspector opens Form 1 from the map
        ▼
  Field inspection           Survey123 → Compliance Case Points
        │                    server mints Case ID
        │                    detection flips to Inspected
        ▼
  Supervisory review         Form 2 → determination + decision
        │
        ▼
  Enforcement action         Form 3 → notice, deadline, resolution
        │
        ▼
  Closure                    every transition logged to Case_History
```

Five stages, five teams, one record. Nothing is edited by clicking the
map — all editing is switched off on every layer and data enters only
through Survey123 or ArcGIS Field Maps.

---

## Architecture

| Layer | Technology |
|---|---|
| Portal | ArcGIS Enterprise 11.5 — `https://gis.rdb.rw/portal` |
| Database | PostgreSQL enterprise geodatabase (release 3,0,0) |
| Business logic | Arcade attribute rules + database sequences |
| Field capture | ArcGIS Survey123, ArcGIS Field Maps |
| Analysis | ArcGIS Dashboards (5) |
| Front door | ArcGIS Experience Builder |
| Provisioning | ArcGIS Pro Notebook (Python 3.9/3.11) |

Spatial reference: **WGS 84 / UTM 36S (EPSG:32736)**.

---

## Data model

Feature dataset `rdb_gis.sde.LULC_Conservation_Platform_Datasets`.

| Class | Type | Role |
|---|---|---|
| `Detection_Points` | Point | Changes detected from imagery |
| `Compliance_Case_Points` | Point | The case record — one per inspection |
| `Main_DLUP` | Polygon | District Land Use Plan zoning |
| `Sector_Boundary` | Polygon | Administrative boundaries |
| `Case_History` | Table | Append-only audit trail |

**Relationship classes**

- `Detections_To_Cases` — a detection to its case
- `Cases_To_Case_History` — a case to its history rows

**Key fields on `Compliance_Case_Points`**

| Group | Fields |
|---|---|
| Identity | `case_id`, `detection_id`, `globalid` |
| Zoning | `zone_id`, `zone_name`, `permitted_use` |
| Administrative | `district`, `sector`, `cell`, `village`, `upi` |
| Inspection | `inspection_date`, `inspector`, `inspection_result`, `confidence` |
| Decision | `determination`, `decision`, `decision_justification`, `reviewed_by`, `review_date`, `decision_deadline` |
| Enforcement | `action_type`, `notice_ref`, `action_officer`, `action_date`, `enforcement_deadline`, `resolution_status`, `follow_up_date`, `closure_date`, `enforcement_notes` |
| State | `current_status`, `priority` |

`current_status` values: `Awaiting review` · `Decision issued` ·
`Under enforcement` · `Closed - resolved` · `Closed - not illegal` ·
`Review required`

`resolution_status` values: `Pending` · `In progress` · `Complied` ·
`Escalated` · `Closed`

---

## Attribute rules

Eight calculation rules, installed by the setup notebook.

| Rule | Trigger | Does |
|---|---|---|
| `VNP_Case_ID` | Insert | Mints the Case ID from a sequence, flips the linked detection to `Inspected`, writes `case_id` back to it |
| `VNP_Zone_ID_Backfill` | Insert | Reads `zone_code` from the DLUP polygon under the point |
| `VNP_Zone_Name_Backfill` | Insert | Reads the zone name |
| `VNP_Admin_District` | Insert | Reads district from `Sector_Boundary` |
| `VNP_Admin_Sector` | Insert | Reads sector |
| `VNP_Admin_Cell` | Insert | Reads cell |
| `VNP_Admin_Village` | Insert | Reads village |
| `VNP_Case_History_Update` | Update | Appends a row to `Case_History` on every status change |

> **Exclude from application evaluation must be enabled** on
> `VNP_Case_ID` and `VNP_Case_History_Update`. The first uses a database
> sequence, the second uses the `edit` keyword. Both are refused by the
> server otherwise (`ERROR 002920`).

**Database sequences** — one per zone family, created by CELL 3 of the
notebook. `NextSequenceValue()` guarantees two inspectors submitting at
the same moment cannot collide.

---

## Case ID

```
VNP-{ZONE}-{YYYY}-{NNNN}
     │       │      └── running number, per zone per year, zero padded
     │       └───────── year of inspection
     └───────────────── zone code from the DLUP polygon
```

Example: `VNP-F1-2026-0012`

The ID is **always minted on the server**, never on the phone and never in
the form. Zone comes from `zone_id` when the detection already carries
one, otherwise from `zone_code` on the DLUP polygon beneath the point.

---

## Applications

### Survey123 forms

| # | Form | Item | Writes to |
|---|---|---|---|
| 1 | Field Inspection | `b3517b4a6f754a5cb09bcb63766d8696` | Compliance Case Points |
| 2 | Compliance Decision | *pending* | Compliance Case Points |
| 3 | Enforcement Action | *pending* | Compliance Case Points |

Form 1 opens from the detection point pop-up as a browser web form with
values pre-filled from the map:

```
https://survey123.arcgis.com/share/<ITEM_ID>?portalUrl=https://gis.rdb.rw/portal
  &field:detection_id=10020
  &field:zone_id=F1
  &field:zone_name=F1-Forest%20plantation%20zone
  &field:permitted_use=Forest
  &field:detection_lat=-1.555435
  &field:detection_lon=29.511664
  &center=-1.555435,29.511664
```

Map Viewer Arcade has no `Project()`, so UTM 36S is converted to WGS 84
by a hand-coded Snyder inverse in the pop-up expression — validated
against `pyproj` to 0.0003 mm.

### Dashboards

| Dashboard | Audience | Source |
|---|---|---|
| Executive Overview | Management, partners | Case layer |
| Reported Cases | Detection and inspection teams | Detection + case layers |
| Case Management | Case officers, supervisors | Case layer |
| Enforcement & Resolution | Enforcement, legal, districts, NLA, RDB | Case layer |

Every dashboard carries a collapsible right sidebar with District, Sector,
Cell, Village, Case ID and a date range, wired to every widget.

### Experience Builder hub

`Land Use Compliance & Enforcement Hub` — the single front door. A
scrolling home page with role cards, four dashboard pages, three form
pages and a user guide.

---

## Repository layout

```
.
├── notebooks/
│   └── VNP_Setup.ipynb            schema, sequences, rules, demo data
├── scripts/
│   ├── 05_detection_demo.py       demo detection generator
│   ├── 07_parcel_rules.py         UPI / admin rules (blocked, see below)
│   └── 08_import_parcels.py       parcel import (blocked, see below)
├── arcade/
│   ├── expr_start_url.txt         detection pop-up → Form 1 pre-fill
│   └── expr_dlup_report.txt       DLUP pop-up zoning summary
├── webmaps/
│   ├── overview_map.json
│   └── vnp_compliance_form.json
├── dashboards/
│   ├── executive_overview_dashboard.json
│   ├── reported_cases_dashboard.json
│   ├── case_management_dashboard.json
│   └── enforcement_dashboard.json
├── experience/
│   ├── hub_scrolling.json         Experience Builder app config
│   └── user_guide.html            User Guide page (Embed → Code)
└── README.md
```

Arcade expressions are kept as `.txt` with CRLF endings. Rendered Markdown
strips underscores from Arcade field names, which silently breaks them.

---

## Deployment

### 1. Geodatabase

Open `notebooks/VNP_Setup.ipynb` in ArcGIS Pro and set the connection at
the top:

```python
SDE = r"C:\path\to\your.sde"
FDS = SDE + r"\rdb_gis.sde.LULC_Conservation_Platform_Datasets"
```

Run the cells in order:

| Cell | Does |
|---|---|
| 1 | Config and preflight checks |
| 2 | Schema — adds the enforcement and administrative fields |
| 2b | Archive-aware unique index on `case_id` |
| 3 | Database sequences |
| 4 | The eight attribute rules |
| 5 | Verification |
| 6 | Demo data (optional) |
| 7 | Cleanup |

**Stop the feature service before running cell 2.** Schema changes need
exclusive access.

### 2. Dashboards

Create an empty dashboard in Portal, then replace its JSON:
Content → the item → **Data** → select all → paste → Save.

Set auto-refresh per widget in the UI afterwards; it is not expressible
in the JSON.

### 3. Experience Builder

Same route — Content → the item → Data → replace → Save. Then open the
builder to upload images and publish.

---

## Known limitations

**Parcels are a hosted service in ITRF_2005.** Attribute rules cannot
reach a hosted service, and Arcade cannot reproject, so `upi` and the
parcel-derived administrative fields are not auto-filled. Three options
remain open: a clipped import into the EGDB, a scheduled backfill, or
passing UPI through the pop-up pre-fill. Not resolved.

**`Musanze Parcels` fails to load** in the web map, which blocks offline
area creation for that district.

**Offline is off in Field Maps.** The parcel layers have no global IDs and
no sync capability.

**Roughly 21 legacy cases (OID 1–21) have empty administrative fields.**
The rules fire on insert only and no backfill was run.

**Average days from decision to resolution is not computed.** There is no
stored duration field, and Dashboards cannot subtract two dates in a
structured filter. Per-case duration is shown in list widgets via Arcade;
the aggregate would need a `days_to_resolve` calculation rule.

**Overdue cannot be a dashboard filter.** Dashboards has no "date is
before today" operator. Open actions are filtered on
`action_date is not null AND closure_date is null`, sorted by deadline,
with Arcade colouring each row by how far past its deadline it is.

**Client/geodatabase version gap.** The geodatabase is release 3,0,0
against ArcGIS Pro 3.x and Enterprise 11.5. `ListDatabaseSequences` fails
and direct-connect inserts are refused; the notebook falls back to the
feature service. Upgrading the geodatabase would likely clear both.

---

## Conventions

- British English throughout the user-facing text.
- Status colours are fixed across every dashboard:
  `Awaiting review #bd7912` · `Decision issued #5350a6` ·
  `Under enforcement #127d86` · `Closed-resolved #2e7d46` ·
  `Closed-not illegal #77857d` · `Review required #b23a2b`
- Hub palette: `#17251d` dark · `#1f7a4d` green · `#09cf74` accent.
- Dashboard JSON is written against the **4.32 `chartConfig` schema**
  (`version: 18.1.0`). The Executive Overview still uses the older
  version-59 chart schema and may need rebuilding.
- Experience Builder config targets **1.17.0**. Hand-written nested
  Row/Column layouts are unreliable — build those in the builder and edit
  the exported JSON, rather than authoring them from scratch.

---

## Licence

Internal. Rwanda Development Board.
