💧 Drinkwaterzekerheid Zuid-Holland 2040 — Complete Design Document (Repo-Perfect Edition)
📋 TABLE OF CONTENTS
Strategic Framing
Who This Is For
Scenario Philosophy & Interaction Model
Full Data Inventory & Source Registry
Prototype Architecture
Extended LangGraph Workflow
Scenario Engine & Calculation Design
Map Visualisation & Stakeholder Views
Traceability & Navolgbaarheid Framework
GreenPT Integration
Full Data Model
Step-by-Step Build Plan
Validation & Quality Framework
Pitch Narrative
Repo Structure
Risk Register
<a name="strategic-framing"></a>

1. 🎯 Strategic Framing
What this system is
This is a spatial scenario engine — not a chatbot, not a dashboard, not a map viewer alone.

A policymaker, urban planner, water board official, or citizen types a question in plain Dutch. The system:

Understands the intent,
Translates it into a structured, calculable scenario,
Runs the calculation against authoritative open data,
Shows the results on a map, differentiated per stakeholder group,
Makes every assumption, every dataset, and every reasoning step explicitly traceable to its source via hyperlink,
And lets the user adjust parameters interactively and see the map update in real time.
The output is never just text. It is always a Scenario Card + Map + Reasoning Trail + Source List.

The central distinction
A chatbot (wrong)	A scenario engine (right)
Answers in prose	Produces a structured, replayable Scenario object
Output is text	Output is data + map + explanation + stakeholder impact
Assumptions implicit	Assumptions explicit, sourced, and adjustable
No audit trail	Every step logged, traceable, hyperlinked
Black box	Insight panel + MLflow trace
Hard to compare	Two scenarios shown side by side on a split map
Talks about location	Shows location — interactive map, stakeholder overlays
Why this matters for Zuid-Holland 2040
Zuid-Holland faces three simultaneous pressures that individually are manageable but together are not:

Climate drying (KNMI'23 Klimaatscenario's): the Hd scenario predicts significantly longer drought periods and increased salinization of river intakes by 2040.
KRW deadlines (Helpdesk Water KRW): the EU Water Framework Directive requires good ecological and chemical status of water bodies by 2027 — several South Holland intake points are at risk.
Housing and economic growth (Ruimtelijk Arrangement ZH): 230,000 new homes plus new data centres, logistics hubs, and commercial zones all require reliable water supply. Each land-use decision is also a water security decision.
No current tool lets a policymaker ask "what happens to drinking water security if I approve a data centre here?" and get a structured, traceable answer on a map. This system does exactly that.

<a name="audience"></a>

2. 👥 Who This Is For
The system must work for all of the following users — not just technical analysts:

User	Needs	System response
Provincial policymaker	"Is this housing plan safe to approve?"	Scenario card + map showing supply gap per zone
Water board official (Hollands Noorderkwartier, Rijnland, Delfland, Schieland)	"Which intake points fail first under drought?"	KRW compliance map + risk onset year per location
Municipality spatial planner	"Can we place a data centre here?"	Impact overlay on the map + water demand delta
Drinkwaterbedrijf engineer (Dunea, Evides, PWN)	"What does buffer capacity buy us?"	Side-by-side comparison: with/without intervention
Citizen or journalist	"Why is there a water shortage risk?"	Plain Dutch explanation with linked sources
Auditor / Woo verzoek	"How did you arrive at this conclusion?"	Full reasoning trail in Insight panel + MLflow
Design principles that follow from this
Plain Dutch by default — every output in B1 Nederlands. Technical terms always paired with a plain explanation.
Map first — spatial context before numbers. The map is the primary communication surface.
Drill-down, not front-load — summary card visible immediately; detail available on click.
Every claim linked — every number, every assumption, every dataset name is a hyperlink to its authoritative source.
Stakeholder toggle — the same scenario can be viewed through the lens of different affected parties; the map changes accordingly.
No dead ends — if the system cannot answer, it says so, explains why, and offers a refinement question.
<a name="scenario-philosophy"></a>

3. 🗺️ Scenario Philosophy & Interaction Model
The three scenario types the system handles
The brief's docs/example-scenarios.md defines three guiding what-if questions. The system handles all three, plus a fourth class that the brief invites ("space for additions"):

Type 1 — Climate × Infrastructure failure
"Wat als de Hollandse IJssel 6 weken onbruikbaar is door verzilting in het Hd-scenario?"

