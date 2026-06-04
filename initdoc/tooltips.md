Below is a **ready-to-paste Markdown patch** that adds an **accessible “Help System”** to your design doc, including:

- **Help Registry schema** (single source of truth for definitions/hints)
- **Tooltip + Popover behavior** (hover/focus for hints; click to pin details)
- **Accessibility acceptance criteria** (WCAG-aligned; no hover-only UX)

This patch assumes you’ve already integrated the earlier patches, so:
- Frontend stores currently go through **15.10**
- Components currently go through **16.13**
- Validation checklist is in **Section 26**

---

# PATCH 4A — Insert **Sections 15.11–15.13** (Help Registry + store + wiring)
**Where to insert:** Immediately after your existing **`15.10 SSE wiring update — map_data → raw overlays → visible overlays`** section and **before** the heading `16. Vue component specifications`.

```md
15.11 Help Registry schema (single source of truth for hints/explanations)

Goal:
- Provide consistent, government-grade microcopy for “what does this mean?” across:
  - map layers (`layer_id`)
  - metrics (`metric_key`)
  - assumptions (`assumption_key`)
  - UI concepts (`ui_key`)
- Avoid fragmented ad-hoc help text inside components.
- Ensure “every explanation links” (source URL is mandatory, same spirit as SourceRegistry).

TypeScript

// src/frontend/src/registries/helpRegistry.ts
export type HelpKey =
  | { type: 'layer'; id: string }        // overlay/layer_id
  | { type: 'metric'; id: string }       // e.g. supply_gap_m3
  | { type: 'assumption'; id: string }   // e.g. demand_per_dwelling_m3_day
  | { type: 'ui'; id: string }           // e.g. feasibility_class

export type HelpEntry = {
  key: HelpKey
  title_nl: string

  // Tooltip content: max ~120 chars, 1–2 lines (B1 Dutch)
  short_hint_nl: string

  // Popover/panel content: max ~500 chars; B1 Dutch
  explanation_nl: string

  // Mandatory for trust and auditability
  source_url: string
  source_label: string

  // Optional metadata (helps with governance)
  last_updated?: string   // ISO date
  tags?: string[]         // e.g. ['krw', 'knmi', 'drinkwater']
}

function k(type: HelpKey['type'], id: string): string {
  return `${type}:${id}`.toLowerCase()
}

/**
 * HELP_REGISTRY is a flat dictionary for O(1) lookups.
 * Key format: `${type}:${id}` normalized to lowercase.
 */
export const HELP_REGISTRY: Record<string, HelpEntry> = {
  // --- UI concepts ---
  [k('ui', 'feasibility_class')]: {
    key: { type: 'ui', id: 'feasibility_class' },
    title_nl: 'Haalbaarheid (GO / RISICO / NIET HAALBAAR)',
    short_hint_nl: 'Samenvatting van haalbaarheid over meerdere KNMI-scenario’s.',
    explanation_nl:
      'Deze indicator vat samen of het scenario haalbaar is in 2040 onder meerdere KNMI-klimaatscenario’s. '
      + 'GO = haalbaar onder alle scenario’s. RISICO = haalbaar in sommige scenario’s, tekort in de droogste. '
      + 'NIET HAALBAAR = tekort in meerdere scenario’s (incl. basisscenario).',
    source_url: '/methodology#feasibility',
    source_label: 'Methodologie (OneGov2 Scenario Engine)',
    last_updated: '2026-06-04'
  },

  // --- Metrics ---
  [k('metric', 'supply_gap_m3')]: {
    key: { type: 'metric', id: 'supply_gap_m3' },
    title_nl: 'Aanvoertekort (m³/dag)',
    short_hint_nl: 'Negatief = tekort. Positief = overschot.',
    explanation_nl:
      'Aanvoertekort is het verschil tussen berekende vraag en beschikbare capaciteit in de leveringszone. '
      + 'Een negatief getal betekent dat er op piekmomenten niet genoeg water kan worden geleverd zonder maatregelen.',
    source_url: '/methodology#supply-gap',
    source_label: 'Methodologie (OneGov2 Scenario Engine)',
    last_updated: '2026-06-04'
  },

  // --- Assumptions ---
  [k('assumption', 'demand_per_dwelling_m3_day')]: {
    key: { type: 'assumption', id: 'demand_per_dwelling_m3_day' },
    title_nl: 'Waterverbruik per woning (aanname)',
    short_hint_nl: 'Bandbreedte omdat huishoudens verschillen in grootte en gedrag.',
    explanation_nl:
      'Dit is een beleidsmatige schatting van gemiddeld waterverbruik per woning per dag. '
      + 'De werkelijke waarde hangt af van huishoudensgrootte, gedrag en efficiëntie. '
      + 'We tonen een bandbreedte en laten de aanname aanpassen om gevoeligheid te verkennen.',
    source_url: 'https://www.vewin.nl/publicaties/waterstatistiek',
    source_label: 'VEWIN Waterstatistiek',
    last_updated: '2026-06-04',
    tags: ['vraag', 'woningbouw']
  },

  // --- Layers (overlays) ---
  [k('layer', 'supply_gap_heatmap')]: {
    key: { type: 'layer', id: 'supply_gap_heatmap' },
    title_nl: 'Kaartlaag: Aanvoertekort (heatmap)',
    short_hint_nl: 'Laat zien waar het tekort het grootst is binnen de leveringszones.',
    explanation_nl:
      'Deze laag visualiseert het berekende aanvoertekort per zone (of per grid/hexagon, afhankelijk van data). '
      + 'Gebruik de legenda om de klassen te interpreteren. Klik op een zone voor details en aannames.',
    source_url: '/methodology#map-layers',
    source_label: 'Methodologie (OneGov2 Scenario Engine)',
    last_updated: '2026-06-04',
    tags: ['kaart', 'tekort']
  }
}


15.12 useHelpStore.ts (lookup + contract enforcement)

Goal:
- Provide a single, safe API for components to fetch help content.
- Guarantee: every help entry rendered has a non-empty source_url (no silent blanks).
- If a key is unknown, return a “Methodologie” fallback (still with a source link).

TypeScript

// src/frontend/src/stores/useHelpStore.ts
import { defineStore } from 'pinia'
import { HELP_REGISTRY, HelpEntry, HelpKey } from '@/registries/helpRegistry'

function keyToString(key: HelpKey): string {
  return `${key.type}:${key.id}`.toLowerCase()
}

export const useHelpStore = defineStore('help', {
  state: () => ({
    // currently opened popover key (only one at a time)
    openPopoverKey: null as null | string
  }),

  actions: {
    getEntry(key: HelpKey): HelpEntry {
      const k = keyToString(key)
      const entry = HELP_REGISTRY[k]

      if (entry && entry.source_url && entry.source_url.trim() !== '') {
        return entry
      }

      // Strict fallback (never return empty source_url)
      return {
        key,
        title_nl: 'Toelichting',
        short_hint_nl: 'Toelichting niet beschikbaar. Zie methodologie.',
        explanation_nl:
          'Er is nog geen specifieke toelichting geregistreerd voor dit onderdeel. '
          + 'Zie het methodologiedocument voor definities, aannames en berekeningen.',
        source_url: '/methodology',
        source_label: 'Methodologie (OneGov2 Scenario Engine)',
        last_updated: '2026-06-04'
      }
    },

    openPopover(key: HelpKey) {
      this.openPopoverKey = keyToString(key)
    },

    closePopover() {
      this.openPopoverKey = null
    },

    togglePopover(key: HelpKey) {
      const k = keyToString(key)
      this.openPopoverKey = (this.openPopoverKey === k) ? null : k
    }
  }
})


15.13 Help behavior contract (Tooltip vs Popover vs FeatureInfo)

Definitions:
- Tooltip = short hint, appears on **hover and keyboard focus**, disappears on blur/leave.
- Popover = richer explanation, opens on **click** (or Enter/Space), stays open until closed.
- FeatureInfoPanel = for map objects; hover only highlights geometry, click pins details.

Rules:
1) **No hover-only help.** Every tooltip-trigger must also work on keyboard focus.
2) Every non-obvious label that could affect interpretation must have a HelpIcon (ⓘ).
   Required coverage:
   - FeasibilityBadge (GO/RISICO/NIET HAALBAAR)
   - MapTitle metric label (“Aanvoertekort”, “Chloride risico”, “Delta”)
   - Legend layer labels
   - Assumption sliders
3) Tooltip content must be B1 Dutch, max ~120 chars (1–2 lines).
4) Popover content must be B1 Dutch, max ~500 chars, and include:
   - what it is
   - why it matters
   - link to authoritative source or methodology
5) Popover must be dismissible via:
   - ESC key
   - close button
   - click outside
6) For map features:
   - hover = highlight only (no critical info hidden behind hover)
   - click = open FeatureInfoPanel with stable content (works on mobile)
```

