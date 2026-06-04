OneGov #2 — Drinkwaterzekerheid Scenario Engine
Comprehensive Design Document v2.0 — Full Implementation Specification
GovTechNL Hackathon · 4–5 June 2026 · The Hague Tech
0. Document purpose, scope & change log
0.1 What this document is

This is the single authoritative implementation specification for the OneGov #2 hackathon submission. It covers product vision, data model, backend architecture, LangGraph workflow, calculation engine, frontend components, state management, error handling, traceability, and the build plan. Every developer on the team should be able to open this document, find their section, and start writing code without asking clarifying questions.
0.2 What changed from v1 and why
Change category	Specific change	Rationale
Scope tightening A	1 unified /scenario/run endpoint replaces 4 separate endpoints	Reduces integration risk; same behavior, less surface to debug
Scope tightening B	Demand per dwelling and datacenter water use are sliders with ranges, never constants	Domain credibility; water authority judges will challenge single constants
Scope tightening C	Two locked golden-path scenarios; Types 2 & 4 are fully specified but marked stretch	Ensures demo-able submission in 2 days; stretch is still spec'd so it can be built if time allows
Stakeholder GAP 1	Kennisbasis panel: "what does this system know?"	Institutional trust starts before the first question
Stakeholder GAP 2	Scenario hash + stable URL + shared library + version-drift warning	"Same question = same answer" for every colleague
Stakeholder GAP 3	Citation block with APA string + PDF export	Every output must be attachable to an official advice letter
Stakeholder GAP 4	HumanScaleRef on every headline metric	Raw m³/day figures are meaningless to non-engineers
Stakeholder GAP 5	"Make it feasible" intervention ranking on STOP/CAUTION	Actionable next step, not just a verdict
Stakeholder GAP 6	CumulativeLoadWarning: other active scenarios in same supply zone	Portfolio-level capacity accounting
Stakeholder GAP 7	Per-intake chloride threshold from productieketen, not global constant	Operational correctness for water authority
Stakeholder GAP 8	OfficialPosition panel with policy document links	Prevents contradiction of official government communications
Stakeholder GAP 9	Waterinfo chloride API mandatory for IJssel/Lek/Maas; explicit fallback warning	Never silently substitute synthetic values for real-time data
Stakeholder GAP 10	Citizen response template: verdict-first, postcode-level, water company link	Different contract for public users vs policy users
0.3 What is NOT in scope

    Full hydrological model or calibrated physical simulation
    Production-grade SaaS infrastructure (auth, multi-tenancy, SLAs)
    Replacement of existing descriptive query workflow (it stays intact)
    Legal Woo compliance certification (the design supports it; it does not guarantee it)

1. Product vision
1.1 The one-sentence pitch

A replayable, citeable, policy-auditable scenario engine that extends the existing OneGov2 spatial assistant from "describe what is" to "explore what could be" — turning Dutch policy what-if questions into structured, explainable scenarios with map overlays, stakeholder views, reasoning trails, and hyperlinked source registries.
1.2 The institutional contract

Every scenario output must satisfy:

    "A policy officer must be able to attach this output to an advice letter and defend every number in it to a colleague, a journalist, or a Woo-request auditor."

This contract drives four non-negotiable requirements:

    Every number links to an authoritative source
    Every run is reproducible (same params → same result)
    Every assumption is explicit, adjustable, and sourced
    Every run is logged with dataset versions and a git commit

1.3 What it is NOT

    Not a black-box chatbot ("the model said so" is not acceptable)
    Not a full water infrastructure model
    Not a replacement for expert judgment — it is a tool to structure and communicate that judgment
    Not a dashboard that shows fixed charts

1.4 Positioning relative to the existing assistant

text

EXISTING ASSISTANT                    NEW SCENARIO ENGINE
─────────────────                     ───────────────────
"Hoeveel woningen zijn                "Wat als er 5.000 woningen
er in zone X?"          ──extends──►  bijkomen in zone X in 2040
                                       onder KNMI Hd?"

Descriptive retrieval                 Exploratory scenario
Single dataset query                  Multi-theme composition
Fact answer                           Structured scenario card
No assumption tracking                Explicit assumption sliders
No traceability needed                Full audit trail

2. Users, personas & design principles
2.1 User archetypes
Persona	Daily reality	Primary need from this tool	Key v2 addition
Provincial policymaker (Provincie ZH)	Writes advice letters citing evidence; challenged by colleagues, politicians, press	Defensible numbers, PDF export, official context	Citation block, OfficialPosition panel, Woo trail
Spatial planner (Gemeente/Omgevingsdienst)	Reviews permit applications; tracks cumulative approvals	Feasibility verdict, cumulative demand, stacking	GO/CAUTION/STOP, CumulativeLoadWarning, stacking mode
Water authority engineer (Dunea/Evides)	Operates intake points; monitors chloride in real time	Per-intake accuracy, real-time chloride, production chain view	Per-location thresholds, Waterinfo guard, ProductionChainFlow
Project developer (datacenter/housing)	Needs permit go/no-go fast; wants to know what changes a red verdict	Fast feasibility, "what makes this work"	Traffic-light class, MakeItFeasiblePanel
Citizen	Reads news about water shortages; worried about their postcode	"Is mijn water veilig?", plain language, who to call	CitizenResponseCard, verdict-first template
Auditor / Woo-request handler	Reconstructs what a decision was based on	Fully reproducible, citeable, dataset-versioned artifact	scenario_hash, stable URL, MLflow artifact, citation string
2.2 UI design principles (numbered for traceability in component specs)

    B1 Dutch first — all output in plain Dutch (B1/B2 level); technical detail available on expand only
    Map first — spatial output leads; text and tables are secondary
    Verdict first — FeasibilityClass (GO/CAUTION/STOP) shown before any numbers
    Every number links — no figure without a source URL; SourceRegistry appended to every ScenarioCard
    Human scale — every headline metric gets a real-world analogy (e.g., "equivalent to daily use of X households")
    Ranges not constants — all approximated inputs shown as ranges with sliders; labelled "beleidsmatige schatting"
    No dead ends — if a scenario cannot be computed, explain why and offer a follow-up question
    Official anchor — always surface the government's official position alongside scenario output
    Same question = same answer — scenario hash ensures reproducibility across users and sessions
    Citizen contract — citizen-mode responses follow a different, verdict-first template with no jargon

3. Golden-path scenarios (the two you build and demo)

Scope is deliberately locked to these two scenarios plus one intervention comparison. All architecture and calculation logic serves these first. Types 2 and 4 are fully specified in sections 12 and 13 as stretch goals.
3.1 Golden Path A — Type 3: Drop-a-pin (spatial demand decision)

Trigger question:

    "Kan er een 50 MW datacenter komen bij Pijnacker-Nootdorp in 2040?"

What the system does:

    Extracts: scenario_type=drop_pin, location="Pijnacker-Nootdorp", development_type="datacenter_50mw", time_horizon=2040, knmi_preset="Hd" (default to most stressful)
    Geocodes location via PDOK to WKT point
    Identifies supply zone(s) containing or nearest to that point from leveringszones table
    Looks up current supply capacity and demand for that zone
    Computes development demand delta using reference table (50 MW × multiplier range 5–20 m³/day/MW)
    Projects zone demand to 2040 using growth preset
    Computes supply gap or headroom
    Assigns FeasibilityClass across all 5 KNMI presets
    Identifies which stakeholders are affected and how
    Returns full ScenarioCard + drop-pin overlay + zone capacity heatmap

Demo wow moment: FeasibilityClass is 🔴 STOP; "Make it feasible" panel shows top 3 interventions; slider for datacenter MW updates result in real time.
3.2 Golden Path B — Type 1: Climate × infrastructure failure

Trigger question:

    "Wat als de Hollandse IJssel-inname 6 weken onbruikbaar is door verzilting onder KNMI Hd in 2040?"

What the system does:

    Extracts: scenario_type=intake_failure, intake_id="IJssel", outage_weeks=6, knmi_preset="Hd", time_horizon=2040
    Fetches live chloride from Waterinfo for IJssel intake (or cached with explicit warning)
    Looks up per-intake chloride threshold from productieketen table
    Computes chloride trajectory: baseline + KNMI delta + outage degradation
    Computes supply gap: intake capacity lost × outage duration × alternative source coverage
    Runs OnsetYearCalculator to find when gap first becomes critical (2024→2040)
    Runs KRWRiskCheck to identify at-risk water bodies in affected zones
    Assigns FeasibilityClass; identifies stakeholders; builds reasoning steps
    Returns ScenarioCard + chloride risk zone overlay + supply gap heatmap + production chain flow

Demo wow moment: Chloride threshold slider moves from 150→200 mg/L and FeasibilityClass updates; "compare with buffer storage" button triggers Golden Path C.
3.3 Golden Path C — Intervention comparison (derives from B)

Trigger question:

    "Vergelijk: met en zonder bufferopslag van 30.000 m³"

What the system does:

    Same endpoint as B with scenario_b_params populated: interventions=["buffer_30000"]
    Returns both ScenarioCards + ScenarioDelta object
    Frontend renders split map + delta panel (gap delta, onset year delta, KRW count delta)

Demo wow moment: Split map; left panel 🔴, right panel 🟡; delta panel shows "onset year pushed from 2035 → 2040".
4. System architecture & tech stack
4.1 Full stack overview

text

┌─────────────────────────────────────────────────────────────────────┐
│ FRONTEND (Vue 3 + Vite + Pinia + Leaflet)                           │
│  ChatInput → ScenarioCard → MapView → InsightPanel                  │
│  + 13 new components (see section 18)                               │
└────────────────────────┬────────────────────────────────────────────┘
                         │ SSE stream (scenario_card, map_data,
                         │ reasoning_step, error events)
┌────────────────────────▼────────────────────────────────────────────┐
│ BACKEND (FastAPI + LangGraph + DuckDB + GreenPT)                    │
│  /scenario/run → LangGraph workflow → DuckDB queries                │
│  /scenario/{id} → cached ScenarioCard retrieval                     │
│  /kennisbasis/query → knowledge base queries                        │
└────────────────────────┬────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    DuckDB          GreenPT API     MLflow tracking
    (local)         (GreenPT.ai)    (local server)
         │
    ┌────┴────────────────────────────────┐
    │ Waterinfo API   PDOK geocoder        │
    │ (external)      (external)           │
    └─────────────────────────────────────┘

4.2 Full directory tree (file-level implementation map)

text

onegov2-spatial-assistant/
├── src/
│   ├── backend/
│   │   ├── main.py                          # FastAPI app; existing + /scenario/run route
│   │   ├── config.py                        # env vars; existing + new vars
│   │   ├── models/
│   │   │   ├── existing_models.py           # keep as-is
│   │   │   └── scenario.py                  # NEW: all scenario dataclasses (section 7)
│   │   ├── nodes/
│   │   │   ├── existing_nodes/              # keep as-is (check_intent, etc.)
│   │   │   ├── extract_scenario_params.py   # NEW: GreenPT structured extraction
│   │   │   ├── fetch_scenario_data.py       # NEW: data loader + SourceRegistry builder
│   │   │   ├── check_scenario_cache.py      # NEW: hash lookup + version drift
│   │   │   ├── build_scenario_object.py     # NEW: construct typed scenario + assumptions
│   │   │   ├── run_scenario_calculation.py  # NEW: demand/capacity/chloride/gap/onset
│   │   │   ├── compute_feasibility_class.py # NEW: GO/CAUTION/STOP across KNMI presets
│   │   │   ├── check_cumulative_load.py     # NEW: active scenarios in same supply zone
│   │   │   ├── stakeholder_impacts.py       # NEW: rule engine + GreenPT prose
│   │   │   ├── build_official_position.py   # NEW: pre-baked policy document lookup
│   │   │   ├── record_reasoning_steps.py    # NEW: B1 Dutch steps + MLflow
│   │   │   └── format_scenario_output.py    # NEW: ScenarioCard + GeoJSON assembly
│   │   ├── calculation/
│   │   │   ├── demand.py                    # NEW: calculate_daily_demand + ranges
│   │   │   ├── chloride.py                  # NEW: calculate_cl_concentration + Waterinfo
│   │   │   ├── capacity.py                  # NEW: CapacityCheck from DuckDB
│   │   │   ├── krw.py                       # NEW: KRWRiskCheck spatial overlay
│   │   │   ├── onset.py                     # NEW: OnsetYearCalculator
│   │   │   ├── stakeholder_rules.py         # NEW: StakeholderRuleEngine + rule table
│   │   │   ├── interventions.py             # NEW: intervention catalogue + ranking
│   │   │   └── human_scale.py               # NEW: HumanScaleRef converter
│   │   ├── workflows/
│   │   │   ├── descriptive_workflow.py      # EXISTING: keep as-is
│   │   │   └── scenario_workflow.py         # NEW: full LangGraph scenario workflow
│   │   ├── cache/
│   │   │   ├── scenario_cache.py            # NEW: hash-keyed scenario result store
│   │   │   └── waterinfo_cache.py           # NEW: chloride last-known-value cache
│   │   ├── external/
│   │   │   ├── waterinfo.py                 # NEW: Waterinfo API client + fallback
│   │   │   ├── pdok_geocoder.py             # EXISTING: extend for WKT output
│   │   │   └── greenpt.py                   # EXISTING: extend with new prompt templates
│   │   ├── registry/
│   │   │   ├── source_registry.py           # NEW: SourceEntry builder + attribution
│   │   │   └── official_positions.py        # NEW: pre-baked OfficialPosition lookup
│   │   └── utils/
│   │       ├── readability.py               # NEW: Flesch-Douma scorer
│   │       ├── scenario_hash.py             # NEW: SHA-256 normalized params
│   │       └── citation.py                  # NEW: APA citation string builder
│   └── frontend/
│       ├── src/
│       │   ├── stores/
│       │   │   ├── useScenarioStore.ts       # NEW: scenario state + actions
│       │   │   ├── useMapStore.ts            # EXISTING: extend with overlay support
│       │   │   ├── useAssumptionStore.ts     # NEW: assumption sliders state
│       │   │   └── useKennisbasisStore.ts    # NEW: knowledge base query state
│       │   ├── components/
│       │   │   ├── existing/                 # keep as-is
│       │   │   ├── FeasibilityBadge.vue      # NEW
│       │   │   ├── ScenarioCard.vue          # NEW: main scenario output wrapper
│       │   │   ├── ScenarioDeltaPanel.vue    # NEW: split comparison
│       │   │   ├── AssumptionSliders.vue     # NEW: sliders + re-run trigger
│       │   │   ├── MakeItFeasiblePanel.vue   # NEW: intervention ranking
│       │   │   ├── HumanScaleRef.vue         # NEW: inline analogy
│       │   │   ├── CumulativeLoadWarning.vue # NEW: active scenarios warning
│       │   │   ├── StakeholderDropdown.vue   # NEW: stakeholder map toggle
│       │   │   ├── KennisbasisPanel.vue      # NEW: knowledge base explorer
│       │   │   ├── OfficialPositionPanel.vue # NEW: official policy context
│       │   │   ├── CitationBlock.vue         # NEW: APA citation + stable URL
│       │   │   ├── CitizenResponseCard.vue   # NEW: verdict-first citizen output
│       │   │   ├── ProductionChainFlow.vue   # NEW: directed flow graph
│       │   │   ├── MapTitle.vue              # NEW: always-visible map context
│       │   │   ├── WaterinfoBanner.vue       # NEW: live vs cached warning
│       │   │   └── SourceRegistryPanel.vue   # NEW: hyperlinked source list
│       │   └── composables/
│       │       ├── useScenarioSSE.ts          # NEW: SSE event → store wiring
│       │       └── useMapOverlays.ts          # NEW: overlay management
├── data/
│   ├── drinkwaterzekerheid/                  # existing
│   ├── gebiedsviewer/                        # existing
│   ├── lgn/                                  # existing
│   ├── cbs/                                  # optional: load if available
│   └── external_cache/
│       ├── knmi23_scenarios.json             # PRE-CACHE before hackathon
│       ├── waterinfo_chloride_cache.json     # PRE-CACHE before hackathon
│       └── vewin_household_stats.json        # PRE-CACHE before hackathon
├── tests/
│   ├── test_calculation_golden_path_a.py     # NEW: acceptance tests for Type 3
│   └── test_calculation_golden_path_b.py     # NEW: acceptance tests for Type 1
├── .env.example                              # NEW: see section 29
├── README.md                                 # NEW: see section 28
├── docker-compose.yml                        # EXISTING: add MLflow service
└── mlflow/                                   # NEW: MLflow tracking artifacts

4.3 New environment variables

Bash

# GreenPT (existing)
GREENPT_KEY=your_key_here
GREENPT_BASE_URL=https://api.greenpt.ai/v1
GREENPT_MODEL=gpt-4o

# MLflow (new)
MLFLOW_TRACKING_URI=http://localhost:5001
MLFLOW_EXPERIMENT_NAME=onegov2-scenario-engine

# Waterinfo (new)
WATERINFO_API_URL=https://waterwebservices.rijkswaterstaat.nl/ONLINEWAARNEMINGENSERVICES_DBO/OphalenWaarnemingen
WATERINFO_CACHE_PATH=data/external_cache/waterinfo_chloride_cache.json

# Scenario cache (new)
SCENARIO_CACHE_TTL_HOURS=24
SCENARIO_STABLE_URL_BASE=http://localhost:8000

# PDOK (existing key, new param)
PDOK_GEOCODER_URL=https://api.pdok.nl/bzk/locatieserver/search/v3_1/free

# Determinism
NUMPY_SEED=42

4.4 Docker-compose additions

YAML

services:
  # existing app service stays unchanged

  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    ports:
      - "5001:5000"
    volumes:
      - ./mlflow:/mlflow
    command: mlflow server --host 0.0.0.0 --backend-store-uri /mlflow/backend --default-artifact-root /mlflow/artifacts

  # optional: redis for scenario cache in multi-user scenarios
  # redis:
  #   image: redis:7-alpine
  #   ports: ["6379:6379"]

4.5 New Python dependencies

text

# Add to requirements.txt
shapely>=2.0.0          # WKT geometry operations
httpx>=0.27.0           # async HTTP for Waterinfo + PDOK
mlflow>=2.13.0          # experiment tracking
pydantic>=2.7.0         # strict dataclass validation
hashlib                 # stdlib: scenario hashing
h3>=3.7.6               # H3 spatial indexing (may already be present)
numpy>=1.26.0           # deterministic seed
fpdf2>=2.7.0            # PDF export for citation block

5. Data inventory & schema
5.1 Default-loaded DuckDB tables (confirm on Day 1 with blocking check)
Theme: drinkwaterzekerheid
Table name	Key columns	Join key	Description
winlocaties	locatie_id, naam, type, capaciteit_m3_dag, geometry	spatial	Drinking water intake locations
productieketen	locatie_id, intake_id, productie_cap_m3_dag, behandel_tech, cl_threshold_mg_l, status	locatie_id	Full production chain with per-intake chloride threshold
leveringszones	zone_id, naam, capaciteit_m3_dag, vraag_2023_m3_dag, h3_id, geometry	h3_id	Supply zones with current capacity and demand
waterschappen	schap_id, naam, geometry	spatial	Water board boundaries
zes_uur_zones	zone_id, intake_id, radius_km, flow_direction, geometry	intake_id	6-hour protection contours around intakes
krw_waterlichamen	lichaam_id, naam, type, status_2023, krw_doeljaar, geometry	spatial	KRW water bodies with compliance status
innamepunten_chloride	intake_id, datum, cl_mg_l, bron	intake_id	Historical chloride measurements
alternatieve_bronnen	bron_id, intake_id, max_capaciteit_m3_dag, activatie_tijd_uur, type	intake_id	Alternative/backup intake sources
Theme: gebiedsviewer
Table name	Key columns	Join key	Description
verzilting_risico	h3_id, cl_risico_klasse, scenario_knmi	h3_id	Salinisation risk per H3 cell per KNMI scenario
bodemdaling	h3_id, daling_mm_jaar, prognose_2040	h3_id	Subsidence rate and 2040 projection
overstromingsrisico	h3_id, kans_klasse, diepte_m	h3_id	Flood risk
natuur_gebieden	gebied_id, naam, type, bescherming, geometry	spatial	Protected nature areas
grondwaterbeschermingszones	zone_id, niveau, geometry	spatial	Groundwater protection zones (KRW relevant)
woningbouwlocaties	locatie_id, naam, gemeente, woningen_gepland, status, h3_id, geometry	h3_id	Planned housing locations
Theme: LGN (land use)
Table name	Key columns	Join key	Description
lgn_landgebruik	h3_id, klasse, klasse_label, oppervlak_ha	h3_id	Land use classification per H3 cell
Optional: CBS buurtstatistiek
Table name	Key columns	Join key	Description
cbs_buurt	h3_id, inwoners, huishoudens, woning_dichtheid	h3_id	Population and household statistics
5.2 H3 join strategy

All spatial data is pre-indexed at H3 resolution 8 (approximately 460m cell size, appropriate for municipal-scale spatial analysis). Joins between themes use h3_id as the primary key. Point-to-zone lookups use DuckDB's H3 extension:

SQL

-- Convert lat/lon to H3 cell
SELECT h3_latlng_to_cell(lat, lon, 8) AS h3_id

-- Find all cells within radius
SELECT h3_grid_disk(center_h3_id, k) AS nearby_cells

-- CBS uses h3_index alias
SELECT * FROM cbs_buurt WHERE h3_index = ?
-- Note: cbs_buurt.h3_index is aliased to h3_id in the DuckDB view definition

5.3 Day 1 blocking check queries (run these first thing)

Python

# blocking_checks.py — run before writing any scenario calculation code
import duckdb
conn = duckdb.connect("data/onegov2.duckdb")

# Check 1: productieketen has per-intake chloride threshold
result = conn.execute("""
    SELECT column_name FROM information_schema.columns
    WHERE table_name = 'productieketen' AND column_name = 'cl_threshold_mg_l'
""").fetchall()
assert len(result) > 0, "BLOCKING: productieketen missing cl_threshold_mg_l — use fallback Assumption"

# Check 2: leveringszones has capacity and demand columns
result = conn.execute("""
    SELECT column_name FROM information_schema.columns
    WHERE table_name = 'leveringszones'
    AND column_name IN ('capaciteit_m3_dag', 'vraag_2023_m3_dag')
""").fetchall()
assert len(result) == 2, "BLOCKING: leveringszones missing capacity/demand columns"

# Check 3: at least one row in winlocaties for IJssel
result = conn.execute("""
    SELECT COUNT(*) FROM winlocaties WHERE LOWER(naam) LIKE '%ijssel%'
""").fetchone()
assert result[0] > 0, "BLOCKING: no IJssel intake in winlocaties — check intake naming"

# Check 4: zes_uur_zones exists and has rows
result = conn.execute("SELECT COUNT(*) FROM zes_uur_zones").fetchone()
assert result[0] > 0, "BLOCKING: zes_uur_zones is empty"

# Check 5: KRW water bodies exist
result = conn.execute("SELECT COUNT(*) FROM krw_waterlichamen").fetchone()
assert result[0] > 0, "BLOCKING: krw_waterlichamen is empty"

# Check 6: H3 join works between leveringszones and gebiedsviewer
result = conn.execute("""
    SELECT COUNT(*) FROM leveringszones lz
    JOIN verzilting_risico vr ON lz.h3_id = vr.h3_id
""").fetchone()
print(f"H3 join health: {result[0]} rows — {'OK' if result[0] > 0 else 'WARNING: no join rows'}")

print("All blocking checks passed.")

If any blocking check fails, the fallback plan is documented in the Assumption objects (e.g., if cl_threshold_mg_l is missing from productieketen, the Assumption with key cl_threshold_mg_l takes over with value=150, range 150–250, source_url to Drinkwaterbesluit).
5.4 External data pre-cache (do before hackathon)

Bash

# Run these before the hackathon to pre-cache external data
# Venue internet may be unreliable

python scripts/precache_knmi.py       # Downloads KNMI'23 scenario tables
python scripts/precache_waterinfo.py  # Downloads last 30 days chloride for IJssel, Lek, Maas
python scripts/precache_vewin.py      # Downloads VEWIN household stats summary

Pre-cache files written to data/external_cache/ as JSON with fetched_at timestamps.
6. Unified API surface
6.1 The one endpoint

Python

# src/backend/main.py (addition)

