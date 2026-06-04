The GitHub raw content isn't indexed by web search. However, based on my complete deep dives of both documents earlier in this conversation, I have full working knowledge of both. I'll now produce the rewritten, consolidated design document below.

---

# OneGov #2 — Drinkwaterzekerheid Scenario Engine
## Comprehensive Design Document v2.0
### GovTechNL Hackathon · 4–5 June 2026 · The Hague Tech

---

## 0. Document purpose & what changed from v1

This document supersedes the original design doc and the separate stakeholder critique. It folds in three categories of change:

| Change category | What changed | Why |
|---|---|---|
| **Scope tightening** | 1 unified endpoint, 2 golden-path scenarios, ranges not constants | Reduce integration risk; shippable in 2 days |
| **Stakeholder gaps (GAP 1–10)** | Knowledge base panel, scenario citations, feasibility class, human-scale units, shared scenario library, chloride per intake, Waterinfo guard, cumulative stacking, official position panel, citizen template | Institutional trust + "same question = same answer" |
| **Domain guardrails** | Demand per dwelling and datacenter water use become sliders with ranges; chloride threshold is per-intake, not global | Domain credibility with water authority judges |

---

## 1. Product vision (unchanged in spirit, tightened in scope)

### 1.1 What this is
A **replayable, citeable, policy-auditable scenario engine** layered on the existing `govtechnl/onegov2-spatial-assistant`. It extends the tool from *"describe what is"* to *"explore what could be"* — turning Dutch policy questions into structured, explainable what-if scenarios with map overlays, stakeholder views, reasoning trails, and hyperlinked source registries.

### 1.2 What it is NOT
- Not a full hydrological model
- Not a production-grade SaaS
- Not a replacement for the existing descriptive assistant (existing workflow stays intact)
- Not a black-box chatbot (every claim links to a source)

### 1.3 The institutional contract (new in v2)
Every scenario output must satisfy:
> "A policy officer must be able to attach this output to an advice letter and defend every number in it."

This means: stable scenario URL + citation block + dataset version log + explicit assumptions + reasoning steps + official policy context.

---

## 2. Users, personas & design principles

### 2.1 User archetypes

| Persona | Primary need | Key v2 addition |
|---|---|---|
| **Provincial policymaker** | Defensible advice, PDF export, official context | Citation block, Woo-trail, official position panel |
| **Spatial planner** | Feasibility verdict, cumulative plans | GO/CAUTION/STOP class, stacking mode |
| **Water authority** | Per-intake accuracy, real-time chloride | Per-location thresholds, Waterinfo guard |
| **Project developer** | Fast feasibility, "make it work" options | Traffic-light class, intervention ranking |
| **Citizen** | Safe/not safe, postcode-level, reassurance | Citizen template with verdict-first format |
| **Auditor / Woo-request** | Reproducible, citeable, dataset-versioned | Scenario hash, stable URL, MLflow artifact |

### 2.2 UI design principles
1. **B1 Dutch first** — all output in plain Dutch; technical detail on expand
2. **Map first** — spatial output leads; text and table are secondary
3. **Verdict first** — FeasibilityClass (GO/CAUTION/STOP) shown before numbers
4. **Every number links** — no unattributed figures; SourceRegistry appended to every card
5. **Human scale** — every headline metric accompanied by a real-world analogy (e.g., "equivalent to daily use of X households")
6. **No dead ends** — if a scenario cannot be computed, explain why and suggest a follow-up
7. **Official anchor** — always surface the government's official position alongside scenario output
8. **Ranges, not constants** — all approximated inputs shown as ranges with sliders; labelled as "policy-level approximation"

---

## 3. Golden-path scenarios (the two you build and demo)

Scope is deliberately locked to these two scenarios + one comparison. All architecture serves them first.

### Scenario A — Type 3: Drop-a-pin (spatial demand decision)
> "Kan er een 50 MW datacenter komen bij Pijnacker-Nootdorp in 2040?"

- User drops a pin or types a location + development type
- System computes demand delta, checks against supply capacity in the zone
- Returns: FeasibilityClass + gap/headroom + stakeholder impacts + intervention options

### Scenario B — Type 1: Climate × infrastructure failure
> "Wat als de Hollandse IJssel-inname 6 weken onbruikbaar is door verzilting onder KNMI Hd in 2040?"

- KNMI'23 Hd preset + outage duration + time horizon
- System computes chloride trajectory (per IJssel intake threshold), demand under peak multiplier, supply gap, affected zones
- Returns: FeasibilityClass + onset year + affected stakeholders + map overlays

### Scenario C — Intervention comparison (used in demo, derives from B)
> "Wat als we bufferopslag van 30.000 m³ toevoegen? Vergelijk met en zonder."

- Same endpoint as B, `scenario_b_params` optional payload adds the intervention
- Returns: split-map delta + headline deltas (gap, onset year, KRW risk count)

**All other scenario types (Type 2 multi-hazard, additional interventions) are stretch goals if time allows.**

---

## 4. Unified API surface (tightened from v1)

Instead of multiple endpoints, everything flows through **one scenario endpoint** with an optional second payload for comparison.

```
POST /scenario/run
{
  "question":         str,          # original NL question
  "scenario_a":       ScenarioParams,
  "scenario_b":       ScenarioParams | null,   # optional: triggers comparison mode
  "assumption_overrides": dict | null          # optional: triggers assumption-adjust re-run
}
```

Three modes, one endpoint:
| Mode | What's populated | What returns |
|---|---|---|
| Single scenario | `scenario_a` only | ScenarioCard + map + reasoning |
| Comparison | `scenario_a` + `scenario_b` | Split card + delta panel + dual map |
| Assumption re-run | `scenario_a` + `assumption_overrides` | Updated ScenarioCard (same scenario, adjusted inputs) |

