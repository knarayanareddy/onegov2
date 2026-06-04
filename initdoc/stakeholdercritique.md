🎭 Stakeholder Critique of the Drinkwaterzekerheid Design Document
Five Voices. One Tool. What's Working, What's Missing, What Needs to Change.
How to read this: I am stepping into the shoes of each stakeholder named in the challenge brief. I am reading the design document as they would — with their actual daily job, their frustrations, their questions, and their blind spots. After each stakeholder's critique, I surface the cross-cutting theme it represents. At the end, I give a consolidated gap list.

The three overheard pointers are woven through every critique:

🏚️ Silo problem: people work in isolation, no shared truth
🪞 Consistency problem: same query should give same answer to everyone
👁️ Intuition + transparency problem: tool must be instantly understandable, every output self-explanatory with sources
STAKEHOLDER 1 — PROVINCIAL POLICYMAKER
"Ik ben beleidsmedewerker Ruimtelijke Ordening, Provincie Zuid-Holland"
My job: I assess spatial plans, write advice letters (adviezen) for permit decisions, prepare Provinciale Staten briefings. I am not a hydrologist. I am not a data scientist. I have 15 years of experience reading policy documents. I use Excel.

What I like about this design
The "PolicymakerSummary" concept — plain Dutch first, detail on expand — is exactly right. The export to PDF briefing note is something I would actually use on Monday morning. The fact that every number links to its source means I can defend my advice letter if challenged.

What I am missing and what I would critique
1. I don't know what the tool "knows" before I ask it anything.

The design shows me a text box and some example questions. But I don't know what data this tool has. Does it know about the new woonakkoord that was signed last month? Does it know about the Hollandse IJssel dossier from 2022? Before I ask a question, I need to understand what universe of knowledge I am querying.

Gap: There is no "Wat weet dit systeem?" (What does this system know?) overview page. A policymaker should be able to see — before asking anything — which datasets are loaded, how fresh they are, and what topics they cover. This is not a nice-to-have. It is the foundation of institutional trust.

Concrete ask: Add a "Kennisbasis" (Knowledge Base) panel — always accessible — showing every loaded dataset, its source organisation, its publication date, and a freshness indicator. This panel should be queryable too: "Welke data heeft dit systeem over woningbouw?" should produce a list of the relevant loaded tables with source links.

2. 🏚️ Silo problem: I produce an advies, my colleague in water produces a different advies, and nobody knows they contradict each other.

The design currently positions this as a single-user tool. But in practice, my water colleague at the same province uses the same tool, asks slightly differently phrased versions of the same question, and gets — potentially — slightly different answers because of LLM variability. Six months later our two adviezen are submitted for the same permit, and they contradict each other.

Gap: The design does not address scenario sharing and institutional reproducibility. There is no concept of a "shared scenario library" — saved scenarios that colleagues can open, inspect, and confirm they are looking at the same calculation. The design mentions MLflow logging, but that is invisible to me as a policymaker.

Concrete ask:

A "Opgeslagen Scenario's" (Saved Scenarios) library — shareable within an organisation
A scenario URL (stable, shareable link) so I can send a colleague: "Kijk naar dit scenario — dit is wat ik heb gebruikt voor mijn advies"
A "Zelfde vraag, zelfde antwoord" guarantee section in the README: explain how deterministic the output is (temperature=0.1, seed, cached results)
3. 🪞 Consistency problem: I need to cite this tool in my official advice. How do I do that?

If I write an official provincial advice letter and I say "according to the drinkwaterscenario tool, a 50MW data centre in Delft will cause a deficit of 169,000 m³/day by 2036," I need to cite that. What is the citation? A URL to a GitHub repo? A timestamp?

Gap: The design has no formal citation / export metadata standard. There is a PDF export but it doesn't specify what makes a scenario "citeable" in a legal/administrative sense — run date, software version, dataset versions, scenario parameter hash.

Concrete ask: Every scenario card should display and include in the export:

Scenario ID: UUID
Aangemaakt op: ISO timestamp
Dataset versies: which version of each dataset was used
Softwareversie: git commit hash or release version
Citeerbaar als: a formatted citation string
4. 👁️ Intuition problem: What am I looking at on the map?

The design specifies five stakeholder view modes. But when I open the tool for the first time and see the map, I do not know which mode I am in, what the colours mean, or what the choropleth is showing. The design describes a legend panel but doesn't specify that it must be always visible, always in plain Dutch, always self-explanatory without needing to click anything.

Gap: The map needs a permanent, plain-language map title that changes with the view mode:

text