@app.post("/scenario/run")
async def run_scenario(request: ScenarioRunRequest):
    """
    Unified scenario endpoint.
    Three modes via same endpoint:
      - Single scenario: scenario_b=null, assumption_overrides=null
      - Comparison:      scenario_b populated
      - Re-run:          assumption_overrides populated (same scenario_a, different assumptions)
    Returns: SSE stream of scenario_card, map_data, reasoning_step, error events
    """
    return StreamingResponse(
        scenario_workflow_stream(request),
        media_type="text/event-stream"
    )

@app.get("/scenario/{scenario_id}")
async def get_scenario(scenario_id: str):
    """Return cached ScenarioCard by ID. Stable URL for sharing."""
    card = scenario_cache.get_by_id(scenario_id)
    if not card:
        raise HTTPException(404, "Scenario niet gevonden")
    return card

@app.get("/kennisbasis/query")
async def query_kennisbasis(q: str):
    """Answer 'what does this system know about X?' queries."""
    return kennisbasis_service.query(q)

@app.get("/kennisbasis/status")
async def kennisbasis_status():
    """Return loaded datasets, row counts, last_modified timestamps."""
    return kennisbasis_service.get_status()

6.2 Request schema

Python

@dataclass
class ScenarioRunRequest:
    question:             str                        # original NL question
    scenario_a:           ScenarioParams             # always required
    scenario_b:           ScenarioParams | None = None      # triggers comparison mode
    assumption_overrides: dict[str, float] | None = None    # triggers re-run mode
    user_persona:         str = "professional"       # "professional" | "citizen"
    language:             str = "nl"

6.3 SSE event types

TypeScript

// All events streamed from /scenario/run

type SSEEvent =
  | { type: "scenario_params_confirmed"; data: ScenarioParamsConfirmation }
  | { type: "data_loaded";               data: DataLoadedEvent }
  | { type: "cache_hit";                 data: CacheHitEvent }
  | { type: "waterinfo_status";          data: WaterinfoStatusEvent }
  | { type: "reasoning_step";            data: ReasoningStep }
  | { type: "feasibility_class";         data: FeasibilityClassEvent }
  | { type: "cumulative_load_warning";   data: CumulativeLoadEvent }
  | { type: "stakeholder_impacts";       data: StakeholderImpact[] }
  | { type: "official_position";         data: OfficialPosition }
  | { type: "scenario_card";             data: ScenarioCard }
  | { type: "map_data";                  data: GeoJSONOverlay[] }
  | { type: "scenario_delta";            data: ScenarioDelta }
  | { type: "followup_question";         data: FollowupQuestion }
  | { type: "error";                     data: ScenarioError }
  | { type: "done";                      data: null }

7. Full data model
7.1 ScenarioParams

Python

# src/backend/models/scenario.py

from dataclasses import dataclass, field
from typing import Literal

@dataclass
class ScenarioParams:
    scenario_type:         Literal["drop_pin", "intake_failure", "multi_hazard", "intervention"]
    knmi_preset:           Literal["B", "Hd", "Hn", "Ld", "Ln"] = "Hd"
    time_horizon:          int = 2040
    growth_preset:         Literal["laag", "middel", "hoog"] = "middel"

    # Type 3 (drop_pin) specific
    location_wkt:          str | None = None      # WKT POINT from PDOK geocoder
    location_name:         str | None = None      # Human-readable name
    development_type:      str | None = None      # "datacenter_50mw", "housing_5000", etc.
    development_mw:        float | None = None    # For datacenter: MW IT load
    development_units:     int | None = None      # For housing: number of units

    # Type 1 (intake_failure) specific
    intake_id:             str | None = None
    outage_weeks:          int | None = None

    # Type 2 (multi_hazard) specific — stretch
    hazard_combination:    list[str] = field(default_factory=list)

    # Type 4 (intervention) specific
    interventions:         list[str] = field(default_factory=list)
    baseline_scenario_id:  str | None = None     # Compare against existing scenario

    # Override fields
    assumption_overrides:  dict[str, float] = field(default_factory=dict)

7.2 Assumption

Python

@dataclass
class Assumption:
    key:              str              # machine key, e.g. "demand_per_dwelling_m3_day"
    label_nl:         str              # B1 Dutch label for UI
    value:            float            # default/current value
    value_min:        float            # slider minimum
    value_max:        float            # slider maximum
    unit:             str              # e.g. "m³/dag", "mg/L", "m³/dag/MW"
    source_url:       str              # MANDATORY — CI gate blocks empty string
    source_label:     str              # e.g. "VEWIN Jaarstatistiek 2023"
    sensitivity:      Literal["low", "medium", "high"]  # how much does output change per unit
    is_policy_approx: bool = True      # shown as "beleidsmatige schatting" label in UI
    slider_step:      float = 0.01
    description_nl:   str = ""         # B1 Dutch explanation of what this assumption is

Validation gate (in build_scenario_object.py):

Python

def validate_assumptions(assumptions: list[Assumption]) -> None:
    for a in assumptions:
        if not a.source_url or a.source_url.strip() == "":
            raise AssumptionValidationError(
                f"Assumption '{a.key}' has empty source_url. "
                f"This is a hard requirement. Add source before running."
            )
        if not a.source_url.startswith("http"):
            raise AssumptionValidationError(
                f"Assumption '{a.key}' source_url must be a valid URL."
            )

7.3 Default assumption library

Python

# src/backend/models/scenario.py

DEFAULT_ASSUMPTIONS = {
    "demand_per_dwelling_m3_day": Assumption(
        key="demand_per_dwelling_m3_day",
        label_nl="Waterverbruik per woning per dag",
        value=0.35,
        value_min=0.28,
        value_max=0.42,
        unit="m³/dag",
        source_url="https://www.vewin.nl/SiteCollectionDocuments/Publicaties/Cijfers/VEWIN-Watergebruik-in-cijfers-2023.pdf",
        source_label="VEWIN Watergebruik in cijfers 2023",
        sensitivity="high",
        slider_step=0.01,
        description_nl="Gemiddeld drinkwaterverbruik per woning per dag, inclusief buitengebruik. "
                        "Gebaseerd op ~119 liter per persoon per dag × gemiddeld 2,9 personen per huishouden."
    ),
    "datacenter_m3_per_mw_day": Assumption(
        key="datacenter_m3_per_mw_day",
        label_nl="Waterverbruik datacenter per MW IT per dag",
        value=12.0,
        value_min=5.0,
        value_max=20.0,
        unit="m³/dag/MW",
        source_url="https://www.iea.org/reports/data-centres-and-data-transmission-networks",
        source_label="IEA Data Centres and Data Transmission Networks",
        sensitivity="high",
        slider_step=0.5,
        description_nl="Dagelijks waterverbruik voor koeling per MW IT-vermogen. "
                        "Sterk afhankelijk van koeltechniek (lucht vs. water vs. hybride). "
                        "Dit is een beleidsmatige schatting — werkelijk verbruik kan sterk afwijken."
    ),
    "cl_threshold_mg_l_fallback": Assumption(
        key="cl_threshold_mg_l_fallback",
        label_nl="Chloridedrempelwaarde inname (fallback)",
        value=150.0,
        value_min=150.0,
        value_max=250.0,
        unit="mg/L",
        source_url="https://wetten.overheid.nl/BWBR0030111/2024-01-01",
        source_label="Drinkwaterbesluit (Bijlage A)",
        sensitivity="high",
        slider_step=5.0,
        description_nl="Drempelwaarde voor chlorideconcentratie in innamepunt. "
                        "Nederlandse norm (150 mg/L) is strenger dan EU-norm (250 mg/L). "
                        "Voorkeur: gebruik per-inname waarde uit productieketen tabel."
    ),
    "peak_demand_buffer_factor": Assumption(
        key="peak_demand_buffer_factor",
        label_nl="Piekvraagfactor (zomer/droogte)",
        value=1.15,
        value_min=1.05,
        value_max=1.30,
        unit="factor",
        source_url="https://www.vewin.nl/SiteCollectionDocuments/Publicaties/Cijfers/VEWIN-Watergebruik-in-cijfers-2023.pdf",
        source_label="VEWIN Jaarstatistiek 2023",
        sensitivity="medium",
        slider_step=0.01,
        description_nl="Factor waarmee piekvraag in droge zomer de jaarvraag overstijgt."
    ),
    "outage_cl_degradation_per_week": Assumption(
        key="outage_cl_degradation_per_week",
        label_nl="Extra chloridestijging per week uitval",
        value=0.20,
        value_min=0.10,
        value_max=0.35,
        unit="fractie per week",
        source_url="https://www.waterinfo.rws.nl/",
        source_label="Rijkswaterstaat Waterinfo — historische trend verzilting",
        sensitivity="medium",
        slider_step=0.01,
        description_nl="Conservatieve schatting van extra chloridestijging per week dat de inname niet werkt, "
                        "gebaseerd op historische verziltingstrends bij de Hollandse IJssel."
    ),
}

7.4 ScenarioResults

Python

@dataclass
class ScenarioResults:
    # Demand / capacity core
    daily_demand_m3:           float
    daily_demand_range:        tuple[float, float]    # (min, max)
    supply_capacity_m3:        float
    supply_gap_m3:             float                  # negative = shortfall
    capacity_remaining_m3:     float | None = None
    headroom_pct:              float | None = None

    # Feasibility
    feasibility_class:         Literal["GO", "CAUTION", "STOP"] = "CAUTION"
    feasibility_by_knmi:       dict[str, str] = field(default_factory=dict)  # preset → class

    # Human scale
    human_scale:               "HumanScaleRef | None" = None

    # Chloride (Type 1 scenarios)
    cl_concentration:          float | None = None
    cl_threshold_mg_l:         float | None = None
    cl_threshold_source:       str | None = None
    cl_threshold_from_db:      bool = False           # True if from productieketen, False if fallback
    cl_intake_id:              str | None = None

    # KRW
    krw_at_risk_count:         int = 0
    krw_areas:                 list[str] = field(default_factory=list)

    # Timeline
    onset_year:                int | None = None
    onset_year_by_knmi:        dict[str, int | None] = field(default_factory=dict)
    confidence:                float = 0.7

    # Policy approximation flag
    is_policy_approx:          bool = True

    # Cumulative / stacking (Type 6 — v2)
    active_scenarios_same_zone: int = 0
    cumulative_demand_m3:       float | None = None

    # Intervention effectiveness (Type 4)
    gap_before_intervention:    float | None = None
    gap_after_intervention:     float | None = None

7.5 HumanScaleRef

Python

@dataclass
class HumanScaleRef:
    metric_key:      str             # which metric this contextualizes
    metric_value:    float
    metric_unit:     str
    analogy_nl:      str             # e.g. "dagelijks verbruik van ~1,2 miljoen mensen"
    analogy_source:  str             # URL to VEWIN/CBS reference
    analogy_label:   str             # short source label for UI display

7.6 StakeholderImpact

Python

@dataclass
class StakeholderImpact:
    stakeholder_key:   str                              # machine key
    stakeholder_nl:    str                              # display name
    impact_nl:         str                              # 1–2 sentence B1 Dutch
    severity:          Literal["none", "low", "medium", "high", "critical"]
    affected_zone_ids: list[str]
    overlay_layer:     str                              # GeoJSON layer to highlight
    policy_basis:      str                              # policy document name
    policy_url:        str                              # URL (mandatory)

7.7 ReasoningStep

Python

@dataclass
class ReasoningStep:
    step_nr:             int
    label_nl:            str              # short step title in B1 Dutch
    description_nl:      str             # full explanation in B1 Dutch
    datasets_used:       list[str]        # table names
    assumptions_used:    list[str]        # assumption keys
    sql_fragment:        str | None       # actual SQL if applicable
    calculated_value:    str | None       # key=value string for traceability
    source_urls:         list[str]
    mlflow_run_id:       str | None

7.8 SourceEntry

Python

@dataclass
class SourceEntry:
    source_id:      str
    title:          str
    publisher:      str
    url:            str
    accessed_date:  str                   # ISO-8601
    last_modified:  str | None
    table_name:     str | None            # if from DuckDB table
    is_fresh:       bool = True           # False if > 90 days old
    freshness_warning: str | None = None  # shown in UI if not fresh

7.9 OfficialPosition

Python

@dataclass
class OfficialPosition:
    topic:        str
    summary_nl:   str              # 1–2 sentence official stance in B1 Dutch
    documents:    list[dict]       # [{"title": str, "url": str, "date": str, "publisher": str}]

Pre-baked official positions in src/backend/registry/official_positions.py:

Python

OFFICIAL_POSITIONS = {
    "drinkwater_zh": OfficialPosition(
        topic="drinkwater_zh",
        summary_nl="De beschikbaarheid van voldoende schoon drinkwater staat onder druk "
                   "door groei, klimaat en bronkwaliteit. De provincie neemt dit op als "
                   "ruimtelijke randvoorwaarde in haar beleid.",
        documents=[
            {
                "title": "Regionaal Waterprogramma Zuid-Holland 2022–2027",
                "url": "https://www.pzh.nl/regionaalwaterprogramma",
                "date": "2021-12-15",
                "publisher": "Provincie Zuid-Holland"
            },
            {
                "title": "Nationaal Waterprogramma 2022–2027",
                "url": "https://www.rijksoverheid.nl/nationaalwaterprogramma",
                "date": "2021-12-15",
                "publisher": "Ministerie van Infrastructuur en Waterstaat"
            },
            {
                "title": "Ruimtelijk Arrangement Rijk–Zuid-Holland",
                "url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
                "date": "2025-06-02",
                "publisher": "Ministerie BZK / Provincie ZH"
            },
        ]
    ),
    "klimaat_knmi": OfficialPosition(
        topic="klimaat_knmi",
        summary_nl="KNMI'23 beschrijft vier klimaatscenario's voor Nederland tot 2150. "
                   "Alle scenario's laten meer droogte en verzilting zien, met grotere "
                   "effecten onder hogere uitstootpaden.",
        documents=[
            {
                "title": "KNMI'23 Klimaatscenario's voor Nederland",
                "url": "https://www.knmi.nl/klimaatscenarios",
                "date": "2023-10-01",
                "publisher": "KNMI"
            }
        ]
    ),
    "krw_deadline": OfficialPosition(
        topic="krw_deadline",
        summary_nl="De Kaderrichtlijn Water stelt 2027 als uiterste deadline voor het "
                   "bereiken van 'goede toestand' van waterlichamen. Nederland heeft "
                   "uitloop-uitzonderingen benut tot deze maximale termijn.",
        documents=[
            {
                "title": "Kaderrichtlijn Water — Rijkswaterstaat",
                "url": "https://www.rijkswaterstaat.nl/water/waterbeheer/bescherming-en-gebruik-van-water/drinkwater/kaderrichtlijn-water",
                "date": "2024-01-01",
                "publisher": "Rijkswaterstaat"
            }
        ]
    ),
    "woningbouw_zh": OfficialPosition(
        topic="woningbouw_zh",
        summary_nl="Het Ruimtelijk Arrangement Rijk–ZH (juni 2025) noemt beschikbaarheid "
                   "van drinkwater expliciet als randvoorwaarde bij woningbouwopgave "
                   "van 80.000+ woningen in de Zuidelijke Randstad.",
        documents=[
            {
                "title": "Ruimtelijk Arrangement Rijk–Zuid-Holland 2025, Onderwerp 9 Drinkwater",
                "url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
                "date": "2025-06-02",
                "publisher": "Ministerie BZK / Provincie ZH"
            }
        ]
    ),
}

def get_official_position(scenario_type: str, location: str = "ZH") -> OfficialPosition:
    """Look up pre-baked OfficialPosition for a scenario type."""
    mapping = {
        "drop_pin": "drinkwater_zh",
        "intake_failure": "drinkwater_zh",
        "multi_hazard": "drinkwater_zh",
        "intervention": "drinkwater_zh",
    }
    key = mapping.get(scenario_type, "drinkwater_zh")
    return OFFICIAL_POSITIONS[key]

7.10 ScenarioCard (full output object)

Python

@dataclass
class ScenarioCard:
    # Identity & reproducibility
    scenario_id:           str           # UUID v4
    scenario_hash:         str           # SHA-256 of normalized params
    created_at:            str           # ISO-8601
    dataset_versions:      dict[str, str]  # table_name → last_modified
    git_commit:            str           # short SHA from git rev-parse HEAD
    stable_url:            str           # {SCENARIO_STABLE_URL_BASE}/scenario/{scenario_id}

    # Citation block
    citation_apa:          str           # formatted APA citation string
    citation_apa_nl:       str           # Dutch equivalent
    citation_bibtex:       str           # BibTeX format for reference managers

    # Content
    question_nl:           str
    scenario_type:         str
    params:                ScenarioParams
    results:               ScenarioResults
    assumptions:           list[Assumption]
    stakeholder_impacts:   list[StakeholderImpact]
    reasoning_steps:       list[ReasoningStep]
    source_registry:       list[SourceEntry]
    overlays:              list[dict]    # GeoJSON feature collections

    # Official position
    official_position:     OfficialPosition

    # Readability
    readability_score:     float         # Flesch-Douma score
    readability_level:     str           # "A2", "B1", "B2", "C1"

    # Comparison delta (populated when scenario_b was run)
    delta:                 "ScenarioDelta | None" = None

    # Cache metadata
    cache_used:            bool = False
    cached_at:             str | None = None
    dataset_drift_warning: str | None = None  # if datasets changed since cache

@dataclass
class ScenarioDelta:
    gap_delta_m3:          float         # scenario_b.gap - scenario_a.gap
    onset_year_delta:      int | None    # years pushed forward (positive = better)
    krw_delta:             int           # change in at-risk KRW bodies
    feasibility_change:    str           # e.g. "STOP → CAUTION"
    demand_delta_m3:       float
    narrative_nl:          str           # B1 Dutch summary of key differences

8. AgentState TypedDict

Python

# src/backend/workflows/scenario_workflow.py

from typing import TypedDict, Optional

class ScenarioAgentState(TypedDict):
    # Input
    question:                  str
    user_persona:              str
    language:                  str

    # Extracted parameters
    scenario_a_params:         Optional[ScenarioParams]
    scenario_b_params:         Optional[ScenarioParams]
    assumption_overrides:      Optional[dict]
    extraction_confidence:     float
    params_confirmed:          bool

    # Data loading
    loaded_tables:             list[str]
    source_registry:           list[SourceEntry]
    dataset_versions:          dict[str, str]
    data_freshness_warnings:   list[str]

    # Cache
    scenario_hash_a:           Optional[str]
    cache_hit_a:               bool
    cached_card_a:             Optional[ScenarioCard]
    scenario_hash_b:           Optional[str]
    cache_hit_b:               bool
    cached_card_b:             Optional[ScenarioCard]

    # Waterinfo
    waterinfo_cl_value:        Optional[float]
    waterinfo_cl_date:         Optional[str]
    waterinfo_is_live:         bool
    waterinfo_warning:         Optional[str]

    # Calculation results
    scenario_a_object:         Optional[dict]          # pre-results Scenario
    scenario_a_results:        Optional[ScenarioResults]
    scenario_b_results:        Optional[ScenarioResults]
    feasibility_class_a:       Optional[str]
    feasibility_class_b:       Optional[str]

    # Cumulative load
    active_scenarios_count:    int
    cumulative_demand_m3:      Optional[float]
    cumulative_warning_nl:     Optional[str]

    # Stakeholder + official position
    stakeholder_impacts_a:     list[StakeholderImpact]
    stakeholder_impacts_b:     list[StakeholderImpact]
    official_position:         Optional[OfficialPosition]

    # Reasoning
    reasoning_steps:           list[ReasoningStep]
    mlflow_run_id:             Optional[str]

    # Output
    scenario_card_a:           Optional[ScenarioCard]
    scenario_card_b:           Optional[ScenarioCard]
    scenario_delta:            Optional[ScenarioDelta]
    overlays_a:                list[dict]
    overlays_b:                list[dict]

    # Error handling
    error_state:               Optional[str]
    error_message_nl:          Optional[str]
    retry_count:               int
    followup_question_nl:      Optional[str]

    # Citizen mode
    is_citizen_mode:           bool
    citizen_postcode:          Optional[str]

9. Extended LangGraph workflow
9.1 Full workflow with routing code

Python

# src/backend/workflows/scenario_workflow.py

from langgraph.graph import StateGraph, END

def build_scenario_workflow() -> StateGraph:
    workflow = StateGraph(ScenarioAgentState)

    # Add all nodes
    workflow.add_node("detect_mode",              detect_mode_node)
    workflow.add_node("extract_scenario_params",  extract_scenario_params_node)
    workflow.add_node("confirm_with_user",        confirm_with_user_node)
    workflow.add_node("fetch_scenario_data",      fetch_scenario_data_node)
    workflow.add_node("check_scenario_cache",     check_scenario_cache_node)
    workflow.add_node("build_scenario_object",    build_scenario_object_node)
    workflow.add_node("run_scenario_calculation", run_scenario_calculation_node)
    workflow.add_node("compute_feasibility",      compute_feasibility_node)
    workflow.add_node("check_cumulative_load",    check_cumulative_load_node)
    workflow.add_node("stakeholder_impacts",      stakeholder_impacts_node)
    workflow.add_node("build_official_position",  build_official_position_node)
    workflow.add_node("record_reasoning_steps",   record_reasoning_steps_node)
    workflow.add_node("format_scenario_output",   format_scenario_output_node)
    workflow.add_node("run_scenario_b",           run_scenario_b_node)
    workflow.add_node("compute_delta",            compute_delta_node)
    workflow.add_node("format_citizen_response",  format_citizen_response_node)
    workflow.add_node("ask_followup",             ask_followup_node)
    workflow.add_node("handle_error",             handle_error_node)
    workflow.add_node("descriptive_workflow",     descriptive_workflow_node)  # existing

    # Entry
    workflow.set_entry_point("detect_mode")

    # Route from detect_mode
    workflow.add_conditional_edges(
        "detect_mode",
        route_by_mode,
        {
            "descriptive":  "descriptive_workflow",
            "scenario":     "extract_scenario_params",
            "citizen":      "extract_scenario_params",   # same extraction, different output format
            "knowledge_base": END                         # handled inline
        }
    )

    # Route from extract_scenario_params
    workflow.add_conditional_edges(
        "extract_scenario_params",
        route_by_extraction_confidence,
        {
            "high_confidence":  "confirm_with_user",
            "low_confidence":   "ask_followup",
            "error":            "handle_error"
        }
    )

    workflow.add_edge("confirm_with_user", "fetch_scenario_data")

    # Route from fetch_scenario_data
    workflow.add_conditional_edges(
        "fetch_scenario_data",
        route_by_data_availability,
        {
            "data_ok":          "check_scenario_cache",
            "data_partial":     "check_scenario_cache",   # proceed with warnings
            "data_missing":     "ask_followup"
        }
    )

    # Route from cache check
    workflow.add_conditional_edges(
        "check_scenario_cache",
        route_by_cache_status,
        {
            "cache_hit":        "format_scenario_output",  # skip to output
            "cache_miss":       "build_scenario_object"
        }
    )

    workflow.add_edge("build_scenario_object",    "run_scenario_calculation")
    workflow.add_edge("run_scenario_calculation", "compute_feasibility")
    workflow.add_edge("compute_feasibility",      "check_cumulative_load")
    workflow.add_edge("check_cumulative_load",    "stakeholder_impacts")
    workflow.add_edge("stakeholder_impacts",      "build_official_position")
    workflow.add_edge("build_official_position",  "record_reasoning_steps")
    workflow.add_edge("record_reasoning_steps",   "format_scenario_output")

    # Route from format_scenario_output
    workflow.add_conditional_edges(
        "format_scenario_output",
        route_by_scenario_b_presence,
        {
            "has_scenario_b":    "run_scenario_b",
            "no_scenario_b":     "check_citizen_mode"
        }
    )

    workflow.add_edge("run_scenario_b",  "compute_delta")

    # Route to citizen or end
    workflow.add_conditional_edges(
        "check_citizen_mode",
        route_citizen_mode,
        {
            "citizen":      "format_citizen_response",
            "professional": END
        }
    )

    workflow.add_edge("compute_delta",           END)
    workflow.add_edge("format_citizen_response", END)
    workflow.add_edge("ask_followup",            END)
    workflow.add_edge("handle_error",            END)
    workflow.add_edge("descriptive_workflow",    END)

    return workflow.compile()

