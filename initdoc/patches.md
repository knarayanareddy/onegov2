Below is a **ready-to-paste Markdown patch** (copy/paste blocks) that integrates **Kadaster’s Generieke Geo Componenten (GGC)** as a **government UI/UX reference baseline** and adds the missing **GGC-equivalent viewer primitives** (layer tree, legend, toolbar, feature info, app shell accessibility) into your existing design doc **at the correct spots**, using the **actual headings/numbering** found in `onegov2designdocumentcompleteversion.md` (notably: `2.2`, `4.1`, `4.2`, `15.4`, `16.9`, `17.1`, `18.`). 

I’m also including **small “surgical edits”** (one-line replacements) to fix the “+ 13 new components (see section 18)” mismatch in your Section `4.1` architecture box, because you’re already using Section `16` for components and Section `18` for trust gaps. 

---

# PATCH A — Insert **Section 2.3 + 2.4** (GGC UX baseline + government app shell)
**Where to insert:** Immediately after your existing **`2.2 UI design principles`** line/block and **before** `3. Golden-path scenarios...` (you currently jump directly from 2.2 → 3). 

```md
2.3 Government map-viewer UX baseline (Kadaster GGC reference)

We use Kadaster’s open-source **Generieke Geo Componenten (GGC)** as a *UX reference baseline* for government-grade map viewers. GGC is described as developer-friendly building blocks to quickly build a map viewer, combining OpenLayers map power with standard viewer functions like: search, legend, choosing map display, and a toolbar for drawing/measuring/editing. GGC also emphasizes responsive design and WCAG 2.1 AA accessibility. We do **not** embed Angular/OpenLayers GGC code in this hackathon build; we replicate the interaction model and accessibility patterns in our Vue + Leaflet UI.

Reference:
- https://github.com/kadaster/generieke-geo-componenten

2.4 Government app shell & accessibility conventions (GGC Home-inspired)

To make the application feel and behave like a government map viewer, we adopt the following shell conventions:

- **Skip-to-content** link ("Naar hoofdinhoud") at the top of the page.
- Persistent **top navigation** with predictable locations for:
  - Home / Scenario
  - Scenario library (stable URLs)
  - Kennisbasis
  - Export (PDF/URL)
  - Help / Toegankelijkheid
- Persistent **footer** with:
  - "Toegankelijkheid" link (WCAG statement/contact)
  - GitHub / feedback link

Accessibility acceptance rules (minimum):
- All interactive controls reachable via keyboard (Tab + Enter).
- Visible focus ring for all controls (no focus hidden).
- Legend always explains colors AND includes text labels (never color-only).
- Layer toggles are keyboard-friendly and ARIA-labeled.

These conventions are implemented via Vue layout + components (see sections 15.5–15.7 and 16.10–16.13).
```

---

# PATCH B — Fix Section **4.1** “+ 13 new components (see section 18)” (one-line replace)
**Where:** In `4.1 Full stack overview` ASCII diagram, replace the single line:

> `+ 13 new components (see section 18)`

with:

```md
+ UI components (see section 16)
```

This prevents a mismatch (your component specs live in Section **16**, and Section **18** is “Trust primitives & institutional UX”). 

---

# PATCH C — Update the **4.2 Full directory tree** (add stores/components/composables)
**Where to edit:** Inside the `4.2 Full directory tree (file-level implementation map)` block.

### C1) Add new stores under `src/frontend/src/stores/`
Find the existing list:

```md
│ │ ├── stores/
│ │ │ ├── useScenarioStore.ts ...
│ │ │ ├── useMapStore.ts ...
│ │ │ ├── useAssumptionStore.ts ...
│ │ │ └── useKennisbasisStore.ts ...
```

Add these two lines (keep the same indentation style):

```md
│ │ │ ├── useLayerStore.ts # NEW: layer tree (dataset-tree) + active overlays + stakeholder layer presets (GGC-equivalent)
│ │ │ └── useLegendStore.ts # NEW: legend items derived from active layers + LAYER_CONFIG (GGC-equivalent)
```

### C2) Add new components under `src/frontend/src/components/`
Find the existing list ending around `WaterinfoBanner.vue` / `SourceRegistryPanel.vue`. Add:

```md
│ │ │ ├── LayerTreePanel.vue # NEW: dataset-tree / layer control (GGC-equivalent)
│ │ │ ├── LegendPanel.vue # NEW: legend (GGC-equivalent)
│ │ │ ├── MapToolbar.vue # NEW: map toolbar (reset/measure/draw) (GGC-equivalent)
│ │ │ └── FeatureInfoPanel.vue # NEW: click feature → attribute panel (GGC object-info direction)
```

### C3) Add new composables under `src/frontend/src/composables/`
Add:

```md
│ │ ├── useLayerLegendBinding.ts # NEW: binds active layers → legend → map overlays
│ │ └── useFeatureInfo.ts # NEW: map click handling → FeatureInfoPanel
```

---

# PATCH D — Insert **15.5 + 15.6 + 15.7** (Layer/Legend/FeatureInfo stores)
**Where to insert:** Immediately after your existing **`15.4 useKennisbasisStore.ts`** section and **before** `16. Vue component specifications`. 

```md
15.5 useLayerStore.ts (GGC dataset-tree equivalent: active layers + stakeholder presets)

TypeScript

// src/frontend/src/stores/useLayerStore.ts
import { defineStore } from 'pinia'
import { STAKEHOLDER_OVERLAY_MAP } from '@/composables/useMapOverlays'

const DEFAULT_BASE_LAYERS = [
  'gemeentegrenzen',
  'waterschapsgrenzen',
  'leveringszones',
  'winlocaties',
  'zes_uur_zones'
]

export type StakeholderKey =
  | 'woningzoekenden'
  | 'drinkwaterbedrijf'
  | 'natuur_krw'
  | 'landbouw'
  | 'zorg_kritisch'
  | 'gemeente'

export const useLayerStore = defineStore('layers', {
  state: () => ({
    // Base layers always start on
    activeLayerIds: new Set<string>(DEFAULT_BASE_LAYERS),

    // Scenario overlays start off; set to on when scenario run returns overlays
    // (user can toggle afterwards)
    hasScenarioOverlays: false,

    // Stakeholder mode drives a recommended overlay preset, but user can override
    activeStakeholder: 'gemeente' as StakeholderKey,
  }),

  actions: {
    setHasScenarioOverlays(has: boolean) {
      this.hasScenarioOverlays = has
      if (has) {
        // Default: turn on supply_gap_heatmap when scenario overlays exist
        this.activeLayerIds.add('supply_gap_heatmap')
      }
    },

    toggleLayer(id: string) {
      if (this.activeLayerIds.has(id)) this.activeLayerIds.delete(id)
      else this.activeLayerIds.add(id)
    },

    setStakeholder(key: StakeholderKey) {
      this.activeStakeholder = key

      // Apply recommended preset (does NOT force user; they can toggle after)
      const recommended = STAKEHOLDER_OVERLAY_MAP[key] ?? []
      for (const layerId of recommended) this.activeLayerIds.add(layerId)
    }
  },

  getters: {
    isActive: (state) => (id: string) => state.activeLayerIds.has(id),
    activeLayersArray: (state) => Array.from(state.activeLayerIds.values()),
  }
})


15.6 useLegendStore.ts (GGC legend equivalent: legend items from active layers)

TypeScript

// src/frontend/src/stores/useLegendStore.ts
import { defineStore } from 'pinia'
import { LAYER_CONFIG } from '@/composables/useMapOverlays'
import { useLayerStore } from '@/stores/useLayerStore'

type LegendItem = {
  layerId: string
  label: string
  swatches: Array<{ color: string; meaning: string }>
}

export const useLegendStore = defineStore('legend', {
  state: () => ({
    legendItems: [] as LegendItem[]
  }),

  actions: {
    recomputeLegend() {
      const layerStore = useLayerStore()
      const items: LegendItem[] = []

      for (const layerId of layerStore.activeLayersArray) {
        const cfg: any = (LAYER_CONFIG as any)[layerId]
        if (!cfg) continue

        // Only show legend for layers that define a colorScale
        if (cfg.colorScale) {
          const swatches = Object.entries(cfg.colorScale).map(([k, v]) => ({
            color: String(v),
            meaning: String(k)
          }))
          items.push({
            layerId,
            label: layerId,
            swatches
          })
        }
      }

      this.legendItems = items
    }
  }
})


15.7 useFeatureInfoStore.ts (GGC object-info direction: clicked feature → panel)

TypeScript

// src/frontend/src/stores/useFeatureInfoStore.ts
import { defineStore } from 'pinia'

export type FeatureInfo = {
  layerId: string
  title: string
  properties: Record<string, any>
} | null

export const useFeatureInfoStore = defineStore('featureInfo', {
  state: () => ({
    selected: null as FeatureInfo
  }),
  actions: {
    setSelected(f: FeatureInfo) {
      this.selected = f
    },
    clear() {
      this.selected = null
    }
  }
})
```