[Current map shows: Supply gap in m³/day per supply zone — KNMI scenario Hd, 2040]
That one line — always visible, always updated — tells a policymaker exactly what they are looking at before they read anything else.

STAKEHOLDER 2 — SPATIAL PLANNER (Gemeente / Omgevingsdienst)
"Ik werk bij een omgevingsdienst en toets ruimtelijke plannen op haalbaarheid"
My job: developers submit spatial plans; I check them against provincial policy, environmental law (Omgevingswet), and sector advice. I have 200+ permit reviews open at any time. I need fast, defensible answers.

What I like about this design
The "housing project impact" scenario (Scenario 4) is exactly what I need. A developer submits a plan for 2,500 homes in Zoetermeer — I type that in and get a water impact assessment. The "headroom remaining" concept is excellent: telling me not just "is this feasible" but "how much capacity is left for other plans" is genuinely useful.

What I am missing and what I would critique
1. 🏚️ Silo problem: I don't know what other permits have already been approved.

The design shows me "residual capacity in the Zoetermeer supply zone." But that residual capacity calculation — does it account for permits already approved? Or permits currently under review? Or only committed woondeals?

In practice: I am the fifth omgevingsdienst employee to check water capacity for Zoetermeer this year. The first four already approved plans consuming 800 m³/day of headroom. If the tool only knows about the woondeals dataset and not the approved permits database, it will tell me there is headroom when there isn't.

Gap: The design has no "al vergund" (already permitted) demand layer. This is the single biggest practical gap for permit-assessment use.

Concrete ask: Add an explicit "al vergund / vergund in behandeling" demand layer, even if it is initially empty and requires manual entry. The tool should show:

text

Beschikbare capaciteit:     1,240 m³/dag (totaal)
Al vergund:                 -810 m³/dag  [Bron: eigen invoer / OLO-koppeling]
In behandeling:             -430 m³/dag  [Bron: eigen invoer]
Beschikbaar voor dit plan:  0 m³/dag ⚠️
2. 👁️ Intuition problem: I need to know "is this plan OK or not" in one second.

The design describes a scenario card with a PolicymakerSummary. But where is the single, unambiguous "GO / CAUTION / STOP" indicator? Policymakers can absorb a traffic light faster than any prose summary.

Gap: There is no top-level Drinkwater Haalbaarheidsklasse (water feasibility class) for a given plan:

🟢 Groen: Voldoende capaciteit beschikbaar in alle KNMI-scenario's tot 2040
🟡 Geel: Capaciteit beschikbaar in Ln/Hn maar krap in Ld/Hd — aanvullende voorwaarden aanbevolen
🔴 Rood: Tekort verwacht vóór 2040 in één of meer KNMI-scenario's zonder interventie
Concrete ask: Add a FeasibilityClass computed field to every project scenario result, shown as the first, most prominent element of the scenario card before any number.

3. The design doesn't handle the "cumulative effect" question.

The brief mentions compound effects explicitly. But the current design treats each project in isolation. In my daily work, the question is almost never "does this one project cause a problem?" It is "do all the plans we are currently processing together cause a problem?"

Gap: No cumulative scenario mode — the ability to add multiple concurrent projects and see their combined effect on a supply zone.

Concrete ask: Allow the user to add multiple project markers to the map simultaneously and see the aggregate demand increase and combined risk assessment with a "stapelingseffect" (stacking effect) summary.

STAKEHOLDER 3 — WATER AUTHORITY (Dunea / Evides)
"Ik ben capaciteitsplanner bij Dunea"
My job: I plan production capacity 10–25 years ahead. I monitor intake quality daily. I advise municipalities and the province on what is and isn't feasible from a supply perspective. I have engineering and hydrology background.

What I like about this design
The calculation engine is directionally right. The KNMI'23 axes (Hd/Hn/Ld/Ln) are correct and I'm relieved the old W+/G/W codes are gone. The interventie catalogue (buffer, alternative intake, demand reduction) is a reasonable first cut. The production chain visualization on the map is useful for internal planning discussions.

What I am missing and what I would critique
1. The chloride threshold is hardcoded at 150 mg/L but that is wrong for some intakes.

The design document uses cl_threshold_mg_l = 150 as a constant. But in practice, different intake points have different thresholds depending on treatment technology. The Gouda intake threshold is different from the Bergambacht intake threshold.

Gap: A single chloride threshold applied to all locations is technically incorrect and will produce wrong results for some locations. The correct value per intake should come from the productieketen dataset or from the drinkwaterbedrijf's own data.