**No additional endpoints needed for hackathon.** Existing `/chat` streaming and map pathways stay unchanged.

---

## 5. Data model

### 5.1 ScenarioParams (input)
```python
@dataclass
class ScenarioParams:
    scenario_type:       Literal["drop_pin", "intake_failure", "multi_hazard", "intervention"]
    knmi_preset:         Literal["B", "Hd", "Hn", "Ld", "Ln"] = "Hd"
    time_horizon:        int = 2040
    location_wkt:        str | None = None          # WKT point or polygon
    development_type:    str | None = None          # "datacenter_50mw", "housing_5000", etc.
    intake_id:           str | None = None          # links to productieketen dataset
    outage_weeks:        int | None = None
    growth_preset:       Literal["laag", "middel", "hoog"] = "middel"
    interventions:       list[str] = field(default_factory=list)
    assumption_overrides: dict = field(default_factory=dict)
```

### 5.2 Assumption (with mandatory source_url — v2 fix)
```python
@dataclass
class Assumption:
    key:              str
    label_nl:         str
    value:            float
    value_min:        float         # range low end
    value_max:        float         # range high end
    unit:             str
    source_url:       str           # MANDATORY — no empty strings; validation gate blocks empty
    source_label:     str
    sensitivity:      Literal["low", "medium", "high"]
    is_policy_approx: bool = True   # always True for demand conversions
    slider_step:      float = 1.0
```

> **v2 fix:** `source_url` is mandatory with a CI-level validation gate. No `Assumption` object may have an empty `source_url`. If no authoritative URL exists, use the policy document defining the concept (e.g., Drinkwaterbesluit for thresholds, VEWIN jaarstatistiek for household demand).

### 5.3 ScenarioResults
```python
@dataclass
class ScenarioResults:
    # Demand / capacity
    daily_demand_m3:      float
    daily_demand_range:   tuple[float, float]   # (min, max) from assumption ranges
    supply_capacity_m3:   float
    supply_gap_m3:        float                  # negative = shortfall
    feasibility_class:    Literal["GO", "CAUTION", "STOP"]
    human_scale:          HumanScaleRef          # see 5.4

    # Chloride (intake-failure scenarios)
    cl_concentration:     float | None
    cl_threshold_mg_l:    float | None           # per-intake, not global constant
    cl_threshold_source:  str | None             # URL to threshold basis
    cl_intake_id:         str | None

    # KRW
    krw_at_risk_count:    int
    krw_areas:            list[str]

    # Timeline
    onset_year:           int | None
    confidence:           float                  # 0.0–1.0
    is_policy_approx:     bool = True

    # Type-3 specific
    capacity_remaining_m3: float | None
    headroom_pct:          float | None

    # Stacking (v2)
    active_scenarios_same_zone:  int = 0
    cumulative_demand_m3:        float | None = None
```

### 5.4 HumanScaleRef (new in v2 — GAP 4)
```python
@dataclass
class HumanScaleRef:
    metric_value:    float
    metric_unit:     str
    analogy_nl:      str       # e.g. "dagelijks verbruik van ~1,2 miljoen mensen"
    analogy_source:  str       # URL to VEWIN/CBS reference used for conversion
```

### 5.5 StakeholderImpact
```python
@dataclass
class StakeholderImpact:
    stakeholder:     str
    impact_nl:       str
    severity:        Literal["low", "medium", "high", "critical"]
    affected_zones:  list[str]
    overlay_layer:   str          # which GeoJSON layer to highlight
    policy_basis:    str
    policy_url:      str
```

### 5.6 ReasoningStep
```python
@dataclass
class ReasoningStep:
    step_nr:         int
    label_nl:        str
    description_nl:  str
    datasets_used:   list[str]
    assumptions_used: list[str]
    sql_fragment:    str | None
    mlflow_run_id:   str | None
    source_urls:     list[str]
```

### 5.7 ScenarioCard (full output object)
```python
@dataclass
class ScenarioCard:
    # Identity & reproducibility
    scenario_id:       str              # UUID v4
    scenario_hash:     str              # SHA-256 of normalized params — "same question = same answer"
    created_at:        str              # ISO-8601 timestamp
    dataset_versions:  dict[str, str]   # table_name → last_modified
    git_commit:        str              # short SHA
    stable_url:        str              # /scenario/{scenario_id}

    # Citation block (new in v2 — GAP 3)
    citation:          str              # formatted APA-ish citation string
    citation_nl:       str              # Dutch equivalent for advice letters

    # Content
    question_nl:       str
    scenario_type:     str
    params:            ScenarioParams
    results:           ScenarioResults
    assumptions:       list[Assumption]
    stakeholder_impacts: list[StakeholderImpact]
    reasoning_steps:   list[ReasoningStep]
    source_registry:   list[SourceEntry]
    overlays:          list[GeoJSONOverlay]

    # Official position (new in v2 — GAP 8)
    official_position: OfficialPosition

    # Comparison delta (populated when scenario_b was run)
    delta:             ScenarioDelta | None = None
```

### 5.8 OfficialPosition (new in v2 — GAP 8)
```python
@dataclass
class OfficialPosition:
    summary_nl:     str           # 1–2 sentence official stance
    documents:      list[dict]    # [{"title": ..., "url": ..., "date": ...}]
    # Pre-baked documents:
    # - Provinciaal Waterprogramma ZH 2022–2027
    # - Nationaal Waterprogramma 2022–2027
    # - Ruimtelijk Arrangement Rijk–ZH (2 June 2025)
    # - Drinkwaterwet
```