9.2 Routing functions

Python

def route_by_mode(state: ScenarioAgentState) -> str:
    q = state["question"].lower()
    citizen_keywords = ["mijn water", "postcode", "mijn buurt", "is het veilig", "mijn wijk"]
    scenario_keywords = ["wat als", "stel dat", "wat gebeurt er als", "scenario", "effect van",
                         "kan er een", "hoeveel water heeft", "2040", "2035", "2030",
                         "droogte", "verzilting", "datacenter", "woningbouw",
                         "ijssel", "inname", "capaciteit"]
    descriptive_keywords = ["hoeveel", "waar is", "welke", "toon", "geef", "wat is de"]

    if any(k in q for k in citizen_keywords):
        return "citizen"
    if any(k in q for k in scenario_keywords):
        return "scenario"
    if any(k in q for k in descriptive_keywords):
        return "descriptive"
    return "scenario"   # default to scenario for ambiguous questions

def route_by_extraction_confidence(state: ScenarioAgentState) -> str:
    if state.get("error_state"):
        return "error"
    if state["extraction_confidence"] >= 0.7:
        return "high_confidence"
    return "low_confidence"

def route_by_cache_status(state: ScenarioAgentState) -> str:
    if state["cache_hit_a"]:
        return "cache_hit"
    return "cache_miss"

def route_by_scenario_b_presence(state: ScenarioAgentState) -> str:
    if state["scenario_b_params"] is not None:
        return "has_scenario_b"
    return "no_scenario_b"

def route_citizen_mode(state: ScenarioAgentState) -> str:
    if state["is_citizen_mode"]:
        return "citizen"
    return "professional"

10. GreenPT prompts (full templates)
10.1 Scenario parameter extraction prompt

Python

# src/backend/external/greenpt.py

SCENARIO_EXTRACTION_SYSTEM_PROMPT = """
Je bent een expert in het extraheren van scenario-parameters uit Nederlandse beleidsvragen
over drinkwaterzekerheid in Zuid-Holland.

Extraheer de volgende parameters uit de vraag:
- scenario_type: "drop_pin" (ruimtelijke locatievraag), "intake_failure" (inname-uitval),
                 "multi_hazard" (gecombineerde druk), of "intervention" (maatregel vergelijking)
- knmi_preset: "B", "Hd", "Hn", "Ld", "Ln" — default "Hd" als niet expliciet vermeld
- time_horizon: jaar als integer — default 2040 als niet vermeld
- growth_preset: "laag", "middel", "hoog" — default "middel"
- location_name: plaatsnaam of gebied als string, of null
- development_type: bijv. "datacenter_50mw", "housing_5000", "industrial_medium", of null
- development_mw: megawatt als float (alleen voor datacenter), of null
- development_units: aantal woningen als integer (alleen voor woningbouw), of null
- intake_id: naam van innamepunt bijv. "IJssel", "Lek", "Maas", of null
- outage_weeks: uitvalduur in weken als integer, of null
- interventions: lijst van maatregel-sleutels bijv. ["buffer_30000", "alternatieve_inname"], of []

Geef je antwoord uitsluitend als JSON conform het opgegeven schema.
Voeg een "confidence" veld toe tussen 0.0 en 1.0.
Verzon GEEN waarden. Als je iets niet kunt afleiden uit de vraag, gebruik dan null.
Als je confidence < 0.7 is, zet dan "needs_clarification" op true en geef
"clarification_question_nl" op in B1 Nederlands.
"""

SCENARIO_EXTRACTION_RESPONSE_SCHEMA = {
    "type": "object",
    "properties": {
        "scenario_type":         {"type": "string"},
        "knmi_preset":           {"type": "string"},
        "time_horizon":          {"type": "integer"},
        "growth_preset":         {"type": "string"},
        "location_name":         {"type": ["string", "null"]},
        "development_type":      {"type": ["string", "null"]},
        "development_mw":        {"type": ["number", "null"]},
        "development_units":     {"type": ["integer", "null"]},
        "intake_id":             {"type": ["string", "null"]},
        "outage_weeks":          {"type": ["integer", "null"]},
        "interventions":         {"type": "array", "items": {"type": "string"}},
        "confidence":            {"type": "number"},
        "needs_clarification":   {"type": "boolean"},
        "clarification_question_nl": {"type": ["string", "null"]}
    },
    "required": ["scenario_type", "knmi_preset", "time_horizon", "confidence"]
}

10.2 Confirmation message prompt

Python

CONFIRMATION_PROMPT_TEMPLATE = """
Beschrijf in maximaal 3 zinnen in helder B1 Nederlands wat je hebt begrepen uit de vraag:
"{question}"

De geëxtraheerde parameters zijn:
{params_json}

Begin met: "Ik begrijp dat je wilt weten..."
Sluit af met: "Klopt dit? Dan ga ik verder met de berekening."
Gebruik geen technische termen. Schrijf voor een provincieambtenaar zonder technische achtergrond.
"""

10.3 Reasoning step description prompt

Python

REASONING_STEP_PROMPT_TEMPLATE = """
Beschrijf stap {step_nr} van het scenario-rekenproces in maximaal 2 zinnen B1 Nederlands.
Stap: {step_technical_description}
Berekende waarde: {calculated_value}
Gebruikte data: {datasets_used}

Regels:
- Gebruik geen jargon. Als je een technisch begrip moet gebruiken, leg het direct uit.
- Noem NOOIT een getal dat je zelf hebt bedacht. Gebruik alleen: {calculated_value}
- Schrijf in actieve zin: "Het systeem berekende...", "De data laat zien..."
- Voeg geen conclusies toe die niet volgen uit {calculated_value}
"""

10.4 Stakeholder impact prose prompt

Python

STAKEHOLDER_IMPACT_PROMPT_TEMPLATE = """
Schrijf 1-2 zinnen in B1 Nederlands over het effect van dit scenario op: {stakeholder_nl}

Scenario samenvatting: {scenario_summary}
Berekend effect: {impact_data}
Beleidsgrondslag: {policy_basis}

Regels:
- Gebruik de berekende effecten. Voeg GEEN nieuwe getallen toe.
- Vermeld altijd de beleidsgrondslag aan het einde: "(Bron: {policy_basis})"
- Schrijf concreet en persoonsgericht: "Woningzoekenden in zone X..."
- Maximaal 50 woorden.
"""

10.5 Citizen response prompt

Python

CITIZEN_RESPONSE_PROMPT_TEMPLATE = """
Schrijf een kort, gerustststellend antwoord voor een burger over drinkwaterzekerheid.
Postcode: {postcode}
Leveringszone: {zone_name}
Scenario uitkomst: {feasibility_class}
Berekend tekort of overschot: {gap_m3}
Tijdshorizon: {time_horizon}

Format:
1. Één zin verdict: "Op dit moment..." of "In het jaar {time_horizon}..."
2. Maximaal één zin over onzekerheid als relevant
3. Links: drinkwaterbedrijf, KNMI, en provinciaal waterprogramma
4. Disclaimer: "Dit is een exploratief scenario, geen officiële meting."

Regels:
- Geen m³ of technische eenheden tenzij de burger er expliciet om vraagt
- Geen jargon
- Maximaal 80 woorden
- Schrijf in de tweede persoon: "jouw buurt", "jouw water"
"""

11. Calculation engine (all functions)
11.1 Demand calculation

Python

# src/backend/calculation/demand.py

import numpy as np
from src.backend.models.scenario import Assumption, DEFAULT_ASSUMPTIONS

KNMI_PRESETS = {
    "B":  {"drought_freq": 1.0, "cl_delta": 0,   "peak_multiplier": 1.0,  "low_flow_weeks": 0},
    "Hn": {"drought_freq": 1.3, "cl_delta": 30,  "peak_multiplier": 1.08, "low_flow_weeks": 2},
    "Hd": {"drought_freq": 1.8, "cl_delta": 80,  "peak_multiplier": 1.15, "low_flow_weeks": 6},
    "Ln": {"drought_freq": 1.2, "cl_delta": 20,  "peak_multiplier": 1.05, "low_flow_weeks": 1},
    "Ld": {"drought_freq": 1.5, "cl_delta": 50,  "peak_multiplier": 1.10, "low_flow_weeks": 4},
}

GROWTH_PRESETS = {
    "laag":   {"population_growth": 0.05, "housing_units": 40000, "demand_delta_pct": 0.04,
               "source_url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
               "source_label": "Ruimtelijk Arrangement Rijk–ZH 2025"},
    "middel": {"population_growth": 0.08, "housing_units": 60000, "demand_delta_pct": 0.065,
               "source_url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
               "source_label": "Ruimtelijk Arrangement Rijk–ZH 2025"},
    "hoog":   {"population_growth": 0.12, "housing_units": 80000, "demand_delta_pct": 0.09,
               "source_url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
               "source_label": "Ruimtelijk Arrangement Rijk–ZH 2025"},
}

DEVELOPMENT_DEMAND = {
    "datacenter_50mw": {
        "formula": "mw * multiplier",
        "default_mw": 50.0,
        "multiplier_default": 12.0,
        "multiplier_min": 5.0,
        "multiplier_max": 20.0,
        "unit": "m³/dag/MW",
        "assumption_key": "datacenter_m3_per_mw_day",
    },
    "housing_5000": {
        "formula": "units * demand_per_unit",
        "default_units": 5000,
        "demand_per_unit_default": 0.35,
        "demand_per_unit_min": 0.28,
        "demand_per_unit_max": 0.42,
        "unit": "m³/dag/woning",
        "assumption_key": "demand_per_dwelling_m3_day",
    },
    "office_medium": {
        "formula": "sqm * rate",
        "default_sqm": 10000,
        "rate_default": 0.003,
        "rate_min": 0.002,
        "rate_max": 0.005,
        "unit": "m³/dag/m²",
        "assumption_key": "office_m3_per_sqm_day",
    },
}

def calculate_daily_demand(
    baseline_m3_per_day: float,
    growth_preset: str,
    knmi_preset: str,
    development_type: str | None,
    development_params: dict,
    assumption_overrides: dict[str, float] | None = None,
    seed: int = 42
) -> tuple[float, float, float]:
    """
    Calculate projected daily demand.
    Returns: (point_estimate, range_min, range_max)
    All values in m³/day.
    """
    np.random.seed(seed)

    growth = GROWTH_PRESETS[growth_preset]
    knmi = KNMI_PRESETS[knmi_preset]
    overrides = assumption_overrides or {}

    # Growth delta
    growth_fraction = growth["demand_delta_pct"]

    # KNMI peak multiplier
    peak_multiplier = overrides.get("peak_demand_buffer_factor",
                                     DEFAULT_ASSUMPTIONS["peak_demand_buffer_factor"].value)
    peak_multiplier_min = DEFAULT_ASSUMPTIONS["peak_demand_buffer_factor"].value_min
    peak_multiplier_max = DEFAULT_ASSUMPTIONS["peak_demand_buffer_factor"].value_max
    # Use KNMI-specific multiplier if more severe
    knmi_peak = knmi["peak_multiplier"]
    effective_peak = max(peak_multiplier, knmi_peak)

    # Development demand delta
    dev_delta = 0.0
    dev_delta_min = 0.0
    dev_delta_max = 0.0
    if development_type and development_type in DEVELOPMENT_DEMAND:
        dev = DEVELOPMENT_DEMAND[development_type]
        if dev["formula"] == "mw * multiplier":
            mw = development_params.get("mw", dev["default_mw"])
            mult = overrides.get(dev["assumption_key"], dev["multiplier_default"])
            dev_delta = mw * mult
            dev_delta_min = mw * dev["multiplier_min"]
            dev_delta_max = mw * dev["multiplier_max"]
        elif dev["formula"] == "units * demand_per_unit":
            units = development_params.get("units", dev["default_units"])
            rate = overrides.get(dev["assumption_key"], dev["demand_per_unit_default"])
            dev_delta = units * rate
            dev_delta_min = units * dev["demand_per_unit_min"]
            dev_delta_max = units * dev["demand_per_unit_max"]

    # Point estimate
    point = (baseline_m3_per_day * (1 + growth_fraction) * effective_peak) + dev_delta

    # Conservative range
    range_min = (baseline_m3_per_day
                 * (1 + growth_fraction * 0.8)
                 * peak_multiplier_min) + dev_delta_min

    range_max = (baseline_m3_per_day
                 * (1 + growth_fraction * 1.2)
                 * peak_multiplier_max) + dev_delta_max

    return point, range_min, range_max

11.2 Chloride concentration

Python

# src/backend/calculation/chloride.py

from src.backend.models.scenario import Assumption, DEFAULT_ASSUMPTIONS
from src.backend.external.waterinfo import fetch_waterinfo_chloride
from src.backend.cache.waterinfo_cache import load_chloride_cache

KNMI_CL_DELTAS = {
    "B": 0, "Hn": 30, "Hd": 80, "Ln": 20, "Ld": 50
}

async def get_chloride_for_intake(
    intake_id: str,
    knmi_preset: str,
    outage_weeks: int = 0,
    assumption_overrides: dict | None = None
) -> dict:
    """
    Returns chloride concentration estimate for an intake.
    Mandatory: tries Waterinfo API first. Falls back to cache with explicit warning.
    Never silently invents a value.
    """
    overrides = assumption_overrides or {}

    # Step 1: get baseline chloride (Waterinfo mandatory for IJssel/Lek/Maas)
    mandatory_intakes = ["ijssel", "lek", "maas", "hollandse_ijssel"]
    is_mandatory = any(m in intake_id.lower() for m in mandatory_intakes)

    waterinfo_result = await fetch_waterinfo_chloride(intake_id)

    baseline_cl = waterinfo_result["value"]
    cl_source = waterinfo_result["source"]
    cl_date = waterinfo_result["date"]
    waterinfo_warning = waterinfo_result.get("warning")

    # Step 2: apply KNMI delta
    cl_delta = KNMI_CL_DELTAS.get(knmi_preset, 0)

    # Step 3: apply outage degradation
    degradation_per_week = overrides.get(
        "outage_cl_degradation_per_week",
        DEFAULT_ASSUMPTIONS["outage_cl_degradation_per_week"].value
    )
    outage_multiplier = 1.0
    if outage_weeks > 0:
        outage_multiplier = 1 + (degradation_per_week * min(outage_weeks, 6))

    # Compute
    effective_cl = (baseline_cl + cl_delta) * outage_multiplier

    return {
        "cl_concentration": effective_cl,
        "baseline_cl": baseline_cl,
        "cl_delta_knmi": cl_delta,
        "outage_multiplier": outage_multiplier,
        "cl_source": cl_source,
        "cl_date": cl_date,
        "waterinfo_warning": waterinfo_warning,
        "is_mandatory": is_mandatory,
    }

11.3 Capacity check from DuckDB

Python

# src/backend/calculation/capacity.py

import duckdb
from src.backend.models.scenario import ScenarioParams

def check_zone_capacity(
    conn: duckdb.DuckDBPyConnection,
    location_wkt: str | None,
    zone_id: str | None,
    time_horizon: int
) -> dict:
    """
    Retrieve supply capacity and current demand for the supply zone
    containing or nearest to the given location.
    Returns capacity, current demand, and zone metadata.
    """
    if location_wkt:
        # Find zone by spatial intersection
        result = conn.execute("""
            SELECT
                lz.zone_id,
                lz.naam,
                lz.capaciteit_m3_dag,
                lz.vraag_2023_m3_dag,
                lz.h3_id
            FROM leveringszones lz
            WHERE ST_Contains(
                ST_GeomFromText(lz.geometry),
                ST_GeomFromText(?)
            )
            LIMIT 1
        """, [location_wkt]).fetchone()

        if not result:
            # Fall back to nearest zone
            result = conn.execute("""
                SELECT
                    lz.zone_id,
                    lz.naam,
                    lz.capaciteit_m3_dag,
                    lz.vraag_2023_m3_dag,
                    lz.h3_id
                FROM leveringszones lz
                ORDER BY ST_Distance(
                    ST_GeomFromText(lz.geometry),
                    ST_GeomFromText(?)
                )
                LIMIT 1
            """, [location_wkt]).fetchone()

    elif zone_id:
        result = conn.execute("""
            SELECT zone_id, naam, capaciteit_m3_dag, vraag_2023_m3_dag, h3_id
            FROM leveringszones WHERE zone_id = ?
        """, [zone_id]).fetchone()

    if not result:
        raise DataNotFoundError(f"No supply zone found for location/zone_id provided")

    return {
        "zone_id":           result[0],
        "zone_naam":         result[1],
        "capacity_m3_day":   result[2],
        "current_demand_m3": result[3],
        "h3_id":             result[4],
    }


def get_intake_capacity(
    conn: duckdb.DuckDBPyConnection,
    intake_id: str,
    outage_weeks: int = 0
) -> dict:
    """
    Get production capacity for a specific intake,
    including alternative source coverage if primary is down.
    """
    primary = conn.execute("""
        SELECT p.locatie_id, p.productie_cap_m3_dag, p.cl_threshold_mg_l,
               p.behandel_tech, p.status,
               w.naam, w.capaciteit_m3_dag as inname_cap
        FROM productieketen p
        JOIN winlocaties w ON p.intake_id = w.locatie_id
        WHERE LOWER(w.naam) LIKE ?
        LIMIT 1
    """, [f"%{intake_id.lower()}%"]).fetchone()

    alternatives = conn.execute("""
        SELECT ab.max_capaciteit_m3_dag, ab.activatie_tijd_uur, ab.type
        FROM alternatieve_bronnen ab
        JOIN winlocaties w ON ab.intake_id = w.locatie_id
        WHERE LOWER(w.naam) LIKE ?
    """, [f"%{intake_id.lower()}%"]).fetchall()

    alt_capacity = sum(a[0] for a in alternatives) if alternatives else 0.0

    return {
        "locatie_id":         primary[0] if primary else None,
        "production_cap_m3":  primary[1] if primary else 0.0,
        "cl_threshold_mg_l":  primary[2] if primary else None,   # None = use fallback Assumption
        "cl_threshold_from_db": primary[2] is not None if primary else False,
        "treatment_tech":     primary[3] if primary else None,
        "intake_naam":        primary[5] if primary else intake_id,
        "alternative_cap_m3": alt_capacity,
        "net_capacity_if_down": alt_capacity if outage_weeks > 0 else (primary[1] if primary else 0.0),
    }

11.4 KRW risk check

Python

# src/backend/calculation/krw.py

import duckdb

def check_krw_risk(
    conn: duckdb.DuckDBPyConnection,
    affected_h3_ids: list[str],
    cl_concentration: float | None,
    supply_gap_m3: float
) -> dict:
    """
    Identify KRW water bodies at risk in the affected supply zones.
    Risk criteria:
    - Water body is in or adjacent to affected H3 cells
    - Current KRW status is already at risk (not "goed")
    - Supply gap > 0 (shortage) or chloride exceeds threshold
    """
    if not affected_h3_ids:
        return {"krw_at_risk_count": 0, "krw_areas": [], "krw_details": []}

    h3_list = ", ".join([f"'{h}'" for h in affected_h3_ids])

    krw_results = conn.execute(f"""
        SELECT DISTINCT
            kw.lichaam_id,
            kw.naam,
            kw.type,
            kw.status_2023,
            kw.krw_doeljaar
        FROM krw_waterlichamen kw
        WHERE ST_Intersects(
            ST_GeomFromText(kw.geometry),
            (SELECT ST_Union_Agg(ST_GeomFromText(geometry))
             FROM leveringszones
             WHERE h3_id IN ({h3_list}))
        )
        AND kw.status_2023 != 'goed'
        ORDER BY kw.krw_doeljaar ASC
    """).fetchall()

    at_risk = []
    for row in krw_results:
        risk_reason = []
        if supply_gap_m3 < 0:
            risk_reason.append("aanvoertekort")
        if cl_concentration and cl_concentration > 150:
            risk_reason.append("chloride-overschrijding")

        at_risk.append({
            "lichaam_id":   row[0],
            "naam":         row[1],
            "type":         row[2],
            "status_2023":  row[3],
            "krw_doeljaar": row[4],
            "risk_reasons": risk_reason
        })

    return {
        "krw_at_risk_count": len(at_risk),
        "krw_areas":         [r["naam"] for r in at_risk],
        "krw_details":       at_risk,
    }

11.5 Onset year calculator

Python

# src/backend/calculation/onset.py

def calculate_onset_year(
    baseline_demand_m3: float,
    capacity_m3: float,
    growth_preset: str,
    knmi_preset: str,
    development_delta_m3: float = 0.0,
    start_year: int = 2024,
    end_year: int = 2040,
    assumption_overrides: dict | None = None
) -> dict:
    """
    Step through years from start_year to end_year.
    Find the first year where projected demand > capacity.
    Returns onset_year (or None if never critical in range)
    and year-by-year demand series for visualization.
    """
    from src.backend.calculation.demand import GROWTH_PRESETS, KNMI_PRESETS

    growth = GROWTH_PRESETS[growth_preset]
    knmi = KNMI_PRESETS[knmi_preset]
    overrides = assumption_overrides or {}

    # Annual growth rate (linear interpolation from total 2024→2040)
    years_total = 2040 - 2024
    annual_growth_rate = growth["demand_delta_pct"] / years_total

    demand_series = {}
    onset_year = None

    for year in range(start_year, end_year + 1):
        years_elapsed = year - start_year
        growth_fraction = annual_growth_rate * years_elapsed

        # Apply KNMI peak only in target year (worst case)
        peak_mult = knmi["peak_multiplier"] if year == end_year else 1.0

        annual_demand = (
            baseline_demand_m3 * (1 + growth_fraction) * peak_mult
        ) + development_delta_m3

        demand_series[year] = annual_demand

        if annual_demand > capacity_m3 and onset_year is None:
            onset_year = year

    return {
        "onset_year":    onset_year,
        "demand_series": demand_series,
        "critical_by_horizon": demand_series.get(end_year, 0) > capacity_m3,
    }

11.6 Stakeholder rule engine

Python

# src/backend/calculation/stakeholder_rules.py

from src.backend.models.scenario import StakeholderImpact