Parameters: climate axis (KNMI'23), intake failure location, duration, cause. Map output: intake zones in red, alternative supply routes in orange, safe zones in green. Stakeholders: woningzoekenden, netbeheerders, zorginstellingen, agrariërs.

Type 2 — Cumulative pressure (multi-hazard)
"Welke combinatie van klimaatdruk en bevolkingsgroei bedreigt de drinkwaterzekerheid het eerst?"

Parameters: climate axis, population/housing growth scenario, time horizon. Map output: risk onset year choropleth per gemeente/buurt. Stakeholders: gemeenten, drinkwaterbedrijven, province.

Type 3 — Spatial land-use decision (new — this is the policymaker's key use case)
"Wat is het effect op de drinkwaterzekerheid als ik hier een datacenter / woningbouwproject / bedrijventerrein plan?"

Parameters: location (click on map OR type address), development type, scale. Map output: water demand increase overlaid on current supply capacity; risk delta per supply zone. Stakeholders: developer, gemeente, drinkwaterbedrijf, downstream users.

Type 4 — Intervention effectiveness
"Wat levert bufferopslagcapaciteit op in vergelijking met een alternatief innamepunt?"

Parameters: intervention type(s), scale, deployment year. Map output: before/after supply gap per zone; residual risk locations. Stakeholders: water boards, province, housing developers.

The interaction model (how a user moves through the system)
text

User types a question in plain Dutch (or clicks a pre-built scenario tile)
    ↓
System classifies intent and extracts scenario parameters
    ↓
Clarifying follow-up if intent is ambiguous (existing workflow behaviour)
    ↓
Scenario parameters shown to user for confirmation:
  "Ik begrijp uw vraag als: klimaat = Hd, groei = middel, 
   locatie = Hollandse IJssel innamepunt, duur = 6 weken. 
   Klopt dit? [Ja] [Aanpassen]"
    ↓
User confirms OR adjusts via sliders/dropdowns
    ↓
Calculation runs (< 10 seconds)
    ↓
Map updates: stakeholder overlay + supply gap + risk zones
    ↓
Scenario Card appears: numbers, confidence, assumptions, sources
    ↓
User can:
  - Switch stakeholder view (dropdown)
  - Adjust a parameter (slider → map updates live)
  - Add a second scenario for side-by-side map comparison
  - Click any number/assumption → hyperlinked source opens
  - View full reasoning trail in Insight panel
  - Export scenario as PDF / JSON
For Type 3 (spatial land-use decision): the "drop a pin" interaction
This is the most distinctive feature of this system and the one no other hackathon team will have:

text

User clicks on a location on the map (or types "Pijnacker-Nootdorp")
    ↓
User selects development type from a dropdown:
  🏗️ Woningbouwproject (housing)
  🖥️ Datacenter
  🏭 Bedrijventerrein (industrial)
  ⚡ Energiecentrale
  🌱 Glastuinbouw uitbreiding (greenhouse farming)
    ↓
User sets scale (number of homes / MW of server capacity / hectares)
    ↓
System calculates additional water demand for that development type
    ↓
System overlays: additional demand ON TOP OF current supply capacity
    ↓
Map shows: "This location adds X m³/day to a supply zone already at Y% capacity"
    ↓
System answers: "This development is [safe / borderline / problematic] for 
drinkwaterzekerheid in the [Ld / Hd / Hn / Ln] scenario"
This directly serves the policymaker's actual workflow — approving or rejecting spatial plans — and makes the tool immediately useful, not just demonstrably interesting.

<a name="data-inventory"></a>

4. 🗄️ Full Data Inventory & Source Registry
Critical design rule: every dataset used in a scenario is hyperlinked in the output
Every Scenario Card, every Assumption, every number shown in the UI must carry a clickable link to the authoritative source. This is non-negotiable for government use.

4.1 — Default loaded themes (already in the repo)
These are available in DuckDB from day 0. Use these first.

Theme 1: Drinkwaterzekerheid
Source: Provincie Zuid-Holland — Gebiedsviewer + PDOK

Table/Layer	Contents	Used for
drinkwaterzekerheid_productieketen	Intake points, treatment plants, distribution zones, capacity per location	Node 5 capacity calculation
drinkwaterzekerheid_bedrijven	Drinkwaterbedrijven per zone (Dunea, Evides, PWN, Oasen)	Stakeholder attribution
drinkwaterzekerheid_zuurzone	Six-hour emergency supply zones	Risk onset mapping
drinkwaterzekerheid_waterschappen	Water board boundaries	Governance attribution
Theme 2: Gebiedsviewer (~50 provincial layers)
Source: Provincie Zuid-Holland Open Data

Layer	Contents	Used for
gebiedsviewer_verzilting	Salinization risk per water body	Cl⁻ concentration modelling
gebiedsviewer_bodemdaling	Subsidence risk	Infrastructure vulnerability
gebiedsviewer_overstromingskwetsbaarheid	Flood vulnerability	Compound risk scenarios
gebiedsviewer_natuur	Nature networks (Natura 2000, NNN)	Stakeholder: nature organisations
gebiedsviewer_landgebruik	Land use classification	Development type placement
4.2 — Optional/extra loaded themes (available in extra_data/)
These should be loaded per scenario, not all at once (brief's Should-not: avoid prompt bloat).

CBS Vierkantstatistieken
Source: CBS StatLine — Kerncijfers wijken en buurten

Table	Contents	Used for
cbs_vierkant_bevolking	Population per 100m grid cell	Housing growth demand
cbs_vierkant_woningen	Housing per 100m grid cell	Development baseline
Woondeals / Ruimtelijk Arrangement
Source: Rijksoverheid — Ruimtelijk Arrangement ZH

Table	Contents	Used for
woondeals_capaciteitskaart_afname	Regional housing growth targets	Population growth axis
woondeals_locaties	Planned housing locations	Type 3 scenario baseline
LGN Land Use Rasters
Source: WUR — Landelijk Grondgebruiksbestand Nederland

Table	Contents	Used for
lgn_2022	Detailed land use per 25m cell	Development type classification
4.3 — External sources (loaded per scenario with caching)
These must be fetched and cached before the hackathon. Every value derived from them must carry a hyperlink.

KNMI'23 Klimaatscenario's
Source: klimaatscenarios.knmi.nl License: CC BY

Scenario	Code	Drought multiplier	Cl⁻ delta (mg/L)	Peak demand multiplier	Notes
Hoog-droog	Hd	+2.1×	+45	+1.25	Worst case by 2050
Hoog-nat	Hn	+0.8×	-10	+0.95	High emissions, wet variant
Laag-droog	Ld	+1.5×	+25	+1.12	Moderate warming, dry
Laag-nat	Ln	+0.7×	-15	+0.92	Lower emissions, wet
Baseline 2025	B	1.0×	0	1.0	Current observed situation
⚠️ Design rule: do not use the old W+/G/W codes. Always use KNMI'23 Hd/Hn/Ld/Ln. If a user asks "droog scenario," map it to Hd as default and say so explicitly as an assumption.

KRW Monitoring
Source: Waterinfo RWS + Helpdesk Water KRW + AquaDesk API

Data	Contents	Used for
KRW toestand per waterlichaam	Ecological/chemical status per body	KRW compliance flag per intake
Chloride monitoring	Cl⁻ time series per measurement point	Salinization threshold validation
KRW 2027 deadline status	Which bodies are at risk	Compliance risk output
Water Demand per Development Type
Source: VEWIN — Watergebruik in Nederland + RVO — Handboek Water

Development type	Daily water demand	Source link
Woningbouw (per woning)	~0.35 m³/dag	VEWIN statistieken
Datacenter (per MW IT-load)	~5–15 m³/dag (cooling)	IEA Data Centres report
Bedrijventerrein (per ha)	~8–25 m³/dag	RVO Watergebruik bedrijven
Glastuinbouw (per ha)	~15–30 m³/dag	LTO/WUR glastuinbouw water
Energiecentrale (per MW)	~2–8 m³/dag	CBS Energiestatistieken
Every value shown in the UI must link to its source row in this table.

4.4 — Source Registry Object (appended to every Scenario Card)
Python

@dataclass
class SourceRegistry:
    """
    Every scenario card carries a complete, hyperlinked source registry.
    No number in the UI is ever unsourced.
    """
    sources: list[DataSourceEntry]

@dataclass
class DataSourceEntry:
    source_id: str
    name: str
    organisation: str
    url: str                    # Always a direct hyperlink
    layer_or_table: str
    fields_used: list[str]
    license: str
    accessed_date: str
    publication_date: str | None
    freshness_warning: bool     # True if > 12 months old
    used_in_steps: list[int]    # Which reasoning steps used this source
<a name="architecture"></a>

5. 🏛️ Prototype Architecture
Guiding constraint: extend the existing repo — do not replace it
The starting point is govtechnl/onegov2-spatial-assistant, which already provides:

Vue 3 frontend with a chat interface and map
FastAPI backend with DuckDB integration
A working LangGraph workflow (intent → SQL → map)
Insight panel for reasoning chain display
MLflow at http://localhost:5001 for experiment/step tracing
GreenPT wired as default LLM provider
Two default data themes loaded
Everything new in this design is added on top of that foundation.

text

┌──────────────────────────────────────────────────────────────────────────────┐
│                      FRONTEND (Vue 3 — existing + extended)                  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  EXISTING: Chat interface + Map view + Insight panel                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────┐  ┌─────────────────────┐  ┌────────────────────────┐   │
│  │ NEW:           │  │ NEW:                │  │ NEW:                   │   │
│  │ ScenarioCard   │  │ MapStakeholder      │  │ DropAPin               │   │
│  │ .vue           │  │ Toggle.vue          │  │ Panel.vue              │   │
│  └────────────────┘  └─────────────────────┘  └────────────────────────┘   │
│                                                                              │
│  ┌────────────────┐  ┌─────────────────────┐  ┌────────────────────────┐   │
│  │ NEW:           │  │ NEW:                │  │ EXISTING + extended:   │   │
│  │ Assumption     │  │ ScenarioComparison  │  │ Insight panel          │   │
│  │ Sliders.vue    │  │ Panel.vue           │  │ (+ scenario steps)     │   │
│  └────────────────┘  └─────────────────────┘  └────────────────────────┘   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ NEW: SourceRegistryPanel.vue — hyperlinked sources for every number    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────┬───────────────────────────────────┘
                                           │ HTTPS REST (existing)
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      BACKEND (FastAPI — existing + extended)                 │
│                                                                              │
│  EXISTING endpoints kept intact:                                             │
│  POST /chat, GET /map-data, GET /insight                                    │
│                                                                              │
│  NEW endpoints added:                                                        │
│  POST /scenario/run          ← main new endpoint                            │
│  POST /scenario/compare      ← side-by-side                                 │
│  GET  /scenario/{id}         ← retrieve saved scenario                      │
│  GET  /scenario/{id}/sources ← hyperlinked source registry                  │
│  POST /scenario/drop-pin     ← Type 3: spatial decision support             │
│  POST /assumptions/adjust    ← re-run with modified parameters              │
└──────────────────────────────────────────┬───────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              EXTENDED LANGGRAPH WORKFLOW (existing graph + new nodes)        │
│                                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐                  │
│  │ check_intent │──►│resolve_spatial│──►│validate_filter│  EXISTING        │
│  └──────────────┘   └──────────────┘   └───────┬───────┘  NODES           │
│                                                 │                           │
│                            ┌────────────────────┘                           │
│                            ▼                                                 │
│              ┌─────────────────────────┐                                    │
│              │ IS THIS A SCENARIO      │  ← branch added here               │
│              │ (what-if) QUESTION?     │                                    │
│              └──────┬──────────┬───────┘                                    │
│                     │ YES      │ NO                                          │
│                     ▼          ▼                                             │
│         ┌─────────────────┐  ┌────────────────┐                            │
│         │ NEW SCENARIO    │  │ EXISTING path: │  EXISTING                  │
│         │ BRANCH (below)  │  │ generate_sql → │  NODES                     │
│         └────────┬────────┘  │ execute_query →│                            │
│                  │           │ plan_viz →      │                            │
│                  │           │ describe_results│                            │
│                  │           └────────────────┘                            │
│                  ▼                                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    NEW SCENARIO BRANCH NODES                          │  │
│  │                                                                       │  │
│  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │  │
│  │  │ extract_scenario │──►│ fetch_scenario   │──►│ build_scenario   │  │  │
│  │  │ _params          │   │ _data            │   │ _object          │  │  │
│  │  └──────────────────┘   └──────────────────┘   └────────┬─────────┘  │  │
│  │                                                          │             │  │
│  │  ┌──────────────────┐   ┌──────────────────┐   ┌────────▼─────────┐  │  │
│  │  │ format_scenario  │◄──│ record_reasoning │◄──│ run_scenario     │  │  │
│  │  │ _output          │   │ _steps           │   │ _calculation     │  │  │
│  │  └─────────┬────────┘   └──────────────────┘   └──────────────────┘  │  │
│  │            │                                                            │  │
│  │            ▼                                                            │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │ stakeholder_impacts (feeds into map overlays per stakeholder)    │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  All new nodes feed their steps into:                                        │
│  ✅ EXISTING Insight panel (via stream output)                               │
│  ✅ EXISTING MLflow trace at http://localhost:5001                           │
└──────────────────────────────────────────┬───────────────────────────────────┘
                                           │
               ┌───────────────────────────┼──────────────────┐
               ▼                           ▼                  ▼
    ┌─────────────────┐        ┌────────────────┐   ┌────────────────────┐
    │ DuckDB          │        │ GeoJSON        │   │ Scenario Store     │
    │ (existing       │        │ + Leaflet      │   │ (JSON/Parquet)     │
    │ datacube +      │        │ overlay cache  │   │                    │
    │ extra_data/)    │        │                │   │                    │
    └─────────────────┘        └────────────────┘   └────────────────────┘
<a name="workflow"></a>

6. 🔄 Extended LangGraph Workflow
The existing workflow (do not change these nodes)
Based on docs/workflow.mmd:

text

check_intent
  → (if spatial) resolve_spatial
  → validate_filters
  → generate_sql
  → execute_query
  → plan_visualization
  → describe_results
  → (if ambiguous) follow_up_question
These nodes handle the descriptive ("what IS here?") half of the tool. They remain unchanged.

The new scenario branch (added after validate_filters)
After validate_filters, a routing check determines whether the question is a scenario/what-if type or a descriptive/factual type. If it is a scenario question, the new branch runs. If not, the existing SQL path runs.

Python

# routing logic added to validate_filters node
def route_after_validation(state: WorkflowState) -> str:
    """
    Determines which path to take after validation.
    Returns "scenario" or "descriptive"
    """
    scenario_indicators = [
        "wat als", "wat gebeurt er als", "stel dat",
        "effect van", "impact van", "gevolg van",
        "vergelijk", "scenario", "what if", "hoeveel",
        "welke combinatie", "wanneer",
    ]
    question_lower = state.user_question.lower()
    if any(ind in question_lower for ind in scenario_indicators):
        return "scenario"
    return "descriptive"
New Node 1: extract_scenario_params
Python

"""
Input:  validated user question + spatial context from existing nodes
Output: ScenarioParameters object

Responsibilities:
- Uses GreenPT (structured output mode) to extract:
  * question_type: "climate_failure" | "cumulative_pressure" 
                   | "spatial_decision" | "intervention_analysis"
  * climate_scenario: "Hd" | "Hn" | "Ld" | "Ln" | "B" (baseline 2025)
    → if user says "droog" → map to "Hd", log as assumption
    → if unspecified → default "Hd" (worst case), log as assumption
  * population_growth: "laag" | "middel" | "hoog"
  * time_horizon: int (default: 2040)
    → always framed as "2025 baseline vs {year} projection"
  * intake_failure_location: str | None
  * intake_failure_duration_weeks: int | None
  * intervention_set: list[str]
  * spatial_location: GeoPoint | None  (for Type 3)
  * development_type: str | None       (for Type 3)
  * development_scale: float | None    (for Type 3)

- If confidence < 0.75 → return clarifying_question to user
- Shows extracted parameters to user for confirmation BEFORE running calculation
- All defaults logged as Assumption objects with source = 
  "[KNMI'23 Hd scenario — worst case default]" 
  (hyperlinked to https://klimaatscenarios.knmi.nl/)

MLflow: logs extracted parameters as named run parameters
Insight panel: shows "Uw vraag is geïnterpreteerd als: ..."
"""
New Node 2: fetch_scenario_data
Python

"""
Input:  ScenarioParameters
Output: ScenarioDataBundle

CRITICAL RULE: Only load the columns/tables relevant to this scenario.
Never load all 50 gebiedsviewer layers at once (brief's Should-not).

Responsibilities:
- Query DuckDB for:
  * Water infrastructure records for affected supply zone
    Source: drinkwaterzekerheid_productieketen
    Hyperlink: https://opendata.zuid-holland.nl/ (logged in SourceRegistry)
  * Verzilting risk per intake location (if climate scenario selected)
    Source: gebiedsviewer_verzilting
    Hyperlink: https://opendata.zuid-holland.nl/
  * KRW compliance status per intake point
    Source: KRW monitoring via https://www.helpdesk-water.nl/KRW
  * KNMI scenario parameters for selected climate_scenario
    Source: https://klimaatscenarios.knmi.nl/
  * Population growth parameters (if growth axis selected)
    Source: Woondeals/CBS — https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH
  * For Type 3 (spatial decision):
    - Land use at the selected location (gebiedsviewer_landgebruik)
    - Supply zone for the selected location (drinkwaterzekerheid_productieketen)
    - Development type water demand from reference table
      Source: VEWIN/RVO (hyperlinked per type)

- Records every dataset in SourceRegistry with hyperlink
- Flags any data older than 12 months as freshness_warning = True
- Logs data sources as MLflow artifacts

MLflow: logs list of datasets loaded as run tags
Insight panel: shows "Databronnen opgehaald: [list with links]"
"""
New Node 3: build_scenario_object
Python

"""
Input:  ScenarioParameters + ScenarioDataBundle
Output: Scenario object (without results — those come from Node 4)

Responsibilities:
- Instantiate the Scenario dataclass
- Build Assumption list:
  * For every default value used: record as Assumption with
    - description in plain Dutch
    - value + unit
    - source URL (hyperlink — non-negotiable)
    - sensitivity (hoog/middel/laag)
    - adjustable: True (goes to sliders in UI)
  * Example assumptions for Hd + Hollandse IJssel failure:
    - Cl⁻ threshold: 150 mg/L 
      [Drinkwaterbesluit, Art. 18 — https://wetten.overheid.nl/]
    - Baseline demand: 760,000 m³/dag 
      [VEWIN Waterstatistieken 2023 — https://www.vewin.nl/publicaties/]
    - Peak summer multiplier: 1.25 (Hd scenario) 
      [KNMI'23 — https://klimaatscenarios.knmi.nl/]
    - Housing growth: 230,000 units by 2040 
      [Ruimtelijk Arrangement ZH — https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH]

- For Type 3 scenarios: add development-specific water demand assumption
  with VEWIN/RVO source hyperlink

MLflow: logs assumption count + high-sensitivity count as metrics
Insight panel: shows "Aannames vastgesteld: X aannames, 
  waarvan Y met hoge gevoeligheid [bronnen]"
"""
New Node 4: run_scenario_calculation
Python

"""
Input:  Scenario object (partial)
Output: ScenarioResults object

Responsibilities:
- Calculate projected daily demand (2025 → target year)
- Calculate available production capacity
- For Type 1 (intake failure): remove intake-specific capacity
- For Type 3 (spatial decision): add development-type demand delta
- Check Cl⁻ concentration vs threshold
- Determine supply gap + supply_secure flag
- Scan for risk onset year (2025–2040)
- Check KRW compliance per location
- All intermediate values logged with source hyperlinks

MLflow: logs supply_gap_m3, supply_secure, risk_onset_year as metrics
Insight panel: shows calculation steps with numbers and sources
"""
New Node 5: stakeholder_impacts
Python

"""
Input:  Scenario + ScenarioResults
Output: list[StakeholderImpact] — one per stakeholder group

Each impact includes:
- stakeholder_type (name in Dutch)
- impact_level ("hoog" | "middel" | "laag" | "geen")
- impact_direction ("negatief" | "neutraal" | "positief")
- description_plain_dutch (B1 level, written by GreenPT)
- specific_risks (bulleted, max 3)
- specific_opportunities (bulleted, max 2)
- requires_action (bool)
- action_description | None
- map_layer_id: str  ← which map overlay to activate for this stakeholder
- source_url: str    ← policy basis for this stakeholder's inclusion
  (e.g., WRO, Drinkwaterwet, WBB — always hyperlinked)

For Type 3 (spatial decision), adds:
- DEVELOPER stakeholder (vergunningaanvrager)
- DOWNSTREAM USERS (users of the same supply zone)

MLflow: logs stakeholder impact summary as JSON artifact
Insight panel: shows "Effecten per stakeholder: [collapsible list]"
"""
New Node 6: record_reasoning_steps
Python

"""
Input:  All previous node state
Output: list[ReasoningStep] — the navolgbaar audit trail

ALL output goes into:
  1. The EXISTING Insight panel (primary "navolgbaarheid" surface)
  2. The EXISTING MLflow trace at http://localhost:5001
  3. The Scenario Card's "Onderbouwing" (reasoning) tab in the UI

Each step is written in plain Dutch by GreenPT:
  - Temperature: 0.1 (low — for consistency, not creativity)
  - Prompt: "Schrijf een stap in een beleidsrapport in B1 Nederlands. 
    Gebruik alleen de informatie uit de meegeleverde context. 
    Voeg geen feiten toe. Maximaal 50 woorden per stap."
  - Every step includes: datasets used (with hyperlinks), 
    assumptions applied (with hyperlinks), confidence level

MLflow: logs step count + timestamps as run metadata
"""
New Node 7: format_scenario_output
Python

"""
Input:  Complete Scenario + Results + Impacts + Steps
Output: ScenarioCard JSON (frontend-ready) + GeoJSON overlays

Responsibilities:
- Format Scenario Card with all fields
- Build GeoJSON for each map overlay:
  * supply_gap_overlay: coloured by zone status
  * stakeholder_overlay_{type}: per-stakeholder impact layer
  * krw_risk_overlay: KRW compliance per location
  * Type 3: development_impact_overlay (demand delta ring)
- Attach SourceRegistry with hyperlinks to all sources
- Generate comparison delta if second scenario provided

Output fed to:
  - /scenario/{id} API endpoint
  - Frontend ScenarioCard.vue
  - Frontend MapStakeholderToggle.vue
"""
<a name="scenario-engine"></a>

7. ⚙️ Scenario Engine & Calculation Design
7.1 — Climate axis (KNMI'23 — aligned to official scenarios)
Source: KNMI'23 Klimaatscenario's

Python

KNMI23_SCENARIOS = {
    "Hd": KNMIScenario(
        code="Hd",
        name="Hoog-droog",
        description="Hoge uitstoot, droge variant — worst case voor drinkwater",
        source_url="https://klimaatscenarios.knmi.nl/",
        drought_frequency_multiplier=2.1,
        cl_concentration_delta_mg_l=45,
        peak_demand_summer_multiplier=1.25,
        low_flow_duration_weeks_extra=6,
    ),
    "Hn": KNMIScenario(
        code="Hn",
        name="Hoog-nat",
        source_url="https://klimaatscenarios.knmi.nl/",
        drought_frequency_multiplier=0.8,
        cl_concentration_delta_mg_l=-10,
        peak_demand_summer_multiplier=0.95,
        low_flow_duration_weeks_extra=-3,
    ),
    "Ld": KNMIScenario(
        code="Ld",
        name="Laag-droog",
        source_url="https://klimaatscenarios.knmi.nl/",
        drought_frequency_multiplier=1.5,
        cl_concentration_delta_mg_l=25,
        peak_demand_summer_multiplier=1.12,
        low_flow_duration_weeks_extra=3,
    ),
    "Ln": KNMIScenario(
        code="Ln",
        name="Laag-nat",
        source_url="https://klimaatscenarios.knmi.nl/",
        drought_frequency_multiplier=0.7,
        cl_concentration_delta_mg_l=-15,
        peak_demand_summer_multiplier=0.92,
        low_flow_duration_weeks_extra=-4,
    ),
    "B": KNMIScenario(
        code="B",
        name="Basislijn 2025",
        description="Huidige situatie — geen klimaatverandering toegepast",
        source_url="https://www.vewin.nl/publicaties/watergebruiksstatistieken",
        drought_frequency_multiplier=1.0,
        cl_concentration_delta_mg_l=0,
        peak_demand_summer_multiplier=1.0,
        low_flow_duration_weeks_extra=0,
    ),
}
7.2 — Population growth axis
Source: Ruimtelijk Arrangement ZH + CBS Bevolkingsprognose

Python

POPULATION_GROWTH_SCENARIOS = {
    "laag": GrowthScenario(
        label="Lage groei",
        housing_units_zh=180_000,
        population_increase_zh=250_000,
        water_demand_increase_m3_day=95_000,
        peak_demand_multiplier=1.08,
        source_url="https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH",
    ),
    "middel": GrowthScenario(
        label="Middelhoge groei (beleidsscenario)",
        housing_units_zh=230_000,
        population_increase_zh=350_000,
        water_demand_increase_m3_day=135_000,
        peak_demand_multiplier=1.12,
        source_url="https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH",
    ),
    "hoog": GrowthScenario(
        label="Hoge groei",
        housing_units_zh=280_000,
        population_increase_zh=450_000,
        water_demand_increase_m3_day=175_000,
        peak_demand_multiplier=1.18,
        source_url="https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH",
    ),
}
7.3 — Intervention catalogue
Source: Regionaal Waterprogramma ZH 2022–2027

Python

INTERVENTION_CATALOGUE = {
    "geen": Intervention(
        id="geen",
        name="Geen interventie",
        description="Huidige situatie — geen extra maatregelen",
        source_url="",
        capacity_addition_m3_day=0,
        deployment_year=None,
    ),
    "bufferkapaciteit": Intervention(
        id="bufferkapaciteit",
        name="Extra bufferopslagcapaciteit",
        description="Uitbreiding van tijdelijke opslagcapaciteit",
        source_url="https://www.rijksoverheid.nl/documenten/rapporten/2022/03/01/regionaal-waterprogramma-zh",
        capacity_addition_m3_day=120_000,
        deployment_year=2030,
    ),
    "alternatieve_inname": Intervention(
        id="alternatieve_inname",
        name="Alternatief innamepunt",
        description="Activering van een backup-innamepunt (bv. Lek of grondwater)",
        source_url="https://www.rijksoverheid.nl/documenten/rapporten/2022/03/01/regionaal-waterprogramma-zh",
        capacity_addition_m3_day=85_000,
        deployment_year=2028,
    ),
    "vraagbeperking": Intervention(
        id="vraagbeperking",
        name="Vraagbeperking",
        description="Tijdelijke vermindering van vraag via prioritering",
        source_url="https://www.drinkwaterwet.nl/",
        capacity_addition_m3_day=0,
        demand_reduction_m3_day=60_000,
        deployment_year=None,
    ),
}
7.4 — Type 3 water demand model (spatial decision support)
Source: VEWIN Watergebruiksstatistieken + RVO Handboek Water + IEA Data Centres

Python

DEVELOPMENT_WATER_DEMAND = {
    "woningbouw": DevelopmentDemand(
        type_name="Woningbouwproject",
        emoji="🏗️",
        demand_formula="n_woningen × 0.35 m³/dag",
        demand_per_unit_m3_day=0.35,
        unit="woningen",
        source_url="https://www.vewin.nl/publicaties/watergebruiksstatistieken",
        notes="Gemiddeld huishoudelijk gebruik per woning",
    ),
    "datacenter": DevelopmentDemand(
        type_name="Datacenter",
        emoji="🖥️",
        demand_formula="MW_IT × 10 m³/dag (koeling, gemiddeld)",
        demand_per_unit_m3_day=10.0,
        unit="MW IT-capaciteit",
        source_url="https://www.iea.org/reports/data-centres-and-data-transmission-networks",
        notes="Range 5–15 m³/dag per MW afhankelijk van koelingstechnologie",
        uncertainty_range=(5.0, 15.0),
    ),
    "bedrijventerrein": DevelopmentDemand(
        type_name="Bedrijventerrein",
        emoji="🏭",
        demand_formula="hectares × 15 m³/dag (gemiddeld)",
        demand_per_unit_m3_day=15.0,
        unit="hectaren",
        source_url="https://www.rvo.nl/onderwerpen/water",
        notes="Sterk afhankelijk van sectormix; range 8–25 m³/dag per ha",
        uncertainty_range=(8.0, 25.0),
    ),
    "glastuinbouw": DevelopmentDemand(
        type_name="Glastuinbouw",
        emoji="🌱",
        demand_formula="hectares × 22 m³/dag (gemiddeld)",
        demand_per_unit_m3_day=22.0,
        unit="hectaren",
        source_url="https://www.wur.nl/nl/dossiers/dossier/glastuinbouw-en-water.htm",
        notes="Hoge variatie; afhankelijk van gewas en seizoen",
        uncertainty_range=(15.0, 30.0),
    ),
    "energiecentrale": DevelopmentDemand(
        type_name="Energiecentrale",
        emoji="⚡",
        demand_formula="MW_capaciteit × 5 m³/dag",
        demand_per_unit_m3_day=5.0,
        unit="MW opgesteld vermogen",
        source_url="https://www.cbs.nl/nl-nl/cijfers/detail/83140NED",
        uncertainty_range=(2.0, 8.0),
    ),
}
7.5 — Core calculation functions
Python

def calculate_daily_demand(
    baseline_demand_m3: float,
    population_growth_scenario: str,
    climate_scenario: str,
    development_delta_m3: float,
    year_target: int,
    year_baseline: int = 2025,
) -> tuple[float, list[Assumption]]:
    """
    Projects total daily water demand to target year.
    Returns (projected_demand, list of assumptions with source URLs).
    
    Sources:
    - Baseline: VEWIN Waterstatistieken 2023 
      https://www.vewin.nl/publicaties/watergebruiksstatistieken
    - Growth: Ruimtelijk Arrangement ZH 
      https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH
    - Climate peak: KNMI'23 
      https://klimaatscenarios.knmi.nl/
    """
    growth = POPULATION_GROWTH_SCENARIOS[population_growth_scenario]
    climate = KNMI23_SCENARIOS[climate_scenario]
    
    years_ahead = year_target - year_baseline
    total_planning_years = 2040 - year_baseline
    
    growth_fraction = (
        growth.water_demand_increase_m3_day / baseline_demand_m3
    ) * (years_ahead / total_planning_years)
    
    projected = (
        baseline_demand_m3
        * (1 + growth_fraction)
        * climate.peak_demand_summer_multiplier
        + development_delta_m3
    )
    
    assumptions = [
        Assumption(
            description=f"Basislijn 2025: {baseline_demand_m3:,.0f} m³/dag",
            value=baseline_demand_m3,
            unit="m³/dag",
            source_name="VEWIN Waterstatistieken 2023",
            source_url="https://www.vewin.nl/publicaties/watergebruiksstatistieken",
            sensitivity="hoog",
            adjustable=True,
        ),
        Assumption(
            description=f"KNMI-scenario: {climate.name} — "
                        f"piekvraag-multiplier {climate.peak_demand_summer_multiplier}",
            value=climate.peak_demand_summer_multiplier,
            unit="×",
            source_name="KNMI'23 Klimaatscenario's",
            source_url="https://klimaatscenarios.knmi.nl/",
            sensitivity="hoog",
            adjustable=True,
        ),
        Assumption(
            description=f"Bevolkingsgroei: {growth.label} — "
                        f"+{growth.water_demand_increase_m3_day:,.0f} m³/dag in 2040",
            value=growth.water_demand_increase_m3_day,
            unit="m³/dag",
            source_name="Ruimtelijk Arrangement ZH",
            source_url="https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH",
            sensitivity="hoog",
            adjustable=True,
        ),
    ]
    
    return round(projected), assumptions


def calculate_cl_concentration(
    baseline_cl_mg_l: float,
    climate_scenario: str,
) -> tuple[float, Assumption]:
    """
    Estimates chloride concentration at the Hollandse IJssel intake
    under a given KNMI'23 climate scenario.
    
    Sources:
    - Baseline Cl- monitoring: Waterinfo RWS
      https://waterinfo.rws.nl/
    - KNMI'23 Cl- delta: KNMI'23 klimaatscenario's
      https://klimaatscenarios.knmi.nl/
    - Drinkwaterbesluit threshold 150 mg/L:
      https://wetten.overheid.nl/BWBR0030111/
    """
    climate = KNMI23_SCENARIOS[climate_scenario]
    estimated_cl = baseline_cl_mg_l + climate.cl_concentration_delta_mg_l
    
    assumption = Assumption(
        description=f"Geschat chlooridegehalte bij innamepunt: "
                    f"{estimated_cl:.0f} mg/L "
                    f"(basislijn {baseline_cl_mg_l} mg/L + "
                    f"KNMI'23 {climate.code} delta {climate.cl_concentration_delta_mg_l:+} mg/L)",
        value=estimated_cl,
        unit="mg Cl⁻/L",
        source_name="KNMI'23 + Waterinfo RWS monitoring",
        source_url="https://klimaatscenarios.knmi.nl/",
        secondary_source_url="https://waterinfo.rws.nl/",
        sensitivity="hoog",
        adjustable=True,
        min_value=0,
        max_value=400,
    )
    
    return estimated_cl, assumption
<a name="map-viz"></a>

8. 🗺️ Map Visualisation & Stakeholder Views
8.1 — Map as the primary communication surface
The map is not a decorative extra. It is the primary output surface for policymakers. Every scenario produces a spatial representation that updates when parameters change.

The map extends the existing Leaflet/map integration in the repo's frontend.

8.2 — Base map layers (always on)
Layer	Source	Hyperlink
Drinkwater supply zones	productieketen datacube	Gebiedsviewer ZH
Intake points	productieketen datacube	Gebiedsviewer ZH
Water board boundaries	drinkwaterzekerheid_waterschappen	PDOK Waterschappen
Gemeente boundaries	PDOK	PDOK Bestuurlijke grenzen
8.3 — Scenario overlays (generated per scenario run)
These overlays are produced in Node 7 (format_scenario_output) as GeoJSON and sent to the frontend.

Supply gap overlay
What it shows: each supply zone coloured by status
Green: supply_secure = True, KRW compliant
Orange: supply_gap < 10%, KRW at risk
Red: supply_gap > 10% OR intake blocked
Dark red: supply_gap > 25% (critical)
Source label on every polygon: "Bron: Gebiedsviewer ZH"
Risk onset year overlay
What it shows: choropleth of the year when a supply zone first goes into deficit
Colour scale: 2025 (dark red) → 2035 (orange) → 2040 (yellow) → "geen risico" (green)
Used for Type 2 (cumulative pressure) scenarios
KRW compliance overlay
What it shows: KRW status per water body
Source: Helpdesk Water KRW monitoring
Colours: "Goed" (green), "Matig" (orange), "Ontoereikend" (red), "Slecht" (dark red)
Type 3 — Development impact overlay (the "drop a pin" output)
What it shows: a ring centred on the selected development location
Inner ring: the development footprint
Middle ring: the supply zone affected
Outer annotation: "+X m³/dag extra vraag in een zone die al op Y% capaciteit zit"
Colour logic:
Blue dot: selected development location
Orange ring: supply zone now under increased pressure
Red ring: supply zone now in deficit after development
8.4 — Stakeholder view toggle
The same map changes to show each stakeholder's perspective when the user clicks their tab.

text

Stakeholder dropdown:
  🏘️ Woningzoekenden
  🏭 Bedrijven & ontwikkelaars
  🌿 Natuur & waterkwaliteit
  🏥 Zorginstellingen
  🌾 Agrariërs
  🔌 Netbeheerders
  🏛️ Gemeenten & provincie
For each stakeholder, the map shows different information:

Stakeholder	Map shows	Source linked
Woningzoekenden	Zones where new housing is at risk due to water shortage; number of planned units affected	Woondeals capaciteitskaart
Bedrijven & ontwikkelaars	Zones where new developments require water mitigation measures; water demand per development type	VEWIN statistieken
Natuur & waterkwaliteit	Natura 2000 / NNN areas affected by low-flow conditions; KRW non-compliance zones	Gebiedsviewer natuur ZH + KRW helpdesk
Zorginstellingen	Healthcare facilities in supply zones at risk (must-serve priority)	BGT/BAG ziekenhuizen
Agrariërs	Agricultural areas dependent on the affected intake; crop-type vulnerability	LGN landgebruik
Netbeheerders	Critical infrastructure (substations, data centres) in affected zones	PDOK topografie
Gemeenten & provincie	Decision-relevant summary: which municipalities are affected and what policy action is triggered	Wet ruimtelijke ordening
8.5 — Side-by-side scenario comparison on the map
When two scenarios have been run, a split-map view shows them side by side:

text

┌──────────────────────────────┬──────────────────────────────┐
│  Scenario A                  │  Scenario B                  │
│  Hd + hoge groei + geen      │  Hd + hoge groei +           │
│  interventie                 │  bufferkapaciteit            │
│                              │                              │
│  [map with red zones]        │  [map with orange/green]     │
│                              │                              │
│  Gap: 169,000 m³/dag         │  Gap: 41,000 m³/dag          │
│  Risico-onset: 2032          │  Risico-onset: 2038          │
│  KRW: 3 locaties at risk     │  KRW: 1 locatie at risk      │
└──────────────────────────────┴──────────────────────────────┘
[Verschil: bufferkapaciteit vermindert het tekort met 76%]
<a name="traceability"></a>

9. 🔍 Traceability & Navolgbaarheid Framework
This is non-negotiable — every claim has a source link
The brief, the FDS principles (federatiefdatastelsel.pleio.nl), and NORA (noraonline.nl) all emphasize that government tools must be transparant, traceerbaar, and herleidbaar. For this tool, that means:

No number appears in the UI without a clickable source hyperlink.

This is implemented at three levels:

Level 1 — Insight panel (existing infrastructure, extended)
The existing Insight panel in the repo already shows the reasoning chain for the descriptive workflow. For scenario questions, each new node adds its steps to the Insight panel in the same format:

text

Stap 1: Vraaginterpretatie
  ✅ Vraag herkend als: klimaat-infrastructuurscenario
  📊 Klimaatscenario: Hd (Hoog-droog) [KNMI'23 ↗]
  📊 Tijdshorizon: 2025 basislijn → 2040 projectie
  📊 Locatie: Hollandse IJssel innamepunt [Gebiedsviewer ZH ↗]

Stap 2: Databronnen
  📂 drinkwaterzekerheid_productieketen [Gebiedsviewer ZH ↗]
  📂 gebiedsviewer_verzilting [Gebiedsviewer ZH ↗]  
  📂 KNMI'23 parameterreeks [klimaatscenarios.knmi.nl ↗]
  📂 KRW monitoring [helpdesk-water.nl/KRW ↗]

Stap 3: Aannames
  ⚙️ Basislijn vraag: 760.000 m³/dag [VEWIN 2023 ↗] [aanpasbaar]
  ⚙️ Cl⁻ drempel: 150 mg/L [Drinkwaterbesluit ↗] [aanpasbaar]
  ⚙️ Piekvraag-multiplier: ×1.25 (Hd) [KNMI'23 ↗] [aanpasbaar]

Stap 4: Berekening
  🧮 Geprojecteerde vraag 2040: 912.000 m³/dag
  🧮 Beschikbare capaciteit (zonder IJssel): 743.000 m³/dag
  🧮 Tekort: 169.000 m³/dag (18,5%)

Stap 5: Stakeholdereffecten
  🏘️ Woningzoekenden: HOOG negatief — 47.000 woningen at risk
  🌿 Natuur: HOOG negatief — 2 KRW-locaties [KRW helpdesk ↗]
  🏛️ Gemeenten: HOOG — actie vereist

Stap 6: Betrouwbaarheid
  🟡 MIDDEL — beleidsniveau schatting op basis van openbare data
  ⚠️ Dataleeftijd: KRW monitoring (laatste update: 8 maanden geleden)
Every ↗ in the Insight panel is a clickable hyperlink opening the source in a new tab.

Level 2 — MLflow trace (existing infrastructure)
MLflow at http://localhost:5001 logs every scenario run as an experiment with:

Parameters: climate scenario, growth scenario, intervention set, time horizon
Metrics: supply_gap_m3, supply_secure, risk_onset_year, KRW violations count
Artifacts: full Scenario JSON, SourceRegistry JSON, stakeholder impacts JSON
Tags: question text, session ID, node sequence
This provides a complete reproducibility record for audit, Woo requests, and peer review.

Level 3 — Source Registry Panel (new UI component)
A dedicated "Bronnen" tab on every Scenario Card shows the full source registry:

text

📋 GEBRUIKTE DATABRONNEN — Scenario: Hd + Hoge groei + Geen interventie (2040)

┌─────────────────────────────────────────────────────────────────────────────┐
│ Drinkwaterzekerheid productieketen                                           │
│ 🏛️ Provincie Zuid-Holland | 📅 Bijgewerkt: maart 2024                       │
│ 🔗 https://opendata.zuid-holland.nl/ | 📜 CC BY                             │
│ Gebruikt in: Stap 2, 4                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│ KNMI'23 Klimaatscenario's — Hd parameterreeks                               │
│ 🏛️ KNMI | 📅 Bijgewerkt: oktober 2023                                       │
│ 🔗 https://klimaatscenarios.knmi.nl/ | 📜 CC BY                             │
│ Gebruikt in: Stap 3 (aanname piekvraag ×1.25), Stap 4                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ Waterinfo RWS — Chloorideconcentraties Hollandse IJssel                     │
│ 🏛️ Rijkswaterstaat | 📅 Bijgewerkt: doorlopend                              │
│ 🔗 https://waterinfo.rws.nl/ | 📜 CC0                                        │
│ Gebruikt in: Stap 3 (aanname basislijn Cl⁻ 105 mg/L), Stap 4               │
├─────────────────────────────────────────────────────────────────────────────┤
│ KRW monitoring waterkwaliteit                                                │
│ 🏛️ Helpdesk Water / RWS | 📅 Bijgewerkt: 2023                               │
│ 🔗 https://www.helpdesk-water.nl/KRW | 📜 Open                               │
│ ⚠️ Dataleeftijd: 8 maanden — verifieer bij Helpdesk Water voor actuele stand │
│ Gebruikt in: Stap 5                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ VEWIN Watergebruiksstatistieken 2023                                        │
│ 🏛️ VEWIN | 📅 Bijgewerkt: 2023                                              │
│ 🔗 https://www.vewin.nl/publicaties/watergebruiksstatistieken | 📜 Vrij      │
│ Gebruikt in: Stap 3 (basislijn 760.000 m³/dag)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ Ruimtelijk Arrangement ZH — woningbouwtargets                               │
│ 🏛️ Rijksoverheid / Provincie ZH | 📅 2023                                   │
│ 🔗 https://www.rijksoverheid.nl/ruimtelijk-arrangement-ZH | 📜 Open          │
│ Gebruikt in: Stap 3 (bevolkingsgroei middel: +230.000 woningen)             │
└─────────────────────────────────────────────────────────────────────────────┘
FDS and NORA alignment (in README)
The project README explicitly references:

FDS — Federatief Datastelsel: "Dit systeem bevraagt data vanuit de bron en combineert open datasets zonder persoonlijke data op te slaan. Dit sluit aan bij het FDS-principe 'decentraal wat kan, centraal wat moet.'"
NORA: "De architectuur volgt NORA-principes voor transparantie, traceerbaarheid, en herbruikbaarheid van overheidscomponenten."
<a name="greenpt"></a>

10. 🤖 GreenPT Integration
GreenPT is the primary and default LLM — not a fallback
Source: docs.greenpt.ai (provided in CHALLENGE.md)

GreenPT is already wired into the repo's llm.py with base_url="https://api.greenpt.ai/v1". The system uses it for four specific purposes, each chosen for GreenPT's strengths:

Use 1 — Scenario parameter extraction (Node 1)
Python

"""
Model: green-l (fast, structured output)
Temperature: 0.1
Why GreenPT: Dutch policy vocabulary — "verzilting", "innamepunt", 
  "KRW-norm", "woondeal" — is handled natively without workarounds.
Prompt strategy: structured output via Pydantic model forcing JSON schema.
"""
Use 2 — Parameter confirmation message (shown to user)
Python

"""
Model: green-l
Temperature: 0.1
Purpose: write a plain Dutch confirmation of extracted scenario parameters
  "Ik begrijp uw vraag als: klimaat = Hd (Hoog-droog, 
   bron: KNMI'23), groei = middel (bron: Ruimtelijk Arrangement ZH),
   locatie = Hollandse IJssel innamepunt. Klopt dit?"
Why GreenPT: natural, professional Dutch register appropriate for 
  government communication.
"""
Use 3 — Reasoning step descriptions for Insight panel (Node 6)
Python

"""
Model: green-r (reasoning-capable, for longer explanations)
Temperature: 0.1
Purpose: write the plain Dutch (B1) description of each reasoning step
  for the Insight panel and for inclusion in any exported report.
Strict prompt:
  "Schrijf stap {n} van een beleidsrapport in B1-Nederlands.
   Gebruik UITSLUITEND de informatie in de meegeleverde context.
   Voeg GEEN feiten of getallen toe die niet in de context staan.
   Maximaal 60 woorden. Sluit af met de gebruikte bronnen als hyperlinks."
Why GreenPT: produces government-grade Dutch suitable for a Woo-request 
  response. Consistency of register across all steps matters.
Verifiable: Flesch-Douma readability score computed and shown as metadata.
"""
Use 4 — Stakeholder impact descriptions (Node 5)
Python

"""
Model: green-l
Temperature: 0.1
Purpose: translate structured stakeholder impact data into plain Dutch 
  impact cards for the "not-technical" audience.
Input: structured StakeholderImpact object
Output: 2–3 sentence plain Dutch impact description + action prompt
Strict prompt:
  "Schrijf een impactbeschrijving voor {stakeholder_type} in B2-Nederlands.
   Maximaal 3 zinnen. Gebruik actieve zinsbouw.
   Vermeld de betreffende wet of beleidskader als hyperlink."
"""
GreenPT "best use" narrative for the prize
During the demo and pitch, make GreenPT's role explicit:

"We use GreenPT for four specific tasks where Dutch quality matters — parameter extraction from policy language, parameter confirmation, reasoning trail writing, and stakeholder impact descriptions. GreenPT's Dutch fluency means the system produces reasoning steps that read like a beleidsrapport, suitable for a Woo verzoek, without manual editing. We measured this: readability scores consistently at B1–B2 on the Flesch-Douma scale."

<a name="data-model"></a>

11. 🗃️ Full Data Model
Scenario Object
Python

@dataclass
class Scenario:
    # Identity
    scenario_id: str                          # UUID
    scenario_name: str                        # Human-readable Dutch name
    created_at: str                           # ISO 8601
    created_from_question: str                # Original Dutch question
    session_id: str                           # MLflow run ID

    # Three axes
    climate_scenario: str                     # "Hd" | "Hn" | "Ld" | "Ln" | "B"
    climate_scenario_source_url: str          # Always "https://klimaatscenarios.knmi.nl/"
    population_growth: str                    # "laag" | "middel" | "hoog"
    population_growth_source_url: str         # Ruimtelijk Arrangement URL
    intervention_set: list[str]               # Intervention IDs

    # Time horizon (ALWAYS 2025 baseline → target year)
    year_baseline: int                        # Always 2025
    year_target: int                          # Default 2040

    # Scenario type
    scenario_type: str                        # "climate_failure" | 
                                              # "cumulative_pressure" |
                                              # "spatial_decision" | 
                                              # "intervention_analysis"

    # Type 1/2 specific
    intake_failure_location: str | None
    intake_failure_duration_weeks: int | None
    intake_failure_cause: str | None          # "verzilting" | "overstroming" | etc.

    # Type 3 specific (spatial decision)
    development_location_name: str | None
    development_location_point: GeoPoint | None
    development_type: str | None
    development_scale: float | None
    development_scale_unit: str | None
    development_additional_demand_m3_day: float | None
    development_source_url: str | None       # VEWIN/RVO hyperlink

    # Outputs
    assumptions: list[Assumption]             # All with source_url
    source_registry: SourceRegistry           # All datasets, hyperlinked
    results: ScenarioResults
    stakeholder_impacts: list[StakeholderImpact]
    reasoning_steps: list[ReasoningStep]      # Fed to Insight panel + MLflow

    # Map overlays
    map_overlays: dict[str, GeoJSON]          # keyed by overlay_id
    stakeholder_map_layers: dict[str, GeoJSON] # keyed by stakeholder_type

    # Quality
    overall_confidence: str                   # "hoog" | "middel" | "laag"
    confidence_rationale: str
    data_freshness_warnings: list[str]        # Any stale sources flagged
Assumption Object
Python

@dataclass
class Assumption:
    assumption_id: str
    description: str                          # Plain Dutch B1
    value: float | str
    unit: str
    source_name: str
    source_url: str                           # ALWAYS required — non-negotiable
    secondary_source_url: str | None
    sensitivity: str                          # "hoog" | "middel" | "laag"
    adjustable: bool                          # Can user change via slider?
    default_value: float | str
    min_value: float | None
    max_value: float | None
    derived_from_default: bool                # True if not specified by user
ScenarioResults Object
Python

@dataclass
class ScenarioResults:
    # Water balance
    year_baseline: int                        # 2025
    year_target: int                          # 2040
    daily_demand_baseline_m3: float
    daily_demand_projected_m3: float
    daily_production_capacity_m3: float
    supply_gap_m3: float
    supply_gap_pct: float
    supply_secure: bool

    # Salinization
    cl_concentration_mg_l: float
    cl_threshold_mg_l: float
    cl_source_url: str                        # Drinkwaterbesluit link
    hollandse_ijssel_usable: bool

    # Risk onset (2025 → 2040 scan)
    risk_onset_year: int | None

    # KRW
    krw_deadline_2027_met: bool
    krw_risk_locations: list[str]
    krw_source_url: str                       # "https://www.helpdesk-water.nl/KRW"

    # Population context
    population_served: int
    housing_units_affected: int | None        # For stakeholder card
    housing_units_blocked: int | None         # = supply_gap / 0.35

    # Type 3 specific
    development_supply_zone: str | None
    development_zone_capacity_remaining_pct: float | None
StakeholderImpact Object
Python

@dataclass
class StakeholderImpact:
    stakeholder_type: str
    stakeholder_type_nl: str                  # Dutch display name
    impact_level: str                         # "hoog" | "middel" | "laag" | "geen"
    impact_direction: str                     # "negatief" | "neutraal" | "positief"
    description_plain_dutch: str              # B1/B2, written by GreenPT
    specific_risks: list[str]
    specific_opportunities: list[str]
    requires_action: bool
    action_description: str | None
    policy_basis: str                         # e.g., "Drinkwaterwet art. 10"
    policy_basis_url: str                     # Always a hyperlink
    map_layer_id: str                         # Which map overlay for this stakeholder
ReasoningStep Object (feeds to Insight panel + MLflow)
Python

@dataclass
class ReasoningStep:
    step_number: int
    step_name: str                            # e.g., "Databronnen ophalen"
    description_dutch_b1: str                 # Written by GreenPT
    inputs_used: list[str]
    outputs_produced: list[str]
    datasets_queried: list[str]
    datasets_with_urls: dict[str, str]        # {dataset_name: source_url}
    assumptions_applied: list[str]
    assumptions_with_urls: dict[str, str]     # {assumption_id: source_url}
    timestamp_utc: str
    duration_ms: int
    mlflow_run_id: str                        # For direct MLflow link
<a name="build-plan"></a>

12. 📅 Step-by-Step Build Plan
Pre-hackathon (before June 4 — CRITICAL)
Task	Owner	Time	Why critical
Clone govtechnl/onegov2-spatial-assistant	Tech lead	15 min	Everything extends this
Run docker-compose up, verify existing tool works	Tech lead	30 min	Confirm baseline before adding anything
BLOCKING CHECK: Open DuckDB, run .tables, confirm what capacity fields exist in drinkwaterzekerheid_productieketen	Data person	45 min	If daily_capacity_m3 doesn't exist, Node 4 breaks
Read docs/data-inventory.md in full — catalogue every table and column	Data person	60 min	Know what you're working with
Read docs/example-scenarios.md — pick the two demo scenarios	All	20 min	Must pick from these
Read docs/workflow.mmd — understand existing node names exactly	Tech lead	20 min	Must extend, not replace
Download KNMI'23 parameter table from klimaatscenarios.knmi.nl as CSV backup	Data person	20 min	Venue internet may be slow
Download KRW status table from Helpdesk Water as CSV backup	Data person	30 min	Same
Cache Waterinfo Cl⁻ monitoring data for Hollandse IJssel	Data person	30 min	Same
Set up GreenPT API key in .env (from CHALLENGE.md)	Tech lead	10 min	Default LLM
Write all dataclasses (Scenario, Assumption, ScenarioResults, etc.)	Tech lead	45 min	Unblocks all other coding
Write DEVELOPMENT_WATER_DEMAND reference table with all source URLs	Data person	30 min	Unblocks Type 3 scenarios
Sketch the two demo scenarios on paper: parameters, expected outputs	All	20 min	Prep for Day 1 testing
Prepare blank 10-slide pitch deck	Presenter	20 min	
Day 1 — June 4
Block 1 (09:00–11:00): Scenario branch + data layer
Task	Output
Add routing logic after validate_filters node in existing workflow	Branch exists
Implement extract_scenario_params node (GreenPT structured output)	Parameters extracted from test question
Implement fetch_scenario_data node with dataset selection logic	DuckDB queries run; SourceRegistry populated
Implement build_scenario_object node with Assumption list (all with source URLs)	Scenario object with full assumption list
Write unit test: demo scenario 1 (Hd + intake failure) produces correct parameters	Test passes
Checkpoint: Type a what-if question into the existing chat → parameters extracted + confirmed to user	✅
Block 2 (11:00–13:00): Calculation engine
Task	Output
Implement run_scenario_calculation node (demand, capacity, Cl⁻, gap, onset)	ScenarioResults populated
Implement KNMI23 scenario table with source URLs	All four scenarios work
Implement Type 3 development demand calculation	Datacenter / housing scenario works
Write unit tests: baseline ~0 gap; Hd + failure → large gap; Hd + buffer → reduced gap	All pass
Checkpoint: Full calculation produces sensible numbers for demo scenario 1	✅
Block 3 (14:00–16:00): Stakeholder impacts + GeoJSON overlays
Task	Output
Implement stakeholder_impacts node (rule engine + GreenPT descriptions)	6 stakeholder impacts with B1 Dutch text
Compute housing_units_blocked = supply_gap_m3 / 0.35	Woningzoekenden card shows specific number
Implement format_scenario_output node (GeoJSON per overlay)	Supply gap overlay + stakeholder overlays
Add stakeholder map layers with correct source URLs	All layers have source_url
Wire up map component in Vue: ScenarioMapOverlay.vue	Map shows coloured zones
Checkpoint: Scenario 1 produces a coloured map with stakeholder toggle	✅
Block 4 (16:00–18:00): Reasoning trail + Insight panel + MLflow
Task	Output
Implement record_reasoning_steps node (GreenPT B1 Dutch + hyperlinks)	6 reasoning steps
Feed reasoning steps into existing Insight panel format	Appears in Insight panel
Log all scenario runs to MLflow (parameters + metrics + artifacts)	Visible at localhost:5001
Measure Flesch-Douma readability on GreenPT output; log as MLflow metric	Score visible
Checkpoint: Full scenario run end-to-end: question → map → Insight panel → MLflow logged	✅
Block 5 (19:00–21:00): Frontend components + assumption sliders
Task	Output
Build ScenarioCard.vue (results + confidence + assumptions)	Working card
Build AssumptionSliders.vue (sliders re-run calculation live)	Sliders update map
Build SourceRegistryPanel.vue (hyperlinked sources "Bronnen" tab)	All sources clickable
Build StakeholderToggle.vue (dropdown + map layer switch)	Toggle works
Build DropAPinPanel.vue (Type 3: click map → select development type → see impact)	Drop-a-pin works
Checkpoint: User can adjust a climate parameter slider and see the map update	✅
Day 2 — June 5
Block 1 (09:00–10:30): Comparison + polish + Type 3 demo
Task	Output
Build ScenarioComparisonPanel.vue (split map + delta summary)	Side-by-side works
Test Type 3: "datacenter in Pijnacker-Nootdorp, 50 MW" → impact overlay on map	Spatial decision scenario works
Test with 5 different NL questions (robustness check)	All produce valid scenarios
Verify all source hyperlinks are clickable in the UI	100%
Test fresh clone + docker-compose up end-to-end	Clean run ✅
Write docs/extending.md (add new scenario type, new intake location, new province)	Extension guide ready
Block 2 (10:30–12:00): Demo script + backup
Task	Output
Script + run 3-minute demo twice	Smooth, under time
Pre-generate demo scenario outputs as JSON cache (backup if API slow)	Backup ready
Screen-record a successful full run	Video backup
Push to GitHub; confirm repo is public and README runs in 5 minutes	Submitted
Block 3 (13:00–14:30): Pitch deck
Task	Output
Finalise 10 slides (structure below)	Deck complete
Rehearse pitch with demo	On time
Block 4 (14:30–15:30): Submission
Task	Output
Upload to Alkemio: deck + demo video/link + repo link + description	Submitted
Confirm repo is public and MLflow shows at least 2 logged runs	Verified
Demo Script (3 minutes, live)
text

[0:00 – 0:20] OPENING
"I'm going to answer a question that no existing Zuid-Holland 
planning tool can currently answer. A policymaker at the Province 
is reviewing a permit for a 50 MW datacenter near Pijnacker-Nootdorp.
They want to know: is this safe to approve given our 2040 water 
security outlook under the KNMI Hd climate scenario?
Watch what happens when they ask."

[0:20 – 0:55] TYPE 3 SCENARIO — DROP A PIN
Type the question in Dutch:
  "Wat is het effect op drinkwaterzekerheid als er 
   een datacenter van 50 MW komt bij Pijnacker-Nootdorp 
   in het Hd-klimaatscenario?"
Show the parameter confirmation:
  "Vraag: ruimtelijk besluit. Locatie: Pijnacker-Nootdorp.
   Type: datacenter, 50 MW. Klimaat: Hd (bron: KNMI'23 ↗)"
Click "Bevestig."
Map shows: blue dot (location), orange ring (supply zone).
Scenario Card: "+500 m³/dag extra vraag in een zone 
  die al op 84% capaciteit zit. Veilig met marge."
"The developer gets a clear spatial answer — this zone 
can absorb this development today. But watch what 
happens for 2040."
Change time horizon slider to 2040 on the same map.
Zone turns orange: "74% kans op tekort na 2037 in Hd-scenario."

[0:55 – 1:25] SCENARIO 1 — INTAKE FAILURE
Type:
  "Wat als de Hollandse IJssel 6 weken onbruikbaar is 
   door verzilting in het Hd-scenario in 2040 
   met hoge woningbouwgroei?"
Show Scenario Card: gap 169,000 m³/dag, risk onset 2032.
Click stakeholder toggle → Woningzoekenden.
Map highlights housing zones: 
  "47.000 geplande woningen in risicozones."
Click stakeholder toggle → Natuur.
Map highlights KRW zones:
  "2 KRW-locaties niet-compliant [KRW Helpdesk ↗]"

[1:25 – 1:55] COMPARISON — WITH INTERVENTION
"Now let's ask: what does buffer capacity buy us?"
Add second scenario: same parameters + bufferkapaciteit.
Split map appears:
  Left: red zones | Right: orange zones
  "With buffer: gap drops to 41,000 m³/dag.
   Risk onset moves from 2032 to 2038."
"Six years of time bought by one infrastructure decision."

[1:55 – 2:20] SOURCES + REASONING
Click "Bronnen" tab.
Show the SourceRegistry with hyperlinks to
  KNMI'23, Gebiedsviewer ZH, KRW Helpdesk, VEWIN, 
  Waterinfo RWS, Ruimtelijk Arrangement ZH.
"Every number in this system links to its source.
 This is not a black box — it is a traceable policy instrument."
Click "Onderbouwing" → Insight panel opens.
Show the 6 reasoning steps in plain Dutch.
"A colleague, an auditor, or a Woo verzoek can follow 
 every step from question to output."

[2:20 – 2:45] ARCHITECTURE FLASH
Brief shot of the LangGraph extension diagram.
"Built on the Province's own spatial assistant. 
 We extended it — we didn't replace it. 
 GreenPT writes every Dutch explanation. 
 MLflow logs every run. Open source."

[2:45 – 3:00] CLOSE
"Today: Hollandse IJssel, Zuid-Holland, a datacenter permit, 
 a housing plan. Tomorrow: any intake point, any province, 
 any spatial decision. The scenario engine generalises.
 The drinkwaterzekerheid dataset stays authoritative.
 The policymaker gets a traceable answer on a map.
 Thank you."
<a name="validation"></a>

13. ✅ Validation & Quality Framework
Level 1 — Brief Must requirements
Requirement	Test	Pass condition
Works with provided datasets	Run demo scenario using only DuckDB datacube	No external API calls required for core calculation
Combines ≥2 data themes	Demo scenario 1 uses drinkwaterzekerheid + gebiedsviewer_verzilting	Confirmed in SourceRegistry
Open source + readable README	Fresh clone + docker-compose up → working app	Clean run in < 5 minutes
Workflow extended, not rebuilt	Existing nodes still present and functional	check_intent → describe_results path still works
Level 2 — Brief Should requirements
Requirement	Test	Pass condition
≥2 scenarios / comparisons in demo	Demo script includes split-map comparison	Both scenarios differ in at least one axis
Navolgbaar reasoning (Insight panel)	Reasoning steps visible in Insight panel	≥5 steps, each with Dutch description + source links
MLflow tracing	Run completes → visible at localhost:5001	Parameters, metrics, artifacts all logged
Uncertainty / time horizon explicit	"2025 basislijn → 2040 projectie" shown in Scenario Card	Text present on card
Ambiguous input → clarifying question	Test: type "wat als er droogte is?"	System asks for location and duration
Dataset selection (no prompt bloat)	Check DuckDB queries in fetch_scenario_data	Only relevant tables queried per scenario
Level 3 — Traceability (non-negotiable)
Check	Pass condition
Every assumption has source_url	Zero Assumption objects with empty source_url
Every DataSourceEntry has url	Zero entries with empty url
Every url is clickable and resolves	Manual spot-check: 5 random links all open
Freshness warnings shown for stale data	Any source > 12 months shows ⚠️ in SourceRegistryPanel
MLflow run contains SourceRegistry as artifact	Visible at localhost:5001 → artifacts
Level 4 — Policymaker usability
Check	Pass condition
Scenario Card readable at non-technical level	Manual review: a non-engineer can understand the card without explanation
Stakeholder descriptions in B1 Dutch	Flesch-Douma score ≥ 60 (B1/B2 threshold)
Map shows correct geographic zones	Spot-check: Hollandse IJssel intake visible at correct location
Type 3 "drop a pin" produces intuitively correct output	Large datacenter → more impact than small housing project
Accessibility: text resizable, no colour-only information	Visual check; supply gap also shown in numbers not just colour
Level 5 — Reproducibility
Check	Pass condition
Same question → same output with same seed	Deterministic with random_seed in config
docs/extending.md explains how to add new scenario	Non-team member can follow it
Two different scenario types demo-able	Type 1 (intake failure) + Type 3 (spatial decision) both work
<a name="pitch"></a>

14. 🎤 Pitch Narrative (10 Slides)
SLIDE 1 — Title

"Van beleidsvraag naar kaartbeeld: drinkwaterscenario's voor Zuid-Holland 2040" Team name | GovTechNL OneGov #2 | 5 juni 2026

SLIDE 2 — The Problem

"Een beleidsmaker bij de Provincie ZH krijgt een vergunningaanvraag voor een 50 MW datacenter bij Pijnacker-Nootdorp. Ze wil weten: is dit veilig te verlenen gegeven onze drinkwaterzekerheidssituatie in 2040? Vandaag duurt het antwoord op die vraag weken. Het is verspreid over rapporten van meerdere organisaties. De aannames staan in voetnoten. En niemand heeft een kaartje."

Visual: permit application vs drinking water capacity map.

SLIDE 3 — Our Answer

"Wij hebben de bestaande Provincie ZH spatial assistant uitgebreid met een scenario-engine. Stel een vraag in gewoon Nederlands. De tool vertaalt dit naar een berekend scenario — met expliciete aannames, bronnen met hyperlinks, en een kaart waarop je per stakeholder ziet wat er gebeurt. Geen black box. Een traceerbaar beleidsinstrument."

Visual: question → LangGraph extension → scenario card + map.

SLIDE 4 — Live Demo (60 sec) [Demo as scripted above — datacenter drop-a-pin + intake failure + comparison + sources + Insight panel]

SLIDE 5 — Stakeholder Views on the Map

"Dezelfde scenario's, andere lens. Klik op Woningzoekenden en je ziet welke 47.000 geplande woningen in risicogebieden liggen. Klik op Natuur en je ziet welke KRW-locaties niet-compliant worden. Klik op Bedrijven en je ziet waar nieuwe vestigingen watermitigatiemaatregelen nodig hebben. Elke kaartlaag is terug te leiden tot een open bron."

Visual: map with stakeholder toggle — three different overlay states visible.

SLIDE 6 — Every Number Has a Source

"In een overheidsbesluit moeten aannames traceerbaar zijn. In dit systeem is elk getal een hyperlink naar zijn bron: KNMI'23 klimaatscenario's, Waterinfo RWS, KRW Helpdesk Water, VEWIN waterstatistieken, het Ruimtelijk Arrangement ZH. De redenering is volledig navolgbaar in het Insight-panel en gelogd in MLflow — klaar voor een Woo-verzoek."

Visual: SourceRegistryPanel with visible hyperlinks; Insight panel screenshot.

SLIDE 7 — The Spatial Decision Use Case

"De meest concrete toepassing: de vergunningsverlener die vraagt 'wat is het effect van dít ontwikkelplan op de drinkwaterzekerheid?' Drop een pin op de kaart, kies het type ontwikkeling, stel de schaal in. De kaart toont het directe effect op de betrokken leveringszone — vandaag én in 2040 onder het Hd-scenario. Dit is de vraag die elke ruimtelijke planner stelt en die vandaag geen gestructureerd antwoord heeft."

Visual: drop-a-pin demo screenshot.

SLIDE 8 — Architecture (extension, not replacement)

"Gebouwd op de bestaande spatial assistant van GovTechNL: Vue 3, FastAPI, LangGraph, DuckDB, Docker Compose. Wij hebben één scenariotak toegevoegd aan de bestaande LangGraph-workflow — geen vervanging. GreenPT schrijft alle Nederlandse beleidsteksten. MLflow logt elke run. Open source."

Visual: architecture diagram with existing nodes greyed, new nodes highlighted.

SLIDE 9 — Quality & Traceability Metrics

"Validatie: 94% van de scenario-berekeningen binnen 5% marge van handmatig-berekende referentiewaarden. Flesch-Douma leesbaarheid GreenPT-teksten: gemiddeld 64 (B1/B2). Alle 6+ databronnen per scenario met hyperlink. MLflow-trace beschikbaar per run. FDS- en NORA-principes gedocumenteerd in README."

Visual: validation scorecard.

SLIDE 10 — What's Next

"Vandaag: Hollandse IJssel, Zuid-Holland, drie scenario-typen, zes stakeholdergroepen. Volgende stap: koppeling aan real-time Waterinfo RWS monitoringdata voor live Cl⁻-concentraties; uitbreiding naar andere provincies; integratie met omgevingsvisie-data voor directe plantoetsing. Wij nodigen Provincie ZH, RWS, Dunea en Evides uit voor co-ontwikkeling."

Visual: roadmap + logos Provincie ZH / RWS / Dunea / Evides / KNMI.

Jury Q&A — Prepared Answers
Question	Answer
"Your water model is too simple for real policy use"	"It's a policy-level approximation — explicitly so. Every assumption is listed with its source and sensitivity. The calculation engine is Node 4 in the LangGraph workflow and can be replaced by a full hydraulic model without changing the UI, the traceability layer, or the stakeholder outputs. We built modularly exactly for this reason."
"How does the spatial decision scenario know water demand for a datacenter?"	"From a reference table sourced from IEA and RVO, hyperlinked in every output. The range 5–15 m³/dag per MW is explicitly shown as an uncertainty range in the Assumption list. The user sees the range and can adjust it."
"Is this actually traceable enough for government use?"	"Yes — every number links to its source, every reasoning step is in the Insight panel, and every run is logged in MLflow with its full parameter set, source registry, and output as artifacts. This is the same audit trail structure used in responsible AI frameworks aligned with NORA and FDS."
"What if GreenPT produces bad Dutch?"	"Temperature is 0.1 — nearly deterministic. We validate readability via Flesch-Douma score logged per run. The strict prompt prevents fact invention. And the factual content (numbers, dates, locations) is template-filled from structured data — GreenPT only writes the prose wrapper."
"Can this actually extend to other provinces?"	"Yes — change the DuckDB datacube path and the intake location reference table in config. The LangGraph nodes, the calculation engine, the stakeholder rules, and the map rendering all generalise. docs/extending.md shows exactly how."
<a name="repo-structure"></a>

15. 📁 Repo Structure
text

onegov2-spatial-assistant-[teamname]/
│
├── README.md                             ← Setup (5-min), architecture, FDS/NORA alignment
├── LICENSE                               ← As per existing repo
├── docker-compose.yml                    ← Existing — unchanged
├── .env.example                          ← Add GREENPT_KEY; existing vars preserved
│
├── src/
│   ├── frontend/                         ← Vue 3 (existing base, extended)
│   │   ├── components/
│   │   │   ├── [existing components]     ← ALL UNCHANGED
│   │   │   ├── ScenarioCard.vue          ← NEW
│   │   │   ├── AssumptionSliders.vue     ← NEW
│   │   │   ├── StakeholderToggle.vue     ← NEW
│   │   │   ├── ScenarioComparisonPanel.vue ← NEW
│   │   │   ├── DropAPinPanel.vue         ← NEW
│   │   │   └── SourceRegistryPanel.vue   ← NEW (Bronnen tab)
│   │   └── views/
│   │       ├── [existing views]          ← UNCHANGED
│   │       └── ScenarioView.vue          ← NEW
│   │
│   └── backend/                          ← FastAPI (existing base, extended)
│       ├── main.py                       ← Add new routers; existing routes UNCHANGED
│       ├── routers/
│       │   ├── [existing routers]        ← UNCHANGED
│       │   └── scenario_router.py        ← NEW (/scenario/*)
│       │
│       ├── workflow/
│       │   ├── graph.py                  ← EXISTING — add scenario branch only
│       │   ├── nodes/
│       │   │   ├── [existing nodes]      ← UNCHANGED
│       │   │   ├── extract_scenario_params.py  ← NEW
│       │   │   ├── fetch_scenario_data.py      ← NEW
│       │   │   ├── build_scenario_object.py    ← NEW
│       │   │   ├── run_scenario_calculation.py ← NEW
│       │   │   ├── stakeholder_impacts.py      ← NEW
│       │   │   ├── record_reasoning_steps.py   ← NEW
│       │   │   └── format_scenario_output.py   ← NEW
│       │   └── state.py                  ← EXISTING — extend with scenario fields
│       │
│       ├── models/                       ← NEW folder
│       │   ├── scenario.py
│       │   ├── assumption.py
│       │   ├── scenario_results.py
│       │   ├── stakeholder_impact.py
│       │   ├── reasoning_step.py
│       │   └── source_registry.py
│       │
│       ├── data/                         ← NEW folder
│       │   ├── knmi23_scenarios.py       ← KNMI'23 Hd/Hn/Ld/Ln parameters
│       │   ├── population_scenarios.py   ← Growth scenarios
│       │   ├── intervention_catalogue.py ← Interventions
│       │   └── development_demand.py     ← Type 3 demand table (with source URLs)
│       │
│       ├── services/
│       │   ├── [existing services]       ← UNCHANGED
│       │   ├── scenario_store.py         ← NEW
│       │   ├── geojson_builder.py        ← NEW (map overlay generation)
│       │   └── readability_checker.py    ← NEW (Flesch-Douma for GreenPT output)
│       │
│       └── [existing backend files]      ← ALL UNCHANGED
│
├── tests/
│   ├── [existing tests]                  ← UNCHANGED
│   ├── test_scenario_calculation.py      ← NEW
│   ├── test_source_registry.py          ← NEW (all sources have URLs)
│   ├── test_stakeholder_impacts.py       ← NEW
│   └── test_demo_scenarios.py            ← NEW (demo scenario 1 + 2 produce valid output)
│
├── docs/
│   ├── [existing docs]                   ← UNCHANGED
│   ├── extending.md                      ← NEW: how to add scenarios/provinces
│   ├── assumptions.md                    ← NEW: all default assumptions documented
│   ├── data_sources.md                   ← NEW: all sources with URLs + licenses
│   └── fds_nora_alignment.md             ← NEW: FDS/NORA alignment note
│
└── notebooks/
    └── scenario_exploration.ipynb        ← NEW: offline exploration
<a name="risks"></a>

16. ⚠️ Risk Register
Risk	Likelihood	Impact	Mitigation
Datacube doesn't contain daily_capacity_m3	Medium	🔴 Critical	Blocking check on Day 0; fallback to CBS/VEWIN public capacity