Concrete ask: cl_threshold_mg_l must be a per-location field in the WaterLocation dataclass, sourced from the productieketen layer, not a global constant. If the value is not available, it should be flagged explicitly as an assumption with the source URL of the default value.

2. 🪞 Consistency problem: If I ask this tool the same question as a developer, do we get the same answer?

This is a real operational problem. A developer runs this tool, gets "feasible," and includes that in their permit application. I run the same tool, get "not feasible," and send back a negative advice. Now we have a legal dispute with contradictory outputs from the same government tool.

Gap: The design does not address output determinism beyond mentioning temperature=0.1. It does not specify whether cached results are served, whether parameter normalization ensures identical questions produce identical outputs, or what happens when the underlying dataset is updated between the developer's query and my query.

Concrete ask:

Implement explicit scenario caching: if the same parameter combination has been computed before, return the cached result with a "berekend op [datum]" timestamp
Add a "Dataset versie gewijzigd" warning if the underlying data has been updated since the cached result was computed
Provide a "Verifieer berekening" button that any party can use to re-run the scenario with current data and compare
3. 👁️ The production chain visualization needs to show actual flow direction, not just locations.

The current map design shows intake points as dots. For a water authority audience, the relevant question is: which intake feeds which production location, which feeds which distribution zone? That is a directed graph, not a scatter plot.

Gap: No production chain flow visualization — the directed "intake → productie → distributie → eindgebruiker" chain is not shown as a flow diagram or a connected map layer.

Concrete ask: Add a "Productieketen view" that renders the supply chain as a connected graph overlaid on the map — arrows showing flow direction, thickness showing capacity, colour showing current/predicted status.

4. 🏚️ Silo problem: This tool doesn't know about Waterinfo's real-time chloride data.

The design mentions waterinfo.rws.nl as an optional external source to be called via API. But it is listed as "optional" and the design's risk register flags it as something to pre-download as a backup. For a water authority, chloride levels at the Hollandse IJssel are not optional — they are the core operational signal.

Concrete ask: Make the waterinfo.rws.nl chloride API call mandatory, not optional, for any scenario involving IJssel/Lek/Maas intakes. If the API is unavailable, show an explicit warning: "Actuele chloridedata niet beschikbaar — berekening gebruikt laatste bekende waarde van [datum]. Bron: waterinfo.rws.nl" rather than silently falling back.

STAKEHOLDER 4 — PROJECT DEVELOPER (Data Centre / Housing)
"Ik ben ontwikkelaar — ik wil weten of mijn project water krijgt"
My job: I am either a housing developer or a large-scale infrastructure developer (data centre, logistics hub). I need to assess site feasibility quickly and cheaply before committing to expensive land purchases. I have no water expertise. I speak business Dutch.

What I like about this design
The "drop a pin on the map" project marker concept is the right interaction model. The before/after toggle (WITH vs WITHOUT my project) is exactly how I think about it. The feasibility class (🟢🟡🔴) — which I requested above — would be perfect for my use.

What I am missing and what I would critique
1. 👁️ Intuition problem: The tool assumes I know what "m³/dag" means.

The scenario card shows "Tekort: 169,000 m³/dag." I do not have an intuitive sense of scale for that. Is 169,000 m³/dag a lot? The design document does not specify any contextualisation of numbers for non-expert users.

Gap: Every key metric needs a plain-language scale reference:

text

Tekort: 169,000 m³/dag
(Dit is gelijk aan het dagelijkse waterverbruik van ongeveer 1,2 miljoen mensen)
[Bron: VEWIN — gemiddeld 140 liter per persoon per dag]
This one addition makes the tool instantly usable for non-experts without changing the underlying calculation at all.

Concrete ask: Add a human_scale_reference field to every SourcedValue that is displayed in the UI — a plain-Dutch analogy computed from the value and sourced to a reference (VEWIN, CBS, etc.).

2. I need to know: "what do I need to do to make my project feasible?"

The current design tells me my project is red (deficit). But it doesn't tell me what I can do about it. Does paying for a buffer capacity extension solve it? Is there an alternative site nearby that is green? Can demand reduction measures (waterbesparende installaties) bring it to yellow?

Gap: The design has an intervention catalogue but it is not surfaced as a "Maak dit project haalbaar" (Make this project feasible) recommendation flow.

Concrete ask: For every 🔴 or 🟡 project scenario, automatically compute and show:

text

Wat maakt dit project haalbaar?

✅ Optie 1: Bufferkapaciteit [Effect: deficit daalt van 169.000 naar 41.000 m³/dag]
   Bron: Regionaal Waterprogramma ZH → [link]

