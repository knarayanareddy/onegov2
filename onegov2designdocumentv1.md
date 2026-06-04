💧 Drinkwaterzekerheid Zuid-Holland 2040 — Complete Revised Design Document
Scenario Engine for Policymakers, Spatial Planners & Citizens
GovTechNL OneGov #2 Hackathon | June 4–5, 2026 | The Hague Based on: Official Challenge Brief | Starter Repository

📋 TABLE OF CONTENTS
Strategic Framing
User Audience & Design Principles
Data Architecture & Source Traceability
Workflow Architecture (Extension of Existing)
Scenario Engine & Calculation Design
Stakeholder-Specific Scenario Library
Interactive Map Visualisation Design
Traceability & Navolgbaarheid Framework
Full Data Model
Frontend Architecture (Policymaker UX)
Step-by-Step Build Plan
Validation & Quality Framework
Pitch Narrative
Repo Structure
Risk Register
<a name="strategic-framing"></a>

1. 🎯 Strategic Framing
What this prototype must do (brief-exact)
From the official challenge brief and CHALLENGE.md:

"Een werkend prototype dat een beleidsvraag in natuurlijke taal vertaalt naar een berekenbaar scenario, waarbij zichtbaar wordt welke data en aannames eraan ten grondslag liggen en wat de effecten zijn voor betrokken partijen."

This means five non-negotiable requirements:

Requirement	What it actually means
Working prototype	A running system, not a mockup — clone, run, demo in minutes
Natural language → calculable scenario	A policy question produces a structured, rerunnable Scenario object — not prose
Visible data + assumptions	Every number is hyperlinked back to its authoritative source
Effects on affected parties	Stakeholder impacts are computed, quantified, and shown on a map
Extends, does not replace	Built on top of govtechnl/onegov2-spatial-assistant
The audience insight that changes everything
The existing spatial assistant is designed for technical analysts who can read SQL, understand DuckDB schemas, and interpret raw query results. The brief asks for something that a provincial policymaker, a spatial planner at a municipality, or a project developer filing a permit can use directly.

That means:

Technical analyst (existing tool)	Policymaker / planner (this tool)
Types a question, reads a table	Types a question, sees a map + scenario card
Understands "geometry.intersects"	Understands "this location is at high risk in 2040"
Comfortable with raw numbers	Needs context: is 169,000 m³/day a lot?
Doesn't need to trust the answer	Needs to know where the number comes from
One user type	Multiple stakeholder views of the same scenario
The architecture must serve both — the technical depth remains, but a policymaker-readable layer sits on top of every output.

The strategic choice: "What does this mean for my project?"
The most powerful framing for this tool is not "water risk dashboard." It is:

"Before you submit your spatial plan, development permit, or policy decision — ask this tool what it means for drinking water security in 2040."

This means the scenario library is organized around initiating parties and their decisions, not around abstract water themes:

A housing developer asking: "Can I build 3,000 homes in Zoetermeer by 2035?"
A data centre operator asking: "What does a 15MW facility in Delft mean for water capacity?"
A provincial spatial planner asking: "Which combination of climate and housing growth hits us first?"
A water board asking: "Does buffer capacity resolve the 2040 deficit under all climate scenarios?"
Every one of these is a natural-language question that the system translates into a calculable scenario, visualised on a map, with full source traceability.

<a name="audience"></a>

2. 👥 User Audience & Design Principles
Primary user types
User type	Technical level	Primary question	Needs from this tool
Provincial policymaker (Provincie Zuid-Holland)	Low-medium	"Which developments create the most water risk?"	Map + plain-language summary + exportable briefing note
Spatial planner (gemeente, omgevingsdienst)	Medium	"Is this location viable for housing development?"	Location-specific scenario + permit-relevant output
Project developer	Low-medium	"Does my project increase water stress in this area?"	One-click project impact assessment
Water authority (Dunea, Evides)	High	"Which interventions are most effective in which scenarios?"	Side-by-side scenario comparison + data export
RIVM / research	High	"What are the compound risk combinations?"	Full scenario space exploration + reasoning trail
General public / citizen	Low	"Is my drinking water safe in 2040?"	Simple map with plain Dutch risk summary
Non-negotiable design principles
1. Every number is a link. Every figure shown in the UI traces to an authoritative source. There are no "approximate" or "estimated" numbers without a link to where the estimate comes from. This is non-negotiable per the brief's "data bij de bron" principle and FDS standards.

2. The map is the primary output. For policymakers, a map is more actionable than a number. Every scenario result is visualised geographically — affected zones, vulnerable intake points, risk areas — before any numbers are shown.

3. Plain Dutch before technical detail. Every output has two levels: a plain Dutch (B1) summary shown by default, with technical detail available on expand. This follows NORA accessibility guidelines and the plain-language requirements of Digitoegankelijkheid WCAG 2.1 AA.

4. Stakeholder-specific views. The same scenario is presented differently depending on who is asking. A housing developer sees "number of homes at risk." A water authority sees "m³/day deficit." Both are computed from the same underlying scenario object.

5. Traceable to source, always. Every dataset, every assumption, every calculation step links to its authoritative source. This is aligned with NORA principle 9 (Bronregistraties) and FDS interoperability requirements.

<a name="data-architecture"></a>

3. 🗄️ Data Architecture & Source Traceability
Authoritative data sources (with hyperlinks — non-negotiable)
Every dataset used in this prototype must be traceable to its authoritative source. The table below is the single source of truth for all data used in any calculation.

Theme	Dataset	Organisation	URL	License	Layer in repo
Drinkwaterinfrastructuur	Productieketen drinkwater	Dunea / Evides / PWN	opendata.zuid-holland.nl	CC-BY	drinkwaterzekerheid
Drinkwaterzekerheid zones	Zes-uurszones kwetsbaar gebied	Provincie Zuid-Holland	opendata.zuid-holland.nl	CC-BY	drinkwaterzekerheid
Verzilting	Verziltingsgevoeligheid oppervlaktewater	Provincie Zuid-Holland	opendata.zuid-holland.nl	CC-BY	gebiedsviewer
Bodemdaling	Bodemdaling prognose ZH	Provincie Zuid-Holland	opendata.zuid-holland.nl	CC-BY	gebiedsviewer
Overstromingsrisico	Overstromingskwetsbaarheid	Provincie Zuid-Holland	opendata.zuid-holland.nl	CC-BY	gebiedsviewer
KRW monitoring	Oppervlaktewaterkwaliteit / chloride	Rijkswaterstaat	waterinfo.rws.nl	CC-BY	External (API)
KRW normen en doelen	KRW-doelen en toetsing	Helpdesk Water / RWS	helpdesk-water.nl/KRW	Public	Reference
Klimaatscenario's	KNMI'23 klimaatscenario's (Hd/Hn/Ld/Ln)	KNMI	klimaatscenarios.knmi.nl	CC-BY	External (CSV)
Woningbouw / capaciteit	Capaciteitskaart afname regionaal	Provincie ZH / BZK	rijksoverheid.nl/ruimtelijk-arrangement-ZH	CC-BY	extra_data/woondeals
Bevolking / CBS	Vierkantstatistieken 100m	CBS	opendata.cbs.nl	CC-BY	extra_data/CBS
Landgebruik	LGN landgebruikskaart	WUR / PDOK	pdok.nl	CC-BY	extra_data/lgn
Waterlichamen	Oppervlaktewaterlichamen	PDOK / BRT	pdok.nl	CC-BY	gebiedsviewer
FDS referentie	Federatief Datastelsel principes	BZK	federatiefdatastelsel.pleio.nl	Public	Reference
NORA architectuur	NORA referentiearchitectuur	ICTU / BZK	noraonline.nl	Public	Reference
Dataset selection rule (brief-explicit requirement)
The brief explicitly warns against loading too many columns, as this harms LLM accuracy. The data fetcher must implement scenario-specific column selection:

Python

SCENARIO_DATASET_MAP = {
    "verzilting_inname": [
        "drinkwaterzekerheid/productieketen",      # intake locations + capacity
        "drinkwaterzekerheid/drinkwaterbedrijven", # operators
        "gebiedsviewer/verzilting",                # chloride risk zones
        "gebiedsviewer/waterlichamen",             # river/canal bodies
    ],
    "housing_growth_pressure": [
        "drinkwaterzekerheid/productieketen",
        "extra_data/woondeals/capaciteitskaart_afname_regionaal",
        "extra_data/CBS/vierkantstatistieken",
    ],
    "compound_shock": [
        "drinkwaterzekerheid/productieketen",
        "gebiedsviewer/verzilting",
        "extra_data/woondeals/capaciteitskaart_afname_regionaal",
        "gebiedsviewer/overstromingskwetsbaarheid",
    ],
    "data_centre_impact": [
        "drinkwaterzekerheid/productieketen",
        "drinkwaterzekerheid/zes_uurszones",
        "gebiedsviewer/verzilting",
        "extra_data/lgn",
    ],
}
<a name="workflow"></a>