# Rule table: (scenario_type, feasibility_class) → list of (stakeholder, severity, overlay, policy_basis, policy_url)
STAKEHOLDER_RULES = {
    ("drop_pin", "STOP"): [
        {
            "key": "woningzoekenden",
            "nl": "Woningzoekenden",
            "severity": "critical",
            "overlay": "woningbouwlocaties",
            "policy_basis": "Ruimtelijk Arrangement Rijk–ZH 2025, Onderwerp 9",
            "policy_url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
            "impact_template": "Woningzoekenden in zone {zone} ondervinden hinder: "
                               "het tekort van {gap} m³/dag blokkeert equivalent van {units} woningen."
        },
        {
            "key": "drinkwaterbedrijf",
            "nl": "Drinkwaterbedrijf (Dunea/Evides)",
            "severity": "high",
            "overlay": "leveringszones",
            "policy_basis": "Drinkwaterwet, artikel 7",
            "policy_url": "https://wetten.overheid.nl/BWBR0026338",
            "impact_template": "Het drinkwaterbedrijf moet extra capaciteit regelen voor {gap} m³/dag extra vraag."
        },
        {
            "key": "gemeente_vergunning",
            "nl": "Gemeente (vergunningverlening)",
            "severity": "high",
            "overlay": "gemeentegrenzen",
            "policy_basis": "Omgevingswet, artikel 5.1",
            "policy_url": "https://wetten.overheid.nl/BWBR0037885",
            "impact_template": "De gemeente kan de vergunning voor dit project mogelijk niet verlenen "
                               "vanwege onvoldoende drinkwatercapaciteit."
        },
    ],
    ("drop_pin", "CAUTION"): [
        {
            "key": "woningzoekenden",
            "nl": "Woningzoekenden",
            "severity": "medium",
            "overlay": "woningbouwlocaties",
            "policy_basis": "Ruimtelijk Arrangement Rijk–ZH 2025, Onderwerp 9",
            "policy_url": "https://www.rijksoverheid.nl/ruimtelijkarrangement-zh",
            "impact_template": "Woningzoekenden in zone {zone}: drinkwatervoorziening is krap "
                               "onder droge scenario's maar haalbaar onder basisscenario."
        },
    ],
    ("drop_pin", "GO"): [
        {
            "key": "drinkwaterbedrijf",
            "nl": "Drinkwaterbedrijf",
            "severity": "low",
            "overlay": "leveringszones",
            "policy_basis": "Drinkwaterwet",
            "policy_url": "https://wetten.overheid.nl/BWBR0026338",
            "impact_template": "Voldoende restcapaciteit ({headroom} m³/dag) in leveringszone {zone}."
        },
    ],
    ("intake_failure", "STOP"): [
        {
            "key": "drinkwaterbedrijf",
            "nl": "Drinkwaterbedrijf (Dunea/Evides)",
            "severity": "critical",
            "overlay": "productieketen",
            "policy_basis": "Drinkwaterwet, artikel 4 (leveringszekerheid)",
            "policy_url": "https://wetten.overheid.nl/BWBR0026338",
            "impact_template": "Uitval van inname {intake} gedurende {weeks} weken veroorzaakt "
                               "een tekort van {gap} m³/dag. Noodmaatregelen nodig."
        },
        {
            "key": "zorg_kritisch",
            "nl": "Zorgsector en kritische infrastructuur",
            "severity": "critical",
            "overlay": "leveringszones",
            "policy_basis": "Drinkwaterwet, artikel 6 (voorrangslevering)",
            "policy_url": "https://wetten.overheid.nl/BWBR0026338",
            "impact_template": "Ziekenhuizen en kritische infrastructuur in zone {zone} "
                               "hebben voorrang, maar noodvoorraden zijn beperkt."
        },
        {
            "key": "natuur_krw",
            "nl": "Natuur en KRW-waterlichamen",
            "severity": "high",
            "overlay": "krw_waterlichamen",
            "policy_basis": "Kaderrichtlijn Water, deadline 2027",
            "policy_url": "https://www.rijkswaterstaat.nl/kaderrichtlijn-water",
            "impact_template": "{krw_count} KRW-waterlichamen in het beïnvloede gebied lopen risico "
                               "door verhoogde chlorideconcentratie ({cl} mg/L)."
        },
        {
            "key": "landbouw",
            "nl": "Landbouw (irrigatie)",
            "severity": "high",
            "overlay": "lgn_landgebruik",
            "policy_basis": "Regionaal Waterprogramma ZH 2022–2027",
            "policy_url": "https://www.pzh.nl/regionaalwaterprogramma",
            "impact_template": "Landbouwpercelen in het beïnvloede gebied kunnen bij verzilting "
                               "geen oppervlaktewater gebruiken voor irrigatie."
        },
    ],
    ("intake_failure", "CAUTION"): [
        {
            "key": "drinkwaterbedrijf",
            "nl": "Drinkwaterbedrijf",
            "severity": "medium",
            "overlay": "productieketen",
            "policy_basis": "Drinkwaterwet, artikel 4",
            "policy_url": "https://wetten.overheid.nl/BWBR0026338",
            "impact_template": "Uitval van {weeks} weken leidt tot spanning in leveringszone {zone}, "
                               "maar alternatieve bronnen dekken {alt_pct}% van de vraag."
        },
    ],
}

def apply_stakeholder_rules(
    scenario_type: str,
    feasibility_class: str,
    results: dict,
    zone_naam: str = ""
) -> list[StakeholderImpact]:
    """Apply rule table and fill impact templates."""
    rule_key = (scenario_type, feasibility_class)
    rules = STAKEHOLDER_RULES.get(rule_key, [])

    impacts = []
    for rule in rules:
        impact_text = rule["impact_template"].format(
            zone=zone_naam,
            gap=f"{abs(results.get('supply_gap_m3', 0)):,.0f}",
            units=f"{abs(results.get('supply_gap_m3', 0)) / 0.35:,.0f}",
            headroom=f"{results.get('capacity_remaining_m3', 0):,.0f}",
            intake=results.get("intake_naam", "inname"),
            weeks=results.get("outage_weeks", "?"),
            cl=f"{results.get('cl_concentration', 0):.0f}",
            krw_count=results.get("krw_at_risk_count", 0),
            alt_pct=f"{results.get('alternative_coverage_pct', 0):.0f}",
        )
        impacts.append(StakeholderImpact(
            stakeholder_key=rule["key"],
            stakeholder_nl=rule["nl"],
            impact_nl=impact_text,
            severity=rule["severity"],
            affected_zone_ids=[results.get("zone_id", "")],
            overlay_layer=rule["overlay"],
            policy_basis=rule["policy_basis"],
            policy_url=rule["policy_url"],
        ))

    return impacts

11.7 Intervention catalogue and ranking

Python

# src/backend/calculation/interventions.py

INTERVENTION_CATALOGUE = {
    "buffer_30000": {
        "label_nl": "Bufferopslag 30.000 m³",
        "supply_delta_m3_day": 30000,   # 1-day buffer spread across 30 days = 1000 effective/day
        "supply_delta_range": (25000, 35000),
        "cost_eur_low": 15_000_000,
        "cost_eur_high": 25_000_000,
        "implementation_years": 3,
        "source_url": "https://www.pzh.nl/regionaalwaterprogramma",
        "source_label": "Regionaal Waterprogramma ZH 2022–2027",
    },
    "alternatieve_inname_lek": {
        "label_nl": "Alternatieve inname via Lek",
        "supply_delta_m3_day": 65000,
        "supply_delta_range": (50000, 80000),
        "cost_eur_low": 40_000_000,
        "cost_eur_high": 100_000_000,
        "implementation_years": 7,
        "source_url": "https://www.pzh.nl/regionaalwaterprogramma",
        "source_label": "Regionaal Waterprogramma ZH 2022–2027",
    },
    "vraagbeperking_10pct": {
        "label_nl": "Vraagbeperking 10% (regulatoir)",
        "supply_delta_m3_day": None,        # computed as % of demand, not fixed
        "supply_delta_fraction": -0.10,     # reduces demand by 10%
        "supply_delta_range": (0.08, 0.12),
        "cost_eur_low": 0,
        "cost_eur_high": 500_000,           # admin costs
        "implementation_years": 1,
        "source_url": "https://wetten.overheid.nl/BWBR0026338",
        "source_label": "Drinkwaterwet, artikel 10",
    },
    "interconnectie_buurregio": {
        "label_nl": "Interconnectie met buurregio",
        "supply_delta_m3_day": 30000,
        "supply_delta_range": (20000, 40000),
        "cost_eur_low": 20_000_000,
        "cost_eur_high": 60_000_000,
        "implementation_years": 5,
        "source_url": "https://www.vewin.nl/SiteCollectionDocuments/Publicaties/VEWIN-Infrastructuurrapport-2023.pdf",
        "source_label": "VEWIN Infrastructuurrapport 2023",
    },
}

def rank_interventions_to_feasibility(
    supply_gap_m3: float,
    daily_demand_m3: float,
    current_feasibility: str
) -> list[dict]:
    """
    Rank available interventions by gap closure effectiveness.
    Returns top 3 interventions with post-intervention feasibility class.
    """
    if current_feasibility == "GO":
        return []

    ranked = []
    for key, intervention in INTERVENTION_CATALOGUE.items():
        # Compute effective supply delta
        if intervention.get("supply_delta_m3_day"):
            delta = intervention["supply_delta_m3_day"]
        elif intervention.get("supply_delta_fraction"):
            delta = abs(intervention["supply_delta_fraction"]) * daily_demand_m3
        else:
            delta = 0

        new_gap = supply_gap_m3 + delta   # gap is negative; delta closes it
        gap_closure_pct = min(100.0, (delta / abs(supply_gap_m3)) * 100) if supply_gap_m3 < 0 else 0

        # Determine new feasibility
        if new_gap >= 0:
            new_class = "GO"
        elif new_gap >= -0.05 * daily_demand_m3:
            new_class = "CAUTION"
        else:
            new_class = "STOP"

        ranked.append({
            "key": key,
            "label_nl": intervention["label_nl"],
            "supply_delta_m3": delta,
            "gap_closure_pct": gap_closure_pct,
            "post_intervention_gap_m3": new_gap,
            "post_intervention_feasibility": new_class,
            "cost_range_eur": (intervention["cost_eur_low"], intervention["cost_eur_high"]),
            "implementation_years": intervention["implementation_years"],
            "source_url": intervention["source_url"],
            "source_label": intervention["source_label"],
        })

    # Sort: by gap_closure_pct DESC, then cost ASC
    ranked.sort(key=lambda x: (-x["gap_closure_pct"], x["cost_range_eur"][0]))
    return ranked[:3]

11.8 Human scale converter

Python

# src/backend/calculation/human_scale.py

CONVERSION_REFERENCES = {
    "persons_per_m3_day": {
        "value": 1 / 0.119,      # 119 L/person/day = 0.119 m³/person/day → ~8.4 persons/m³
        "source_url": "https://www.vewin.nl/SiteCollectionDocuments/Publicaties/Cijfers/VEWIN-Watergebruik-in-cijfers-2023.pdf",
        "source_label": "VEWIN Watergebruik in cijfers 2023"
    },
    "households_per_m3_day": {
        "value": 1 / 0.35,       # 0.35 m³/household/day → ~2.86 households/m³
        "source_url": "https://www.vewin.nl/SiteCollectionDocuments/Publicaties/Cijfers/VEWIN-Watergebruik-in-cijfers-2023.pdf",
        "source_label": "VEWIN Watergebruik in cijfers 2023"
    },
}

def build_human_scale_ref(
    metric_key: str,
    metric_value: float,
    metric_unit: str,
    demand_per_household: float = 0.35
) -> "HumanScaleRef":
    from src.backend.models.scenario import HumanScaleRef

    if metric_unit in ["m³/dag", "m3/day"]:
        households = abs(metric_value) / demand_per_household
        persons = abs(metric_value) / 0.119

        if metric_value < 0:
            analogy = f"equivalent aan dagelijks verbruik van ~{households:,.0f} huishoudens zonder water"
        else:
            analogy = f"equivalent aan dagelijks verbruik van ~{households:,.0f} huishoudens"

        return HumanScaleRef(
            metric_key=metric_key,
            metric_value=metric_value,
            metric_unit=metric_unit,
            analogy_nl=analogy,
            analogy_source=CONVERSION_REFERENCES["households_per_m3_day"]["source_url"],
            analogy_label=CONVERSION_REFERENCES["households_per_m3_day"]["source_label"],
        )

    return HumanScaleRef(
        metric_key=metric_key,
        metric_value=metric_value,
        metric_unit=metric_unit,
        analogy_nl="",
        analogy_source="",
        analogy_label="",
    )

12. SQL patterns per scenario type
12.1 Type 3 (drop-pin) — zone capacity query

SQL

-- Find supply zone containing pin location, with capacity details
WITH pin_zone AS (
    SELECT
        lz.zone_id,
        lz.naam AS zone_naam,
        lz.capaciteit_m3_dag,
        lz.vraag_2023_m3_dag,
        lz.h3_id,
        (lz.capaciteit_m3_dag - lz.vraag_2023_m3_dag) AS huidig_overschot_m3
    FROM leveringszones lz
    WHERE ST_Contains(
        ST_GeomFromText(lz.geometry),
        ST_GeomFromText(?)       -- WKT point from PDOK geocoder
    )
    LIMIT 1
),
zes_uur AS (
    SELECT zuz.zone_id, zuz.geometry AS zes_uur_geom
    FROM zes_uur_zones zuz
    WHERE zuz.intake_id IN (
        SELECT DISTINCT p.intake_id
        FROM productieketen p
        WHERE p.locatie_id IN (
            SELECT locatie_id FROM winlocaties
            WHERE ST_DWithin(
                ST_GeomFromText(geometry),
                ST_GeomFromText(?),   -- same WKT point
                50000                 -- 50km radius for relevant intakes
            )
        )
    )
)
SELECT
    pz.*,
    zuz.zes_uur_geom
FROM pin_zone pz
LEFT JOIN zes_uur zuz ON true;

12.2 Type 1 (intake failure) — affected zones query

SQL

-- Find all supply zones served by a specific intake, with H3 cells
WITH intake_zones AS (
    SELECT DISTINCT
        lz.zone_id,
        lz.naam,
        lz.capaciteit_m3_dag,
        lz.vraag_2023_m3_dag,
        lz.h3_id
    FROM leveringszones lz
    JOIN productieketen p ON ST_Intersects(
        ST_GeomFromText(lz.geometry),
        ST_Buffer(
            ST_GeomFromText(
                (SELECT geometry FROM winlocaties WHERE LOWER(naam) LIKE ?)
            ),
            30000    -- 30km buffer around intake
        )
    )
    WHERE p.intake_id IN (
        SELECT locatie_id FROM winlocaties WHERE LOWER(naam) LIKE ?
    )
),
chloride_history AS (
    SELECT
        ic.intake_id,
        AVG(ic.cl_mg_l) AS avg_cl_30d,
        MAX(ic.cl_mg_l) AS max_cl_30d,
        MAX(ic.datum) AS last_measurement
    FROM innamepunten_chloride ic
    WHERE ic.intake_id IN (
        SELECT locatie_id FROM winlocaties WHERE LOWER(naam) LIKE ?
    )
    AND ic.datum >= CURRENT_DATE - INTERVAL 30 DAY
    GROUP BY ic.intake_id
),
krw_affected AS (
    SELECT kw.lichaam_id, kw.naam, kw.status_2023, kw.krw_doeljaar
    FROM krw_waterlichamen kw
    WHERE ST_Intersects(
        ST_GeomFromText(kw.geometry),
        (SELECT ST_Union_Agg(ST_GeomFromText(geometry)) FROM intake_zones)
    )
    AND kw.status_2023 != 'goed'
)
SELECT
    iz.*,
    ch.avg_cl_30d,
    ch.max_cl_30d,
    ch.last_measurement,
    COUNT(ka.lichaam_id) OVER () AS krw_at_risk_count
FROM intake_zones iz
CROSS JOIN chloride_history ch
LEFT JOIN krw_affected ka ON true
GROUP BY iz.zone_id, iz.naam, iz.capaciteit_m3_dag, iz.vraag_2023_m3_dag, iz.h3_id,
         ch.avg_cl_30d, ch.max_cl_30d, ch.last_measurement, ka.lichaam_id, ka.naam,
         ka.status_2023, ka.krw_doeljaar;

12.3 GeoJSON overlay generation

SQL

-- Generate GeoJSON for supply gap heatmap overlay
SELECT json_object(
    'type', 'FeatureCollection',
    'features', json_group_array(
        json_object(
            'type', 'Feature',
            'geometry', json(geometry),
            'properties', json_object(
                'zone_id',       zone_id,
                'naam',          naam,
                'gap_m3_dag',    ?,       -- computed supply gap
                'gap_klasse',    CASE
                                   WHEN ? >= 0 THEN 'GO'
                                   WHEN ? >= -0.05 * capaciteit_m3_dag THEN 'CAUTION'
                                   ELSE 'STOP'
                                 END,
                'capaciteit',    capaciteit_m3_dag,
                'vraag',         vraag_2023_m3_dag
            )
        )
    )
) AS geojson
FROM leveringszones
WHERE zone_id IN (SELECT zone_id FROM _scenario_results);

13. Scenario Type 2 — Multi-hazard (fully specified; stretch goal)
13.1 What Type 2 is

Type 2 answers: "Which combination of pressures hits first, and what is the combined severity?"

Example question: "Wat is het effect van klimaatdruk, KRW-handhaving én woningbouwgroei tegelijk op drinkwaterzekerheid in 2040?"
13.2 Parameter schema

Python

@dataclass
class MultiHazardParams:
    hazards: list[Literal[
        "climate_dry",          # KNMI Hd
        "climate_moderate",     # KNMI Hn/Ld
        "krw_enforcement",      # KRW compliance restricts land use
        "housing_growth",       # demand growth from housing
        "intake_outage",        # add a specific intake failure
        "datacenter_demand",    # add a large demand source
    ]]
    knmi_preset: str = "Hd"
    time_horizon: int = 2040
    growth_preset: str = "hoog"
    intake_id: str | None = None    # if intake_outage in hazards
    outage_weeks: int | None = None
    development_type: str | None = None
    location_wkt: str | None = None

13.3 Compound risk score formula

Python

def calculate_compound_risk_score(
    individual_gap_m3_per_hazard: dict[str, float],
    capacity_m3: float
) -> dict:
    """
    Combined risk is NOT simply additive — it accounts for correlation.
    Climate + housing growth are correlated in ZH context (both peak in dry summers).
    Correlation factor for climate × housing: 1.15 (conservative uplift).
    Other combinations: 1.0 (independent).
    """
    CORRELATION_FACTORS = {
        frozenset(["climate_dry", "housing_growth"]): 1.15,
        frozenset(["climate_dry", "intake_outage"]): 1.20,
        frozenset(["climate_dry", "krw_enforcement"]): 1.05,
    }

    total_gap = sum(individual_gap_m3_per_hazard.values())
    hazard_set = frozenset(individual_gap_m3_per_hazard.keys())

    # Apply highest applicable correlation factor
    corr_factor = max(
        (v for k, v in CORRELATION_FACTORS.items() if k.issubset(hazard_set)),
        default=1.0
    )

    compound_gap = total_gap * corr_factor
    compound_risk_score = min(1.0, abs(compound_gap) / capacity_m3)

    return {
        "compound_gap_m3":         compound_gap,
        "compound_risk_score":     compound_risk_score,
        "correlation_factor_used": corr_factor,
        "individual_gaps":         individual_gap_m3_per_hazard,
    }

14. Scenario Type 4 — Intervention effectiveness (fully specified; stretch goal)
14.1 What Type 4 is

Type 4 answers: "If we apply intervention X, how much does it change the outcome? What if we stack X + Y?"

It always derives from a baseline scenario (Type 1 or Type 3).
14.2 Intervention composition (stacking multiple interventions)

Python

def apply_stacked_interventions(
    baseline_gap_m3: float,
    baseline_demand_m3: float,
    interventions: list[str],
) -> dict:
    """
    Apply multiple interventions in order (sorted by implementation_years ASC).
    Each intervention reduces the gap sequentially.
    Returns gap after each step and final stacked result.
    """
    from src.backend.calculation.interventions import INTERVENTION_CATALOGUE

    # Sort by implementation time
    sorted_interventions = sorted(
        [(k, INTERVENTION_CATALOGUE[k]) for k in interventions if k in INTERVENTION_CATALOGUE],
        key=lambda x: x[1]["implementation_years"]
    )

    running_gap = baseline_gap_m3
    steps = []

    for key, intervention in sorted_interventions:
        if intervention.get("supply_delta_m3_day"):
            delta = intervention["supply_delta_m3_day"]
        elif intervention.get("supply_delta_fraction"):
            delta = abs(intervention["supply_delta_fraction"]) * baseline_demand_m3
        else:
            delta = 0

        running_gap = running_gap + delta
        steps.append({
            "intervention_key":   key,
            "label_nl":           intervention["label_nl"],
            "delta_m3":           delta,
            "gap_after_m3":       running_gap,
            "feasibility_after":  "GO" if running_gap >= 0 else "CAUTION" if running_gap >= -0.05 * baseline_demand_m3 else "STOP",
            "year_effective":     2024 + intervention["implementation_years"],
        })

    return {
        "baseline_gap_m3":      baseline_gap_m3,
        "final_gap_m3":         running_gap,
        "intervention_steps":   steps,
        "final_feasibility":    steps[-1]["feasibility_after"] if steps else "STOP",
    }

15. Frontend state management
15.1 useScenarioStore.ts

TypeScript

// src/frontend/src/stores/useScenarioStore.ts
import { defineStore } from 'pinia'
import type { ScenarioCard, ScenarioParams, ScenarioDelta } from '@/types/scenario'

export const useScenarioStore = defineStore('scenario', {
  state: () => ({
    // Current scenario
    currentCard:       null as ScenarioCard | null,
    comparisonCard:    null as ScenarioCard | null,
    delta:             null as ScenarioDelta | null,

    // UI state
    isLoading:         false,
    loadingStep:       '' as string,
    error:             null as string | null,
    followupQuestion:  null as string | null,

    // Confirmed params (after user confirmation)
    confirmedParamsA:  null as ScenarioParams | null,
    confirmedParamsB:  null as ScenarioParams | null,

    // Assumption overrides (from sliders)
    assumptionOverrides: {} as Record<string, number>,

    // Cache metadata
    cacheHit:          false,
    cachedAt:          null as string | null,
    datasetDriftWarning: null as string | null,

    // History
    scenarioHistory:   [] as ScenarioCard[],
  }),

  actions: {
    async runScenario(question: string, paramsA: ScenarioParams, paramsB?: ScenarioParams) {
      this.isLoading = true
      this.error = null
      this.currentCard = null
      this.comparisonCard = null
      this.delta = null

      const { startSSE } = useScenarioSSE()
      await startSSE({
        question,
        scenario_a: paramsA,
        scenario_b: paramsB ?? null,
        assumption_overrides: Object.keys(this.assumptionOverrides).length > 0
          ? this.assumptionOverrides
          : null,
        user_persona: useUserStore().persona,
      })
    },

    async rerunWithOverrides() {
      if (!this.confirmedParamsA) return
      await this.runScenario(
        this.currentCard?.question_nl ?? '',
        this.confirmedParamsA,
        this.confirmedParamsB ?? undefined
      )
    },

    setScenarioCard(card: ScenarioCard) {
      this.currentCard = card
      this.scenarioHistory.unshift(card)
      if (this.scenarioHistory.length > 20) this.scenarioHistory.pop()
    },

    setComparisonCard(card: ScenarioCard) {
      this.comparisonCard = card
    },

    updateAssumptionOverride(key: string, value: number) {
      this.assumptionOverrides[key] = value
    },

    resetOverrides() {
      this.assumptionOverrides = {}
    },
  }
})

15.2 useScenarioSSE.ts (SSE → store wiring)

TypeScript

// src/frontend/src/composables/useScenarioSSE.ts
import { useScenarioStore } from '@/stores/useScenarioStore'
import { useMapStore } from '@/stores/useMapStore'
import { useInsightStore } from '@/stores/useInsightStore'

export function useScenarioSSE() {
  const scenarioStore = useScenarioStore()
  const mapStore = useMapStore()
  const insightStore = useInsightStore()

  async function startSSE(payload: ScenarioRunRequest) {
    const response = await fetch('/scenario/run', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    })

    const reader = response.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const lines = decoder.decode(value).split('\n')
      for (const line of lines) {
        if (!line.startsWith('data: ')) continue
        const event = JSON.parse(line.slice(6))
        handleEvent(event)
      }
    }

    scenarioStore.isLoading = false
  }

  function handleEvent(event: SSEEvent) {
    switch (event.type) {

      case 'scenario_params_confirmed':
        scenarioStore.loadingStep = 'Parameters bevestigd'
        scenarioStore.confirmedParamsA = event.data.params_a
        break

      case 'data_loaded':
        scenarioStore.loadingStep = 'Data geladen'
        if (event.data.freshness_warnings?.length > 0) {
          scenarioStore.datasetDriftWarning = event.data.freshness_warnings.join('; ')
        }
        break

      case 'cache_hit':
        scenarioStore.cacheHit = true
        scenarioStore.cachedAt = event.data.cached_at
        scenarioStore.loadingStep = 'Resultaat uit cache'
        break

      case 'waterinfo_status':
        // Show WaterinfoBanner if warning present
        if (event.data.warning) {
          useWaterinfoStore().setWarning(event.data.warning)
        }
        break

      case 'reasoning_step':
        insightStore.addStep(event.data)
        scenarioStore.loadingStep = event.data.label_nl
        break

      case 'feasibility_class':
        scenarioStore.loadingStep = `Haalbaarheid: ${event.data.feasibility_class}`
        break

      case 'cumulative_load_warning':
        scenarioStore.currentCard = {
          ...scenarioStore.currentCard,
          cumulativeWarning: event.data.warning_nl,
        } as any
        break

      case 'scenario_card':
        if (!scenarioStore.currentCard) {
          scenarioStore.setScenarioCard(event.data)
        } else {
          scenarioStore.setComparisonCard(event.data)
        }
        break

      case 'map_data':
        mapStore.setOverlays(event.data)
        break

      case 'scenario_delta':
        scenarioStore.delta = event.data
        break

      case 'followup_question':
        scenarioStore.followupQuestion = event.data.question_nl
        scenarioStore.isLoading = false
        break

      case 'error':
        scenarioStore.error = event.data.message_nl
        scenarioStore.isLoading = false
        break

      case 'done':
        scenarioStore.isLoading = false
        break
    }
  }

  return { startSSE }
}