---

# PATCH 4B — Insert **Component specs 16.14–16.16** (Tooltip, InfoPopover, HelpIcon)
**Where to insert:** Immediately after your current **`16.13 FeatureInfoPanel.vue`** section and **before** `17. Map layer configuration`.

```md
16.14 Tooltip.vue (hover + focus micro-hints; not hover-only)

Purpose:
- Show 1–2 line hints on hover AND keyboard focus.
- Must be accessible: role=tooltip + aria-describedby binding from trigger.

vue

<script setup lang="ts">
import { computed, ref } from 'vue'

const props = defineProps<{
  id: string
  text: string
  isVisible: boolean
}>()

const tooltipId = computed(() => props.id)
</script>

Template requirements:
- `<div :id="tooltipId" role="tooltip" v-show="isVisible"> {{ text }} </div>`
- Trigger element must include: `:aria-describedby="tooltipId"` when visible.


16.15 InfoPopover.vue (click-to-pin explanations)

Purpose:
- Show richer help text on click (or Enter/Space).
- Stays open until dismissed.
- Must be keyboard and screen-reader friendly.

Accessibility requirements:
- popover container has `role="dialog"` or `role="region"` with `aria-label`
- focus moves into popover when opened; returns to trigger when closed
- ESC closes popover

vue

<script setup lang="ts">
defineProps<{
  title: string
  body: string
  sourceUrl: string
  sourceLabel: string
  isOpen: boolean
}>()

defineEmits<{
  (e: 'close'): void
}>()
</script>

Template requirements:
- Close button with visible label ("Sluiten")
- Source link always visible: `📚 {sourceLabel}`


16.16 HelpIcon.vue (single UX affordance: tooltip + popover + source link)

Purpose:
- Standardizes help affordance across the app.
- Given a HelpKey, it renders:
  - tooltip on hover/focus (short_hint_nl)
  - popover on click (title + explanation + source link)
- Used in:
  - FeasibilityBadge
  - MapTitle
  - LegendPanel
  - AssumptionSliders
  - Metric cards

Behavior:
- Hover/focus shows tooltip (short_hint_nl)
- Click toggles popover (explanation_nl)
- ESC closes popover
- Works on touch (click only, tooltip optional)

vue

<script setup lang="ts">
import { computed, ref } from 'vue'
import { useHelpStore } from '@/stores/useHelpStore'
import Tooltip from '@/components/Tooltip.vue'
import InfoPopover from '@/components/InfoPopover.vue'

const props = defineProps<{
  helpKey: { type: 'layer' | 'metric' | 'assumption' | 'ui'; id: string }
}>()

const helpStore = useHelpStore()
const entry = computed(() => helpStore.getEntry(props.helpKey as any))

const isHovering = ref(false)
const isFocused = ref(false)

const tooltipVisible = computed(() => isHovering.value || isFocused.value)
const popoverKey = computed(() => `${props.helpKey.type}:${props.helpKey.id}`.toLowerCase())
const popoverOpen = computed(() => helpStore.openPopoverKey === popoverKey.value)

function onClick() {
  helpStore.togglePopover(props.helpKey as any)
}

function onKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault()
    onClick()
  }
  if (e.key === 'Escape') {
    helpStore.closePopover()
  }
}
</script>

Template requirements:
- Trigger is a `<button>` with label "Info" (visually can be ⓘ)
- `@mouseenter/@mouseleave` set hovering
- `@focusin/@focusout` set focused
- `@keydown` handles Enter/Space/Escape
- Tooltip receives `entry.short_hint_nl`
- Popover receives `entry.title_nl`, `entry.explanation_nl`, `entry.source_url`, `entry.source_label`
```