---

# PATCH E — Insert **16.10–16.13** (GGC-equivalent components)
**Where to insert:** Immediately after your existing **`16.9 WaterinfoBanner.vue`** and before `17. Map layer configuration`. 

```md
16.10 LayerTreePanel.vue (GGC dataset-tree equivalent)

vue

<script setup lang="ts">
defineProps<{
  availableLayers: Array<{ id: string; label: string; group: string; defaultOn: boolean }>
  activeLayerIds: string[]
}>()

defineEmits<{
  (e: 'toggle-layer', id: string): void
  (e: 'set-stakeholder', key: string): void
}>()
</script>


16.11 LegendPanel.vue (GGC legend equivalent)

vue

<script setup lang="ts">
defineProps<{
  legendItems: Array<{
    layerId: string
    label: string
    swatches: Array<{ color: string; meaning: string }>
  }>
}>()
</script>


16.12 MapToolbar.vue (GGC toolbar equivalent)

vue

<script setup lang="ts">
defineProps<{
  enabledTools: Array<'reset_view' | 'measure_distance' | 'draw_aoi'>
}>()

defineEmits<{
  (e: 'tool-selected', toolId: 'reset_view' | 'measure_distance' | 'draw_aoi'): void
}>()
</script>


16.13 FeatureInfoPanel.vue (GGC object-info direction)

vue

<script setup lang="ts">
defineProps<{
  selectedFeature: null | {
    layerId: string
    title: string
    properties: Record<string, any>
  }
}>()

defineEmits<{
  (e: 'close'): void
}>()
</script>
```

---

# PATCH F — Extend **17. Map layer configuration** with layer groups + legend binding + non-color cues
**Where to insert:** In Section `17. Map layer configuration`, after your existing `LAYER_CONFIG` / `STAKEHOLDER_OVERLAY_MAP` definitions.

```md
17.2 Layer groups (for LayerTreePanel.vue) and legend binding (for LegendPanel.vue)

TypeScript

// src/frontend/src/composables/useMapOverlays.ts (additions)
export const LAYER_GROUPS: Array<{ groupId: string; label: string; layerIds: string[] }> = [
  { groupId: 'admin', label: 'Bestuurlijke grenzen', layerIds: ['gemeentegrenzen', 'waterschapsgrenzen'] },
  { groupId: 'drinkwater_basis', label: 'Drinkwater basislagen', layerIds: ['leveringszones', 'winlocaties', 'zes_uur_zones'] },
  { groupId: 'scenario', label: 'Scenario overlays', layerIds: ['supply_gap_heatmap', 'chloride_risk_zones', 'drop_pin_marker', 'krw_risk_overlay'] },
  { groupId: 'comparison', label: 'Vergelijking', layerIds: ['delta_layer'] },
]

// Legend binding rule:
// - LegendPanel shows items for all ACTIVE layers that define LAYER_CONFIG[layerId].colorScale
// - Legend recompute is triggered on:
//   - layer toggle
//   - stakeholder selection
//   - scenario_card/map_data arrival


17.3 Accessibility: non-color cues for risk classes

Rule:
- Any layer that uses a color scale MUST have:
  - text labels in the legend (low/medium/high, GO/CAUTION/STOP)
  - optional pattern/dash styles for borders to support color-vision deficiency
- FeasibilityBadge and ScenarioCard must repeat the verdict in text (not only color).
```