✅ Optie 2: Alternatieve inname activeren [Effect: deficit daalt naar 12.000 m³/dag]
   Bron: Productieketen drinkwater → [link]

⚠️ Optie 3: Vraagbeperking (waterrecycling) [Effect: -15% demand = 27.000 m³/dag bespaard]
   Bron: VEWIN Waterstatistieken → [link]

❌ Geen combinatie dekt het tekort volledig onder scenario Hd zonder infrastructuurinvestering.
3. 🏚️ Silo problem: I don't know if another developer already asked about the same location.

Two developers can ask about the same site. The first gets "green." The second — a week later — gets "green" too, because the tool doesn't know the first developer's plan exists. In reality, both plans together would make it red.

Gap: Same as the spatial planner's "cumulative effect" gap — but from the developer's perspective this is even more acute because they are making financial decisions.

Concrete ask: When a developer drops a pin, show them: "Er zijn X andere actieve scenario's voor deze supply zone. Gezamenlijk effect: [Y] m³/dag extra vraag." This requires the shared scenario library from the policymaker's request.

STAKEHOLDER 5 — GENERAL PUBLIC / CITIZEN
"Ik woon in Zoetermeer en ik wil weten of mijn drinkwater veilig is in 2040"
My background: I read about the drinkwater crisis in the newspaper. I am worried. I am not a specialist. I am 45, I use Google, I want a straight answer.

What I like about this design
That a "general public" stakeholder view mode exists at all is positive. The accessibility-first principles (plain Dutch, no jargon) are good.

What I am missing and what I would critique
1. 👁️ Intuition problem: The tool is currently designed for professionals, not citizens.

The example questions in the design are: "Wat is het effect van een 50MW datacentrum in Delft?" and "2.500 woningen in Zoetermeer — drinkwater haalbaar?" These are professional questions. A citizen asks: "Is mijn drinkwater veilig?" or "Wat betekent die droogtemelding voor mijn kraan?"

Gap: The design does not specify what a citizen-appropriate question looks like or how the system should respond to one. There is a "Burger" view mode in the stakeholder switcher but it has no distinct UX or response format specified.

Concrete ask: Define the citizen response format explicitly:

text

[Map: Your postcode area highlighted in green/amber/red]

🟢 Uw drinkwater in Zoetermeer is naar verwachting veilig tot 2040.

In een gemiddeld klimaatscenario blijft er voldoende capaciteit.
In het droogste scenario (KNMI scenario Hd) is de marge kleiner,
maar er is geen tekort verwacht voor uw gebied.

Meer informatie: Dunea — uw drinkwaterbedrijf → dunea.nl
Klimaatinformatie: KNMI → klimaatscenarios.knmi.nl
No numbers unless the citizen asks. No jargon. Always with a link to the responsible drinkwaterbedrijf.

2. 🪞 Consistency problem: What I read in the newspaper and what this tool says should match.

If Provincie ZH publishes a press release saying "water security is a risk in 2040," and I run this tool and it says "green — safe until 2040," I will not trust this tool. Conversely, if this tool is more alarming than official communications, it may cause unnecessary concern.

Gap: No "Officiële positie" (official position) anchor in the tool output. The tool currently shows its own scenario calculations without relating them to official government statements or published risk assessments.

Concrete ask: Add a "Wat zegt de overheid hierover?" section to every scenario that links to official published policy positions:

Provinciaal Waterprogramma Zuid-Holland 2022–2027
Nationaal Water Programma 2022–2027
Relevant Kamerstukken / policy letters
This is not about changing the calculation — it is about contextualising it against official sources so the tool's output is legible within the broader public information ecosystem.

THE MISSING FEATURES: CONSOLIDATED GAP LIST
Drawing all five critiques together, here are the gaps that appear across multiple stakeholders — these are the highest priority because they affect everyone:

GAP 1: No "What does this system know?" panel (affects all 5 stakeholders)
The problem: Users don't know what data the tool has before asking. The fix: A permanent, always-accessible "Kennisbasis" panel listing every loaded dataset, its freshness, its source, and what topics it covers — queryable in natural language. Why critical: Without this, users cannot calibrate their trust in the tool's answers. This is the foundation of the silo-breaking promise.

GAP 2: No shared scenario library / scenario URL (policymaker + spatial planner + water authority + developer)
The problem: Different users asking the same question at different times get answers that may be inconsistent, and there is no way to share or compare. The fix:

Saved scenarios with stable URLs
Organisation-level scenario library
"Same question, same answer" guarantee documented and implemented
Dataset version tracking per scenario run Why critical: This is the direct response to the "silo" problem. The tool currently creates information asymmetry — the first person who asks gets a different answer than the last — rather than resolving it.
GAP 3: No GO / CAUTION / STOP feasibility class (spatial planner + developer + citizen)
The problem: The tool produces numbers but not decisions. Policymakers and developers need an actionable signal, not just data. The fix: A FeasibilityClass (🟢🟡🔴) computed across all four KNMI scenarios, shown as the first and most prominent element of every project scenario output. Why critical: This is the "intuition" problem. Without a top-level signal, users have to interpret numbers themselves — and they will interpret them differently, recreating the silo problem in individual heads.

GAP 4: No human-scale contextualisation of numbers (developer + citizen + policymaker)
The problem: "169,000 m³/day" means nothing to a non-expert. The fix: Every key metric displays a plain-Dutch analogy:

m³/day → "equivalent to the daily use of X people" (VEWIN source)
housing units at risk → "equivalent to the population of [gemeente]" (CBS source)
cl concentration → "X times above the safe drinking level" (KRW source) Why critical: This is the "transparency" and "intuition" problem. Data that isn't understood isn't actionable.
GAP 5: No "make this feasible" recommendation flow (developer + policymaker)
The problem: The tool tells users their project is red but doesn't tell them what to do. The fix: For every non-green scenario, automatically compute and rank interventions by effectiveness, with source links for each. Why critical: A government tool that identifies a problem but offers no path forward is less useful than one that identifies the problem AND points toward the solution space.

GAP 6: No cumulative / stacking effect mode (spatial planner + developer + water authority)
The problem: Each project is assessed in isolation, but real-world capacity is consumed cumulatively. The fix: Multi-project overlay: add multiple project markers, see combined demand, see combined risk. Why critical: This is the deepest silo problem. Individual projects "pass" while the combined load fails — a pattern that only becomes visible when you can see all plans simultaneously.

GAP 7: Per-location chloride thresholds (water authority)
The problem: The tool uses a single global chloride threshold (150 mg/L) which is technically wrong. The fix: cl_threshold_mg_l as a per-location field, sourced from the productieketen dataset, with a fallback default that is explicitly flagged as an assumption. Why critical: Technically incorrect thresholds produce incorrect risk assessments — which in a government context can have legal and safety implications.

GAP 8: "What does the government officially say?" context panel (citizen + policymaker)
The problem: The tool's outputs exist in isolation from official government positions. The fix: A "Officiële positie" section linking every scenario to relevant published policy documents (Waterprogramma, Kamerstukken, etc.). Why critical: In a government context, a tool whose outputs contradict official communications — without explanation — will not be trusted by either policymakers or citizens.

GAP 9: Waterinfo chloride API is "optional" — it should be mandatory (water authority + spatial planner)
The problem: The design treats real-time chloride data as optional with a "pre-download as backup" fallback. For intake-failure scenarios this is the core input. The fix: Make the waterinfo.rws.nl call mandatory for IJssel/Lek/Maas intake scenarios, with an explicit "data unavailable — using last known value from [date]" warning if the API fails. Why critical: A scenario about intake failure due to salinization that doesn't use actual chloride data is a significant credibility gap in front of a water authority jury.

GAP 10: No citizen-specific question format and response template (citizen)
The problem: The tool is designed for professionals. Citizens asking simple questions get professional-grade outputs they cannot interpret. The fix: Detect citizen-type questions (by tone/vocabulary) and route them to a simplified response template: postcode → map view → one-sentence verdict → link to drinkwaterbedrijf. Why critical: The brief explicitly asks for a tool usable by the general public. Without a defined citizen response format, the "Burger" view mode is a label, not a feature.

Summary: The Three Overheard Pointers Against the Design Doc
Pointer	Current state in design doc	What's missing
🏚️ "They work in silos"	The tool is single-user; no shared scenarios; no cumulative effect mode; no "already approved" demand layer	Shared scenario library + stable URLs + cumulative project overlay + "al vergund" layer
🪞 "Same query = same answer"	Temperature=0.1 is mentioned; MLflow logs runs; but no caching guarantee, no dataset-version tracking, no scenario URL for sharing	Explicit scenario caching + dataset version per run + shareable scenario URL + "verify calculation" button
👁️ "Intuitive, transparent, easy to follow"	Plain Dutch summary exists; stakeholder tabs exist; source links exist	Missing: GO/CAUTION/STOP signal; human-scale number analogies; "What does this system know?" panel; permanent map title; citizen response template; "make this feasible" recommendation flow