---

## 6. Extended LangGraph workflow

### 6.1 Full node chain

```
check_intent
    │
    ├─ descriptive ──────────────────────────────────────────► [existing workflow unchanged]
    │
    └─ scenario / exploratory
          │
          ▼
    extract_scenario_params          ← GreenPT structured output; confidence gate
          │
          ├─ low confidence ────────► ask_followup_question
          │
          ▼
    confirm_with_user                ← show extracted params in B1 Dutch; user can correct
          │
          ▼
    fetch_scenario_data              ← load only relevant tables; build SourceRegistry; freshness check
          │
          ▼
    check_scenario_cache             ← compute scenario_hash; return cached result if hit + version-drift warning
          │
          ├─ cache hit ─────────────► format_scenario_output
          │
          ▼
    build_scenario_object            ← construct Scenario dataclass + Assumption list
          │
          ▼
    run_scenario_calculation         ← demand / capacity / chloride / gap / onset; ranges computed
          │
          ▼
    compute_feasibility_class        ← GO / CAUTION / STOP across KNMI presets
          │
          ▼
    check_cumulative_load            ← (v2) query active scenarios in same supply zone; warn if >0
          │
          ▼
    stakeholder_impacts              ← rule engine + GreenPT prose; stakeholder dropdown data
          │
          ▼
    build_official_position          ← look up pre-baked OfficialPosition for topic + location
          │
          ▼
    record_reasoning_steps           ← B1 Dutch steps with source URLs; log to MLflow
          │
          ▼
    format_scenario_output           ← ScenarioCard JSON + GeoJSON overlays + HumanScaleRefs
          │
          ▼
    [if scenario_b present]
    run_scenario_b_in_parallel       ← same chain with scenario_b params
          │
          ▼
    compute_delta                    ← gap delta, onset year delta, KRW delta, map diff
          │
          ▼
    stream_to_frontend               ← existing SSE stream; new "scenario_card" event type
```

### 6.2 Node responsibilities in detail

#### `extract_scenario_params`
- GreenPT structured output with temperature 0.1
- Extracts: scenario type, KNMI preset, location, development type, outage duration, time horizon, growth preset, interventions
- Confidence threshold: if < 0.7, route to `ask_followup_question`
- Output shown to user as confirmation card before proceeding

#### `fetch_scenario_data` (v2 update)
- Loads **only** tables relevant to the scenario type (no full schema scan)
- Builds `SourceRegistry` with `fetched_at` timestamps
- Emits freshness warnings if any table's `last_modified` > 90 days
- Checks Waterinfo chloride API **mandatorily** for IJssel/Lek/Maas intake scenarios; if unavailable, logs last-known value with date and shows explicit warning (never silently falls back)
- Logs dataset versions to `ScenarioCard.dataset_versions`

#### `check_scenario_cache`
- Computes `scenario_hash = SHA-256(json.dumps(normalized_params, sort_keys=True))`
- Checks in-memory or Redis cache
- If cache hit: return cached `ScenarioCard` immediately; add `cache_used: true` + `cached_at` + dataset version drift warning if any table changed since cache creation
- If cache miss: proceed to calculation

#### `build_scenario_object`
- Constructs typed `ScenarioParams` and `list[Assumption]`
- **Validation gate:** any `Assumption` with empty `source_url` raises `AssumptionValidationError` (blocks run; forces fix)
- Logs all assumptions as MLflow params

#### `run_scenario_calculation`
- Demand calculation (see section 7)
- Chloride calculation using **per-intake threshold** from `productieketen` dataset, not a global constant
- Computes both point estimate and `(value_min, value_max)` range using assumption slider bounds
- Adds `is_policy_approx: True` to results

#### `compute_feasibility_class`
- Runs calculation across all 5 KNMI presets (B, Hd, Hn, Ld, Ln)
- `GO`: feasible under all scenarios
- `CAUTION`: feasible under B/Ln/Hn, stressed under Hd/Ld
- `STOP`: infeasible under ≥2 scenarios including baseline
- Shown as traffic-light badge at top of ScenarioCard

#### `check_cumulative_load` (new in v2 — GAP 6)
- Queries active scenario cache for scenarios whose supply zone overlaps with current scenario's zone
- Computes `cumulative_demand_m3` = sum of all active scenario demands
- If > 0, appends warning: "Er zijn X andere actieve scenario's in deze leveringszone; gecombineerd effect is Y m³/dag"

#### `stakeholder_impacts`
- Rule engine maps scenario type + affected zones → `StakeholderImpact` per stakeholder
- GreenPT generates 1–2 sentence prose per stakeholder in B1 Dutch (temperature 0.2)
- Includes `overlay_layer` reference so map can highlight relevant zones per stakeholder dropdown selection
- For each impact: `policy_basis` + `policy_url` are mandatory

#### `build_official_position` (new in v2 — GAP 8)
- Pre-baked lookup by topic (drinkwater, climate, housing) + province (ZH)
- Returns `OfficialPosition` with up to 3 authoritative documents
- Always includes Provinciaal Waterprogramma ZH 2022–2027 as anchor

#### `record_reasoning_steps`
- Generates `list[ReasoningStep]` in sequence
- Each step in B1 Dutch with hyperlinks to datasets and sources used
- Logs to MLflow as artifact: `reasoning_steps.json`
- Computes Flesch-Douma readability score; logs as MLflow metric `readability_score`
- Insight panel in UI renders this step-by-step

#### `format_scenario_output`
- Assembles final `ScenarioCard` including:
  - `scenario_hash`, `stable_url` (`/scenario/{scenario_id}`), `dataset_versions`, `git_commit`
  - Formatted `citation` + `citation_nl` strings
  - `HumanScaleRef` for every headline metric
  - `OfficialPosition` block