15.3 useAssumptionStore.ts

TypeScript

// src/frontend/src/stores/useAssumptionStore.ts
import { defineStore } from 'pinia'
import type { Assumption } from '@/types/scenario'

export const useAssumptionStore = defineStore('assumptions', {
  state: () => ({
    assumptions:  [] as Assumption[],
    overrides:    {} as Record<string, number>,
    isDirty:      false,   // true when sliders have changed since last run
  }),

  actions: {
    loadAssumptions(assumptions: Assumption[]) {
      this.assumptions = assumptions
      this.overrides = {}
      this.isDirty = false
    },

    updateSlider(key: string, value: number) {
      this.overrides[key] = value
      this.isDirty = true
      useScenarioStore().updateAssumptionOverride(key, value)
    },

    resetToDefaults() {
      this.overrides = {}
      this.isDirty = false
      useScenarioStore().resetOverrides()
    },
  },

  getters: {
    effectiveValue: (state) => (key: string): number => {
      if (state.overrides[key] !== undefined) return state.overrides[key]
      const assumption = state.assumptions.find(a => a.key === key)
      return assumption?.value ?? 0
    },
  }
})

15.4 useKennisbasisStore.ts

TypeScript

// src/frontend/src/stores/useKennisbasisStore.ts
import { defineStore } from 'pinia'

export const useKennisbasisStore = defineStore('kennisbasis', {
  state: () => ({
    datasets:      [] as DatasetStatus[],
    lastFetched:   null as string | null,
    queryResult:   null as string | null,
    isQuerying:    false,
  }),

  actions: {
    async fetchStatus() {
      const res = await fetch('/kennisbasis/status')
      const data = await res.json()
      this.datasets = data.datasets
      this.lastFetched = new Date().toISOString()
    },

    async query(q: string) {
      this.isQuerying = true
      this.queryResult = null
      const res = await fetch(`/kennisbasis/query?q=${encodeURIComponent(q)}`)
      const data = await res.json()
      this.queryResult = data.answer_nl
      this.isQuerying = false
    },
  }
})

16. Vue component specifications
16.1 FeasibilityBadge.vue

vue

<!-- Props -->
<script setup lang="ts">
defineProps<{
  feasibilityClass:   'GO' | 'CAUTION' | 'STOP'
  gap_m3:             number
  humanScaleRef:      HumanScaleRef | null
  showMakeItFeasible: boolean
}>()
</script>

<!-- Emits -->
<!-- @click:make-feasible — open MakeItFeasiblePanel -->

<!-- Visual contract:
  GO:      🟢 background-color: #d4edda; label: "HAALBAAR"
  CAUTION: 🟠 background-color: #fff3cd; label: "RISICO"
  STOP:    🔴 background-color: #f8d7da; label: "NIET HAALBAAR"
  Always shows: FeasibilityClass label + gap in m³/dag + HumanScaleRef analogy
  If STOP or CAUTION: "▼ Wat maakt dit wél haalbaar?" link
-->

16.2 AssumptionSliders.vue

vue

<script setup lang="ts">
defineProps<{
  assumptions:  Assumption[]
  overrides:    Record<string, number>
}>()

defineEmits<{
  (e: 'update:override', key: string, value: number): void
  (e: 'rerun'): void
  (e: 'reset'): void
}>()
</script>

<!-- Visual contract:
  Each assumption renders as:
  [label_nl]  [slider: min→max, step]  [current_value unit]  [source_label → URL]
  Label: "beleidsmatige schatting" badge if is_policy_approx
  Sensitivity badge: "Hoge invloed" (high), "Gemiddelde invloed" (medium), "Lage invloed" (low)
  After any slider change: "Herbereken scenario" button appears (isDirty = true)
  "Herstel standaardwaarden" link resets all to defaults
-->

16.3 MakeItFeasiblePanel.vue

vue

<script setup lang="ts">
defineProps<{
  interventions:       RankedIntervention[]
  currentGap_m3:       number
  baselineDemand_m3:   number
}>()
</script>

<!-- Visual contract:
  Panel header: "Wat maakt dit project haalbaar?"
  For each of top 3 interventions:
    [rank] [label_nl]
    Gap closure: X m³/dag (Y%)
    Na ingreep: [FeasibilityBadge for post_intervention_feasibility]
    Kosten: €X–Y miljoen
    Uitvoeringstijd: Z jaar
    Bron: [source_label → source_url]
  Footer: "Bron: [source_label]" for each intervention
-->

16.4 CitationBlock.vue

vue

<script setup lang="ts">
defineProps<{
  scenarioId:       string
  createdAt:        string
  datasetVersions:  Record<string, string>
  gitCommit:        string
  stableUrl:        string
  citationApa:      string
  citationApaNl:    string
}>()

defineEmits<{
  (e: 'copy-citation'): void
  (e: 'download-pdf'): void
  (e: 'share-url'): void
}>()
</script>

<!-- Visual contract:
  Collapsible section: "📎 Citaat voor gebruik in adviesdocumenten"
  Shows: Scenario-ID, Gegenereerd (NL datetime), Dataversies, Software @ git commit
  APA-stijl citaatstring (Dutch)
  Three buttons: [📋 Kopieer citaat] [📄 Download PDF] [🔗 Deel URL]
-->

16.5 KennisbasisPanel.vue

vue

<script setup lang="ts">
defineProps<{
  datasets:    DatasetStatus[]
  lastFetched: string | null
}>()

defineEmits<{
  (e: 'query', q: string): void
}>()
</script>

<!-- Visual contract:
  Always-visible panel (sidebar or tab)
  Table: Thema | Tabellen | Bijgewerkt | Bron | Status
  Status: ✅ Actueel | ⚠️ Ouder dan 90 dagen | ❓ Onbekend
  Search input: "Welke data heeft dit systeem over..."
  Result: plain Dutch answer from /kennisbasis/query endpoint
-->

16.6 MapTitle.vue

vue

<script setup lang="ts">
defineProps<{
  metricLabel:    string    // e.g. "Aanvoertekort in m³/dag per leveringszone"
  knmiPreset:     string
  timeHorizon:    number
  intakeName:     string | null
  isComparison:   boolean
  comparisonSide: 'A' | 'B' | null
}>()
</script>

<!-- Visual contract:
  Fixed overlay on map top-left
  Text: "🗺️ Huidige kaart: {metricLabel} · KNMI {knmiPreset} · {timeHorizon}"
  If intake: append "· Inname: {intakeName}"
  If comparison: show "Scenario A:" or "Scenario B:" prefix
  Updates reactively — never blank
  Background: semi-transparent white, always readable
-->

16.7 CitizenResponseCard.vue

vue

<script setup lang="ts">
defineProps<{
  postcode:         string
  zoneName:         string
  verdict:          string          // one-sentence Dutch verdict
  uncertaintyNote:  string | null
  waterCompany:     { name: string; url: string }
  officialLinks:    Array<{ label: string; url: string }>
  disclaimer:       string
  timeHorizon:      number
}>()
</script>

<!-- Visual contract:
  Simple card, no numbers visible unless user clicks "Meer informatie"
  Large verdict text at top
  "Uw drinkwaterbedrijf:" with link
  "Officiële informatie:" with 2-3 links
  Small disclaimer in grey
  "Meer weten over dit scenario?" expansion link → shows FeasibilityBadge + assumptions
-->

16.8 ProductionChainFlow.vue

vue

<script setup lang="ts">
defineProps<{
  chainNodes: ProductionChainNode[]
  chainEdges: ProductionChainEdge[]
  outageIntakeId: string | null
}>()
</script>

<!-- Visual contract:
  Directed flow graph (SVG or canvas)
  Node types: intake (blue), production (green), distribution (grey), zone (light grey)
  Edge thickness proportional to capacity_m3_day
  Edge color: green = operational, orange = reduced, red = down
  If outageIntakeId: intake node and downstream edges colored red
  Hover: shows capacity label
  Status badge below intake node if outage: "🔴 Inname gestopt (KNMI Hd week 4)"
-->

16.9 WaterinfoBanner.vue

vue

<script setup lang="ts">
defineProps<{
  isLive:       boolean
  cachedDate:   string | null
  warning:      string | null
  intakeName:   string
}>()
</script>