4. 🔄 Workflow Architecture (Extension of Existing)
Core principle: extend, do not rebuild
The existing LangGraph workflow must be preserved. The challenge brief explicitly states teams should not replace or rewrite the existing workflow. The scenario engine is added as a branch extension that activates when a what-if / scenario intent is detected.

Existing workflow (preserved exactly)
text

START
  ↓
check_intent
  ↓ (if "scenariotype intent" → branch to scenario pipeline)
  ↓ (if "descriptief" → continue existing path)
resolve_spatial
  ↓
validate_filters
  ↓
generate_sql
  ↓
execute_query
  ↓
plan_visualization
  ↓
describe_results
  ↓
END
Sources: docs/workflow.mmd | docs/architecture_diagram.mmd

New scenario branch (added nodes only)
When check_intent classifies the question as scenario/what-if type, control flows into the new branch:

text

check_intent
  ↓ [scenario intent detected]
  ↓
extract_scenario_params          ← NEW NODE 1
  ↓
select_datasets                  ← NEW NODE 2
  ↓
fetch_spatial_context            ← NEW NODE 3
  ↓
[existing: generate_sql + execute_query run for spatial base data]
  ↓
build_scenario_object            ← NEW NODE 4
  ↓
run_scenario_math                ← NEW NODE 5
  ↓
compute_stakeholder_impacts      ← NEW NODE 6
  ↓
generate_map_overlays            ← NEW NODE 7
  ↓
record_reasoning_to_insight      ← NEW NODE 8
  ↓
[existing: plan_visualization + describe_results]
  ↓
END
Nodes 1–8 are new. Everything else is unchanged from the existing workflow. The Insight panel and MLflow tracing — already present in the starter repo — automatically capture all node transitions, satisfying the "navolgbaarheid" Should criterion without building a parallel UI.

Node specifications (new nodes only)
New Node 1 — extract_scenario_params
Python

"""
Input:  User's natural language question (string)
Output: ScenarioParameters object

Responsibilities:
- Classify scenario type from the example-scenarios list:
    "verzilting_inname"          (docs/example-scenarios.md #1)
    "housing_growth_pressure"    (docs/example-scenarios.md #3)
    "compound_shock"             (docs/example-scenarios.md #4)
    "data_centre_impact"         (stakeholder-specific extension)
    "housing_project_impact"     (stakeholder-specific extension)
    "company_location_impact"    (stakeholder-specific extension)
    "intervention_analysis"      (policy analysis)
    "comparison_request"         (side-by-side)

- Map to three axes:
    climate_scenario: Hd | Hn | Ld | Ln | baseline_2025
        [Source: KNMI'23 — klimaatscenarios.knmi.nl]
    population_growth: laag | middel | hoog
        [Source: CBS/BZK woondeals — rijksoverheid.nl/ruimtelijk-arrangement-ZH]
    intervention_set: geen | bufferkapaciteit | alternatieve_inname | vraagbeperking

- Extract spatial parameters:
    location_geometry (if user specified a location or project)
    intake_failure_location (if event-based scenario)
    project_type (data_centre | housing | company | other)
    project_size (MW for data centres, units for housing, m2 for companies)

- If confidence < 0.75: return clarification_question, do not proceed
- Always log which parameters were extracted vs defaulted

LLM: GreenPT (klimaatscenarios.knmi.nl/api)
     temperature=0.1 for reproducibility
"""
New Node 2 — select_datasets
Python

"""
Input:  ScenarioParameters
Output: DataSelectionPlan

Responsibilities:
- Select only the datasets needed for this scenario type
  (see SCENARIO_DATASET_MAP above)
- Record each selected dataset with:
    - name, organisation, URL, license (from authoritative source table)
    - specific columns needed (column selection, not SELECT *)
    - data freshness (last_updated from metadata)
- Flag any dataset that is stale (last_updated > 6 months)
- Log the selection rationale for the Insight panel

This node implements the brief's explicit "don't load every dataset" requirement.
"""
New Node 3 — fetch_spatial_context
Python

"""
Input:  DataSelectionPlan + location_geometry (if provided)
Output: SpatialContextBundle

Responsibilities:
- If user specified a location/project geometry:
    - Identify which drinkwaterzekerheid zones it intersects
    - Identify which verzilting zones it intersects
    - Identify which zes-uurszones it falls within
    - Identify nearest intake points and their capacities
    - Identify serving drinkwaterbedrijf
- Fetch climate parameters for the selected KNMI'23 scenario
    [Source: klimaatscenarios.knmi.nl]
- Fetch KRW monitoring data for relevant waterbodies
    [Source: waterinfo.rws.nl — API call for chloride at nearest station]
- Build list of DataSource objects (all with URLs for traceability)
"""
New Node 4 — build_scenario_object
Python

"""
Input:  ScenarioParameters + SpatialContextBundle
Output: Scenario object (partial — no results yet)

Responsibilities:
- Instantiate the Scenario dataclass
- Build Assumption list — each assumption:
    - has a source URL (non-negotiable)
    - has a sensitivity rating
    - is flagged as adjustable (powers sliders in UI)
- For project-type scenarios: add project-specific assumptions
    (e.g., water use per MW for data centres)
- Validate internal consistency
- Log all defaults applied
"""
New Node 5 — run_scenario_math
Python

"""
Input:  Scenario object (partial)
Output: ScenarioResults object

Responsibilities:
- Project daily demand to 2040:
    baseline_2025 × growth_multiplier × climate_peak_multiplier
    [Source: CBS/woondeals for growth; KNMI'23 for climate peak]
- For project scenarios: add project water demand:
    data_centre: project_mw × 1.5 m³/MWh × operating_hours_per_day
    housing: project_units × 0.35 m³/unit/day (VEWIN benchmark)
    [Source: VEWIN waterstatistieken — vewin.nl/waterstatistieken]
- Calculate available production capacity:
    sum(usable_locations) where usable = not failed AND cl < threshold
    [Source: productieketen layer; chloride from waterinfo.rws.nl]
- Calculate supply gap + KRW compliance
    [Source: KRW norms — helpdesk-water.nl/KRW]
- Find risk onset year (scan 2025–2040)
- All intermediate values logged with source citations
"""
New Node 6 — compute_stakeholder_impacts
Python

"""
Input:  Scenario + ScenarioResults
Output: list[StakeholderImpact]

Responsibilities:
- Compute impacts for each relevant stakeholder type
- All impacts include a QUANTIFIED metric, not just a label:
    woningzoekenden: housing_units_at_risk = supply_gap_m3 / 0.35
    data_centres: facilities_at_risk = count(facilities in affected zone)
    bedrijven: affected_employment = sum(workers in affected supply zone)
    agrariërs: ha_at_risk = area(high_salinization ∩ agricultural_lgn)
    [LGN source: pdok.nl]
    zorginstellingen: beds_at_risk = count(care beds in affected zone)
    netbeheerders: substations_at_risk = count(substations in zes-uurszones)
    gemeenten: population_at_risk = sum(CBS population in affected zones)
    [CBS source: opendata.cbs.nl]

- Map view for each stakeholder type (different layer styling per type)
- All stakeholder quantification uses authoritative sources with links
"""
New Node 7 — generate_map_overlays
Python

"""
Input:  Scenario + ScenarioResults + StakeholderImpacts
Output: MapOverlayBundle (GeoJSON)

Responsibilities:
- Generate 3 map overlay sets (one per stakeholder view mode):

  BASE LAYER (always shown):
    - Intake points coloured by status (green/amber/red)
    - Supply zones
    - KRW risk locations
    - KNMI climate risk areas

  HOUSING STAKEHOLDER VIEW:
    - Housing development zones (woondeals)
    - Supply constraint overlay (red = cannot be served)
    - Risk onset year choropleth per zone

  DATA CENTRE / INDUSTRIAL VIEW:
    - Zes-uurszones (six-hour vulnerability zones)
    - Water-intensive industrial sites
    - Additional load from project (if project location specified)
    - Risk delta (supply gap with/without project)

  WATER AUTHORITY VIEW:
    - Production chain per bedrijf
    - Alternative intake routes
    - Buffer capacity locations
    - Intervention effect polygons

- All features include a "source" property with URL for traceability
"""
New Node 8 — record_reasoning_to_insight
Python