---

# PATCH G — (Optional but recommended) Add a small note in Section 18 (trust primitives)
Your Section **18** is “Trust primitives & institutional UX (all 10 stakeholder gaps)”. You don’t need an 11th gap, but adding a *note* helps judges connect the dots:

**Where:** At the very start of Section `18.` (just after the title line).

```md
Note: In addition to the 10 stakeholder gaps below, we adopt Kadaster GGC as a reference UX baseline for government map viewers (Section 2.3–2.4) and implement GGC-equivalent primitives in Vue: dataset-tree (LayerTreePanel), legend (LegendPanel), toolbar (MapToolbar), and feature info (FeatureInfoPanel).
```

---

## Why this patch is “correct” for GGC
GGC’s README explicitly frames the library as Angular components to build map viewers fast, combining OpenLayers with standard viewer functions: search, legend, map display selection, and a toolbar for drawing/measuring/editing, and it emphasizes WCAG 2.1 AA accessibility. 

---

If you want, I can also generate a **second patch** that updates your **frontend wiring narrative** to explicitly show how `useLayerStore` filters what `useMapStore.setOverlays()` renders (so “map_data arrives” ≠ “everything is shown”). That makes the design doc feel even more “government viewer” and less “hackathon overlay dump.”





Below is a **second ready-to-paste Markdown patch** that updates your **frontend wiring narrative** (and adds concrete store/composable code) so it’s explicit that:

> **`map_data` arrives → mapStore stores raw overlays → layerStore decides what’s visible → map renders only visible overlays**  
(i.e., **not** “dump everything onto the map”).

This patch assumes the numbering from the first patch you applied (where you added **15.5–15.7** and **16.10–16.13**). It introduces **15.8–15.10**, and small edits to your existing `useScenarioSSE` / `useMapStore` sections.

---

# PATCH 2A — Insert **Section 15.8** (the narrative + event flow)
**Where to insert:** Immediately after your new `15.7 useFeatureInfoStore.ts` section and **before** `16. Vue component specifications`.

```md
15.8 Frontend overlay rendering pipeline (raw overlays ≠ visible overlays)

Goal:
- When the backend emits `map_data`, the frontend must NOT automatically render every overlay.
- Instead, we store ALL overlays as **raw overlays** and render only those that are currently **active** in the UI layer tree.
- This matches government map viewer behavior (GGC-style): user always controls which layers are visible, and the legend matches visible layers.

Event/data flow (SSE → store → map):

1) Backend emits SSE event `map_data`:
   - payload contains `{ overlays: GeoJSONOverlay[] }`
   - overlays may include: scenario overlays, stakeholder overlays, delta overlays, debug overlays

2) `useScenarioSSE.ts` receives `map_data` and calls:
   - `mapStore.setRawOverlays(payload.overlays)`
   - `layerStore.setHasScenarioOverlays(true)` (enables recommended scenario layers)
   - `mapStore.recomputeVisibleOverlays(layerStore.activeLayerIds)`
   - `legendStore.recomputeLegend()` (legend matches currently active layers)

3) Map component (`MapView.vue`) renders ONLY:
   - `mapStore.visibleOverlays` (filtered subset)
   - base layers always available
   - no overlay renders unless its `overlay.layer_id` is active in `useLayerStore.activeLayerIds`

Key guarantee:
- “map_data arrived” does NOT imply “everything is shown”.
- User must always be able to answer: “Welke lagen staan aan, en waarom?”
```

---

# PATCH 2B — Add/replace **useMapStore** overlay state with raw vs visible
**Where:** In your existing `useMapStore.ts` section (likely under Section 15.x; in your doc it’s usually `15.2` or `15.3`).  
You can either (1) paste this as a replacement for the overlay portion of your store, or (2) append these fields/methods to your existing store.