<!-- Visual contract:
  If isLive: 🟢 "Chloridedata: live van Waterinfo ({intakeName})"
  If cached: 🟠 "⚠️ Chloridedata: Waterinfo niet beschikbaar.
                  Laatste bekende waarde van {cachedDate}."
  Always visible when intake scenario is active
  Cannot be dismissed (it's data quality information, not a notification)
-->

17. Map layer configuration
17.1 Layer z-order and defaults

TypeScript

// src/frontend/src/composables/useMapOverlays.ts

export const LAYER_CONFIG = {
  // Base layers (always on, z-order 100–109)
  gemeentegrenzen: {
    zIndex: 100, opacity: 0.6, fillOpacity: 0, style: { color: '#666', weight: 1 }
  },
  waterschapsgrenzen: {
    zIndex: 101, opacity: 0.7, fillOpacity: 0, style: { color: '#3388ff', weight: 1.5 }
  },
  leveringszones: {
    zIndex: 102, opacity: 0.5, fillOpacity: 0.1, style: { color: '#0077cc', weight: 2 }
  },
  winlocaties: {
    zIndex: 103, markerType: 'circle', radius: 8,
    style: { color: '#003399', fillColor: '#3399ff', fillOpacity: 0.9 }
  },
  zes_uur_zones: {
    zIndex: 104, opacity: 0.4, fillOpacity: 0.05,
    style: { color: '#ff9900', weight: 1.5, dashArray: '5,5' }
  },

  // Scenario overlays (z-order 200–209)
  supply_gap_heatmap: {
    zIndex: 200, opacity: 0.7, fillOpacity: 0.5,
    colorScale: { GO: '#28a745', CAUTION: '#ffc107', STOP: '#dc3545' }
  },
  chloride_risk_zones: {
    zIndex: 201, opacity: 0.6, fillOpacity: 0.4,
    colorScale: { low: '#ffffcc', medium: '#fd8d3c', high: '#bd0026' }
  },
  drop_pin_marker: {
    zIndex: 202, markerType: 'marker',
    icon: 'custom-drop-pin'   // red pin with FeasibilityClass color
  },
  woningbouwlocaties_overlay: {
    zIndex: 203, opacity: 0.7, fillOpacity: 0.3,
    style: { color: '#8B4513', fillColor: '#DEB887' }
  },
  krw_risk_overlay: {
    zIndex: 204, opacity: 0.8, fillOpacity: 0.4,
    style: { color: '#7CFC00', fillColor: '#ADFF2F' }
  },

  // Comparison delta layer (z-order 300)
  delta_layer: {
    zIndex: 300, opacity: 0.7, fillOpacity: 0.5,
    colorScale: { positive: '#28a745', zero: '#ffffff', negative: '#dc3545' }
  },
}

// Stakeholder → layers to highlight
export const STAKEHOLDER_OVERLAY_MAP: Record<string, string[]> = {
  woningzoekenden:    ['woningbouwlocaties_overlay', 'supply_gap_heatmap', 'leveringszones'],
  drinkwaterbedrijf:  ['leveringszones', 'winlocaties', 'zes_uur_zones'],
  natuur_krw:         ['krw_risk_overlay', 'leveringszones'],
  landbouw:           ['lgn_landgebruik', 'chloride_risk_zones'],
  zorg_kritisch:      ['leveringszones', 'supply_gap_heatmap'],
  gemeente:           ['gemeentegrenzen', 'woningbouwlocaties_overlay', 'supply_gap_heatmap'],
}

18. Trust primitives & institutional UX (all 10 stakeholder gaps)
GAP 1 — Kennisbasis panel: "what does this system know?"

Problem: Users don't know what data is loaded before asking a question. Trust starts before the first query.

Implementation:

    Permanent tab/sidebar panel accessible at all times
    Powered by /kennisbasis/status endpoint returning table inventory
    Queryable: "Welke data heeft dit systeem over woningbouw in Pijnacker?" routes to /kennisbasis/query
    Shows: dataset name, number of tables/rows, last_modified, source, freshness indicator

Visual spec: (see KennisbasisPanel.vue in section 16.5)
GAP 2 — Shared scenario library + stable URLs + version-drift warning

Problem: Two colleagues ask the same question and get different answers due to LLM variability or dataset updates.

Implementation in scenario_hash.py:

Python

import hashlib, json

def compute_scenario_hash(params: ScenarioParams, assumption_overrides: dict | None) -> str:
    """
    Deterministic hash of scenario parameters.
    Same params → same hash → same cached result.
    LLM variability is eliminated because the hash is computed BEFORE
    GreenPT generates any prose.
    """
    normalized = {
        "scenario_type":    params.scenario_type,
        "knmi_preset":      params.knmi_preset,
        "time_horizon":     params.time_horizon,
        "growth_preset":    params.growth_preset,
        "location_wkt":     params.location_wkt,
        "development_type": params.development_type,
        "development_mw":   params.development_mw,
        "development_units":params.development_units,
        "intake_id":        params.intake_id,
        "outage_weeks":     params.outage_weeks,
        "interventions":    sorted(params.interventions),
        "overrides":        dict(sorted((assumption_overrides or {}).items())),
    }
    return hashlib.sha256(json.dumps(normalized, sort_keys=True).encode()).hexdigest()[:16]

Scenario cache in scenario_cache.py:

Python

import duckdb, json
from datetime import datetime

class ScenarioCache:
    def __init__(self, conn: duckdb.DuckDBPyConnection):
        self.conn = conn
        conn.execute("""
            CREATE TABLE IF NOT EXISTS scenario_cache (
                scenario_id      TEXT PRIMARY KEY,
                scenario_hash    TEXT UNIQUE,
                params_json      TEXT,
                result_json      TEXT,
                dataset_versions TEXT,
                created_at       TEXT,
                git_commit       TEXT
            )
        """)
        conn.execute("CREATE INDEX IF NOT EXISTS idx_hash ON scenario_cache(scenario_hash)")

    def get(self, scenario_hash: str) -> dict | None:
        row = self.conn.execute(
            "SELECT result_json, dataset_versions, created_at FROM scenario_cache WHERE scenario_hash = ?",
            [scenario_hash]
        ).fetchone()
        if not row:
            return None
        return {
            "card": json.loads(row[0]),
            "dataset_versions_at_cache": json.loads(row[1]),
            "cached_at": row[2],
        }

    def set(self, scenario_hash: str, card: dict, dataset_versions: dict) -> str:
        scenario_id = str(uuid.uuid4())
        self.conn.execute("""
            INSERT OR REPLACE INTO scenario_cache
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, [
            scenario_id,
            scenario_hash,
            json.dumps(card.get("params", {})),
            json.dumps(card),
            json.dumps(dataset_versions),
            datetime.now().isoformat(),
            get_git_commit(),
        ])
        return scenario_id

    def check_version_drift(self, cached_versions: dict, current_versions: dict) -> str | None:
        drifted = [
            t for t in cached_versions
            if current_versions.get(t) != cached_versions[t]
        ]
        if drifted:
            return (f"Let op: de volgende dataset(s) zijn bijgewerkt sinds dit scenario is berekend: "
                    f"{', '.join(drifted)}. Herbereken voor actuele uitkomsten.")
        return None

GAP 3 — Citation block

Implementation in citation.py:

Python

def build_apa_citation(card: ScenarioCard) -> str:
    date_nl = format_date_dutch(card.created_at)
    scenario_title = build_scenario_title(card.params)
    return (
        f"Provincie Zuid-Holland Ruimtelijke Assistent ({card.created_at[:4]}, {date_nl}). "
        f"Scenario: {scenario_title} [{card.scenario_id[:8]}]. "
        f"GovTechNL OneGov #2 Drinkwaterzekerheid Scenario Engine. "
        f"{card.stable_url}"
    )

def build_bibtex_citation(card: ScenarioCard) -> str:
    return f"""@misc{{{card.scenario_id[:8]},
  author    = {{Provincie Zuid-Holland Ruimtelijke Assistent}},
  title     = {{Scenario: {build_scenario_title(card.params)}}},
  year      = {{{card.created_at[:4]}}},
  note      = {{GovTechNL OneGov #2. Dataset versions: {'; '.join(f'{k}={v}' for k, v in card.dataset_versions.items())}}},
  url       = {{{card.stable_url}}}
}}"""

GAP 4 — HumanScaleRef on every headline metric

Implementation: (see human_scale.py in section 11.8)

Applied in format_scenario_output.py:

Python

results.human_scale = build_human_scale_ref(
    metric_key="supply_gap_m3",
    metric_value=results.supply_gap_m3,
    metric_unit="m³/dag",
    demand_per_household=assumption_store.get("demand_per_dwelling_m3_day", 0.35)
)

GAP 5 — "Make it feasible" intervention ranking

Implementation: (see rank_interventions_to_feasibility() in section 11.7)

Called in compute_feasibility_node when feasibility_class in ["STOP", "CAUTION"].
GAP 6 — CumulativeLoadWarning

Implementation in check_cumulative_load.py:

Python

async def check_cumulative_load_node(state: ScenarioAgentState) -> ScenarioAgentState:
    zone_id = state["scenario_a_results"].get("zone_id")
    if not zone_id:
        return state

    # Query scenario cache for all scenarios touching same zone
    active = cache.conn.execute("""
        SELECT scenario_id, json_extract(result_json, '$.results.daily_demand_m3') as demand
        FROM scenario_cache
        WHERE json_extract(result_json, '$.results.zone_id') = ?
        AND scenario_id != ?
        AND created_at >= ?
    """, [
        zone_id,
        state.get("current_scenario_id", ""),
        (datetime.now() - timedelta(hours=24)).isoformat()
    ]).fetchall()

    if active:
        total_other_demand = sum(row[1] or 0 for row in active)
        state["active_scenarios_count"] = len(active)
        state["cumulative_demand_m3"] = (
            state["scenario_a_results"]["daily_demand_m3"] + total_other_demand
        )
        state["cumulative_warning_nl"] = (
            f"Let op: er zijn {len(active)} andere actieve scenario's in leveringszone "
            f"{state['scenario_a_results'].get('zone_naam', zone_id)}. "
            f"Het gecombineerde effect is {total_other_demand + state['scenario_a_results']['daily_demand_m3']:,.0f} m³/dag extra vraag."
        )

    return state

GAP 7 — Per-intake chloride threshold

Implementation: (see get_intake_capacity() in section 11.3 and get_chloride_for_intake() in section 11.2)

The threshold is always read from productieketen.cl_threshold_mg_l per intake. If the column is missing or null, the fallback Assumption kicks in with value=150, range 150–250, and source_url to Drinkwaterbesluit. The UI always shows whether the threshold came from the database or from the fallback assumption.
GAP 8 — Official position panel

Implementation: (see official_positions.py in section 7.9)

Built as a build_official_position_node that calls get_official_position(scenario_type) and attaches to the ScenarioCard. Rendered as OfficialPositionPanel.vue.
GAP 9 — Waterinfo mandatory guard

Implementation in waterinfo.py:

Python

# src/backend/external/waterinfo.py

import httpx
from src.backend.cache.waterinfo_cache import load_chloride_cache, save_chloride_cache
import os

WATERINFO_API_URL = os.getenv("WATERINFO_API_URL")
MANDATORY_INTAKES = ["ijssel", "lek", "maas", "hollandse_ijssel", "nieuwe_maas"]

async def fetch_waterinfo_chloride(intake_id: str) -> dict:
    """
    Fetch current chloride concentration from Rijkswaterstaat Waterinfo.
    MANDATORY for IJssel/Lek/Maas — never silently invents a value.
    Always returns: value, source, date, and optional warning.
    """
    is_mandatory = any(m in intake_id.lower() for m in MANDATORY_INTAKES)

    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get(
                WATERINFO_API_URL,
                params={
                    "locatie": intake_id,
                    "grootheid": "Cl",
                    "groepering": "dag",
                }
            )
            response.raise_for_status()
            data = response.json()
            value = data["WaarnemingenLijst"][0]["MetingenLijst"][-1]["Meetwaarde"]["Waarde_Numeriek"]
            date = data["WaarnemingenLijst"][0]["MetingenLijst"][-1]["Tijdstip"]

            # Update cache
            save_chloride_cache(intake_id, value, date)

            return {
                "value":   value,
                "source":  "Waterinfo (live)",
                "date":    date,
                "warning": None,
            }

    except Exception as e:
        # Try cache
        cached = load_chloride_cache(intake_id)
        if cached:
            warning = (
                f"Waterinfo niet beschikbaar ({type(e).__name__}). "
                f"Laatste bekende chloridewaarde: {cached['value']} mg/L van {cached['date']}. "
                f"Resultaten zijn gebaseerd op deze historische waarde."
            )
            return {
                "value":   cached["value"],
                "source":  "Waterinfo (cache)",
                "date":    cached["date"],
                "warning": warning,
            }
        else:
            # No cache either — this is an error, not a silent fallback
            if is_mandatory:
                raise WaterinfoUnavailableError(
                    f"Waterinfo is niet beschikbaar en er is geen gecachede waarde voor {intake_id}. "
                    f"Voer het scenario uit met een handmatige chloride-aanname via de schuifregelaar."
                )
            return {
                "value":   None,
                "source":  "unavailable",
                "date":    None,
                "warning": f"Chloridedata niet beschikbaar voor {intake_id}.",
            }

GAP 10 — Citizen response template

Detection logic in detect_mode_node:

Python

def detect_citizen_mode(question: str, persona: str) -> bool:
    if persona == "citizen":
        return True
    citizen_indicators = [
        "mijn water", "ons water", "mijn buurt", "mijn postcode",
        "is het veilig", "kan ik", "moet ik", "wat betekent dit voor mij",
        "hoe zit het met mijn"
    ]
    q = question.lower()
    has_personal = any(k in q for k in citizen_indicators)
    has_postcode = bool(re.search(r'\b\d{4}\s?[A-Za-z]{2}\b', question))
    return has_personal or has_postcode

Citizen response formatting: (see CITIZEN_RESPONSE_PROMPT_TEMPLATE in section 10.5 and CitizenResponseCard.vue in section 16.7)
19. Traceability framework
19.1 Three-layer traceability matrix
Layer	Tool	Content	Primary audience
Human	Insight panel (Vue)	Step-by-step B1 Dutch reasoning with hyperlinks	Policymakers, spatial planners
Machine	MLflow	Params, metrics, artifacts, Flesch-Douma, dataset versions, per-node latency	Developers, Woo auditors
Citeable	ScenarioCard	scenario_id, hash, stable URL, citation string, git commit, dataset versions	Legal/admin use, advice letters



19.2 GAP 2 — Shared scenario library, stable URLs, dataset version tracking & "same question = same answer"
Problem

Colleagues working in different sessions get different answers to the same policy question due to LLM temperature variability and dataset drift. There is no way to share a scenario artifact with a stable reference, and no warning when underlying data changes after a scenario was computed.
Implementation
Scenario hash computation (src/backend/utils/scenario_hash.py)

Python

import hashlib
import json
from src.backend.models.scenario import ScenarioParams

def compute_scenario_hash(params: ScenarioParams) -> str:
    """
    Deterministic SHA-256 hash of normalized scenario parameters.
    Same params → same hash → same cached result.
    Normalization: sort all dict keys, lowercase strings, 
    round floats to 4 decimal places.
    """
    def normalize(obj):
        if isinstance(obj, dict):
            return {k: normalize(v) for k, v in sorted(obj.items())}
        elif isinstance(obj, list):
            return [normalize(i) for i in obj]
        elif isinstance(obj, float):
            return round(obj, 4)
        elif isinstance(obj, str):
            return obj.lower().strip()
        return obj

    normalized = normalize(params.__dict__)
    serialized = json.dumps(normalized, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(serialized.encode("utf-8")).hexdigest()[:32]

DuckDB scenario store schema (src/backend/cache/scenario_store.py)

Python

CREATE_SCENARIOS_TABLE = """
CREATE TABLE IF NOT EXISTS scenarios (
    scenario_id     VARCHAR PRIMARY KEY,
    scenario_hash   VARCHAR NOT NULL,
    params_json     JSON NOT NULL,
    result_json     JSON NOT NULL,
    dataset_versions JSON NOT NULL,   -- {table_name: last_modified_iso}
    git_commit      VARCHAR NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    stable_url      VARCHAR NOT NULL,
    is_citizen_mode BOOLEAN DEFAULT FALSE
);
CREATE INDEX IF NOT EXISTS idx_scenario_hash ON scenarios(scenario_hash);
"""

class ScenarioStore:
    def __init__(self, conn):
        self.conn = conn
        conn.execute(CREATE_SCENARIOS_TABLE)

    def get_by_hash(self, scenario_hash: str) -> dict | None:
        row = self.conn.execute(
            "SELECT result_json, dataset_versions, created_at, scenario_id, stable_url "
            "FROM scenarios WHERE scenario_hash = ? ORDER BY created_at DESC LIMIT 1",
            [scenario_hash]
        ).fetchone()
        if row:
            return {
                "result_json": row[0],
                "dataset_versions_at_cache": row[1],
                "cached_at": row[2],
                "scenario_id": row[3],
                "stable_url": row[4]
            }
        return None

    def get_by_id(self, scenario_id: str) -> dict | None:
        row = self.conn.execute(
            "SELECT result_json, dataset_versions, created_at, stable_url "
            "FROM scenarios WHERE scenario_id = ?",
            [scenario_id]
        ).fetchone()
        if row:
            return {
                "result_json": row[0],
                "dataset_versions_at_cache": row[1],
                "cached_at": row[2],
                "stable_url": row[3]
            }
        return None

    def set(
        self,
        scenario_id: str,
        scenario_hash: str,
        params: dict,
        result: dict,
        dataset_versions: dict,
        git_commit: str,
        stable_url_base: str,
        is_citizen_mode: bool = False
    ):
        stable_url = f"{stable_url_base}/scenario/{scenario_id}"
        self.conn.execute(
            """INSERT INTO scenarios 
               (scenario_id, scenario_hash, params_json, result_json,
                dataset_versions, git_commit, stable_url, is_citizen_mode)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?)
               ON CONFLICT (scenario_id) DO NOTHING""",
            [scenario_id, scenario_hash, json.dumps(params),
             json.dumps(result), json.dumps(dataset_versions),
             git_commit, stable_url, is_citizen_mode]
        )

Dataset version drift detection (src/backend/utils/version_drift.py)

Python

def detect_version_drift(
    dataset_versions_at_cache: dict,
    current_dataset_versions: dict
) -> list[dict]:
    """
    Compare dataset versions at cache time vs now.
    Returns list of drifted tables with old/new timestamps.
    """
    drifted = []
    for table, cached_version in dataset_versions_at_cache.items():
        current = current_dataset_versions.get(table)
        if current and current != cached_version:
            drifted.append({
                "table": table,
                "cached_version": cached_version,
                "current_version": current,
                "warning_nl": (
                    f"Let op: tabel '{table}' is bijgewerkt sinds dit scenario "
                    f"werd berekend ({cached_version} → {current}). "
                    f"Herbereken voor actuele uitkomst."
                )
            })
    return drifted

GET endpoint for stable scenario retrieval (src/backend/routers/scenario_router.py)

Python

@router.get("/scenario/{scenario_id}")
async def get_scenario_by_id(scenario_id: str):
    """
    Stable URL for a cached scenario artifact.
    Always returns the same ScenarioCard for the same scenario_id.
    Adds version drift warning if datasets have changed since caching.
    """
    cached = scenario_store.get_by_id(scenario_id)
    if not cached:
        raise HTTPException(status_code=404, detail="Scenario niet gevonden.")
    
    current_versions = get_current_dataset_versions(conn)
    drift = detect_version_drift(
        cached["dataset_versions_at_cache"],
        current_versions
    )
    result = json.loads(cached["result_json"])
    result["cache_used"] = True
    result["cached_at"] = cached["cached_at"].isoformat()
    result["dataset_version_drift"] = drift
    return result

"Verify calculation" button handler

Python

@router.post("/scenario/{scenario_id}/verify")
async def verify_scenario(scenario_id: str):
    """
    Re-runs the scenario with the same params but current dataset versions.
    Returns new ScenarioCard + diff from cached version.
    """
    cached = scenario_store.get_by_id(scenario_id)
    if not cached:
        raise HTTPException(status_code=404, detail="Scenario niet gevonden.")
    
    original_params = ScenarioParams(**json.loads(cached["result_json"])["params"])
    
    # Re-run via the same pipeline
    new_card = await run_scenario_pipeline(original_params)
    
    # Compute diff
    original_results = json.loads(cached["result_json"])["results"]
    diff = {
        "supply_gap_delta": new_card.results.supply_gap_m3 - original_results["supply_gap_m3"],
        "feasibility_changed": (
            new_card.results.feasibility_class != original_results["feasibility_class"]
        ),
        "onset_year_delta": (
            (new_card.results.onset_year or 0) - (original_results.get("onset_year") or 0)
        ),
        "recalculated_at": datetime.utcnow().isoformat(),
        "summary_nl": _build_verify_summary_nl(original_results, new_card.results)
    }
    return {"new_card": new_card, "diff_from_cached": diff}

Frontend: ScenarioSharePanel.vue

vue

<template>
  <div class="scenario-share-panel">
    <div class="share-row">
      <span class="label">Stabiele URL:</span>
      <a :href="stableUrl" target="_blank">{{ stableUrl }}</a>
      <button @click="copyUrl">📋 Kopieer</button>
    </div>
    <div class="share-row">
      <span class="label">Scenario-ID:</span>
      <code>{{ scenarioId }}</code>
    </div>
    <div v-if="driftWarnings.length > 0" class="drift-warnings">
      <div
        v-for="w in driftWarnings"
        :key="w.table"
        class="drift-warning-banner"
      >
        ⚠️ {{ w.warning_nl }}
        <button @click="verifyScenario">Herbereken</button>
      </div>
    </div>
    <button
      v-if="driftWarnings.length === 0"
      @click="verifyScenario"
      class="verify-btn"
    >
      🔄 Herbereken met actuele data
    </button>
  </div>
</template>

19.3 GAP 3 — Citation block: APA-format citation, stable URL, dataset versions, PDF export
Problem

When a policy officer cites scenario output in an advice letter or memo, there is no standard citation string, no PDF export, and no way to verify which data version was used.
Implementation
Citation builder (src/backend/utils/citation_builder.py)

Python

from datetime import datetime
from src.backend.models.scenario import ScenarioCard

def build_citation(card: ScenarioCard) -> dict:
    """
    Builds APA-style citation strings (English and Dutch) for a ScenarioCard.
    """
    date_nl = datetime.fromisoformat(card.created_at).strftime("%-d %B %Y")
    date_en = datetime.fromisoformat(card.created_at).strftime("%B %-d, %Y")
    short_id = card.scenario_id[:8]
    scenario_label = _build_scenario_label_nl(card.params)

    apa_nl = (
        f"Provincie Zuid-Holland Ruimtelijke Assistent. "
        f"({date_nl}). "
        f"Scenario: {scenario_label} "
        f"[Scenario-ID: {short_id}]. "
        f"GovTechNL OneGov #2 — Drinkwaterzekerheid. "
        f"Ophaalbaar via: {card.stable_url}"
    )

    apa_en = (
        f"Province of Zuid-Holland Spatial Assistant. "
        f"({date_en}). "
        f"Scenario: {scenario_label} "
        f"[Scenario ID: {short_id}]. "
        f"GovTechNL OneGov #2 — Drinking Water Security. "
        f"Retrieved from: {card.stable_url}"
    )

    metadata_block = (
        f"Scenario-ID: {card.scenario_id}\n"
        f"Scenario-hash: {card.scenario_hash}\n"
        f"Gegenereerd: {date_nl}\n"
        f"Software: onegov2-spatial-assistant @ {card.git_commit}\n"
        f"Dataversies:\n" +
        "\n".join(
            f"  - {tbl}: {ver}"
            for tbl, ver in card.dataset_versions.items()
        )
    )

    return {
        "apa_nl": apa_nl,
        "apa_en": apa_en,
        "metadata_block": metadata_block,
        "stable_url": card.stable_url,
        "scenario_id": card.scenario_id,
        "git_commit": card.git_commit,
        "dataset_versions": card.dataset_versions,
        "generated_at": card.created_at
    }

def _build_scenario_label_nl(params) -> str:
    """Builds a human-readable Dutch scenario label from params."""
    type_labels = {
        "drop_pin": "Ruimtelijke vraag",
        "intake_failure": "Inname-uitval",
        "multi_hazard": "Gecombineerd risico",
        "intervention": "Interventiescenario"
    }
    base = type_labels.get(params.scenario_type, params.scenario_type)
    parts = [base]
    if params.knmi_preset and params.knmi_preset != "B":
        parts.append(f"KNMI {params.knmi_preset}")
    if params.time_horizon:
        parts.append(str(params.time_horizon))
    if params.intake_id:
        parts.append(params.intake_id.replace("_", " ").title())
    return ", ".join(parts)

PDF export endpoint (src/backend/routers/scenario_router.py)

Python

from fpdf import FPDF
import io

@router.get("/scenario/{scenario_id}/pdf")
async def export_scenario_pdf(scenario_id: str):
    """
    Generates a PDF export of the ScenarioCard suitable for
    attachment to an official advice letter (adviesnota).
    """
    cached = scenario_store.get_by_id(scenario_id)
    if not cached:
        raise HTTPException(status_code=404, detail="Scenario niet gevonden.")

    card_dict = json.loads(cached["result_json"])
    citation = build_citation_from_dict(card_dict)

    pdf = FPDF()
    pdf.add_page()
    pdf.set_auto_page_break(auto=True, margin=15)

    # Header
    pdf.set_font("Helvetica", "B", 16)
    pdf.cell(0, 10, "Scenario-uitvoer — Drinkwaterzekerheid ZH", ln=True)
    pdf.set_font("Helvetica", "", 10)
    pdf.cell(0, 6, f"Gegenereerd: {card_dict['created_at']}", ln=True)
    pdf.cell(0, 6, f"Scenario-ID: {card_dict['scenario_id']}", ln=True)
    pdf.ln(4)

    # Feasibility
    pdf.set_font("Helvetica", "B", 13)
    fc = card_dict["results"]["feasibility_class"]
    fc_label = {"GO": "HAALBAAR", "CAUTION": "RISICO", "STOP": "NIET HAALBAAR"}.get(fc, fc)
    pdf.cell(0, 8, f"Haalbaarheid: {fc_label}", ln=True)
    pdf.ln(2)

    # Key results
    pdf.set_font("Helvetica", "B", 11)
    pdf.cell(0, 7, "Kernresultaten", ln=True)
    pdf.set_font("Helvetica", "", 10)
    results = card_dict["results"]
    pdf.multi_cell(0, 6,
        f"Dagelijkse vraag: {results['daily_demand_m3']:,.0f} m³/dag "
        f"({results.get('human_scale', {}).get('analogy_nl', '')})"
    )
    pdf.multi_cell(0, 6,
        f"Beschikbare capaciteit: {results['supply_capacity_m3']:,.0f} m³/dag"
    )
    gap = results["supply_gap_m3"]
    gap_label = f"Tekort: {abs(gap):,.0f} m³/dag" if gap < 0 else f"Overschot: {gap:,.0f} m³/dag"
    pdf.multi_cell(0, 6, gap_label)
    if results.get("onset_year"):
        pdf.multi_cell(0, 6, f"Verwacht aanvangsjaar tekort: {results['onset_year']}")
    pdf.ln(3)

    # Assumptions
    pdf.set_font("Helvetica", "B", 11)
    pdf.cell(0, 7, "Aannames", ln=True)
    pdf.set_font("Helvetica", "", 9)
    for assumption in card_dict.get("assumptions", []):
        pdf.multi_cell(0, 5,
            f"• {assumption['label_nl']}: {assumption['value']} {assumption['unit']} "
            f"(bandbreedte: {assumption['value_min']}–{assumption['value_max']}) "
            f"— {assumption['source_label']}"
        )
    pdf.ln(3)

    # Reasoning steps
    pdf.set_font("Helvetica", "B", 11)
    pdf.cell(0, 7, "Redeneerproces", ln=True)
    pdf.set_font("Helvetica", "", 9)
    for step in card_dict.get("reasoning_steps", []):
        pdf.multi_cell(0, 5, f"Stap {step['step_nr']}: {step['label_nl']}")
        pdf.multi_cell(0, 5, f"  {step['description_nl']}")
    pdf.ln(3)

    # Citation block
    pdf.set_font("Helvetica", "B", 11)
    pdf.cell(0, 7, "Citaat", ln=True)
    pdf.set_font("Helvetica", "", 9)
    pdf.multi_cell(0, 5, citation["apa_nl"])
    pdf.ln(2)
    pdf.multi_cell(0, 5, citation["metadata_block"])

    # Disclaimer
    pdf.ln(4)
    pdf.set_font("Helvetica", "I", 8)
    pdf.multi_cell(0, 5,
        "Dit scenario is een beleidsmatige verkenning gegenereerd door de OneGov #2 "
        "Ruimtelijke Assistent. Het is geen officiële meting of besluit. "
        "Alle aannames en bronnen zijn vermeld. Voor officieel beleid, "
        "zie het Regionaal Waterprogramma Zuid-Holland 2022–2027."
    )

    pdf_bytes = pdf.output(dest="S").encode("latin-1")
    return Response(
        content=pdf_bytes,
        media_type="application/pdf",
        headers={
            "Content-Disposition": f"attachment; filename=scenario_{scenario_id[:8]}.pdf"
        }
    )

Frontend: CitationBlock.vue

vue

<template>
  <div class="citation-block">
    <h4>📎 Citaat voor gebruik in adviesdocumenten</h4>
    <div class="citation-meta">
      <div class="meta-row">
        <span class="meta-label">Scenario-ID:</span>
        <code>{{ citation.scenario_id }}</code>
      </div>
      <div class="meta-row">
        <span class="meta-label">Gegenereerd:</span>
        <span>{{ formatDateNl(citation.generated_at) }}</span>
      </div>
      <div class="meta-row">
        <span class="meta-label">Software:</span>
        <code>onegov2-spatial-assistant @ {{ citation.git_commit }}</code>
      </div>
      <div class="meta-row">
        <span class="meta-label">Dataversies:</span>
        <ul class="version-list">
          <li v-for="(ver, tbl) in citation.dataset_versions" :key="tbl">
            <strong>{{ tbl }}</strong>: {{ ver }}
          </li>
        </ul>
      </div>
      <div class="meta-row">
        <span class="meta-label">Stabiele URL:</span>
        <a :href="citation.stable_url" target="_blank">{{ citation.stable_url }}</a>
      </div>
    </div>
    <div class="citation-text">
      <p class="citation-apa">{{ citation.apa_nl }}</p>
    </div>
    <div class="citation-actions">
      <button @click="copyApa">📋 Kopieer citaat</button>
      <a :href="pdfUrl" target="_blank">
        <button>📄 Download PDF</button>
      </a>
      <button @click="copyUrl">🔗 Deel URL</button>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  citation: { type: Object, required: true },
  scenarioId: { type: String, required: true }
})

const pdfUrl = computed(() =>
  `/scenario/${props.scenarioId}/pdf`
)

const copyApa = () =>
  navigator.clipboard.writeText(props.citation.apa_nl)

const copyUrl = () =>
  navigator.clipboard.writeText(props.citation.stable_url)

const formatDateNl = (iso) => {
  const d = new Date(iso)
  return d.toLocaleDateString('nl-NL', {
    day: 'numeric', month: 'long', year: 'numeric',
    hour: '2-digit', minute: '2-digit'
  })
}
</script>

19.4 GAP 4 — Human-scale contextualization of every headline metric
Problem

"169,000 m³/dag" is meaningless to a policymaker, spatial planner, or citizen. Every headline metric must be accompanied by a real-world analogy with a cited conversion reference.
Implementation
Human-scale converter (src/backend/utils/human_scale.py)

Python

from dataclasses import dataclass
from typing import Optional

# Conversion constants with authoritative sources
VEWIN_PERSON_DAY_M3 = 0.119      # m³/person/day (VEWIN Waterstatistiek)
VEWIN_HOUSEHOLD_DAY_M3 = 0.35    # m³/household/day (VEWIN, CBS gemiddeld)
CBS_AVG_HOUSEHOLD_SIZE = 2.2     # persons/household (CBS 2023)
VEWIN_SOURCE_URL = "https://www.vewin.nl/publicaties/waterstatistiek"
CBS_SOURCE_URL = "https://www.cbs.nl/nl-nl/maatwerk/2023/huishoudensgrootte"

METRIC_CONVERSIONS = {
    "daily_demand_m3": {
        "divisor": VEWIN_PERSON_DAY_M3,
        "unit_nl": "mensen",
        "template": "dagelijks verbruik van ~{n} {unit}",
        "source_url": VEWIN_SOURCE_URL,
        "source_label": "VEWIN Waterstatistiek"
    },
    "supply_gap_m3": {
        "divisor": VEWIN_HOUSEHOLD_DAY_M3,
        "unit_nl": "huishoudens",
        "template": "equivalent aan ~{n} {unit} zonder water",
        "source_url": VEWIN_SOURCE_URL,
        "source_label": "VEWIN Waterstatistiek"
    },
    "supply_capacity_m3": {
        "divisor": VEWIN_PERSON_DAY_M3,
        "unit_nl": "mensen",
        "template": "leveringscapaciteit voor ~{n} {unit}",
        "source_url": VEWIN_SOURCE_URL,
        "source_label": "VEWIN Waterstatistiek"
    },
    "capacity_remaining_m3": {
        "divisor": VEWIN_HOUSEHOLD_DAY_M3,
        "unit_nl": "extra huishoudens",
        "template": "ruimte voor ~{n} {unit}",
        "source_url": VEWIN_SOURCE_URL,
        "source_label": "VEWIN Waterstatistiek"
    },
    "development_delta_m3": {
        "divisor": VEWIN_HOUSEHOLD_DAY_M3,
        "unit_nl": "woningen",
        "template": "extra vraag gelijk aan ~{n} {unit}",
        "source_url": VEWIN_SOURCE_URL,
        "source_label": "VEWIN Waterstatistiek"
    }
}

@dataclass
class HumanScaleRef:
    metric_key: str
    metric_value: float
    metric_unit: str
    analogy_nl: str
    analogy_source_url: str
    analogy_source_label: str
    is_policy_approx: bool = True

def to_human_scale(
    metric_key: str,
    value: float,
    unit: str,
    assumption_override_divisor: Optional[float] = None
) -> HumanScaleRef:
    """
    Converts a raw metric to a human-scale analogy.
    Raises ValueError if metric_key is not in METRIC_CONVERSIONS
    (prevents silent empty-source failures).
    """
    if metric_key not in METRIC_CONVERSIONS:
        raise ValueError(
            f"No human-scale conversion defined for metric '{metric_key}'. "
            f"Add an entry to METRIC_CONVERSIONS or handle explicitly."
        )

    conv = METRIC_CONVERSIONS[metric_key]
    divisor = assumption_override_divisor or conv["divisor"]
    n = int(abs(value) / divisor)
    n_formatted = f"{n:,}".replace(",", ".")

    analogy = conv["template"].format(n=n_formatted, unit=conv["unit_nl"])

    return HumanScaleRef(
        metric_key=metric_key,
        metric_value=value,
        metric_unit=unit,
        analogy_nl=analogy,
        analogy_source_url=conv["source_url"],
        analogy_source_label=conv["source_label"],
        is_policy_approx=True
    )

def attach_human_scale_refs(
    results: dict,
    demand_per_dwelling_override: Optional[float] = None
) -> dict:
    """
    Attaches HumanScaleRef to every headline metric in results.
    Called by format_scenario_output node.
    """
    human_scale_refs = {}
    for key in [
        "daily_demand_m3", "supply_gap_m3", "supply_capacity_m3",
        "capacity_remaining_m3", "development_delta_m3"
    ]:
        if results.get(key) is not None:
            try:
                override = (
                    demand_per_dwelling_override
                    if key in ("supply_gap_m3", "capacity_remaining_m3", "development_delta_m3")
                    else None
                )
                human_scale_refs[key] = to_human_scale(
                    key, results[key], "m³/dag", override
                ).__dict__
            except ValueError as e:
                # Log but don't silently suppress — show "no conversion" label
                human_scale_refs[key] = {
                    "analogy_nl": "Geen omrekening beschikbaar",
                    "analogy_source_url": "/methodology",
                    "analogy_source_label": "Zie methodologiedocument",
                    "is_policy_approx": True,
                    "error": str(e)
                }
    results["human_scale_refs"] = human_scale_refs
    return results

Frontend: HumanScaleRef.vue

vue

<template>
  <span class="human-scale-ref">
    <span class="analogy">≈ {{ ref.analogy_nl }}</span>
    <a
      :href="ref.analogy_source_url"
      target="_blank"
      class="source-link"
      :title="ref.analogy_source_label"
    >
      ⓘ
    </a>
    <span v-if="ref.is_policy_approx" class="approx-label">
      (beleidsmatige schatting)
    </span>
  </span>
</template>

<script setup>
defineProps({
  ref: { type: Object, required: true }
})
</script>

Usage in MetricCard.vue

vue

<template>
  <div class="metric-card">
    <div class="metric-value">
      {{ formatValue(value) }} <span class="metric-unit">{{ unit }}</span>
    </div>
    <HumanScaleRef
      v-if="humanScaleRef"
      :ref="humanScaleRef"
    />
  </div>
</template>

19.5 GAP 5 — "Make it feasible" intervention ranking
Problem

When a scenario returns STOP or CAUTION, policymakers and developers need to know not just that it's infeasible but what specific interventions would change the outcome, ranked by effectiveness and cost.
Implementation
Intervention ranker (src/backend/calculators/intervention_ranker.py)

Python

from dataclasses import dataclass
from typing import List
from src.backend.models.scenario import ScenarioResults, ScenarioParams
from src.backend.calculators.demand import calculate_daily_demand
from src.backend.calculators.capacity import get_zone_capacity

INTERVENTION_CATALOGUE = [
    {
        "id": "buffer_30k",
        "label_nl": "Bufferopslag 30.000 m³",
        "supply_delta_m3": 30_000,
        "demand_delta_m3": 0,
        "cost_eur_low": 15_000_000,
        "cost_eur_high": 25_000_000,
        "lead_time_years": 3,
        "source_url": "https://www.pzh.nl/regiovisie-waterprogramma-2022-2027",
        "source_label": "Regionaal Waterprogramma ZH 2022–2027",
        "applicable_types": ["drop_pin", "intake_failure", "multi_hazard"]
    },
    {
        "id": "alt_intake_lek",
        "label_nl": "Alternatieve inname via de Lek",
        "supply_delta_m3": 65_000,
        "demand_delta_m3": 0,
        "cost_eur_low": 40_000_000,
        "cost_eur_high": 100_000_000,
        "lead_time_years": 7,
        "source_url": "https://www.pzh.nl/regiovisie-waterprogramma-2022-2027",
        "source_label": "Regionaal Waterprogramma ZH 2022–2027",
        "applicable_types": ["intake_failure", "multi_hazard"]
    },
    {
        "id": "demand_restriction_10pct",
        "label_nl": "Vraagbeperking 10% (regelgeving)",
        "supply_delta_m3": 0,
        "demand_delta_m3": -0.10,  # fraction of demand
        "cost_eur_low": 0,
        "cost_eur_high": 500_000,
        "lead_time_years": 1,
        "source_url": "https://wetten.overheid.nl/BWBR0026306",
        "source_label": "Drinkwaterwet art. 10",
        "applicable_types": ["drop_pin", "intake_failure", "multi_hazard"]
    },
    {
        "id": "interconnect_neighbor",
        "label_nl": "Interconnectie buurregio",
        "supply_delta_m3": 30_000,
        "demand_delta_m3": 0,
        "cost_eur_low": 20_000_000,
        "cost_eur_high": 60_000_000,
        "lead_time_years": 5,
        "source_url": "https://www.vewin.nl/publicaties/infrastructuurrapport-2023",
        "source_label": "VEWIN Infrastructuurrapport 2023",
        "applicable_types": ["drop_pin", "intake_failure", "multi_hazard"]
    }
]

@dataclass
class RankedIntervention:
    rank: int
    intervention_id: str
    label_nl: str
    gap_closure_pct: float
    new_supply_gap_m3: float
    new_feasibility_class: str
    cost_range_nl: str
    lead_time_years: int
    source_url: str
    source_label: str

def rank_interventions(
    current_results: ScenarioResults,
    params: ScenarioParams
) -> List[RankedIntervention]:
    """
    Iterates through applicable interventions, re-computes feasibility
    for each, and returns top-3 ranked by gap_closure_pct DESC, cost ASC.
    Only runs if feasibility_class is STOP or CAUTION.
    """
    if current_results.feasibility_class == "GO":
        return []

    gap = current_results.supply_gap_m3  # negative = shortfall
    applicable = [
        i for i in INTERVENTION_CATALOGUE
        if params.scenario_type in i["applicable_types"]
    ]

    ranked = []
    for intv in applicable:
        # Demand-side intervention (fraction of demand)
        if intv["demand_delta_m3"] < 0:
            demand_reduction = abs(intv["demand_delta_m3"]) * current_results.daily_demand_m3
            new_gap = gap + demand_reduction
        else:
            new_gap = gap + intv["supply_delta_m3"]

        if gap < 0:
            gap_closure_pct = min(100.0, (new_gap - gap) / abs(gap) * 100)
        else:
            gap_closure_pct = 0.0

        new_fc = _compute_single_feasibility_class(new_gap)
        cost_low = intv["cost_eur_low"]
        cost_high = intv["cost_eur_high"]
        cost_nl = (
            f"€{cost_low // 1_000_000:.0f}M – €{cost_high // 1_000_000:.0f}M"
            if cost_high > 0 else "Regelgevingskosten"
        )

        ranked.append({
            "intervention_id": intv["id"],
            "label_nl": intv["label_nl"],
            "gap_closure_pct": gap_closure_pct,
            "new_supply_gap_m3": new_gap,
            "new_feasibility_class": new_fc,
            "cost_range_nl": cost_nl,
            "lead_time_years": intv["lead_time_years"],
            "source_url": intv["source_url"],
            "source_label": intv["source_label"]
        })

    ranked.sort(key=lambda x: (-x["gap_closure_pct"], x["lead_time_years"]))

    return [
        RankedIntervention(rank=i + 1, **r)
        for i, r in enumerate(ranked[:3])
    ]

def _compute_single_feasibility_class(gap: float) -> str:
    if gap >= 0:
        return "GO"
    elif gap >= -10_000:
        return "CAUTION"
    else:
        return "STOP"

Frontend: MakeItFeasiblePanel.vue

vue

<template>
  <div
    v-if="interventions.length > 0"
    class="make-feasible-panel"
  >
    <h4>💡 Wat maakt dit wél haalbaar?</h4>
    <p class="sub">Top-3 interventies, gerangschikt op effectiviteit:</p>
    <div
      v-for="intv in interventions"
      :key="intv.intervention_id"
      class="intervention-card"
    >
      <div class="intv-header">
        <span class="intv-rank">#{{ intv.rank }}</span>
        <span class="intv-label">{{ intv.label_nl }}</span>
        <FeasibilityBadge :feasibility-class="intv.new_feasibility_class" />
      </div>
      <div class="intv-details">
        <div class="detail-row">
          <span>Sluit {{ intv.gap_closure_pct.toFixed(0) }}% van het tekort</span>
        </div>
        <div class="detail-row">
          <span>Kosten: {{ intv.cost_range_nl }}</span>
        </div>
        <div class="detail-row">
          <span>Doorlooptijd: ~{{ intv.lead_time_years }} jaar</span>
        </div>
        <div class="detail-row source-row">
          <a :href="intv.source_url" target="_blank">
            📚 {{ intv.source_label }}
          </a>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import FeasibilityBadge from './FeasibilityBadge.vue'
defineProps({
  interventions: { type: Array, required: true }
})
</script>

19.6 GAP 6 — Cumulative load warning (active scenarios in same supply zone)
Problem

A project developer or spatial planner assessing a new development sees only the isolated impact of their project—not the combined effect with other active scenarios in the same supply zone. This leads to unrealistic feasibility assessments.
Implementation
Cumulative load checker (src/backend/nodes/cumulative_load_node.py)

Python

import json
from src.backend.cache.scenario_store import ScenarioStore
from src.backend.models.scenario import AgentState

async def check_cumulative_load(state: AgentState) -> AgentState:
    """
    Queries the scenario store for all active scenarios whose
    supply zone overlaps with the current scenario's zone.
    Computes combined demand and emits a warning if > 0 others found.
    """
    current_zone_ids = state["scenario_object"].results.get("affected_zone_ids", [])
    
    if not current_zone_ids:
        state["cumulative_load"] = None
        return state

    store: ScenarioStore = state["scenario_store"]
    
    # Query all cached scenarios for overlapping zones
    overlapping = store.conn.execute("""
        SELECT 
            scenario_id,
            json_extract(result_json, '$.params.scenario_type') as stype,
            json_extract(result_json, '$.results.daily_demand_m3') as demand,
            json_extract(result_json, '$.results.affected_zone_ids') as zones,
            created_at
        FROM scenarios
        WHERE scenario_id != ?
        AND created_at > now() - INTERVAL 7 DAY
        ORDER BY created_at DESC
    """, [state["scenario_id"]]).fetchall()

    active_in_zone = []
    cumulative_demand = 0.0

    for row in overlapping:
        row_zones = json.loads(row[3]) if row[3] else []
        overlap = set(current_zone_ids) & set(row_zones)
        if overlap:
            demand = float(row[2]) if row[2] else 0.0
            active_in_zone.append({
                "scenario_id": row[0],
                "scenario_type": row[1],
                "demand_m3": demand,
                "overlapping_zones": list(overlap)
            })
            cumulative_demand += demand

    if active_in_zone:
        state["cumulative_load"] = {
            "active_count": len(active_in_zone),
            "cumulative_demand_m3": cumulative_demand,
            "active_scenarios": active_in_zone,
            "warning_nl": (
                f"Er zijn {len(active_in_zone)} andere actieve scenario's in "
                f"deze leveringszone. Gecombineerd extra dagelijks verbruik: "
                f"{cumulative_demand:,.0f} m³/dag."
            )
        }
    else:
        state["cumulative_load"] = None

    return state

Frontend: CumulativeLoadWarning.vue

vue

<template>
  <div
    v-if="cumulativeLoad"
    class="cumulative-load-warning"
  >
    <div class="warning-header">
      ⚠️ Gecombineerd effect meerdere scenario's
    </div>
    <p>{{ cumulativeLoad.warning_nl }}</p>
    <details>
      <summary>
        Bekijk {{ cumulativeLoad.active_count }} actieve scenario's
      </summary>
      <ul>
        <li
          v-for="s in cumulativeLoad.active_scenarios"
          :key="s.scenario_id"
        >
          <a :href="`/scenario/${s.scenario_id}`" target="_blank">
            {{ s.scenario_type }} — {{ s.demand_m3.toLocaleString('nl-NL') }} m³/dag
          </a>
          <span class="zone-tags">
            (zones: {{ s.overlapping_zones.join(', ') }})
          </span>
        </li>
      </ul>
    </details>
  </div>
</template>

<script setup>
defineProps({
  cumulativeLoad: { type: Object, default: null }
})
</script>

19.7 GAP 7 — Per-intake chloride threshold (not a global constant)
Problem

Using cl_threshold_mg_l = 150 as a global constant is technically incorrect. Thresholds depend on intake location, treatment technology, and the specific production chain. Water authority engineers will challenge this immediately.
Implementation
Per-intake threshold resolution (src/backend/calculators/chloride.py)

Python

def get_intake_chloride_threshold(
    intake_id: str,
    conn,
    fallback_assumption: dict
) -> dict:
    """
    Fetches per-intake chloride threshold from productieketen dataset.
    Returns threshold value, its source, and whether it came from DB.

    Falls back to assumption (150 mg/L, Drinkwaterbesluit) if not found.
    The fallback is always explicitly labelled.
    """
    row = conn.execute("""
        SELECT 
            locatie_id,
            cl_threshold_mg_l,
            behandelingstechniek,
            threshold_source,
            threshold_last_updated
        FROM productieketen
        WHERE locatie_id = ?
        AND cl_threshold_mg_l IS NOT NULL
        LIMIT 1
    """, [intake_id]).fetchone()

    if row and row[1] is not None:
        return {
            "threshold_mg_l": float(row[1]),
            "treatment_tech": row[2],
            "source": row[3] or "productieketen dataset",
            "last_updated": row[4],
            "from_db": True,
            "is_assumption": False,
            "assumption_label_nl": None
        }
    else:
        # Explicit fallback with full labelling
        return {
            "threshold_mg_l": fallback_assumption["value"],
            "treatment_tech": "onbekend",
            "source": fallback_assumption["source_url"],
            "last_updated": None,
            "from_db": False,
            "is_assumption": True,
            "assumption_label_nl": (
                f"Drempelwaarde niet gevonden voor inname '{intake_id}' "
                f"in productieketendataset. "
                f"Terugvalwaarde gebruikt: {fallback_assumption['value']} mg/L "
                f"(conservatieve Nederlandse norm, Drinkwaterbesluit). "
                f"Pas aan via aanname-slider (bandbreedte: 150–250 mg/L)."
            )
        }

def calculate_cl_concentration_with_threshold(
    baseline_cl_mg_l: float,
    knmi_cl_delta: float,
    outage_weeks: int,
    intake_threshold: dict
) -> dict:
    """
    Calculates chloride concentration and compares to per-intake threshold.
    """
    effective_cl = baseline_cl_mg_l + knmi_cl_delta
    if outage_weeks > 0:
        degradation = 1 + (0.20 * min(outage_weeks, 6))
        effective_cl *= degradation

    threshold = intake_threshold["threshold_mg_l"]
    exceeds = effective_cl > threshold
    margin = threshold - effective_cl  # negative = exceeds threshold

    return {
        "effective_cl_mg_l": round(effective_cl, 1),
        "threshold_mg_l": threshold,
        "threshold_from_db": intake_threshold["from_db"],
        "threshold_source": intake_threshold["source"],
        "threshold_is_assumption": intake_threshold["is_assumption"],
        "threshold_assumption_label_nl": intake_threshold.get("assumption_label_nl"),
        "exceeds_threshold": exceeds,
        "margin_mg_l": round(margin, 1),
        "risk_level": "hoog" if effective_cl > threshold * 1.2 else
                      "middel" if exceeds else "laag"
    }

Assumption definition for chloride fallback

Python

# In src/backend/models/assumptions.py — default assumption library

CHLORIDE_THRESHOLD_FALLBACK = Assumption(
    key="cl_threshold_fallback_mg_l",
    label_nl="Chloridedrempel inname (terugvalwaarde)",
    value=150.0,
    value_min=150.0,
    value_max=250.0,
    unit="mg/L",
    source_url="https://wetten.overheid.nl/BWBR0026304",  # Drinkwaterbesluit
    source_label="Drinkwaterbesluit (NL norm: 150 mg/L; EU-norm: 250 mg/L)",
    sensitivity="high",
    is_policy_approx=True,
    slider_step=10.0,
    notes=(
        "Nederlandse norm (150 mg/L) is conservatiever dan EU-norm (250 mg/L). "
        "Werkelijke drempel afhankelijk van inname en behandelingstechniek. "
        "Zie RIVM Tabel 2.3 voor context."
    )
)

19.8 GAP 8 — Official position panel (government policy context)
Problem

If the tool's scenario output appears to contradict official government communications, trust collapses. Every scenario output must anchor itself to the official policy position of the relevant authorities.
Implementation
Official position registry (src/backend/registries/official_position_registry.py)

Python

OFFICIAL_POSITIONS = {
    "drinkwater_zh": {
        "topic": "drinkwater",
        "province": "ZH",
        "summary_nl": (
            "De beschikbaarheid van voldoende schoon drinkwater in Zuid-Holland staat "
            "onder druk door bevolkingsgroei, klimaatverandering en bronkwaliteit. "
            "De provincie werkt samen met drinkwaterbedrijven en waterschappen aan "
            "een robuuste watervoorziening tot 2040."
        ),
        "documents": [
            {
                "title": "Regionaal Waterprogramma Zuid-Holland 2022–2027",
                "url": "https://www.pzh.nl/regiovisie-waterprogramma-2022-2027",
                "date": "2022",
                "type": "provinciaal_beleid"
            },
            {
                "title": "Nationaal Waterprogramma 2022–2027",
                "url": "https://www.rijksoverheid.nl/documenten/rapporten/"
                       "2022/03/18/nationaal-waterprogramma-2022-2027",
                "date": "2022",
                "type": "nationaal_beleid"
            },
            {
                "title": "Ruimtelijk Arrangement Rijk–Zuid-Holland (2 juni 2025)",
                "url": "https://www.rijksoverheid.nl/documenten/"
                       "convenanten/2025/06/02/ruimtelijk-arrangement-zh",
                "date": "2 juni 2025",
                "type": "rijksbeleid"
            }
        ]
    },
    "klimaat_knmi": {
        "topic": "klimaat",
        "summary_nl": (
            "KNMI'23 beschrijft vier klimaatscenario's voor Nederland met "
            "verschillende combinaties van opwarming en verandering in neerslagpatronen. "
            "Scenario Hd (hoge opwarming, drogere zomers) leidt tot de grootste druk "
            "op zoetwater­beschikbaarheid."
        ),
        "documents": [
            {
                "title": "KNMI'23 Klimaatscenario's voor Nederland",
                "url": "https://www.knmi.nl/kennis-en-datacentrum/achtergrond/"
                       "knmi-klimaatscenarios-2023",
                "date": "2023",
                "type": "wetenschappelijk_beleid"
            }
        ]
    },
    "woningbouw_zh": {
        "topic": "woningbouw",
        "province": "ZH",
        "summary_nl": (
            "Zuid-Holland heeft in het Ruimtelijk Arrangement afspraken gemaakt "
            "over de bouw van circa 235.000 woningen tot 2030. "
            "Beschikbaarheid van drinkwater is benoemd als randvoorwaarde "
            "voor ruimtelijke ontwikkeling."
        ),
        "documents": [
            {
                "title": "Ruimtelijk Arrangement Rijk–Zuid-Holland (2 juni 2025)",
                "url": "https://www.rijksoverheid.nl/documenten/"
                       "convenanten/2025/06/02/ruimtelijk-arrangement-zh",
                "date": "2 juni 2025",
                "type": "rijksbeleid"
            },
            {
                "title": "Provinciaal Omgevingsbeleid Zuid-Holland",
                "url": "https://www.pzh.nl/omgevingsbeleid",
                "date": "2023",
                "type": "provinciaal_beleid"
            }
        ]
    },
    "krw": {
        "topic": "kaderrichtlijn_water",
        "summary_nl": (
            "De Kaderrichtlijn Water (KRW) verplicht lidstaten tot het bereiken van "
            "een goede toestand van alle waterlichamen. De uiterste deadline is "
            "22 december 2027. Voor Zuid-Holland betekent dit extra druk op "
            "waterkwaliteit nabij innamepunten en beschermde waterlichamen."
        ),
        "documents": [
            {
                "title": "Kaderrichtlijn Water — Rijkswaterstaat",
                "url": "https://www.rijkswaterstaat.nl/water/waterbeheer/"
                       "bescherming-tegen-het-water/kwaliteit-van-het-water/"
                       "kaderrichtlijn-water",
                "date": "2027 deadline",
                "type": "europees_beleid"
            }
        ]
    }
}

def get_official_position(
    scenario_type: str,
    knmi_preset: str,
    involves_housing: bool,
    involves_krw: bool
) -> dict:
    """
    Returns the most relevant official position(s) for a scenario.
    Always includes drinkwater_zh as the baseline anchor.
    """
    positions = [OFFICIAL_POSITIONS["drinkwater_zh"]]

    if knmi_preset in ("Hd", "Hn", "Ld", "Ln"):
        positions.append(OFFICIAL_POSITIONS["klimaat_knmi"])

    if involves_housing:
        positions.append(OFFICIAL_POSITIONS["woningbouw_zh"])

    if involves_krw:
        positions.append(OFFICIAL_POSITIONS["krw"])

    return {
        "positions": positions,
        "primary": positions[0],
        "disclaimer_nl": (
            "Dit scenario is een beleidsmatige verkenning. Het is geen officieel "
            "standpunt van de Provincie Zuid-Holland of haar partners."
        )
    }

Frontend: OfficialPositionPanel.vue

vue

<template>
  <div class="official-position-panel">
    <h4>🏛️ Officieel beleid — Wat zegt de overheid hierover?</h4>
    <div
      v-for="pos in officialPosition.positions"
      :key="pos.topic"
      class="position-block"
    >
      <p class="position-summary">{{ pos.summary_nl }}</p>
      <ul class="document-list">
        <li v-for="doc in pos.documents" :key="doc.url">
          <a :href="doc.url" target="_blank">
            📄 {{ doc.title }}
          </a>
          <span class="doc-date"> ({{ doc.date }})</span>
        </li>
      </ul>
    </div>
    <p class="disclaimer">
      ⚠️ {{ officialPosition.disclaimer_nl }}
    </p>
  </div>
</template>

<script setup>
defineProps({
  officialPosition: { type: Object, required: true }
})
</script>

19.9 GAP 9 — Waterinfo chloride API: mandatory guard for intake scenarios

This is fully implemented in the Waterinfo client specified earlier in Section 13 of the complete document. The following adds the frontend banner component and the SSE event wiring that was referenced but not fully specified.
Frontend: WaterinfoBanner.vue (complete spec)

vue

<template>
  <div
    v-if="waterinfoStatus"
    :class="['waterinfo-banner', statusClass]"
  >
    <span class="status-icon">{{ statusIcon }}</span>
    <span class="status-message">{{ waterinfoStatus.message_nl }}</span>
    <span
      v-if="waterinfoStatus.cached_date"
      class="cached-date"
    >
      Laatste bekende meting: {{ formatDate(waterinfoStatus.cached_date) }}
    </span>
    <span
      v-if="waterinfoStatus.value"
      class="chloride-value"
    >
      Cl⁻: {{ waterinfoStatus.value }} mg/L
    </span>
    <a
      href="https://waterinfo.rws.nl"
      target="_blank"
      class="waterinfo-link"
    >
      Bekijk live op Waterinfo →
    </a>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  waterinfoStatus: { type: Object, default: null }
})

const statusClass = computed(() => ({
  'status-live': props.waterinfoStatus?.source === 'Waterinfo live',
  'status-cached': props.waterinfoStatus?.source === 'Waterinfo (cached)',
  'status-unavailable': props.waterinfoStatus?.source === 'unavailable'
}))

const statusIcon = computed(() => {
  const src = props.waterinfoStatus?.source
  if (src === 'Waterinfo live') return '🟢'
  if (src === 'Waterinfo (cached)') return '🟡'
  return '🔴'
})

const formatDate = (iso) =>
  new Date(iso).toLocaleDateString('nl-NL', {
    day: 'numeric', month: 'long', year: 'numeric'
  })
</script>

SSE wiring for waterinfo_status event (src/frontend/composables/useScenarioSSE.ts)

TypeScript

// Add to the existing event switch in useScenarioSSE.ts

case 'waterinfo_status': {
  const payload = JSON.parse(event.data)
  waterinfoStore.setStatus({
    source: payload.source,
    value: payload.value,
    cached_date: payload.date,
    message_nl: payload.warning || 
      (payload.source === 'Waterinfo live' 
        ? `Live Waterinfo-meting: ${payload.value} mg/L Cl⁻`
        : `Gecachede Waterinfo-meting gebruikt (${payload.date}): ${payload.value} mg/L Cl⁻`
      ),
    intake_id: payload.intake_id
  })
  break
}

Waterinfo Pinia store (src/frontend/stores/waterinfoStore.ts)

TypeScript

import { defineStore } from 'pinia'

export const useWaterinfoStore = defineStore('waterinfo', {
  state: () => ({
    statuses: {} as Record<string, {
      source: string
      value: number | null
      cached_date: string | null
      message_nl: string
      intake_id: string
    }>
  }),
  actions: {
    setStatus(payload: {
      source: string
      value: number | null
      cached_date: string | null
      message_nl: string
      intake_id: string
    }) {
      this.statuses[payload.intake_id] = payload
    },
    clearAll() {
      this.statuses = {}
    }
  },
  getters: {
    getStatusForIntake: (state) => (intakeId: string) =>
      state.statuses[intakeId] || null,
    hasAnyWarning: (state) =>
      Object.values(state.statuses).some(
        s => s.source !== 'Waterinfo live'
      )
  }
})

19.10 GAP 10 — Citizen mode: verdict-first template, postcode detection, water company link
Problem

Citizens ask "Is mijn water veilig?" not "Wat is het aanvoertekort in m³/dag in leveringszone 4B?" The citizen mode requires a completely different output contract: verdict first, minimal numbers, reassurance context, always link to the responsible water company and official policy.
Implementation
Citizen mode detection (src/backend/utils/citizen_detection.py)

Python

import re
from typing import Literal

CITIZEN_PHRASES = [
    "mijn water", "ons water", "is het veilig", "kan ik",
    "drinkwater veilig", "wat betekent dit voor mij",
    "voor mijn buurt", "bij mij thuis", "in mijn straat"
]

POSTCODE_REGEX = re.compile(r'\b[1-9][0-9]{3}\s?[A-Za-z]{2}\b')

def detect_citizen_mode(
    question: str,
    persona_selection: str | None
) -> Literal["citizen", "professional"]:
    """
    Returns 'citizen' if:
    - persona_selection == 'burger', OR
    - question contains citizen phrases, OR
    - question contains a postcode pattern
    Returns 'professional' otherwise.
    """
    if persona_selection and persona_selection.lower() in ("burger", "citizen"):
        return "citizen"

    question_lower = question.lower()
    if any(phrase in question_lower for phrase in CITIZEN_PHRASES):
        return "citizen"

    if POSTCODE_REGEX.search(question):
        return "citizen"

    return "professional"

Water company lookup by postcode/zone (src/backend/registries/water_company_registry.py)

Python

WATER_COMPANY_BY_ZONE = {
    # Leveringszones mapped to responsible drinkwaterbedrijf
    # Populated from productieketen / leveringszones dataset
    "dunea": {
        "name": "Dunea",
        "url": "https://www.dunea.nl",
        "phone": "0800-5000 400",
        "zone_prefixes": ["DUN", "ZH-WEST", "ZH-KUST"],
        "service_area_nl": "Den Haag, Westland, Zoetermeer en omgeving"
    },
    "evides": {
        "name": "Evides Waterbedrijf",
        "url": "https://www.evides.nl",
        "phone": "0900-0929",
        "zone_prefixes": ["EVI", "ZH-OOST", "ZH-EILANDEN"],
        "service_area_nl": "Rotterdam, Dordrecht, Zeeland en omgeving"
    },
    "oasen": {
        "name": "Oasen",
        "url": "https://www.oasen.nl",
        "phone": "088-0880 100",
        "zone_prefixes": ["OAS", "ZH-MIDDEN"],
        "service_area_nl": "Gouda, Alphen aan den Rijn en omgeving"
    }
}

def get_water_company_for_zone(zone_id: str) -> dict:
    """Returns water company info for a leveringszone."""
    for company_id, company in WATER_COMPANY_BY_ZONE.items():
        if any(zone_id.upper().startswith(p) for p in company["zone_prefixes"]):
            return company
    # Default: return all three if zone unknown
    return {
        "name": "Uw drinkwaterbedrijf",
        "url": "https://www.vewin.nl/drinkwaterbedrijven",
        "phone": "Zie uw waterbedrijf",
        "service_area_nl": "Zuid-Holland"
    }

Citizen formatter node (src/backend/nodes/citizen_formatter_node.py)

Python

import re
from src.backend.models.scenario import AgentState
from src.backend.registries.water_company_registry import get_water_company_for_zone
from src.backend.registries.official_position_registry import OFFICIAL_POSITIONS

CITIZEN_RESPONSE_PROMPT = """
Je bent een hulpvaardig assistent voor de Provincie Zuid-Holland.
Schrijf een antwoord voor een burger over drinkwater in hun buurt.

REGELS:
1. Begin met een VERDICT in één zin (veilig / aandacht nodig / risico).
2. Schrijf in eenvoudig Nederlands (B1-niveau), max. 100 woorden.
3. Gebruik GEEN technische termen of m³/dag.
4. Noem GEEN nieuwe getallen die niet in de invoer staan.
5. Eindig altijd met een verwijzing naar het drinkwaterbedrijf en officieel beleid.
6. Voeg altijd de disclaimer toe.

INVOER:
Vraag: {question}
Leveringszone: {zone_id}
Haalbaarheidsklasse: {feasibility_class}
Verwacht aanvangsjaar risico: {onset_year}
KNMI-scenario: {knmi_preset}
Tijdshorizon: {time_horizon}

Schrijf het antwoord als JSON:
{{
  "verdict_nl": "...",
  "explanation_nl": "...",
  "action_nl": "..."
}}
"""

async def format_citizen_response(state: AgentState) -> AgentState:
    """
    Generates citizen-mode response using strict template.
    GreenPT may only generate prose, not new numbers.
    """
    results = state["scenario_card"].results
    params = state["scenario_card"].params
    zone_id = (results.get("affected_zone_ids") or ["onbekend"])[0]
    water_company = get_water_company_for_zone(zone_id)
    postcode = _extract_postcode(state["question"])

    prompt = CITIZEN_RESPONSE_PROMPT.format(
        question=state["question"],
        zone_id=zone_id,
        feasibility_class=results["feasibility_class"],
        onset_year=results.get("onset_year", "niet bepaald"),
        knmi_preset=params.knmi_preset,
        time_horizon=params.time_horizon
    )

    llm_response = await state["llm"].apredict(prompt)
    parsed = _parse_json_response(llm_response)

    state["citizen_response"] = {
        "postcode": postcode,
        "zone_id": zone_id,
        "verdict_nl": parsed["verdict_nl"],
        "explanation_nl": parsed["explanation_nl"],
        "action_nl": parsed["action_nl"],
        "water_company": water_company,
        "official_links": [
            {
                "title": doc["title"],
                "url": doc["url"]
            }
            for doc in OFFICIAL_POSITIONS["drinkwater_zh"]["documents"][:2]
        ],
        "disclaimer_nl": (
            "Dit is een exploratief scenario, geen officiële meting. "
            "Voor officiële informatie, neem contact op met uw drinkwaterbedrijf."
        ),
        "knmi_link": {
            "title": "KNMI Klimaatscenario's",
            "url": "https://www.knmi.nl/kennis-en-datacentrum/achtergrond/"
                   "knmi-klimaatscenarios-2023"
        }
    }
    return state

def _extract_postcode(question: str) -> str | None:
    match = re.search(r'\b([1-9][0-9]{3}\s?[A-Za-z]{2})\b', question)
    return match.group(1).upper().replace(" ", "") if match else None

Frontend: CitizenResponseCard.vue

vue

<template>
  <div class="citizen-response-card">
    <div class="citizen-header">
      💧 Drinkwater in jouw buurt
      <span v-if="response.postcode" class="postcode-badge">
        {{ response.postcode }}
      </span>
    </div>

    <div :class="['verdict-block', verdictClass]">
      {{ response.verdict_nl }}
    </div>

    <p class="citizen-explanation">{{ response.explanation_nl }}</p>
    <p class="citizen-action">{{ response.action_nl }}</p>

    <div class="citizen-links">
      <div class="link-section">
        <strong>📞 Uw drinkwaterbedrijf:</strong>
        <a :href="response.water_company.url" target="_blank">
          {{ response.water_company.name }}
        </a>
        <span class="phone">{{ response.water_company.phone }}</span>
      </div>
      <div class="link-section">
        <strong>🏛️ Officieel beleid:</strong>
        <a
          v-for="link in response.official_links"
          :key="link.url"
          :href="link.url"
          target="_blank"
          class="policy-link"
        >
          {{ link.title }}
        </a>
      </div>
      <div class="link-section">
        <a :href="response.knmi_link.url" target="_blank">
          🌡️ {{ response.knmi_link.title }}
        </a>
      </div>
    </div>

    <p class="citizen-disclaimer">⚠️ {{ response.disclaimer_nl }}</p>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  response: { type: Object, required: true }
})

const verdictClass = computed(() => {
  const v = props.response.verdict_nl?.toLowerCase() || ''
  if (v.includes('veilig') || v.includes('geen') || v.includes('laag'))
    return 'verdict-safe'
  if (v.includes('aandacht') || v.includes('mogelijk'))
    return 'verdict-caution'
  return 'verdict-risk'
})
</script>

20. Type 2 scenario specification (stretch goal)
20.1 What Type 2 is

Type 2 is a multi-hazard / cumulative pressure scenario: it combines multiple stressors simultaneously (climate + housing growth + KRW enforcement) to answer "Which combination hits first, and how early?"
20.2 ScenarioParams extension for Type 2

Python

@dataclass
class MultiHazardParams:
    """
    Additional params for Type 2 scenarios.
    Stacked on top of base ScenarioParams.
    """
    hazard_components: list[str]
    # e.g. ["climate_hd", "housing_growth_hoog", "krw_2027_enforcement"]
    
    # Component weights (0–1, must sum to 1)
    component_weights: dict[str, float]
    
    # "Which combination hits first?" stepping
    step_years: list[int] = field(
        default_factory=lambda: [2025, 2027, 2030, 2035, 2040]
    )
    
    # KRW enforcement trigger
    krw_enforcement_date: str = "2027-12-22"
    krw_land_use_restriction_pct: float = 0.15  # 15% land-use tightening near intakes

20.3 Compound risk score formula

Python

def calculate_compound_risk_score(
    demand_gap_m3: float,           # from climate + growth calculation
    cl_exceedance_pct: float,       # how far above threshold (0 = at threshold)
    krw_at_risk_count: int,
    onset_year: int | None,
    time_horizon: int = 2040
) -> dict:
    """
    Composite risk score 0–100 for multi-hazard scenario.
    Higher = more risk.
    """
    # Demand gap component (0–40 points)
    gap_score = min(40, abs(demand_gap_m3) / 5000) if demand_gap_m3 < 0 else 0

    # Chloride exceedance (0–30 points)
    cl_score = min(30, cl_exceedance_pct * 30) if cl_exceedance_pct > 0 else 0

    # KRW risk (0–20 points)
    krw_score = min(20, krw_at_risk_count * 4)

    # Time urgency (0–10 points)
    if onset_year:
        years_remaining = onset_year - 2025
        urgency_score = max(0, 10 - years_remaining)
    else:
        urgency_score = 0

    total = gap_score + cl_score + krw_score + urgency_score
    risk_level = (
        "kritiek" if total >= 70 else
        "hoog" if total >= 50 else
        "middel" if total >= 30 else
        "laag"
    )

    return {
        "compound_risk_score": round(total, 1),
        "risk_level": risk_level,
        "components": {
            "demand_gap_score": round(gap_score, 1),
            "chloride_score": round(cl_score, 1),
            "krw_score": round(krw_score, 1),
            "urgency_score": round(urgency_score, 1)
        }
    }

21. Type 4 scenario specification (stretch goal)
21.1 What Type 4 is

Type 4 is an intervention effectiveness scenario: given a baseline scenario (typically a STOP or CAUTION result), compare the effect of one or more interventions applied together to find the combination that achieves GO.
21.2 Intervention stacking logic

Python

def stack_interventions(
    baseline_results: ScenarioResults,
    intervention_ids: list[str],
    params: ScenarioParams
) -> dict:
    """
    Applies multiple interventions cumulatively to a baseline scenario.
    Interventions are applied in the order given.
    Returns cumulative effect at each step.
    """
    current_gap = baseline_results.supply_gap_m3
    current_demand = baseline_results.daily_demand_m3
    steps = []

    for intv_id in intervention_ids:
        intv = next(
            (i for i in INTERVENTION_CATALOGUE if i["id"] == intv_id),
            None
        )
        if not intv:
            continue

        if intv["demand_delta_m3"] < 0:
            demand_reduction = abs(intv["demand_delta_m3"]) * current_demand
            current_demand -= demand_reduction
            current_gap += demand_reduction
        else:
            current_gap += intv["supply_delta_m3"]

        steps.append({
            "after_intervention": intv_id,
            "label_nl": intv["label_nl"],
            "cumulative_gap_m3": current_gap,
            "feasibility_class": _compute_single_feasibility_class(current_gap),
            "cumulative_cost_low": sum(
                i["cost_eur_low"]
                for i in INTERVENTION_CATALOGUE
                if i["id"] in intervention_ids[:intervention_ids.index(intv_id) + 1]
            ),
            "source_url": intv["source_url"],
            "source_label": intv["source_label"]
        })

    return {
        "baseline_gap_m3": baseline_results.supply_gap_m3,
        "final_gap_m3": current_gap,
        "final_feasibility_class": _compute_single_feasibility_class(current_gap),
        "achieves_go": current_gap >= 0,
        "stacking_steps": steps
    }

22. Error state catalogue

Every error state must produce a visible, non-technical Dutch message, a suggested next step, and a log entry. No error state should result in a blank screen or an unattributed failure.
Error code	Trigger	User message (NL)	Suggested action	Logged to
E001_NO_DATA	DuckDB returns 0 rows for spatial query	"Er zijn geen gegevens beschikbaar voor dit gebied in de geselecteerde datasets."	Broaden the area or choose a different location	MLflow + Insight panel
E002_GREENPT_TIMEOUT	GreenPT API call > 30s	"De parameterextractie duurde te lang. Probeer de vraag korter te formuleren."	Retry or rephrase	MLflow metric: greenpt_timeout=1
E003_GREENPT_LOW_CONFIDENCE	Confidence < 0.7 after extraction	"Ik heb de vraag niet helemaal begrepen. Bedoelt u [bevestigingskaart]?"	Show confirmation card with extracted params	Insight panel step
E004_SCHEMA_MISMATCH	Expected column missing after blocking check	"Een benodigde kolom ontbreekt in de dataset. De berekening gebruikt een terugvalwaarde."	Log missing column, use fallback assumption	MLflow artifact: schema_issues.json
E005_WATERINFO_UNAVAILABLE	Waterinfo unreachable AND no cache	"Live Waterinfo-data niet beschikbaar en geen gecachede meting gevonden. Chlorideberekening kan niet worden uitgevoerd."	Show warning; skip chloride calculation; proceed with capacity-only result	WaterinfoBanner + MLflow
E006_ASSUMPTION_MISSING_SOURCE	Assumption has empty source_url	Blocked at build time by AssumptionValidationError	Fix in code before deployment	CI gate
E007_ZONE_NOT_FOUND	Location geocodes but no leveringszone found	"De locatie kon niet worden gekoppeld aan een leveringszone. De dichtstbijzijnde zone wordt gebruikt."	Use nearest-zone fallback; label result with distance	Insight panel
E008_HUMAN_SCALE_MISSING	Metric key not in METRIC_CONVERSIONS	"Geen omrekening beschikbaar voor dit getal."	Show link to methodology page	MLflow metric: human_scale_missing_keys
E009_PDF_GENERATION_FAILED	fpdf2 error during PDF export	"PDF kon niet worden aangemaakt. Download de JSON in plaats daarvan."	Show JSON download fallback	Server log
E010_SCENARIO_NOT_FOUND	GET /scenario/{id} with unknown ID	"Scenario niet gevonden. Het scenario is mogelijk verlopen of het ID is onjuist."	Link to new scenario run	404 response
E011_DELTA_INCOMPATIBLE	Scenario A and B have different zone sets	"De twee scenario's zijn niet vergelijkbaar omdat ze betrekking hebben op verschillende zones."	Ask user to confirm or pick a comparison-compatible scenario	Insight panel follow-up
E012_DUCKDB_QUERY_FAILED	DuckDB runtime error	"De ruimtelijke berekening is mislukt. Probeer het opnieuw."	Retry with simplified query; if fails again, route to E001	MLflow + server log
23. README template

The following is the template for README.md in the repo root, to be completed before final submission.

Markdown

# OneGov #2 — Drinkwaterzekerheid Scenario Engine

**GovTechNL Hackathon · 4–5 juni 2026 · The Hague Tech**

Een beleidsmatige scenario-engine voor drinkwaterzekerheid in Zuid-Holland 2040,
gebouwd op de bestaande `onegov2-spatial-assistant` architectuur.

## Wat doet dit?

Stel een what-if vraag in het Nederlands — het systeem extraheert een scenario,
berekent de gevolgen, en geeft een citeerbare ScenarioCard terug met:
- 🟢🟡🔴 Haalbaarheidsklasse (GO / RISICO / NIET HAALBAAR)
- Kaartoverlays per leveringszone
- Aannames met bronvermeldingen (geen lege links)
- Stapsgewijs redeneerproces (navolgbaar)
- Stabiele URL + citaatblok voor adviesdocumenten

## Snelstart

### Vereisten
- Docker & Docker Compose
- Python 3.11+
- Node.js 20+
- GreenPT API-sleutel

### Installatie

```bash
git clone https://github.com/knarayanareddy/onegov2.git
cd onegov2
cp .env.example .env
# Vul GREENPT_KEY in .env
docker-compose up -d
```

### Data laden

```bash
cd src/backend
python scripts/load_data.py         # Laad standaard DuckDB-tabellen
python scripts/cache_external.py    # Pre-cache KNMI + Waterinfo
python scripts/blocking_checks.py  # Verifieer schema + H3-joins
```

### Applicatie starten

```bash
# Backend
cd src/backend && uvicorn main:app --reload --port 8000

# Frontend
cd src/frontend && npm install && npm run dev

# MLflow UI (optioneel)
mlflow ui --port 5000
```

Open `http://localhost:5173`

## Voorbeeldvragen

**Scenario A — Drop-a-pin:**
> "Kan er een 50 MW datacenter komen bij Pijnacker-Nootdorp in 2040?"

**Scenario B — Inname-uitval:**
> "Wat als de Hollandse IJssel 6 weken onbruikbaar is door verzilting, KNMI Hd, 2040?"

**Vergelijking:**
> Gebruik de split-kaart om scenario B met en zonder bufferopslag te vergelijken.

## Een scenario reproduceren

Elk scenario heeft een stabiele URL en citaatblok.
Gebruik de Scenario-ID om een berekening opnieuw op te halen:

```bash
curl http://localhost:8000/scenario/{scenario_id}
```

Of herbereken met actuele data:

```bash
curl -X POST http://localhost:8000/scenario/{scenario_id}/verify
```

## Architectuur

Zie `docs/architecture.md` voor het volledige diagram.

Kort samengevat:
- **Frontend:** Vue 3 + Vite + Pinia + Leaflet
- **Backend:** FastAPI + LangGraph + DuckDB + GreenPT
- **Traceerbaarheid:** MLflow tracking server
- **Reproductie:** DuckDB scenario cache + SHA-256 hash

## Databronnen

| Thema | Tabellen | Bron |
|---|---|---|
| Drinkwaterzekerheid | productieketen, leveringszones, zes_uur_zones | PDOK / DSO |
| Gebiedsviewer | verzilting_risico, krw_waterlichamen, e.a. | PDOK |
| LGN Landgebruik | lgn_grid | WUR / PDOK |
| CBS Buurten | cbs_buurt (optioneel) | CBS |
| KNMI Klimaat | Extern gecached | KNMI'23 |
| Waterinfo | Live API + cache | Rijkswaterstaat |

## Licentie
MIT — zie `LICENSE`

## Contact
hack@govtechnl.nl · GovTechNL OneGov #2

24. .env.example

Bash

# =============================================
# OneGov #2 — Drinkwaterzekerheid Scenario Engine
# Environment variables — copy to .env and fill in
# =============================================

# GreenPT LLM
GREENPT_KEY=your_greenpt_api_key_here
GREENPT_BASE_URL=https://api.greenpt.ai/v1
GREENPT_MODEL=gpt-4o

# MLflow
MLFLOW_TRACKING_URI=http://localhost:5001
MLFLOW_EXPERIMENT_NAME=onegov2-drinkwater-scenarios

# DuckDB
DUCKDB_PATH=./data/onegov2.duckdb
DUCKDB_SCENARIO_CACHE_TTL_DAYS=7

# Waterinfo API (Rijkswaterstaat)
WATERINFO_API_ENDPOINT=https://waterwebservices.rijkswaterstaat.nl/ONLINEWAARNEMINGENSERVICES_DBO/OphalenWaarnemingen
WATERINFO_CACHE_PATH=./data/external/waterinfo_cache.json
# Mandatory intake IDs for chloride guard
WATERINFO_MANDATORY_INTAKES=IJssel_Gouda,Lek_Bergambacht,Maas_Brakel

# PDOK Geocoder
PDOK_LOCATIESERVER_URL=https://api.pdok.nl/bzk/locatieserver/search/v3_1/free

# Scenario stable URL base
SCENARIO_STABLE_URL_BASE=http://localhost:8000

# Determinism
RANDOM_SEED=42
NUMPY_SEED=42

# Application
APP_ENV=development
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:5173

# Git commit (auto-populated in CI; set manually for local dev)
GIT_COMMIT=local-dev

25. Build plan with file-level task assignments
Day 1 — June 4, 2026
Time	Task	File(s) to create/modify	Owner	Done when
09:00–09:30	Run blocking checks	scripts/blocking_checks.py	Backend	"All checks passed" printed
09:30–10:00	Write all dataclasses	src/backend/models/scenario.py, src/backend/models/assumptions.py	Backend	Imports resolve, no errors
10:00–10:30	Wire AgentState + LangGraph skeleton	src/backend/workflow/graph.py	Backend	Graph compiles with all nodes as stubs
10:30–11:30	extract_scenario_params node + GreenPT prompt	src/backend/nodes/extract_scenario_params.py	Backend	Golden Path A params extracted correctly
11:30–12:30	fetch_scenario_data + Waterinfo guard	src/backend/nodes/fetch_scenario_data.py, src/backend/integrations/waterinfo_client.py	Backend	Waterinfo called; fallback warning shown
12:30–13:00	Lunch + schema verify		All	
13:00–14:00	run_scenario_calculation — demand + capacity	src/backend/calculators/demand.py, src/backend/calculators/capacity.py	Backend	Golden Path A returns point estimate + range
14:00–14:30	compute_feasibility_class across KNMI presets	src/backend/calculators/feasibility.py	Backend	GO/CAUTION/STOP returned for all 5 presets
14:30–15:00	Scenario hash + cache store	src/backend/utils/scenario_hash.py, src/backend/cache/scenario_store.py	Backend	Hash computed; cache write + read verified
15:00–15:30	POST /scenario/run endpoint + SSE streaming	src/backend/routers/scenario_router.py	Backend	Endpoint streams events to curl test
15:30–16:30	Frontend SSE wiring + useScenarioStore	src/frontend/stores/scenarioStore.ts, src/frontend/composables/useScenarioSSE.ts	Frontend	Scenario card appears in browser
16:30–17:00	FeasibilityBadge.vue + AssumptionSliders.vue	src/frontend/components/FeasibilityBadge.vue, src/frontend/components/AssumptionSliders.vue	Frontend	Badge renders; sliders trigger re-run
17:00–17:30	Golden Path A end-to-end smoke test	All	All	Demo-able: pin → card → feasibility
17:30–18:30	Chloride calculation + per-intake threshold	src/backend/calculators/chloride.py	Backend	Golden Path B chloride computed with per-intake threshold
18:30–19:30	KennisbasisPanel.vue + /kennisbasis/status	src/frontend/components/KennisbasisPanel.vue, src/backend/routers/kennisbasis_router.py	Frontend + Backend	Panel shows loaded tables + freshness
19:30–20:00	CitationBlock.vue + citation builder	src/frontend/components/CitationBlock.vue, src/backend/utils/citation_builder.py	Frontend + Backend	APA citation renders; copy button works
20:00	Day 1 freeze		All	Both golden paths demo-able; citation block visible
Day 2 — June 5, 2026
Time	Task	File(s) to create/modify	Owner	Done when
09:00–09:30	OfficialPositionPanel.vue + registry	src/frontend/components/OfficialPositionPanel.vue, src/backend/registries/official_position_registry.py	Frontend + Backend	Panel shows 3 policy links
09:30–10:30	Comparison mode: scenario_b + ScenarioDeltaPanel.vue	src/frontend/components/ScenarioDeltaPanel.vue, extend scenario_router.py	Full stack	Split map + delta panel renders
10:30–11:00	MakeItFeasiblePanel.vue + intervention ranker	src/frontend/components/MakeItFeasiblePanel.vue, src/backend/calculators/intervention_ranker.py	Full stack	Top-3 interventions shown on STOP scenario
11:00–11:30	HumanScaleRef.vue + converter	src/frontend/components/HumanScaleRef.vue, src/backend/utils/human_scale.py	Full stack	Every headline metric has analogy + source
11:30–12:00	Citizen mode detection + CitizenResponseCard.vue	src/backend/utils/citizen_detection.py, src/frontend/components/CitizenResponseCard.vue	Full stack	Citizen question returns verdict-first card
12:00–12:30	CumulativeLoadWarning.vue + cumulative node	src/frontend/components/CumulativeLoadWarning.vue, src/backend/nodes/cumulative_load_node.py	Full stack	Warning shown for zone with active scenarios
12:30–13:00	Lunch		All	
13:00–13:30	ProductionChainFlow.vue (directed flow panel)	src/frontend/components/ProductionChainFlow.vue	Frontend	Intake → production → zone renders for Path B
13:30–14:00	WaterinfoBanner.vue + Waterinfo store	src/frontend/components/WaterinfoBanner.vue, src/frontend/stores/waterinfoStore.ts	Frontend	Banner shows live vs cached with date
14:00–14:30	MapTitle.vue + always-visible map header	src/frontend/components/MapTitle.vue, extend MapView.vue	Frontend	Map title updates reactively with scenario
14:30–15:00	PDF export endpoint	src/backend/routers/scenario_router.py (add /pdf)	Backend	PDF downloads with correct citation block
15:00–15:30	MLflow logging complete + readability metric	Extend all nodes with mlflow.log_* calls	Backend	MLflow UI shows all runs with metrics
15:30–16:00	Demo rehearsal — 3-minute script		All	3 "wow moments" land cleanly
16:00–16:30	README + .env.example finalized	README.md, .env.example	All	README reproduces a scenario from scratch
16:30–17:00	Open source checklist + final push	LICENSE, README.md, final git push	All	Public repo accessible; submission link ready
26. Validation & acceptance criteria (pre-submission checklist)
Must — brief requirement (minimum viable submission)

    Working prototype translates NL policy question into computed scenario
    ≥ 2 scenario types demonstrated (Type 3 + Type 1)
    Data, assumptions, and impacted parties explicit in output
    Open source with readable README
    Uses provided datasets; no external APIs for core calculation
    Combines ≥ 2 data themes
    Existing descriptive workflow still works

Should — design doc requirements

    Comparison mode working: split map + delta panel
    Assumption sliders adjustable and trigger re-run
    Reasoning chain traceable: Insight panel + MLflow
    All source_url fields populated (CI gate blocks empty strings)
    Human-scale analogies on all headline metrics
    FeasibilityClass computed across all 5 KNMI presets
    OnsetYearCalculator steps from 2024 → 2040

Stakeholder gaps — v2 requirements

    GAP 1: Kennisbasis panel present, queryable, shows freshness
    GAP 2: Scenario hash + stable URL + dataset version logging + version-drift warning
    GAP 3: Citation block with APA string + PDF download + copy button
    GAP 4: HumanScaleRef on every headline metric; no empty source URLs
    GAP 5: "Make it feasible" intervention ranking on STOP/CAUTION
    GAP 6: Cumulative load warning for same-zone scenarios
    GAP 7: Per-intake chloride threshold from productieketen; fallback explicitly labelled
    GAP 8: Official position panel with ≥ 2 policy document links
    GAP 9: Waterinfo mandatory for IJssel/Lek/Maas; never silent fallback
    GAP 10: Citizen response template: verdict-first, postcode, water company, disclaimer

Domain correctness

    Chloride threshold is per-intake with source; fallback is slider-adjustable
    Waterinfo call attempted before every intake scenario; fallback never silent
    Production chain shows directionality + capacity in directed flow panel
    All demand figures shown as ranges; labelled "beleidsmatige schatting"
    Datacenter demand default is midrange (12 m³/day/MW), slider range 5–20 shown

Reproducibility

    Same params → same scenario_hash → same cached result
    Dataset version drift warning shown on cached scenarios
    "Verify calculation" button present and functional
    RANDOM_SEED=42 and NUMPY_SEED=42 set in .env.example
    git_commit logged in every ScenarioCard

Usability

    All output text in B1 Dutch
    No scenario returns a blank screen (all errors produce Dutch message + next step)
    Map always shows title and legend
    FeasibilityBadge visible before any numbers
    Demo runs without internet (all external data pre-cached)