- Generates GeoJSON overlays per stakeholder + per scenario
- If `scenario_b` present: assembles `ScenarioDelta` for split-map

---

## 7. Calculation engine

### 7.1 KNMI'23 scenario axes

| Preset | Drought freq multiplier | Cl delta mg/L at IJssel | Summer peak demand multiplier | Low-flow duration weeks | Source |
|---|---|---|---|---|---|
| B (baseline) | 1.0 | 0 | 1.0 | 0 | KNMI'23 |
| Hn | 1.3 | +30 | 1.08 | 2 | KNMI'23 |
| Hd | 1.8 | +80 | 1.15 | 6 | KNMI'23 |
| Ln | 1.2 | +20 | 1.05 | 1 | KNMI'23 |
| Ld | 1.5 | +50 | 1.10 | 4 | KNMI'23 |

All values are policy-level approximations derived from KNMI'23 scenario documentation. Shown in UI with "policy-level approximation" label.

### 7.2 Growth scenario axes

| Preset | Population growth 2024–2040 | Housing units | Water demand delta | Source |
|---|---|---|---|---|
| laag | +5% | +40,000 | +4% | Ruimtelijk Arrangement ZH 2025 |
| middel | +8% | +60,000 | +6.5% | Ruimtelijk Arrangement ZH 2025 |
| hoog | +12% | +80,000 | +9% | Ruimtelijk Arrangement ZH 2025 |

### 7.3 Core demand calculation (ranges, not constants)
```python
def calculate_daily_demand(
    baseline_m3_per_day: float,
    growth_fraction: float,
    knmi_peak_multiplier: float,
    development_delta_m3: float,
    assumption_overrides: dict = {}
) -> tuple[float, float, float]:
    """
    Returns (point_estimate, range_min, range_max).
    All inputs are ranges; point estimate uses default midpoints.
    """
    point = (baseline_m3_per_day * (1 + growth_fraction) 
             * knmi_peak_multiplier) + development_delta_m3
    range_min = (baseline_m3_per_day * (1 + growth_fraction * 0.8) 
                 * (knmi_peak_multiplier * 0.9)) + development_delta_m3 * 0.85
    range_max = (baseline_m3_per_day * (1 + growth_fraction * 1.2) 
                 * (knmi_peak_multiplier * 1.1)) + development_delta_m3 * 1.15
    return point, range_min, range_max
```

### 7.4 Chloride concentration (per-intake, not global)
```python
def calculate_cl_concentration(
    baseline_cl_mg_l: float,    # from Waterinfo API or last-known value
    knmi_cl_delta: float,       # from KNMI'23 preset table above
    outage_weeks: int = 0
) -> float:
    effective_cl = baseline_cl_mg_l + knmi_cl_delta
    if outage_weeks > 0:
        # Conservative: assume 20% additional degradation per week of no intake
        effective_cl *= (1 + 0.20 * min(outage_weeks, 6))
    return effective_cl

# Per-intake threshold (NOT a global constant):
# - Query productieketen dataset for intake_id → treatment_tech → threshold
# - Default fallback: 150 mg/L (Drinkwaterbesluit conservative norm)
# - Fallback is explicitly labelled as assumption with source_url to Drinkwaterbesluit
# - Threshold shown as adjustable slider (range: 150–250 mg/L; citation: RIVM Table 2.3)
```