```md
15.9 useMapStore.ts update — store raw overlays, render only visible overlays

TypeScript

// src/frontend/src/stores/useMapStore.ts (additions / refactor)
import { defineStore } from 'pinia'

export type GeoJSONOverlay = {
  layer_id: string            // REQUIRED: used for toggling + legend binding
  label_nl: string
  geojson: any                // FeatureCollection
  style?: any                 // optional style metadata
  z_index?: number
  default_on?: boolean
  group?: string
}

export const useMapStore = defineStore('map', {
  state: () => ({
    // All overlays received from backend (unfiltered)
    rawOverlays: [] as GeoJSONOverlay[],

    // Overlays currently visible on map (filtered by useLayerStore.activeLayerIds)
    visibleOverlays: [] as GeoJSONOverlay[],

    // Optional: last map_data timestamp (for debugging / trust)
    lastMapDataAt: null as null | string
  }),

  actions: {
    setRawOverlays(overlays: GeoJSONOverlay[], receivedAtIso?: string) {
      this.rawOverlays = overlays || []
      this.lastMapDataAt = receivedAtIso || new Date().toISOString()
    },

    recomputeVisibleOverlays(activeLayerIds: Set<string>) {
      // Filter overlays strictly by layer_id membership
      const active = new Set(Array.from(activeLayerIds.values()))

      this.visibleOverlays = (this.rawOverlays || []).filter(o => {
        if (!o?.layer_id) return false
        return active.has(o.layer_id)
      })

      // Optional: stable ordering by z-index then label
      this.visibleOverlays.sort((a, b) => {
        const za = a.z_index ?? 0
        const zb = b.z_index ?? 0
        if (za !== zb) return za - zb
        return (a.label_nl || a.layer_id).localeCompare(b.label_nl || b.layer_id)
      })
    },

    clearOverlays() {
      this.rawOverlays = []
      this.visibleOverlays = []
      this.lastMapDataAt = null
    }
  },

  getters: {
    getRawOverlayById: (state) => (layerId: string) =>
      state.rawOverlays.find(o => o.layer_id === layerId) || null
  }
})
```

**Important note (backend contract):** every overlay emitted by backend must contain a stable `layer_id` string. Without that, layer toggling/legend binding cannot work consistently.

---

# PATCH 2C — Update `useScenarioSSE.ts` map_data handling to filter overlays through `useLayerStore`
**Where to insert/edit:** In the SSE switch-case for `map_data` inside `src/frontend/src/composables/useScenarioSSE.ts`.

Replace your current `map_data` handling with:

```md
15.10 SSE wiring update — map_data → raw overlays → visible overlays

TypeScript

// src/frontend/src/composables/useScenarioSSE.ts (map_data case)
import { useMapStore } from '@/stores/useMapStore'
import { useLayerStore } from '@/stores/useLayerStore'
import { useLegendStore } from '@/stores/useLegendStore'

...

case 'map_data': {
  const payload = JSON.parse(event.data)

  const mapStore = useMapStore()
  const layerStore = useLayerStore()
  const legendStore = useLegendStore()

  // 1) Store raw overlays
  mapStore.setRawOverlays(payload.overlays || [], payload.received_at)

  // 2) Mark that scenario overlays exist (enables recommended scenario layers)
  layerStore.setHasScenarioOverlays(true)

  // 3) Compute visible overlays from active layer set
  mapStore.recomputeVisibleOverlays(layerStore.activeLayerIds)

  // 4) Recompute legend to match visible/active layers
  legendStore.recomputeLegend()

  break
}
```

Also add recompute calls when the user toggles layers or switches stakeholder:

- When `LayerTreePanel` emits `toggle-layer`:
  - `layerStore.toggleLayer(layerId)`
  - `mapStore.recomputeVisibleOverlays(layerStore.activeLayerIds)`
  - `legendStore.recomputeLegend()`

- When user switches stakeholder:
  - `layerStore.setStakeholder(key)`
  - `mapStore.recomputeVisibleOverlays(layerStore.activeLayerIds)`
  - `legendStore.recomputeLegend()`

This ensures the map always reflects the visible layer set and the legend always matches.
```

---

# PATCH 2D — Small MapView rendering change (render visible overlays, not raw)
**Where to edit:** In your `MapView.vue` spec or implementation notes (often in Section 16 or a Map section).  
Add this explicit rule:

```md
MapView rendering rule:
- Render `mapStore.visibleOverlays` only.
- Do not render `mapStore.rawOverlays` directly.
- Base layers remain separate and are not part of overlay filtering.