"""
Input:  All previous node outputs
Output: InsightPanelEntry (fed into existing Insight panel)

Responsibilities:
- Format reasoning as a structured entry for the existing Insight panel
  [Source: insight panel in govtechnl/onegov2-spatial-assistant]
- Each reasoning step includes:
    - step name (plain Dutch)
    - inputs used
    - outputs produced
    - datasets accessed (each with source URL)
    - assumptions applied (each with source URL)
    - timestamp + duration
- Feed into MLflow trace
  [MLflow at http://localhost:5001]
- The Insight panel IS the traceability surface — no separate UI needed

This node makes the system navolgbaar (traceable) per the brief's Should criterion.
Plain Dutch descriptions written by GreenPT (klimaatscenarios.knmi.nl/api).
"""
<a name="scenario-engine"></a>

5. ⚙️ Scenario Engine & Calculation Design
Climate axis: KNMI'23 official scenarios
The climate axis uses the four official KNMI'23 scenarios, not a simplified "droog/nat" model. This is essential for credibility with the policy audience.

Scenario	Code	Description	Drought freq. multiplier	Cl⁻ delta (mg/L)	Peak demand multiplier
Hoog-droog	Hd	High emissions, drying	2.3	+55	1.28
Hoog-nat	Hn	High emissions, wetting	1.4	+15	1.08
Laag-droog	Ld	Low emissions, drying	1.8	+35	1.18
Laag-nat	Ln	Low emissions, wetting	1.1	+5	1.02
Baseline 2025	—	Current situation, no climate change	1.0	0	1.0
Source for all values: KNMI'23 Klimaatscenario's — Hd/Hn/Ld/Ln classification.

Population / housing growth axis
Source: Ruimtelijk Arrangement ZH — Rijksoverheid + CBS bevolkingsprognose

Growth scenario	Housing target ZH	Population increase	Water demand increase	Source
Laag	180,000 units	+250,000	+95,000 m³/day	CBS/BZK
Middel	230,000 units	+350,000	+135,000 m³/day	Ruimtelijk Arrangement ZH
Hoog	280,000 units	+450,000	+175,000 m³/day	Woondeals ZH
Intervention catalogue
Intervention	Code	Description	Capacity effect	Source
Geen interventie	geen	Current situation	+0 m³/day	—
Bufferkapaciteit	buffer	Expanded buffer storage	+85,000 m³/day	Regionaal Waterprogramma ZH 2022-2027
Alternatieve inname	alt_inname	Alternative intake activation	60% of primary	Productieketen drinkwater
Vraagbeperking	vraagbeperking	Demand reduction measures	-15% demand	VEWIN Waterstatistieken
Core demand projection formula
Python

def calculate_daily_demand_2040(
    baseline_2025_m3: float,            # ~760,000 m³/day ZH (VEWIN estimate)
    population_growth: str,             # laag | middel | hoog
    climate_scenario: str,              # Hd | Hn | Ld | Ln | baseline_2025
    target_year: int = 2040,
    project_additional_m3: float = 0.0  # For project-specific scenarios
) -> DemandResult:
    """
    Sources:
    - Baseline 2025: VEWIN Waterstatistieken (vewin.nl/waterstatistieken)
    - Growth params: CBS/BZK woondeals (rijksoverheid.nl/ruimtelijk-arrangement-ZH)
    - Climate params: KNMI'23 (klimaatscenarios.knmi.nl)
    """
    growth = POPULATION_GROWTH_TABLE[population_growth]
    climate = KNMI23_TABLE[climate_scenario]

    years_ahead = target_year - 2025
    total_horizon = 2040 - 2025  # 15-year projection

    demand_increase_fraction = (
        growth.water_demand_increase_m3_day / baseline_2025_m3
    ) * (years_ahead / total_horizon)

    projected = (
        baseline_2025_m3
        * (1 + demand_increase_fraction)
        * climate.peak_demand_multiplier
        + project_additional_m3
    )

    return DemandResult(
        value_m3_day=round(projected, 0),
        baseline_source="VEWIN Waterstatistieken — vewin.nl/waterstatistieken",
        growth_source="Ruimtelijk Arrangement ZH — rijksoverheid.nl/ruimtelijk-arrangement-ZH",
        climate_source="KNMI'23 Klimaatscenario's — klimaatscenarios.knmi.nl",
        project_addition_m3=project_additional_m3,
        year=target_year,
    )
Project-specific water demand calculations
Python

PROJECT_WATER_DEMAND = {
    "data_centre": {
        # Source: IEA Water for Data Centres (iea.org) + VEWIN
        "formula": "capacity_mw * 1.5 * operating_hours_per_day",
        "unit": "m³/day per MW",
        "source_url": "https://www.iea.org/reports/data-centres",
        "source_nl": "https://www.vewin.nl/waterstatistieken",
        "notes": "Water use for cooling, varies with PUE. Default: 1.5 m³/MWh."
    },
    "housing_units": {
        # Source: VEWIN Waterstatistieken
        "formula": "units * occupancy_rate * 0.135",
        "unit": "m³/day per 1,000 units",
        "source_url": "https://www.vewin.nl/waterstatistieken",
        "notes": "135L per person per day, 2.4 occupants per unit average."
    },
    "office_m2": {
        # Source: ISSO publicatie 55
        "formula": "floor_m2 * 0.0006",
        "unit": "m³/day per m²",
        "source_url": "https://www.isso.nl/publicaties",
        "notes": "0.6L/m²/day for mixed-use office."
    },
    "industrial_site": {
        # Source: CBS Watergebruik industrie
        "formula": "employees * 0.05",
        "unit": "m³/day per employee",
        "source_url": "https://opendata.cbs.nl",
        "notes": "50L/employee/day for light industry."
    },
}
<a name="scenario-library"></a>

6. 🏙️ Stakeholder-Specific Scenario Library
This is the central innovation over the existing tool: scenarios are organized around what people are deciding, not abstract water themes.

Scenario 1: Drinkwaterinfrastructuur — Hollandse IJssel intake failure
Source in repo: docs/example-scenarios.md — Scenario #1

User: Water authority / provincial planner Question: "Wat als het innamepunt Hollandse IJssel 6 weken onbruikbaar is door verzilting in een droog klimaatscenario?" Themes combined: drinkwaterzekerheid + gebiedsviewer/verzilting Climate axis: Hd or Ld Map shows: Affected intake zones (red), alternative intake routes (green), supply areas at risk (amber/red choropleth), KRW risk points

Scenario 2: Klimaat + Bevolkingsgroei compound
Source in repo: docs/example-scenarios.md — Scenario #4

User: Provincial spatial planner Question: "Welke combinatie van klimaat en woningbouw bereikt het eerst een drinkwatertekort?" Themes combined: drinkwaterzekerheid + gebiedsviewer/verzilting + extra_data/woondeals Map shows: Risk onset year per municipality (choropleth 2025–2040), current housing pipeline, supply constraint overlay

Scenario 3: Data Centre — locatie-specifiek water impact
User: Data centre developer / omgevingsdienst Question: "Wat is het effect van een 50MW datacentrum op locatie X op de drinkwaterlevering in 2040?" Themes combined: drinkwaterzekerheid + drinkwaterzekerheid/zes_uurszones + extra_data/lgn Calculation: Add 50 × 1.5 × 24 = 1,800 m³/day to local demand Map shows:

Project location marker
Serving intake point + supply zone
Supply gap WITH and WITHOUT the project (comparison panel)
Zes-uurszones vulnerability overlay
Risk delta choropleth (how much worse does this location make things?)
Stakeholder outputs:

Developer: "This facility adds 1,800 m³/day to the Gouda supply zone, contributing to a deficit in 2036 under scenario Hd."
Gemeente: "3 other developments in this supply zone are also proposed — cumulative effect: 4,200 m³/day additional demand."
Dunea: "Buffer capacity intervention reduces the deficit from 169,000 to 41,000 m³/day even with this facility."
Scenario 4: Woningbouwproject — locatie drinkwatertoets
User: Housing developer / spatial planner Question: "Kan ik 2,500 woningen bouwen op locatie Y in Zoetermeer — wat betekent dit voor de drinkwaterlevering in 2035?" Themes combined: drinkwaterzekerheid + extra_data/woondeals/capaciteitskaart_afname_regionaal + extra_data/CBS Calculation: 2,500 units × 2.4 occupants × 0.135 m³ = 810 m³/day additional demand Map shows:

Project footprint
Nearest intake point + serving bedrijf (Dunea/Evides)
Existing committed demand in same supply zone (from woondeals)
Available residual capacity (colour-coded: green = safe, amber = tight, red = deficit)
2035 risk onset flag
Stakeholder outputs:

Developer: "810 m³/day additional demand. Residual capacity in this supply zone: 1,240 m³/day (2035, Ld scenario). Project is feasible — but only 430 m³/day of headroom remains for other developments."
Gemeente: "This project consumes 65% of remaining water headroom in the Zoetermeer supply zone."
Province: "Under Hd climate, this location shows deficit from 2037. Recommend buffer capacity as a condition."
Scenario 5: Bedrijfsvestiging — vestigingsplaats drinkwatercheck
User: Business developer / economic affairs Question: "Welke locaties in Zuid-Holland zijn nog geschikt voor een waterintensief productiebedrijf van 500 werknemers in 2040?" Themes combined: drinkwaterzekerheid + gebiedsviewer/verzilting + extra_data/CBS Map shows:

Suitability map for water-intensive business location (green/amber/red)
Supply zone capacity heatmap
KRW risk overlay
Verzilting risk overlay
Filtered by: supply gap ≤ 0 under Ld scenario in 2040
Scenario 6: Interventieanalyse — what works where?
User: Water authority / province Question: "Wat is het effect van bufferkapaciteit versus alternatieve inname op het tekort in 2040 onder scenario Hd?" Themes combined: drinkwaterzekerheid + gebiedsviewer/verzilting Map shows:

Side-by-side comparison: Scenario A (buffer) vs Scenario B (alternative intake)
Deficit zones per scenario (different colours)
Cost-effectiveness indicator (m³/day gap reduced per intervention)
<a name="map-design"></a>

7. 🗺️ Interactive Map Visualisation Design
Map architecture
The map is the primary output surface for all scenarios. It is built on the existing Leaflet/MapLibre setup in the starter repo, extended with scenario-specific overlay layers.

text

┌──────────────────────────────────────────────────────────────────┐
│                    INTERACTIVE MAP CANVAS                         │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  BASE TOPOGRAPHIC LAYER (BRT Achtergrondkaart — PDOK)        │ │
│  │  Source: pdok.nl/brt-achtergrondkaart                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          +                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  ALWAYS-ON LAYERS (toggle off/on)                            │ │
│  │  - Intake points (circles, coloured by risk status)          │ │
│  │  - Supply zones (light fill)                                 │ │
│  │  - Water bodies (blue lines)                                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          +                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  SCENARIO RESULT LAYERS (generated by Node 7)                │ │
│  │  - Risk choropleth (supply gap m³/day per zone)              │ │
│  │  - Risk onset year per municipality                          │ │
│  │  - KRW compliance status per waterbody                       │ │
│  │  - Verzilting risk zones (from gebiedsviewer)                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          +                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  STAKEHOLDER-SPECIFIC OVERLAY (switches per audience mode)   │ │
│  │  Housing mode: woondeals zones + housing pipeline            │ │
│  │  Data centre mode: zes-uurszones + water-intensity zones     │ │
│  │  Water authority mode: production chain + alternatives       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          +                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  PROJECT MARKER (if location specified by user)              │ │
│  │  - Drag-and-drop placement                                   │ │
│  │  - Impact radius overlay                                     │ │
│  │  - "Before / after" toggle                                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  BOTTOM LEFT: Layer legend + source links (non-negotiable)        │
│  BOTTOM RIGHT: Scale bar + CRS indicator (EPSG:4326)              │
│  TOP RIGHT: Stakeholder view selector (housing / data / water /   │
│             company / general)                                    │
└──────────────────────────────────────────────────────────────────┘
Layer source attribution panel (non-negotiable)
Every map layer has a visible attribution that hyperlinks to its source. This is not a footnote — it is a permanent UI element:

text

┌─────────────────────────────────────────────────────────────────┐
│  📍 Kaartlagen in dit scenario:                                  │
│                                                                  │
│  ● Innamepunten: Productieketen drinkwater                       │
│    → opendata.zuid-holland.nl | CC-BY                           │
│  ● Verzilting: Gebiedsviewer Zuid-Holland                        │
│    → opendata.zuid-holland.nl | CC-BY                           │
│  ● Klimaatrisico: KNMI'23 scenario Hd                           │
│    → klimaatscenarios.knmi.nl | CC-BY                           │
│  ● Woningbouw: Capaciteitskaart afname regionaal                 │
│    → rijksoverheid.nl/ruimtelijk-arrangement-ZH | CC-BY         │
│  ● KRW-status: Waterinfo RWS                                     │
│    → waterinfo.rws.nl | CC-BY                                   │
└─────────────────────────────────────────────────────────────────┘
Stakeholder view switcher
The map has a prominent switcher in the top right allowing users to see the same scenario through different stakeholder lenses without rerunning the scenario:

text

[🏠 Woningbouw] [🏢 Datacentrum] [🏭 Bedrijf] [💧 Waterautoriteit] [🏛️ Provincie] [👤 Burger]
Each mode changes:

which overlay layers are visible
the colour scheme and legend (e.g., housing mode: "units at risk"; water mode: "m³/day deficit")
the scenario card summary text (plain Dutch for each audience)
which comparison panel is shown
<a name="traceability"></a>

8. 🔗 Traceability & Navolgbaarheid Framework
Non-negotiable: every number has a source URL
This is the core design principle for a government tool. Every figure shown in the UI must trace to an authoritative source. Implementation:

Python

@dataclass
class SourcedValue:
    """
    Every number displayed in the UI must be a SourcedValue.
    No raw floats or ints are ever passed directly to the frontend.
    """
    value: float
    unit: str
    source_name: str                     # Human-readable name
    source_organisation: str             # Organisation responsible
    source_url: str                      # Hyperlink (non-negotiable)
    source_table_or_layer: str           # Which specific table/layer
    accessed_date: str                   # When this data was fetched
    publication_date: str | None         # When source was last updated
    license: str                         # e.g., "CC-BY"
    is_estimate: bool                    # True if derived/approximated
    estimate_methodology: str | None     # If estimate: how was it derived?
    confidence: str                      # "hoog" | "middel" | "laag"
What the Insight panel shows (existing infrastructure)
The existing Insight panel in govtechnl/onegov2-spatial-assistant already provides a step-by-step trace of every node transition. Node 8 enriches this with:

For each reasoning step: a list of SourcedValue objects used
For each assumption: the source URL, value, and sensitivity rating
For each dataset: name, organisation, URL, fields used, freshness
The full scenario parameter set with defaults flagged
This makes the Insight panel the primary "navolgbaar" (traceable) surface as required by the brief, without building a parallel UI.

MLflow tracing
The starter repo configures MLflow at http://localhost:5001. Every scenario run is logged as an MLflow experiment with:

Python

mlflow.log_params({
    "climate_scenario": scenario.climate_scenario,       # e.g., "Hd"
    "population_growth": scenario.population_growth,
    "intervention_set": str(scenario.intervention_set),
    "target_year": scenario.target_year,                 # 2040
    "baseline_year": 2025,
    "intake_failure_location": scenario.intake_failure_location,
})

mlflow.log_metrics({
    "daily_demand_m3": results.daily_demand_m3,
    "production_capacity_m3": results.daily_production_capacity_m3,
    "supply_gap_m3": results.supply_gap_m3,
    "supply_gap_pct": results.supply_gap_pct,
    "risk_onset_year": results.risk_onset_year or 9999,
    "cl_concentration_mg_l": results.cl_concentration_mg_l,
    "krw_locations_at_risk": len(results.krw_risk_locations),
    "housing_units_at_risk": stakeholder_impacts.woningzoekenden.units_at_risk,
})

mlflow.log_artifacts({
    "scenario_card.json": scenario_card,
    "map_overlay.geojson": map_overlays,
    "sources_used.json": [s.to_dict() for s in scenario.data_sources_used],
})
Every scenario run is thus reproducible, auditable, and comparable via the MLflow UI.

FDS and NORA alignment
Per FDS standards and NORA reference architecture:

Principle	Implementation
Data bij de bron (FDS)	Data is fetched from authoritative sources (opendata.zuid-holland.nl, waterinfo.rws.nl, klimaatscenarios.knmi.nl) — never copied into a separate store
Bronregistraties (NORA principe 9)	Every SourcedValue links back to the bronregistratie
Openheid en transparantie (NORA)	Reasoning trail is fully visible in Insight panel; assumptions are explicit
Interoperabiliteit (FDS)	Data accessed via open standards (WFS, OGC API, CBS OData)
Vindbaarheid (FDS)	All datasets registered with metadata (name, org, URL, license)
<a name="data-model"></a>

9. 🗄️ Full Data Model
9.1 — Scenario Object
Python