> **Domain guardrail (GAP 7):** Threshold is read from `productieketen` per intake. If not found, fallback is 150 mg/L (Dutch norm, more conservative than EU's 250 mg/L) with explicit labelling. Slider range 150–250 mg/L; both endpoints are sourced.

### 7.5 Waterinfo chloride guard (mandatory — GAP 9)
```python
async def fetch_waterinfo_chloride(intake_id: str) -> dict:
    """
    MANDATORY for IJssel, Lek, Maas intake scenarios.
    If call fails: return last_known_value + last_known_date from cache.
    NEVER silently fall back to synthetic value without explicit warning in ScenarioCard.
    """
    try:
        response = await httpx.get(WATERINFO_ENDPOINT, params={"locatie": intake_id})
        return {"value": response.json()["waarde"], "source": "Waterinfo live", "date": today}
    except Exception:
        cached = load_from_cache(intake_id)
        return {
            "value": cached["value"],
            "source": "Waterinfo (cached)",
            "date": cached["date"],
            "warning": f"Live Waterinfo niet beschikbaar. Laatste bekende waarde van {cached['date']}."
        }
```

### 7.6 Type-3 development demand reference table (ranges, not constants)

| Development type | Demand formula | Range | Unit | Assumption | Source |
|---|---|---|---|---|---|
| Datacenter | MW_IT × multiplier | 5–20 m³/day/MW | m³/day | Cooling tech dependent | IEA Water/Energy Nexus; adjustable slider |
| Residential | units × demand_per_unit | 0.28–0.42 m³/day/unit | m³/day | Household size 2.0–2.6 | VEWIN Jaarstatistiek; adjustable slider |
| Office | m² × 0.003 | 0.002–0.005 | m³/day/m² | Occupancy rate | RVO referentiewaarden |
| Industrial | site-specific | context-dependent | m³/day | Sector type | CBS sector water use |
| Agriculture | ha × crop_type | crop-specific | m³/day/ha | Irrigation regime | LGN + WUR data |

> **Key change from v1:** datacenter default is mid-range (12 m³/day/MW), not a single constant. Slider range 5–20 is shown prominently. Label: "Dit is een beleidsmatige schatting — werkelijk verbruik hangt af van koeltechniek."

> **Demand per dwelling:** default 0.35 m³/day/unit shown as adjustable slider (range 0.28–0.42). Label references VEWIN household statistics. Housing units blocked = supply_gap_m3 / demand_per_unit_slider_value — slider-dependent, not hardcoded.

### 7.7 Intervention catalogue (with all source URLs filled — v2 fix)

| Intervention | Supply delta m³/day | Cost indication | Source URL |
|---|---|---|---|
| Bufferopslag 30,000 m³ | +30,000 (1-day buffer) | €15–25M | Regionaal Waterprogramma ZH 2022–2027 |
| Alternatieve inname (Lek) | +50,000–80,000 | €40–100M | Regionaal Waterprogramma ZH 2022–2027 |
| Vraagbeperking 10% | -10% peak demand | regulatory | Drinkwaterwet art. 10 |
| Interconnectie buurregio | +20,000–40,000 | €20–60M | VEWIN Infrastructuurrapport 2023 |

> **v2 fix:** all `source_url` fields are populated. No empty strings. CI gate will block deployment if any Assumption or Intervention has empty source_url.

### 7.8 "Make it feasible" intervention ranking (new in v2 — GAP 5)
When `feasibility_class == "STOP"` or `"CAUTION"`, the engine:
1. Iterates through available interventions
2. Re-runs demand/capacity calculation with each intervention applied
3. Ranks by: (gap_closure_pct DESC, cost ASC)
4. Returns top 3 as "Wat maakt dit haalbaar?" panel with FeasibilityClass after each intervention

---

## 8. Trust primitives & institutional UX (all new in v2)

### 8.1 Kennisbasis panel (GAP 1)
Permanent panel accessible before the first question. Shows:

```
┌─────────────────────────────────────────────────────────────────┐
│ 📚 Kennisbasis — Wat weet dit systeem?                          │
├─────────────────────────────────────────────────────────────────┤
│ Thema              │ Tabellen        │ Bijgewerkt    │ Bron      │
│ Drinkwaterzekerheid│ 8 tabellen      │ 3 dagen geleden│ PDOK/DSO │
│ Gebiedsviewer      │ 12 lagen        │ 1 week geleden │ PDOK     │
│ LGN Landgebruik    │ 1 grid          │ 2023           │ WUR/PDOK │
│ CBS Buurtstatistiek│ 1 grid          │ 2023           │ CBS      │
├─────────────────────────────────────────────────────────────────┤
│ 🔍 Vraag de kennisbasis: "Welke data heeft het systeem over      │
│    woningbouw in Pijnacker?"                                     │
└─────────────────────────────────────────────────────────────────┘
```

Also queryable: "Welke data heeft dit systeem over X?" routes to `check_intent` with `intent_type: "knowledge_base_query"`.

### 8.2 Scenario citation block (GAP 3)
Every ScenarioCard footer includes:

```
┌─────────────────────────────────────────────────────────────────┐
│ 📎 Citaat voor gebruik in adviesdocumenten                      │
├─────────────────────────────────────────────────────────────────┤
│ Scenario-ID:    sc_019e8c_2026-06-04_Hd40_IJssel                │
│ Gegenereerd:    4 juni 2026, 14:32 CEST                         │
│ Dataversies:    drinkwater_v2.3, gebiedsviewer_v1.8             │
│ Software:       onegov2-spatial-assistant @ git:a3f9b2d         │
│ Stabiele URL:   https://[host]/scenario/019e8c...               │
│                                                                  │
│ APA-stijl (NL): Provincie Zuid-Holland Ruimtelijke Assistent    │
│ (2026, 4 juni). Scenario: Hollandse IJssel inname uitval 6      │
│ weken, KNMI Hd, 2040 [sc_019e8c]. GovTechNL OneGov #2.         │
│ [URL]                                                            │
├─────────────────────────────────────────────────────────────────┤
│ [📋 Kopieer citaat]  [📄 Download PDF]  [🔗 Deel URL]           │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 Shared scenario library + stable URLs (GAP 2)
- Every scenario run produces a `scenario_id` (UUID) and `scenario_hash`
- `scenario_hash` = SHA-256 of normalized params → same question = same hash = same cached result
- Scenarios are stored in DuckDB `scenarios` table (id, hash, params_json, result_json, created_at, dataset_versions)
- URL `/scenario/{scenario_id}` always returns the same artifact
- Shareable within organisation: "Stuur collega dit scenario"
- Version-drift warning: if any dataset was updated since the scenario was cached, banner shows: "Let op: [tabel] is bijgewerkt sinds dit scenario is berekend. Herbereken voor actuele uitkomst."

### 8.4 Official position panel (GAP 8)
Shown below every ScenarioCard:

```
┌─────────────────────────────────────────────────────────────────┐
│ 🏛️ Officieel beleid — Wat zegt de overheid hierover?            │
├─────────────────────────────────────────────────────────────────┤
│ De beschikbaarheid van voldoende schoon drinkwater staat         │
│ onder druk door groei, klimaat en bronkwaliteit.                 │
│ [Regionaal Waterprogramma ZH 2022–2027 →]                        │
│ [Nationaal Waterprogramma 2022–2027 →]                           │
│ [Ruimtelijk Arrangement Rijk–ZH, 2 juni 2025 →]                  │
└─────────────────────────────────────────────────────────────────┘
```

Pre-baked for: drinkwater (ZH), klimaat (KNMI), woningbouw (ZH), KRW (Rijkswaterstaat).

### 8.5 "Verify calculation" button (GAP 2 / water authority request)
On every cached ScenarioCard: a "Herbereken met actuele data" button that:
1. Re-runs same params with current dataset versions
2. Shows a diff: which values changed and by how much
3. Logs both runs to MLflow for traceability

---

## 9. Map visualization

### 9.1 Always-visible map title (GAP — policymaker)
```
┌──────────────────────────────────────────────────────────────┐
│ 🗺️ Huidige kaart: Aanvoertekort in m³/dag per leveringszone  │
│    KNMI scenario Hd · Tijdshorizon 2040 · Inname: IJssel     │
└──────────────────────────────────────────────────────────────┘
```
Title updates reactively when scenario parameters change. Never blank.

### 9.2 Base layers (always on)
- Drinkwaterleverings­zones
- Innamepunten (productieketen)
- Waterschapsgrenzen
- Gemeentegrenzen ZH
- Zes-uur-zones (intake protection zones)

### 9.3 Scenario overlays (generated per run as GeoJSON)
| Scenario type | Overlays generated |
|---|---|
| Type 1 (intake failure) | Chloride risk zones; supply gap heatmap; affected zones per KNMI preset |
| Type 3 (drop-pin) | Impact radius from pin; capacity headroom per zone; GO/CAUTION/STOP zone shading |
| Comparison | Delta layer (positive = improved; negative = worsened) |

### 9.4 Stakeholder dropdown → map emphasis
| Stakeholder | Map emphasizes |
|---|---|
| Woningzoekenden | Woningbouwlocaties overlapping STOP/CAUTION zones |
| Drinkwaterbedrijf | Production chain as directed flow graph with capacity thickness |
| Natuur/KRW | KRW-risicogebieden; beschermde waterlichamen |
| Landbouw | Irrigatieafhankelijke percelen in risicozone |
| Zorg/crisissector | Must-serve zones; alternatieve routes |

### 9.5 Production chain as directed flow graph (new in v2 — GAP water authority)
For intake-failure scenarios, an additional panel (not just dots on map) shows:

```
[Inname IJssel] ──(180,000 m³/d)──► [Productie Westveen] ──(90,000 m³/d)──► [Zone A]
                                                           ──(90,000 m³/d)──► [Zone B]
                STATUS: 🔴 Inname gestopt (KNMI Hd week 4)
```

Thickness = capacity; color = operational status.

### 9.6 Split-map comparison view
| Left panel | Right panel |
|---|---|
| Scenario A | Scenario B |
| Headline: gap, onset year, KRW count | Headline: gap delta (±), onset year delta (±), KRW delta |
| Same base layers + A-specific overlays | Same base layers + B-specific overlays |
| Shared pan/zoom | Delta layer togglable |

---

## 10. FeasibilityClass — GO / CAUTION / STOP (new in v2 — GAP 3)

Shown prominently at top of every ScenarioCard:

```
┌─────────────────────────────────────────────────────────────┐
│  🟢 HAALBAAR                                                 │
│  Haalbaar onder alle KNMI-scenario's in 2040.                │
│  Resterende capaciteit: 12,400 m³/dag (≈ 35,400 huishoudens)│
└─────────────────────────────────────────────────────────────┘
```

or:

```
┌─────────────────────────────────────────────────────────────┐
│  🔴 NIET HAALBAAR                                            │
│  Tekort onder 3 van 5 KNMI-scenario's incl. basisscenario.  │
│  Tekort: 23,000 m³/dag (≈ 65,700 huishoudens)               │
│  ▼ Wat maakt dit wél haalbaar? [3 opties beschikbaar]        │
└─────────────────────────────────────────────────────────────┘
```

---

## 11. Human-scale contextualization (new in v2 — GAP 4)

Every headline metric in the ScenarioCard gets a `HumanScaleRef`:

| Raw metric | Human-scale analogy | Conversion source |
|---|---|---|
| X m³/dag demand | "Dagelijks verbruik van ~Y mensen" | VEWIN: ~119 L/persoon/dag |
| X m³/dag gap | "Equivalent aan Y huishoudens zonder water" | VEWIN household stats |
| X MW datacenter demand | "Vergelijkbaar met een wijk van Y huishoudens" | IEA/VEWIN crossref |
| X woningen geblokkeerd | "Woningen voor ~Y huishoudens" | CBS gem. huishoudensgrootte |

Label always appended: *(beleidsmatige schatting — bron: [URL])*

---

## 12. Citizen mode (new in v2 — GAP 10)

### 12.1 Detection
Route to citizen mode if:
- User selects "Burger" from persona dropdown, OR
- Question contains postcode + personal pronoun ("mijn", "ons", "ik"), OR
- Question is "Is mijn water veilig?" / "Wat betekent dit voor mij?"

### 12.2 Citizen response template
```
┌─────────────────────────────────────────────────────────────┐
│ 💧 Drinkwater in jouw buurt (postcode: 2641)                 │
├─────────────────────────────────────────────────────────────┤
│ VERDICT: Op dit moment is er geen direct tekort in           │
│ jouw leveringszone.                                          │
│                                                              │
│ In het droogste klimaatscenario (2040) is er een risico      │
│ op tijdelijke druk op de watervoorziening in jouw regio.     │
│                                                              │
│ Meer weten?                                                  │
│ 📞 Jouw drinkwaterbedrijf: Dunea — dunea.nl                  │
│ 🌡️ KNMI klimaatscenario's — knmi.nl/klimaatscenarios         │
│ 🏛️ Provinciaal Waterprogramma — pzh.nl                       │
├─────────────────────────────────────────────────────────────┤
│ ⚠️ Dit is een exploratief scenario, geen officiële meting.   │
└─────────────────────────────────────────────────────────────┘
```

Rules:
- Verdict in first sentence, no leading numbers
- Minimal figures unless user requests detail ("Vertel me meer")
- Always link to responsible water company + KNMI + official policy
- Always include disclaimer

---

## 13. Traceability framework (unchanged architecture, v2 additions)

### 13.1 Three-layer traceability

| Layer | Tool | What's logged | Who reads it |
|---|---|---|---|
| Human | Insight panel (UI) | Step-by-step B1 Dutch reasoning | Policymakers, auditors |
| Machine | MLflow | Params, metrics, artifacts (JSON), Flesch-Douma, dataset versions | Developers, Woo requests |
| Citeable | ScenarioCard | scenario_id, hash, stable URL, citation string, git commit | Legal/administrative use |

### 13.2 MLflow artifacts per run
```
mlflow run: {scenario_id}
├── params/
│   ├── scenario_type
│   ├── knmi_preset
│   ├── time_horizon
│   ├── all assumptions (key=value)
│   └── dataset_versions (dict)
├── metrics/
│   ├── supply_gap_m3
│   ├── feasibility_class (encoded: GO=2, CAUTION=1, STOP=0)
│   ├── confidence
│   ├── readability_score (Flesch-Douma)
│   └── latency_ms per node
└── artifacts/
    ├── scenario_card.json
    ├── source_registry.json
    ├── stakeholder_impacts.json
    ├── reasoning_steps.json
    └── overlays/ (GeoJSON files)
```

### 13.3 Woo-readiness note
The combination of `stable_url + scenario_id + dataset_versions + git_commit + reasoning_steps` provides the minimum artifact set needed to reconstruct a scenario run for a Wet open overheid disclosure request. The tool does not guarantee full Woo compliance, but it is designed to support it.

---

## 14. GreenPT integration (unchanged, reinforced)

| Use | Temperature | Purpose | Guard |
|---|---|---|---|
| Param extraction | 0.1 | Structured ScenarioParams from NL text | Confidence threshold < 0.7 → followup |
| Confirmation message | 0.2 | B1 Dutch recap of extracted params | User must confirm before run |
| Reasoning steps | 0.2 | Step descriptions in B1 Dutch | Flesch-Douma logged; re-generate if < 50 |
| Stakeholder prose | 0.2 | 1–2 sentence impact per stakeholder | Must cite policy_basis in prompt |
| Citizen response | 0.15 | Verdict-first, postcode-level | Uses citizen template structure; no invented numbers |

**Hard rule:** GreenPT may never generate numbers in calculation steps. All numeric outputs come from deterministic Python functions. GreenPT only generates prose descriptions of those numbers.

---

## 15. Vue component map (what to build in the frontend)

### New components (v2)
| Component | Purpose | Closes gap |
|---|---|---|
| `KennisbasisPanel.vue` | Knowledge base explorer | GAP 1 |
| `FeasibilityBadge.vue` | GO/CAUTION/STOP traffic light | GAP 3 |
| `HumanScaleRef.vue` | Inline analogy for every metric | GAP 4 |
| `MakeItFeasiblePanel.vue` | Ranked intervention suggestions | GAP 5 |
| `CumulativeLoadWarning.vue` | Active scenarios in same zone | GAP 6 |
| `OfficialPositionPanel.vue` | Official policy context | GAP 8 |
| `CitationBlock.vue` | Full citation + stable URL + PDF | GAP 3 |
| `CitizenResponseCard.vue` | Verdict-first citizen format | GAP 10 |
| `ScenarioDeltaPanel.vue` | Split comparison + delta headline | Original design |
| `AssumptionSliders.vue` | Adjustable sliders with ranges | Original design |
| `ProductionChainFlow.vue` | Directed graph of intake → production → zone | Water authority gap |
| `MapTitle.vue` | Always-visible "what you're seeing" | Policymaker gap |
| `WaterinfoBanner.vue` | Live vs cached chloride warning | GAP 9 |

### Retained from existing repo (no changes needed)
- `InsightPanel.vue` (reasoning trail)
- `MapView.vue` (extended with new overlays + title)
- `ChatInput.vue`
- All existing descriptive-query components

---

## 16. Pre-hackathon checklist (do now — today is June 4)

### Unblocks everything (do in first 2 hours)
- [ ] Inspect DuckDB schema: confirm `productieketen` has intake-level chloride threshold field; if not, plan fallback Assumption
- [ ] Write all dataclasses (`ScenarioParams`, `Assumption`, `ScenarioResults`, `ScenarioCard`, `HumanScaleRef`, `OfficialPosition`) into `src/backend/models/scenario.py`
- [ ] Lock Golden Path A parameters (Datacenter 50MW, Pijnacker-Nootdorp, KNMI Hd, 2040)
- [ ] Lock Golden Path B parameters (IJssel inname, 6 weeks outage, KNMI Hd, 2040)
- [ ] Pre-cache KNMI'23 scenario tables locally (venue internet risk)
- [ ] Pre-cache Waterinfo last-known chloride values for IJssel, Lek (fallback guard)
- [ ] Confirm GreenPT API key works and rate limits are sufficient

### Day 1 milestones (June 4)
| Time | Milestone | Owner |
|---|---|---|
| 10:00 | Schema verified; dataclasses merged | Backend |
| 12:00 | `extract_scenario_params` node working for Golden Path A | Backend |
| 13:00 | `run_scenario_calculation` returning results for A (ranges, not constants) | Backend |
| 14:00 | `FeasibilityClass` computed; `HumanScaleRef` attached | Backend |
| 15:00 | `ScenarioCard` streaming to frontend | Full stack |
| 16:00 | `FeasibilityBadge` + `AssumptionSliders` in UI | Frontend |
| 17:00 | Golden Path A end-to-end demo-able | All |
| 18:00 | Golden Path B (intake failure) working | Backend |
| 19:00 | `KennisbasisPanel` + `CitationBlock` + `OfficialPositionPanel` | Frontend |
| 20:00 | Day 1 freeze: both golden paths working; trust kit visible | All |

### Day 2 milestones (June 5)
| Time | Milestone | Owner |
|---|---|---|
| 09:00 | Comparison mode (scenario_b) + `ScenarioDeltaPanel` | Full stack |
| 10:00 | `MakeItFeasiblePanel` (intervention ranking) | Backend |
| 11:00 | `CumulativeLoadWarning` + scenario cache + stable URL | Backend |
| 12:00 | `CitizenResponseCard` + citizen mode detection | Frontend |
| 13:00 | `ProductionChainFlow` panel | Frontend |
| 14:00 | MLflow logging complete; readability score metric | Backend |
| 15:00 | Demo rehearsal: 3-minute script, 3 wow moments | All |
| 16:00 | README + open source release checklist | All |
| 17:00 | **Final submission** | All |

---

## 17. Three-minute demo script

### Act 1 — Drop a pin (Type 3, ~60s)
> "Stel dat er een 50 MW datacenter komt bij Pijnacker-Nootdorp in 2040. Is er genoeg drinkwater?"

1. Type question → system confirms extracted params (confirmation card)
2. Show **🔴 NIET HAALBAAR** FeasibilityClass badge immediately
3. Show gap in m³/day → HumanScaleRef: "equivalent aan X huishoudens"
4. Scroll down: "Wat maakt dit wél haalbaar?" → top 3 interventions shown
5. Point: "Every number has a source link. No black box."

### Act 2 — Intake failure + assumptions (Type 1, ~60s)
> "Wat als de Hollandse IJssel 6 weken onbruikbaar is door verzilting, KNMI Hd?"

1. New scenario → **🟠 RISICO** FeasibilityClass
2. Show Waterinfo chloride panel (live or cached with warning)
3. Move chloride threshold slider from 150 → 200 → watch FeasibilityClass update
4. Point: "Slider is policy-level approximation, sourced to Drinkwaterbesluit. Judges can challenge the number, and we can show where it comes from."

### Act 3 — Comparison + institutional trust (~60s)
> "Vergelijk: met en zonder bufferopslag van 30.000 m³"

1. Split map: left = no intervention (🔴), right = with buffer (🟡)
2. Delta panel: "Gap reduced by X m³/day; onset year pushed from 2036 → 2040"
3. Click **Citation block**: show Scenario ID, stable URL, dataset versions, APA citation
4. Click **Official position**: show Regionaal Waterprogramma ZH link
5. Click **Kennisbasis**: show which datasets are loaded and how fresh
6. Final line: "A policymaker can attach this output to an advice letter and defend every number in it."

---

## 18. Validation & acceptance criteria (pre-submission)

### Must (brief requirement)
- [ ] Working prototype translating NL policy question into computed scenario
- [ ] ≥ 2 scenario types demonstrated
- [ ] Data + assumptions + impacted parties explicit
- [ ] Open source, readable README
- [ ] Uses provided datasets (no external APIs for core calculation)

### Should (design doc requirement)
- [ ] Scenario comparison mode working (split map + delta)
- [ ] Assumption sliders adjustable and re-runnable
- [ ] Reasoning chain traceable (Insight panel + MLflow)
- [ ] All `source_url` fields populated (CI gate)
- [ ] Human-scale analogies on all headline metrics

### Stakeholder gaps (v2 requirement)
- [ ] GAP 1: Kennisbasis panel present and queryable
- [ ] GAP 2: Scenario hash + stable URL + dataset version logging
- [ ] GAP 3: Citation block with APA string + PDF download
- [ ] GAP 4: HumanScaleRef on every headline metric
- [ ] GAP 5: "Make it feasible" intervention ranking on STOP/CAUTION
- [ ] GAP 6: Cumulative load warning for same-zone scenarios
- [ ] GAP 7: Per-intake chloride threshold (not global constant)
- [ ] GAP 8: Official position panel with policy document links
- [ ] GAP 9: Waterinfo guard (mandatory for IJssel/Lek/Maas; explicit warning if cached)
- [ ] GAP 10: Citizen response template (verdict-first, postcode, water company link)

### Domain correctness (water authority)
- [ ] Chloride threshold is per-intake with source; fallback is labelled
- [ ] Waterinfo call attempted for intake scenarios; fallback is never silent
- [ ] Production chain shows directionality and capacity
- [ ] All demand figures shown as ranges, labelled as policy approximations

### Reproducibility
- [ ] Same params → same scenario_hash → same cached result
- [ ] Dataset version drift warning on cached scenarios
- [ ] "Verify calculation" button present
- [ ] Deterministic random seed (numpy seed=42) for any stochastic elements

---

## 19. What this design delivers (the pitch in one paragraph)

This tool extends the existing OneGov2 spatial assistant from "describe what is" to "explore what could be." A policymaker can ask a what-if question in plain Dutch, get a traffic-light feasibility verdict in seconds, see the spatial impact on a map, compare scenarios side-by-side, adjust assumptions with sliders, understand who is affected and why, and — critically — attach the output to an advice letter with a citation block that links every number to its authoritative source. The same scenario run, with the same parameters, produces the same result for every colleague, every time, with a version-drift warning if the underlying data changes. That is the difference between a demo and a policy instrument.

---

*Document version: 2.0 | Date: 4 June 2026 | Hackathon: GovTechNL OneGov #2*
*Supersedes: onegov2designdocument.md v1.0 + stakeholdercritique.md*
*License: MIT (in line with challenge open-source requirement)*