Reason:
- Ensures “government viewer” behavior: user controls visibility via layer tree; map_data is not a dump.
```

If you want a concrete snippet to paste into your MapView section:

```md
vue

// MapView.vue (conceptual)
const mapStore = useMapStore()

// Render loop:
for (const overlay of mapStore.visibleOverlays) {
  renderGeoJsonLayer(overlay.geojson, overlay.style, overlay.z_index)
}
```

---

## Optional (but very clean): add a composable `useLayerLegendBinding.ts`
If you already added it in the first patch’s directory tree, you can specify it here as a convenience wrapper:

```md
Optional helper:
- `useLayerLegendBinding.ts` centralizes the repeated calls:
  - recomputeVisibleOverlays(activeLayerIds)
  - recomputeLegend()
- called after SSE map_data, layer toggles, and stakeholder switches
```

---

## Result (what judges/mentors will notice)
With this patch, you can confidently say in the demo:

- “The backend can send 15 overlays, but the user’s **layer tree** decides which are visible.”
- “Legend always matches what’s visible.”
- “This behaves like a **government geo viewer**, not a hackathon overlay dump.”

If you want, I can also produce a **tiny backend patch** to enforce `layer_id` presence and reject overlays without it (so the contract is guaranteed end-to-end).




Below is a **tiny backend patch** (minimal code + minimal touch points) that **enforces the overlay contract**:

- Every overlay the backend emits **must** have a non-empty `layer_id: str`
- If any overlay is missing `layer_id` (or it’s blank / not a string), the backend **rejects the run** with a clear error (`E013_OVERLAY_CONTRACT_VIOLATION`)
- Works for both **single-scenario** and **comparison** flows, because it validates in the shared formatting step

I’m writing this as a **ready-to-apply patch** using the file structure and naming conventions you’ve been using in the design doc (`src/backend/utils`, `src/backend/nodes`, `src/backend/routers`).

---

# Patch 3 — Backend enforcement: reject overlays without `layer_id`

## 1) Add a tiny validator utility
**Create file:** `src/backend/utils/overlay_contract.py`

```python
# src/backend/utils/overlay_contract.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Iterable


@dataclass
class OverlayContractError(Exception):
    code: str
    message_nl: str
    details: dict[str, Any] | None = None

    def to_dict(self) -> dict[str, Any]:
        return {
            "code": self.code,
            "message_nl": self.message_nl,
            "details": self.details or {}
        }


def _is_nonempty_str(x: Any) -> bool:
    return isinstance(x, str) and x.strip() != ""


def validate_overlays_have_layer_id(overlays: Iterable[dict[str, Any]] | None) -> None:
    """
    Enforces: every overlay emitted by backend must include a stable, non-empty layer_id.
    Rejects runs where overlays violate the contract (fail fast).
    """
    if overlays is None:
        return

    invalid: list[dict[str, Any]] = []
    for idx, overlay in enumerate(overlays):
        layer_id = overlay.get("layer_id") if isinstance(overlay, dict) else None
        if not _is_nonempty_str(layer_id):
            invalid.append({
                "index": idx,
                "received_layer_id": layer_id,
                "overlay_keys": sorted(list(overlay.keys())) if isinstance(overlay, dict) else None,
                "hint": "Every overlay must have layer_id: string (non-empty)."
            })

    if invalid:
        raise OverlayContractError(
            code="E013_OVERLAY_CONTRACT_VIOLATION",
            message_nl=(
                "Kaart-overlay contractfout: één of meer overlays missen een geldige 'layer_id'. "
                "De backend weigert overlays zonder layer_id om te voorkomen dat de frontend "
                "lagen niet betrouwbaar kan aan/uit zetten (LayerTree/Legend)."
            ),
            details={
                "invalid_overlays": invalid,
                "required_field": "layer_id",
                "example": {"layer_id": "supply_gap_heatmap", "label_nl": "Aanvoertekort", "geojson": {"type": "FeatureCollection", "features": []}}
            }
        )