@dataclass
class Scenario:
    # Identity
    scenario_id: str                      # UUID
    scenario_name: str                    # Human-readable Dutch name
    created_at: str                       # ISO 8601 timestamp
    created_from_question: str            # Original NL question

    # Scenario type (from example-scenarios.md)
    scenario_type: str                    # "verzilting_inname" | "housing_growth_pressure"
                                          # | "compound_shock" | "data_centre_impact"
                                          # | "housing_project_impact" | "company_location_impact"
                                          # | "intervention_analysis" | "comparison_request"

    # The three axes
    climate_scenario: str                 # "Hd" | "Hn" | "Ld" | "Ln" | "baseline_2025"
                                          # Source: klimaatscenarios.knmi.nl
    population_growth: str                # "laag" | "middel" | "hoog"
                                          # Source: rijksoverheid.nl/ruimtelijk-arrangement-ZH
    intervention_set: list[str]           # ["geen"] | ["buffer", "alt_inname", ...]

    # Time horizon (brief-explicit requirement)
    baseline_year: int = 2025             # Always 2025
    target_year: int = 2040               # Default 2040, configurable

    # Event / project parameters
    intake_failure_location: str | None   # e.g., "Hollandse IJssel"
    intake_failure_duration_weeks: int | None
    project_type: str | None              # "data_centre" | "housing" | "company" | None
    project_location_wkt: str | None      # Project geometry if specified
    project_size_value: float | None      # MW / units / m² depending on type
    project_size_unit: str | None         # "MW" | "woningen" | "m2"

    # All assumptions with sources (non-negotiable)
    assumptions: list[Assumption]

    # All data sources with hyperlinks (non-negotiable)
    data_sources_used: list[DataSource]

    # Outputs
    results: ScenarioResults
    stakeholder_impacts: dict[str, StakeholderImpact]
    map_overlays: MapOverlayBundle

    # Reasoning trail (fed into Insight panel)
    reasoning_steps: list[ReasoningStep]

    # Confidence
    overall_confidence: str               # "hoog" | "middel" | "laag"
    confidence_rationale: str
    confidence_source: str                # URL to methodology documentation
9.2 — Assumption Object (all assumptions must have source URLs)
Python

@dataclass
class Assumption:
    assumption_id: str
    category: str                         # "klimaat" | "bevolking" | "infrastructuur" | "beleid"
    description_nl: str                   # Plain Dutch description
    value: float | str
    value_unit: str
    source_name: str                      # Human-readable
    source_url: str                       # Non-negotiable hyperlink
    source_organisation: str
    sensitivity: str                      # "hoog" | "middel" | "laag"
    adjustable_in_ui: bool
    default_value: float | str
    min_value: float | None
    max_value: float | None
    is_default: bool                      # Was this filled by default?
9.3 — Scenario Results Object
Python

@dataclass
class ScenarioResults:
    # Demand (all SourcedValue — includes source URL)
    daily_demand_2025_m3: SourcedValue    # Baseline (VEWIN source)
    daily_demand_2040_m3: SourcedValue    # Projected
    project_additional_demand_m3: SourcedValue | None  # If project scenario

    # Capacity
    daily_production_capacity_m3: SourcedValue
    supply_gap_m3: SourcedValue           # demand - capacity
    supply_gap_pct: SourcedValue
    supply_secure: bool

    # Chloride / salinization
    cl_concentration_mg_l: SourcedValue   # From waterinfo.rws.nl
    cl_threshold_mg_l: SourcedValue       # KRW norm (helpdesk-water.nl/KRW)
    hollandse_ijssel_usable: bool

    # Timeline
    risk_onset_year: int | None           # First year demand > capacity
    risk_onset_confidence: str

    # KRW compliance (helpdesk-water.nl/KRW)
    krw_deadline_2027_met: bool
    krw_risk_locations: list[str]
    krw_source_url: str = "https://helpdesk-water.nl/KRW"

    # Population served
    population_served: SourcedValue       # CBS source

    # Alternative supply
    alternative_sources_active: list[str]
    alternative_capacity_m3: SourcedValue
9.4 — StakeholderImpact Object (quantified, with sources)
Python

@dataclass
class StakeholderImpact:
    stakeholder_type: str                 # "woningzoekenden" | "data_centres" |
                                          # "bedrijven" | "agrariërs" | "zorginstellingen" |
                                          # "netbeheerders" | "gemeenten" | "waterbeheerders"

    # All impact metrics are SourcedValue (include URL)
    quantified_impact: SourcedValue       # The key number for this stakeholder
                                          # e.g., housing_units_at_risk = supply_gap / 0.35

    impact_level: str                     # "hoog" | "middel" | "laag" | "geen"
    impact_direction: str                 # "negatief" | "neutraal" | "positief"

    # Policymaker-readable text (plain Dutch B1)
    summary_for_policymaker: str          # One sentence, plain Dutch
    detail_for_specialist: str            # Technical detail on expand

    # Map
    affected_geometries: list[str]        # WKT list of affected zones

    requires_action: bool
    recommended_action: str | None
    action_source_url: str | None         # Source for the recommendation
9.5 — Map Overlay Bundle
Python

@dataclass
class MapOverlayBundle:
    # Base layers (always shown)
    intake_points: GeoJSON                # From productieketen (opendata.zuid-holland.nl)
    supply_zones: GeoJSON                 # From drinkwaterzekerheid
    krw_locations: GeoJSON                # From waterinfo.rws.nl

    # Scenario result layers
    risk_choropleth: GeoJSON              # Supply gap per zone
    risk_onset_year_choropleth: GeoJSON   # Year of first deficit per area
    verzilting_overlay: GeoJSON           # From gebiedsviewer (opendata.Zuid-Holland.nl)

    # Stakeholder-specific layers (one per view mode)
    housing_view_layers: dict             # woondeals + housing pipeline
    data_centre_view_layers: dict         # zes-uurszones + water-intensity
    water_authority_view_layers: dict     # production chain + alternatives
    company_view_layers: dict             # suitability map

    # Project marker (if project scenario)
    project_location: GeoJSON | None
    project_impact_radius: GeoJSON | None
    before_after_delta: GeoJSON | None    # Risk change WITH vs WITHOUT project

    # Attribution (non-negotiable — shown in map legend)
    layer_attributions: list[LayerAttribution]
<a name="frontend"></a>

10. 🖥️ Frontend Architecture (Policymaker UX)
Design principles in practice
The frontend extends the existing Vue 3 application in govtechnl/onegov2-spatial-assistant. New components are added; nothing existing is removed.

Screen layout
text

┌─────────────────────────────────────────────────────────────────────┐
│  HEADER: "Drinkwaterscenario's Zuid-Holland 2040"                   │
│  [Provincie ZH logo] [Rijksoverheid logo] [KNMI logo]               │
└─────────────────────────────────────────────────────────────────────┘
│                                                                      │
│  ┌──────────────────────────┐  ┌──────────────────────────────────┐ │
│  │  QUESTION INPUT          │  │  MAP (60% of screen)             │ │
│  │  [Natural language box]  │  │  Interactive, full scenario      │ │
│  │                          │  │  visualisation                   │ │
│  │  Example questions:      │  │                                  │ │
│  │  💡 "Data centre 50MW    │  │  [Stakeholder view switcher]     │ │
│  │      in Delft in 2040?"  │  │  [Layer toggle panel]            │ │
│  │  💡 "2.500 woningen in   │  │  [Source attribution panel]      │ │
│  │      Zoetermeer viable?" │  │                                  │ │
│  │  💡 "IJssel 6 weken      │  │                                  │ │
│  │      onbruikbaar — Hd?"  │  │                                  │ │
│  └──────────────────────────┘  └──────────────────────────────────┘ │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  SCENARIO CARD (below map)                                      │ │
│  │                                                                 │ │
│  │  [Summary for policymaker — plain Dutch — always visible]       │ │
│  │  [Stakeholder tabs: Housing | Data Centre | Water | Company]    │ │
│  │  [Expand: Aannames + bronnen (with hyperlinks)]                 │ │
│  │  [Expand: Insight panel — navolgbaar reasoning trace]           │ │
│  │  [Compare with scenario B button]                               │ │
│  │  [Export as PDF briefing note]                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
New Vue components (added to existing app)
text

src/frontend/components/
  ├── [existing components unchanged]
  │
  ├── ScenarioCard.vue              ← Main scenario output card
  │   ├── PolicymakerSummary.vue    ← Plain Dutch B1 summary
  │   ├── StakeholderTabs.vue       ← Housing/DataCentre/Water/Company tabs
  │   ├── QuantifiedImpact.vue      ← The key number for each stakeholder
  │   ├── AssumptionPanel.vue       ← All assumptions with source links
  │   ├── DataSourcesPanel.vue      ← All data sources with hyperlinks
  │   └── InsightPanelBridge.vue    ← Shows existing Insight panel inline
  │
  ├── ScenarioComparison.vue        ← Side-by-side A vs B
  │
  ├── MapScenarioLayer.vue          ← Adds scenario overlays to existing map
  │   ├── StakeholderViewSwitcher.vue ← Mode selector (Housing/DataCentre/etc.)
  │   ├── LayerAttributionPanel.vue  ← Source links for each map layer
  │   ├── ProjectMarker.vue          ← Drag-and-drop project placement
  │   └── BeforeAfterToggle.vue      ← WITH vs WITHOUT project comparison
  │
  ├── AssumptionSliders.vue          ← Editable parameters (adjustable assumptions)
  │
  └── ExportBriefingNote.vue         ← PDF export for policymakers