---

# PATCH 4C — Add **Help System accessibility acceptance criteria** to Section 26
**Where to insert:** In Section `26. Validation & acceptance criteria`, add a new subsection (near “Usability” / “Accessibility”)—use the next available number. If your doc currently ends at `26.x`, paste this as the final subsection.

```md
26.X Help System & accessibility acceptance criteria (Tooltip/Popover)

Required:
- [ ] No critical information is hover-only. Every tooltip trigger works on keyboard focus.
- [ ] Tooltip:
  - [ ] Appears on hover AND focus
  - [ ] Has `role="tooltip"`
  - [ ] Trigger uses `aria-describedby` when tooltip is visible
  - [ ] Tooltip text is max ~120 chars, B1 Dutch
- [ ] Popover:
  - [ ] Opens on click and on Enter/Space
  - [ ] Closes on ESC, close button, and click outside
  - [ ] Has `role="dialog"` or `role="region"` with accessible label
  - [ ] Focus moves into popover on open and returns to trigger on close
  - [ ] Includes a visible source link (no empty source_url)
- [ ] Help Registry:
  - [ ] All HelpEntries rendered in UI have a non-empty `source_url`
  - [ ] Unknown keys fall back to `/methodology` (never blank)
- [ ] Map interactions:
  - [ ] Hover only highlights geometry (does not hide key meaning)
  - [ ] Click pins details in FeatureInfoPanel (works on mobile)
```

---

## Notes (why this matches your doc style)
- Uses the same structure you’ve been using: **Goal → TypeScript/Vue → wiring rules → acceptance criteria**
- Aligns with your “trust primitives” philosophy: **help content has sources and is centrally governed**
- Aligns with GGC-style government UX: **progressive disclosure, keyboard-first, predictable panels**

If you paste your current Section 26 numbering (the last subsection number), I can replace `26.X` with the exact correct number so you can drop it in without any manual renumbering.