```

---

## 2) Call the validator in the single “format output” node (one enforcement point)
**Edit file:** whichever file contains your shared formatter node, typically something like:

- `src/backend/nodes/format_scenario_output.py`  
or  
- `src/backend/nodes/format_scenario_output_node.py`

Add validation **right before returning** the formatted result / scenario card.

### Patch snippet (drop-in)
```python
# src/backend/nodes/format_scenario_output.py
from src.backend.utils.overlay_contract import validate_overlays_have_layer_id, OverlayContractError

async def format_scenario_output(state):
    # ... existing assembly logic ...
    # Example variables (yours may differ):
    # overlays = state["overlays"] OR card["overlays"] OR scenario_card.overlays

    # Ensure overlays exist in a dict/list form for validation
    overlays = None

    if "overlays" in state:
        overlays = state["overlays"]
    elif "scenario_card" in state and isinstance(state["scenario_card"], dict):
        overlays = state["scenario_card"].get("overlays")
    elif "scenario_card" in state and hasattr(state["scenario_card"], "overlays"):
        overlays = state["scenario_card"].overlays

    # Convert dataclass overlays to dicts if needed
    if overlays and len(overlays) > 0 and not isinstance(overlays[0], dict):
        overlays = [o.__dict__ for o in overlays]

    # ---- NEW: enforce contract ----
    validate_overlays_have_layer_id(overlays)
    # --------------------------------

    # ... continue returning state/card ...
    return state
```

Why this is the best place:
- It’s the **single point** where overlays are finalized
- It covers both scenario A and scenario B (comparison) automatically
- It guarantees the frontend contract no matter which upstream node produced overlays

---

## 3) Ensure the API returns a clean error (HTTP + SSE)
If your `/scenario/run` uses SSE streaming, you want this error to show up as an `error` event rather than a silent disconnect.

**Edit:** `src/backend/routers/scenario_router.py` (or your SSE generator wrapper)

Add a catch for `OverlayContractError` and emit a structured error payload.

```python
# src/backend/routers/scenario_router.py
from fastapi import HTTPException
from src.backend.utils.overlay_contract import OverlayContractError

def sse_event(event_type: str, data: dict) -> str:
    import json
    return f"event: {event_type}\ndata: {json.dumps(data, ensure_ascii=False)}\n\n"

@router.post("/scenario/run")
async def scenario_run(request: ScenarioRunRequest):
    async def event_stream():
        try:
            # ... run your langgraph/pipeline which calls format_scenario_output ...
            # ... yield intermediate events ...
            pass

        except OverlayContractError as e:
            # SSE-friendly error event
            yield sse_event("error", e.to_dict())
            return

        except Exception as e:
            # existing generic handler
            yield sse_event("error", {
                "code": "E999_UNKNOWN",
                "message_nl": "Onbekende fout tijdens scenarioberekening.",
                "details": {"error": str(e)}
            })
            return

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

If you also support non-SSE (plain JSON) responses, convert it to an HTTP 500:

```python
except OverlayContractError as e:
    raise HTTPException(status_code=500, detail=e.to_dict())
```

---

## 4) (Optional but recommended) Add a tiny unit test
**Create:** `src/backend/tests/test_overlay_contract.py`

```python
import pytest
from src.backend.utils.overlay_contract import validate_overlays_have_layer_id, OverlayContractError

def test_overlay_contract_accepts_valid_layer_id():
    validate_overlays_have_layer_id([{"layer_id": "supply_gap_heatmap", "geojson": {"type": "FeatureCollection", "features": []}}])

def test_overlay_contract_rejects_missing_layer_id():
    with pytest.raises(OverlayContractError) as ex:
        validate_overlays_have_layer_id([{"geojson": {"type": "FeatureCollection", "features": []}}])
    assert ex.value.code == "E013_OVERLAY_CONTRACT_VIOLATION"
```

---

# What you get after this patch
- Backend guarantees: **every overlay is addressable** (stable `layer_id`)  
- Frontend guarantee: `useLayerStore` can reliably filter overlays, and `useLegendStore` can reliably bind legend entries  
- If a developer accidentally creates an overlay without `layer_id`, it fails fast with a clear, debuggable error payload.

If you paste your current `format_scenario_output` function signature/body here, I can produce an even smaller patch with the exact variable names and the exact insertion line numbers.