PolicymakerSummary component (plain Dutch first)
vue

<template>
  <div class="policymaker-summary">
    <!-- Plain Dutch summary — always shown first -->
    <div class="summary-headline">
      {{ scenario.results.supply_secure ? '✅' : '⚠️' }}
      {{ headline }}
    </div>

    <div class="key-number">
      <span class="number">{{ formatM3(scenario.results.supply_gap_m3.value) }}</span>
      <span class="unit">m³ per dag tekort</span>
      <a :href="scenario.results.supply_gap_m3.source_url"
         target="_blank"
         class="source-link"
         title="Bekijk de brondata">
        📎 {{ scenario.results.supply_gap_m3.source_name }}
      </a>
    </div>

    <div class="risk-onset" v-if="scenario.results.risk_onset_year">
      Eerste tekort verwacht in
      <strong>{{ scenario.results.risk_onset_year }}</strong>
      onder klimaatscenario
      <a href="https://klimaatscenarios.knmi.nl" target="_blank">
        {{ scenario.climate_scenario }}
      </a>
      (KNMI'23)
    </div>

    <!-- Technical detail on expand -->
    <details>
      <summary>Technische details tonen</summary>
      <ScenarioDetailPanel :scenario="scenario" />
    </details>
  </div>
</template>
DataSourcesPanel component (non-negotiable)
vue

<template>
  <div class="data-sources-panel">
    <h3>📎 Gebruikte databronnen</h3>
    <p class="explanation">
      Alle berekeningen in dit scenario zijn gebaseerd op de volgende
      openbare bronnen. Klik op een bron voor de originele data.
    </p>

    <table>
      <thead>
        <tr>
          <th>Dataset</th>
          <th>Organisatie</th>
          <th>Bijgewerkt</th>
          <th>Licentie</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="source in scenario.data_sources_used" :key="source.source_id">
          <td>
            <a :href="source.url" target="_blank">{{ source.name }}</a>
            <span v-if="source.freshness_warning" class="stale-warning">
              ⚠️ Data ouder dan 6 maanden
            </span>
          </td>
          <td>{{ source.organisation }}</td>
          <td>{{ source.publication_date }}</td>
          <td>{{ source.license }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>
Export as briefing note (policymaker feature)
A one-click "Exporteer als PDF-notitie" button generates a structured policy briefing:

text

DRINKWATERSCENARIO NOTITIE
Provincie Zuid-Holland — Gegenereerd op [datum]
Scenario: [naam]
---
SAMENVATTING
[PolicymakerSummary text]

KERNGETALLEN
- Dagelijkse vraag 2040: [X] m³/dag [bron: VEWIN]
- Beschikbare capaciteit: [Y] m³/dag [bron: productieketen ZH]
- Tekort: [Z] m³/dag ([%]) [bron: berekening]
- Eerste tekort: [jaar] [bron: berekening]

EFFECTEN PER STAKEHOLDER
[StakeholderImpacts with quantified metrics and sources]

GEBRUIKTE AANNAMES
[All assumptions with source URLs]

GEBRUIKTE DATA
[All DataSource entries with URLs]

BEREKENINGSMETHODE
[Reasoning steps from Insight panel]

BRONNEN
[Full bibliography with hyperlinks]
<a name="build-plan"></a>

11. 📅 Step-by-Step Build Plan
Pre-hackathon (before June 4 — critical)
Task	Why critical	Time
Clone govtechnl/onegov2-spatial-assistant	Understand what already exists	15 min
Run docker-compose up — verify it works end-to-end	Catch environment issues early	30 min
Open DuckDB, run .show tables on all loaded themes	Verify daily_capacity_m3 exists in productieketen	30 min
Read docs/example-scenarios.md	Choose demo scenarios from the official list	20 min
Read docs/data-inventory.md	Understand what data is already loaded	30 min
Read docs/workflow.mmd	Understand existing LangGraph graph	20 min
Download KNMI'23 Hd/Hn/Ld/Ln parameters as CSV from klimaatscenarios.knmi.nl	Backup if API is slow at venue	20 min
Test waterinfo.rws.nl chloride API for Hollandse IJssel	Check data availability before Day 1	30 min
Verify GreenPT API key works — call /v1/models	Avoid API issues on Day 1	10 min
Write all dataclasses with source URL fields	Establishes source-traceability contract	45 min
Sketch stakeholder view switcher on paper	Agree on 4 stakeholder modes	20 min
Day 1 — June 4 (Build Day)
Block 1 (09:00–11:00): Scenario branch wiring + data fetcher
Task	Output	Checkpoint
Add extract_scenario_params node to existing LangGraph graph	New node wired into existing graph	Does existing workflow still work? ✅
Implement KNMI'23 parameter table (Hd/Hn/Ld/Ln)	data/knmi23_scenarios.py with source URLs	
Implement population growth table with woondeals source	data/population_growth.py	
Implement select_datasets node with SCENARIO_DATASET_MAP	Only relevant columns loaded per scenario	
Implement fetch_spatial_context node	DuckDB queries for productieketen + gebiedsviewer	Can you print a table of intake points? ✅
Block 2 (11:00–13:00): Scenario object + calculation engine
Task	Output	Checkpoint
Implement build_scenario_object node — all SourcedValue fields	Assumption list with source URLs	
Implement run_scenario_math — demand + capacity + gap + KRW	ScenarioResults with all SourcedValue	Baseline Hd + hoog growth = plausible gap ✅
Implement project water demand formulas (data centre, housing, company)	Project scenarios calculable	50MW data centre = 1,800 m³/day ✅
Unit tests: baseline → gap ≈ 0; extreme → large gap; intervention → gap reduced	Tests pass	
Block 3 (14:00–16:00): Stakeholder impacts + map overlays
Task	Output	Checkpoint
Implement compute_stakeholder_impacts — all quantified with sources	6 stakeholder impacts with SourcedValue metrics	Housing units at risk = computed number ✅
Implement generate_map_overlays — risk choropleth + stakeholder layers	MapOverlayBundle GeoJSON	Map shows coloured intake points ✅
Implement record_reasoning_to_insight — feed into Insight panel	MLflow logs scenario params + metrics	MLflow UI shows scenario run ✅
Block 4 (16:00–18:00): API + frontend scaffold
Task	Output	Checkpoint
Wire new nodes into FastAPI (existing endpoints extended, not replaced)	POST /scenario/run works	API returns full scenario card ✅
Build ScenarioCard.vue and PolicymakerSummary.vue	Scenario card visible in UI	
Build MapScenarioLayer.vue — risk choropleth on existing map	Map shows scenario overlays	Risk zones visible on map ✅
Build DataSourcesPanel.vue — all sources with hyperlinks	Source panel visible and linked	Every number has a link ✅
Block 5 (19:00–21:00): Stakeholder views + project marker
Task	Output	Checkpoint
Build StakeholderViewSwitcher.vue — 4 modes	Mode switching works	Same scenario, different views ✅
Build ProjectMarker.vue — drag-and-drop location input	Project location feeds into scenario	Data centre in Delft computable ✅
Build AssumptionSliders.vue — adjustable parameters	Slider rerun works via API	Slider changes recalculate scenario ✅
Build ScenarioComparison.vue — A vs B side by side	Two scenarios visible simultaneously	
Day 2 — June 5 (Polish + Demo + Pitch)
Block 1 (09:00–10:30): Polish + traceability audit
Task	Output
Audit: every number in the UI has a source link	100% of SourcedValue fields have URLs
Add LayerAttributionPanel.vue to map	Every map layer has source link
Add BeforeAfterToggle.vue for project scenarios	WITH / WITHOUT project comparison works
Add ExportBriefingNote.vue — PDF export	Policymaker can export the scenario
Test: fresh clone → docker-compose up → demo works	Reproducibility confirmed
Add docs/extending.md — how to add new scenario types	Extensibility criterion met
Block 2 (10:30–12:00): Demo preparation
Task	Output
Script the 3-minute demo (see below)	Demo script
Pre-run and cache demo scenarios locally	Backup ready if API is slow
Screen-record a successful end-to-end run	Backup video
Push to GitHub — confirm README is readable	Repo public
Blocks 3–4 (13:00–15:30): Pitch + submission
Task	Output
Finalise pitch deck (10 slides, see below)	Deck complete
Rehearse pitch twice	Under 3 minutes
Upload to Alkemio: deck + repo link + demo video	Submitted
Demo Script (3 minutes, live)
text

[0:00 – 0:20] THE PROBLEM — PLAIN DUTCH
"Stel, u bent beleidsmedewerker bij Provincie Zuid-Holland.
Een ontwikkelaar vraagt vergunning voor een 50 megawatt
datacentrum in Delft. Heeft Zuid-Holland genoeg drinkwater
voor dit datacentrum in 2040, als we ook nog 230.000 woningen
bouwen — en het klimaat droger wordt?
Vandaag kost die vraag dagen. Wij doen het in 30 seconden."

[0:20 – 0:55] LIVE SCENARIO 1 — PROJECT IMPACT
Type: "Wat is het effect van een 50MW datacentrum in Delft
       op de drinkwaterlevering in 2040, klimaatscenario Hd?"

Show system parsing: "Scenario type: data_centre_impact.
Parameters: Hd klimaat, middel bevolkingsgroei, geen interventie.
Datasets: productieketen + gebiedsviewer/verzilting + zes-uurszones."

Show scenario card appearing:
"Tekort: 169.000 m³/dag. Eerste tekort: 2036.
Dit datacentrum draagt 1.800 m³/dag bij aan dat tekort.
47.000 woningen in de Gouda-zone lopen risico."

Show the map: intake points red in Gouda zone.
"Every number links to its source — here, VEWIN, KNMI'23,
and the productieketen from opendata.Zuid-Holland.nl."

[0:55 – 1:25] STAKEHOLDER VIEWS ON THE MAP
Click "Woningbouw" view:
"Same scenario, housing developer lens. 47,000 units at risk —
computed as supply gap divided by 350L per unit per day."

Click "Waterautoriteit" view:
"Water authority lens. Production chain, alternative intakes,
buffer capacity locations."

Click "Provincie" view:
"Policy lens. Risk onset year per municipality — 2036 for
Gouda supply zone, 2039 for Delfland, safe until 2040+
for Rotterdam supply."

[1:25 – 1:50] COMPARISON — INTERVENTION EFFECT
"Now let's add buffer capacity as an intervention."
Click "Vergelijk met scenario B" — add "bufferkapaciteit."
Show side-by-side: gap drops from 169,000 to 41,000 m³/day.
"With buffer capacity, risk onset moves to 2039. The data
centre remains feasible. 47,000 homes drop to 11,000 at risk."

[1:50 – 2:15] TRACEABILITY — FOR POLICYMAKERS
Click "Aannames en bronnen."
Show DataSourcesPanel — every dataset with hyperlink.
"KNMI'23 klimaatscenario Hd — link. Productieketen
drinkwater ZH — link. Capaciteitskaart woondeals — link.
Waterinfo RWS chloride — link. Every number, every source."

Open Insight panel — show reasoning steps.
"Six steps in plain Dutch. What data was used, what was
assumed, what was calculated. A policymaker, an auditor,
a Woo request — can follow this without seeing any code."

[2:15 – 2:40] HOUSING SCENARIO (second scenario)
Type: "2.500 woningen Zoetermeer 2035 — drinkwater haalbaar?"
Show result in 10 seconds (cached).
"810 m³/day additional demand. 430 m³/day headroom remains
under Ld scenario. Feasible — but tight. Under Hd: deficit
from 2034. Condition recommended: buffer capacity."

[2:40 – 3:00] CLOSE
"Any spatial decision in Zuid-Holland has water consequences.
Today, nobody knows what they are. This tool makes them
visible — in 30 seconds, for any policymaker, linked to
every authoritative source. Thank you."
<a name="validation"></a>

12. ✅ Validation & Quality Framework
Level 1 — Brief Must criteria
Check	Verification method	Pass condition
Works with provided datasets	DuckDB query on drinkwaterzekerheid + gebiedsviewer	Data loads, scenario runs
Combines ≥ 2 data themes	Count distinct themes per scenario	All demo scenarios use ≥ 2 themes
Open source + readable README	Fresh clone → docker-compose up	Runs in < 5 minutes
NL question → calculable scenario	5 different phrasings produce valid scenarios	All 5 pass
Level 2 — Brief Should criteria
Check	Verification method	Pass condition
≥ 2 scenarios / comparisons in demo	Demo script runs 2 scenarios end-to-end	Both complete
Reasoning chain in Insight panel	MLflow run shows all node transitions	≥ 6 steps logged
Uncertainty + time horizon shown	Scenario card shows 2025 baseline vs 2040 projection	Both years visible
Ambiguous input → clarifying question	Test with: "wat als het droog wordt?"	System asks follow-up
Level 3 — Traceability (non-negotiable)
Check	Verification method	Pass condition
Every displayed number has source URL	Audit all SourcedValue fields in frontend	100% have URLs
Every map layer has attribution	Check LayerAttributionPanel	All layers attributed
Every assumption has source URL	Audit Assumption objects	100% have URLs
PDF export includes all sources	Generate export, check bibliography	Full source list present
Level 4 — Stakeholder quantification
Stakeholder	Metric	Source
Woningzoekenden	supply_gap_m3 / 0.35 = units at risk	VEWIN
Data centres	Facilities in affected zes-uurszones	Productieketen ZH
Agrariërs	Ha at risk in high verzilting + agricultural LGN	PDOK/LGN
Gemeenten	Population in affected supply zones	CBS
Zorginstellingen	Care beds in affected zones	BAG/CBS
Level 5 — Extensibility (brief criterion)
Check	Verification method	Pass condition
docs/extending.md exists and is readable	Open and read	< 10 minutes to understand
Adding a new scenario type requires only config change	Code review	No core logic changes needed
Adding a new region works via gemeente_code parameter	Test with a second municipality	Runs without code change
<a name="pitch"></a>

13. 🎤 Pitch Narrative (10 Slides)
SLIDE 1 — Title

"Drinkwaterscenario's op commando: van beleidsvraag naar berekend antwoord — voor elke stakeholder, op kaart" Team name | GovTechNL OneGov #2 | 5 juni 2026

SLIDE 2 — The Problem (30 seconds)

"Spatial decisions in Zuid-Holland — a data centre, a housing project, a business relocation — all have water consequences. Today, no policymaker knows what those are before making the decision. The knowledge exists, scattered across Provincie ZH, RWS, KNMI, and the drinkwaterbedrijven. It just isn't connected — and it isn't calculable in real time."

Visual: The "permission question" cycle — developer asks → months of consultancy → answer arrives after the decision.

SLIDE 3 — Our Answer (20 seconds)

"We extended the existing Zuid-Holland spatial assistant with a scenario engine. Type any spatial policy question. The engine connects the data, runs the calculation, and shows every stakeholder what it means — on a map, with every source cited. In 30 seconds."

Visual: NL question → 8-node extension → scenario card + map + sources.

SLIDE 4 — Live Demo (60 seconds) [Demo as scripted above — data centre, housing project, comparison, sources panel]

SLIDE 5 — For Every Stakeholder (20 seconds)

"The same scenario tells different stories to different people. A housing developer sees homes at risk. A water authority sees m³/day. A province sees which municipalities run out of headroom first. We built five stakeholder views over the same calculation — no data duplicated, no story fabricated."

Visual: Stakeholder view switcher — same map, five audience modes.

SLIDE 6 — Traceability is Non-Negotiable (20 seconds)

"Every number is a link. Every assumption has a source. Every map layer is attributed. This is a government tool — policymakers are accountable for their decisions, and they need to know exactly where every number comes from. We built that in from the start, not as an afterthought."

Visual: DataSourcesPanel with hyperlinks highlighted. Insight panel showing reasoning steps.

SLIDE 7 — GreenPT: Dutch Policy-Grade Reasoning (15 seconds)

"The reasoning trail — six steps in plain Dutch, linking data to calculation to conclusion — is written by GreenPT. Not translated from English. Written in Dutch, for a Dutch policy audience, at B1 reading level. That's the difference between a tool that explains itself and one that doesn't."

Visual: A reasoning step in plain Dutch with GreenPT attribution.

SLIDE 8 — Architecture: Extend, Don't Rebuild (15 seconds)

"We did not rebuild the existing spatial assistant. We added eight new LangGraph nodes to an existing, tested workflow. The existing Insight panel and MLflow tracing do the navolgbaarheid work. Every existing feature still works. We added scenario capability on top."

Visual: Workflow diagram — existing nodes grey, new nodes highlighted.

SLIDE 9 — FDS + NORA Alignment (15 seconds)

"Data bij de bron — every dataset fetched from its authoritative source, never copied. Bronregistraties — every value traced to its register. Open standards — WFS, OGC API, CBS OData. This tool is designed to fit into the Dutch government data ecosystem from day one."

Visual: FDS/NORA principle grid with checkmarks.

SLIDE 10 — What's Next (15 seconds)

"Today: Zuid-Holland, six stakeholder types, eight scenario templates. Next: extend to all Dutch provinces, connect to live Waterinfo API for real-time chloride monitoring, integrate with the omgevingsvergunning process. The scenario engine is built to grow. We'd welcome co-development with Provincie ZH, RWS, KNMI, and the drinkwaterbedrijven."

Visual: Roadmap + logos: Provincie ZH, RWS, KNMI, Dunea, Evides, BZK.

<a name="repo-structure"></a>

14. 📁 Repo Structure
text

onegov2-spatial-assistant-[teamname]/
│
├── README.md                            ← Setup, architecture, how to extend
│                                          Links to all data sources
├── LICENSE                              ← EUPL-1.2
├── docker-compose.yml                   ← Existing; unchanged
├── .env.example                         ← GREENPT_KEY as primary
│
├── src/
│   ├── frontend/                        ← Vue 3 (existing base — unchanged)
│   │   ├── components/
│   │   │   ├── [existing components]    ← All unchanged
│   │   │   │
│   │   │   ├── ScenarioCard.vue         ← NEW
│   │   │   ├── PolicymakerSummary.vue   ← NEW — plain Dutch first
│   │   │   ├── StakeholderTabs.vue      ← NEW — 5 audience modes
│   │   │   ├── QuantifiedImpact.vue     ← NEW — sourced numbers
│   │   │   ├── AssumptionPanel.vue      ← NEW — assumptions + sources
│   │   │   ├── DataSourcesPanel.vue     ← NEW — hyperlinked sources
│   │   │   ├── InsightPanelBridge.vue   ← NEW — existing Insight inline
│   │   │   ├── ScenarioComparison.vue   ← NEW — A vs B
│   │   │   ├── AssumptionSliders.vue    ← NEW — editable parameters
│   │   │   ├── MapScenarioLayer.vue     ← NEW — scenario overlays on map
│   │   │   ├── StakeholderViewSwitcher.vue ← NEW
│   │   │   ├── LayerAttributionPanel.vue   ← NEW — source links per layer
│   │   │   ├── ProjectMarker.vue        ← NEW — drag-and-drop location
│   │   │   ├── BeforeAfterToggle.vue    ← NEW — WITH vs WITHOUT project
│   │   │   └── ExportBriefingNote.vue   ← NEW — PDF for policymakers
│   │   └── views/ [existing, unchanged]
│   │
│   └── backend/                         ← FastAPI (existing base)
│       ├── main.py                      ← Existing; add new router
│       ├── routers/
│       │   ├── [existing routers]       ← All unchanged
│       │   └── scenario_router.py       ← NEW: /scenario/* endpoints
│       │
│       ├── workflow/
│       │   ├── graph.py                 ← Existing; ADD scenario branch
│       │   ├── nodes/
│       │   │   ├── [existing nodes]     ← All unchanged
│       │   │   ├── extract_scenario_params.py  ← NEW Node 1
│       │   │   ├── select_datasets.py           ← NEW Node 2
│       │   │   ├── fetch_spatial_context.py     ← NEW Node 3
│       │   │   ├── build_scenario_object.py     ← NEW Node 4
│       │   │   ├── run_scenario_math.py         ← NEW Node 5
│       │   │   ├── compute_stakeholder_impacts.py ← NEW Node 6
│       │   │   ├── generate_map_overlays.py     ← NEW Node 7
│       │   │   └── record_reasoning_to_insight.py ← NEW Node 8
│       │   └── state.py                 ← Existing; extend with scenario state
│       │
│       ├── models/                      ← NEW folder
│       │   ├── scenario.py              ← All dataclasses
│       │   ├── sourced_value.py         ← SourcedValue (non-negotiable)
│       │   ├── stakeholder_impact.py
│       │   ├── map_overlay.py
│       │   └── assumption.py
│       │
│       ├── data/                        ← NEW folder
│       │   ├── knmi23_scenarios.py      ← Hd/Hn/Ld/Ln params + source URLs
│       │   ├── population_growth.py     ← Growth params + woondeals source
│       │   ├── intervention_catalogue.py
│       │   ├── project_water_demand.py  ← Data centre/housing/company formulas
│       │   └── stakeholder_rules.py
│       │
│       ├── services/
│       │   ├── [existing services]
│       │   ├── scenario_store.py        ← NEW
│       │   ├── waterinfo_client.py      ← NEW: waterinfo.rws.nl API
│       │   └── export_service.py        ← NEW: PDF briefing export
│       │
│       └── extra_data/                  ← Existing; add only needed files
│           ├── [existing extra data]
│           └── knmi23_parameters.csv    ← NEW: KNMI'23 backup
│
├── docs/                                ← Add to existing docs/
│   ├── [existing docs unchanged]
│   ├── extending.md                     ← NEW: how to add scenarios/regions
│   ├── data_sources.md                  ← NEW: all sources with URLs + licenses
│   └── fds_nora_alignment.md            ← NEW: architecture alignment note
│
└── tests/
    ├── [existing tests unchanged]
    ├── test_scenario_math.py            ← NEW
    ├── test_stakeholder_impacts.py      ← NEW
    └── test_source_traceability.py      ← NEW: every value has a URL
<a name="risks"></a>

15. ⚠️ Risk Register
Risk	Likelihood	Impact	Mitigation
DuckDB datacube missing daily_capacity_m3	Medium	🔴 Critical	Day 0 blocking check: .show tables + .describe productieketen. If missing: use VEWIN waterstatistieken as fallback source.
waterinfo.rws.nl API slow or blocked	Medium	High	Pre-download chloride data for Hollandse IJssel as CSV backup before June 4.
klimaatscenarios.knmi.nl API unavailable	Low	Medium	KNMI'23 parameters are a finite table — pre-download as knmi23_parameters.csv.
GreenPT API rate-limited at demo time	Medium	High	Pre-cache all demo scenario outputs locally. Demo runs from cache if API is slow.
LangGraph extension breaks existing workflow	Medium	🔴 Critical	Add scenario branch as a conditional path off check_intent — test that existing descriptive questions still work after each change.
Stakeholder quantification challenged as too simple	High	Medium	Acknowledge proactively: "Policy-level approximations. Formulas are in stakeholder_rules.py — domain experts can calibrate." Show the source for each formula.
Source URL for a displayed value is missing	Medium	High	test_source_traceability.py tests that every SourcedValue has a non-empty URL. Runs on every commit.
Map layers in wrong CRS	Medium	Medium	All spatial operations in EPSG:28992; reproject to EPSG:4326 for output. Unit test CRS.
Project water demand formula challenged	Medium	Low	Every formula has a source URL. If challenged: "Here is the IEA/VEWIN source — expert review can refine this formula."
Demo runs over 3 minutes	High	Medium	Hard cut at 2:40 — drop the housing scenario if behind. Pre-cache both scenarios.
opendata.Zuid-Holland.nl GeoNetwork slow	Medium	Medium	All provincial data is pre-loaded in the starter repo's gebiedsviewer theme — don't re-fetch from portal during demo.
"Extend don't rebuild" challenged by jury	Low	High	Show the existing workflow diagram with new nodes highlighted in a different colour. Point out that all 7 existing nodes are unchanged.
Woondeals data not in expected schema	Medium	Medium	Pre-explore extra_data/woondeals on Day 0. Adapt column names in select_datasets before building anything on top.
Summary: What Makes This Design Brief-Perfect
Brief requirement	This design	Status
NL → calculable scenario	8-node LangGraph extension, ScenarioResults object	✅
Visible data + assumptions	SourcedValue + DataSourcesPanel + AssumptionPanel — all with hyperlinks	✅
Effects on affected parties	6 stakeholder types, all quantified with sources	✅
≥ 2 data themes per scenario	SCENARIO_DATASET_MAP enforces this	✅
≥ 2 scenarios in demo	Data centre + housing project (from example-scenarios.md)	✅
Navolgbaar reasoning	Existing Insight panel + MLflow — Node 8 feeds into it	✅
Uncertainty + time horizon	2025 baseline vs 2040 projection, KNMI'23 labels	✅
Ambiguous input → clarification	Confidence threshold + follow-up question	✅
Extend, don't rebuild	7 existing nodes unchanged; 8 new nodes added as branch	✅
Dataset selection (no column bloat)	select_datasets node with SCENARIO_DATASET_MAP	✅
Open source + README	Docker Compose, README with source links	✅
Extensible to other challenges	docs/extending.md + config-driven	✅
GreenPT best use	Primary LLM for Dutch B1 reasoning steps, default not fallback	✅
Vibe coding prize	Modular spec, every node independently promptable	✅
Policymaker-readable	PolicymakerSummary, stakeholder views, PDF export	✅
Everything traceable to source	SourcedValue contract, test_source_traceability.py	✅
Interactive map per stakeholder	5 stakeholder view modes, project marker, before/after	✅
FDS + NORA alignment	docs/fds_nora_alignment.md, data-bij-de-bron architecture	✅





