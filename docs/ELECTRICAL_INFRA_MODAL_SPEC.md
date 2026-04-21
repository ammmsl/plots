# Electrical Infrastructure Modal — Specification

**Document version:** 1.0  
**Status:** Ready for implementation  
**Supersedes:** Sections 7–8 of `POWERHOUSE_LAYOUT_ELECTRICAL_SPEC.md` (Parts D and E), which are replaced in full by this document. Parts A, B, and C of that document are also revised — see §9.  
**Read alongside:** `ELECTRICAL_INFRASTRUCTURE.md` (cost constants and sizing equations), `CONTROLLER_KNOWLEDGE_BASE.md` (brand product stacks and decision rationale)

---

## Table of Contents

1. [Purpose and design intent](#1-purpose-and-design-intent)
2. [Scope boundaries](#2-scope-boundaries)
3. [Design principles](#3-design-principles)
4. [Data interface](#4-data-interface)
5. [Cost model — three tiers](#5-cost-model--three-tiers)
6. [Flag trigger conditions](#6-flag-trigger-conditions)
7. [Controller brand model](#7-controller-brand-model)
8. [UI specification](#8-ui-specification)
9. [Powerhouse modal change — trench shading only](#9-powerhouse-modal-change--trench-shading-only)
10. [app.jsx integration](#10-appjsx-integration)
11. [TCO integration](#11-tco-integration)
12. [Recomputation behaviour](#12-recomputation-behaviour)
13. [New assumptions ledger entries](#13-new-assumptions-ledger-entries)
14. [What is explicitly out of scope](#14-what-is-explicitly-out-of-scope)

---

## 1. Purpose and design intent

The Electrical Infrastructure Modal answers one question for the skilled user:

> **"By choosing this fleet configuration, what electrical infrastructure class am I committing to, who controls it, and what is the order-of-magnitude cost?"**

It is a decision-support tool, not a detailed design tool. The output is not a BoM for procurement. It is a cost characterisation and technology consequence that makes the fleet comparison visible across a dimension that is otherwise invisible — electrical infrastructure is a significant, non-negotiable cost that changes shape materially between fleet strategies and is almost always underestimated or omitted from early project comparisons.

The modal contributes one combined electrical CapEx line to the main TCO comparison. The detail behind that line is visible inside the modal.

---

## 2. Scope boundaries

### In scope

- Generator ACBs — one per generator, frame selected by rated current
- Generator cable runs — sized by generator kWe and layout mode, worst-case overspec length
- Cable trenching — characterised by trench depth class, not metered precisely
- Per-generator cubicle assembly — enclosure, ACB, controller module, CTs, metering, terminations
- Main distribution ACB — sized to full installed fleet
- Bus coupler ACB — when split bus is triggered
- Busbar assembly — bars per phase by total fleet FLA
- Controller selection — per-gen paralleling module and site master, by brand and fleet size
- Hybrid integration layer — when BESS or solar is active and controller brand requires it
- Earthing and surge protection — tiered by fleet size
- Factory acceptance testing (FAT) — tiered by fleet size, with or without witnessed option

### Explicitly out of scope

| Item | Reason |
|---|---|
| Precise cable run lengths from geometry | Layout-mode overspec is sufficient for planning grade |
| Outgoing load ACBs downstream of MDB | Load distribution scope — site-specific |
| Transformer scope (MV/LV) | Not applicable — island microgrid is LV throughout |
| BESS inverter connections and DC cabling | BESS scope |
| Solar array cabling and string inverters | Solar scope |
| On-site commissioning and testing labour | Site-specific, not estimable from modal inputs |
| Specialist island travel for installation team | Too variable |
| Harmonic filtering | Requires a power quality study |
| WebSupervisor / DEIF Cloud subscription OpEx | Optional; on-premise monitoring is achievable at comparable cost |
| Mains parallel / grid-tie controller features | Island microgrid IS the mains — mains supervision features do not apply |
| Quote overrides for electrical line items | Electrical infrastructure always exists regardless of configuration — no override needed |
| Genset AMF local panel | Generator scope |

---

## 3. Design principles

**P-1 — Shape of the problem, order of magnitude of cost.**  
The modal shows cost tier and cost estimate. It does not show line-by-line BoM detail on the primary view. Line items are available on expand per cubicle. The primary output is a characterisation: which cost cliff has been triggered, what that means, and approximately what it costs.

**P-2 — All three strategies use the same control architecture.**  
The controller brand selection is one choice per site. Comparing strategies with different controller brands would compare commercial decisions alongside technical ones, making the fleet shape comparison incoherent. Brand is selected once; its cost consequence differs per strategy because fleet size differs.

**P-3 — You change, you lose.**  
The modal recomputes from scratch on every open. There is no cached state, no quote overrides, no staleness detection. If the user changes fleet configuration in the sidebar and reopens the modal, new numbers appear. This is intentional — preserving stale results after a fleet change would be misleading.

**P-4 — Worst-case cable overspec.**  
Cable run lengths are derived from the user's selected layout mode using the longest possible route in that layout — the furthest generator from the MDB room. A routing overhead factor is then applied on top. This consistently over-estimates cable cost, which is the correct direction for planning-grade estimates: as-built cable runs never come in shorter than the layout implies; they often come in longer due to bends, obstructions, and deviations from the design.

**P-5 — Full fleet always, clearly communicated.**  
The main distribution ACB is sized to carry the full installed fleet simultaneously, not the URA design load minimum. Power creep across a 10-year resort horizon is normal. Sizing to full fleet eliminates distribution upgrade events within the TCO horizon. The URA minimum is noted for reference. No user override is provided below the full fleet figure.

**P-6 — Honest about DSE hybrid limitations.**  
When DSE is selected and BESS or solar is active, the modal surfaces a phasing narrative — not a warning penalty. DSE is a legitimate, commercially dominant choice. The hybrid gap is a phasing decision, not a disqualification. The integration budget for Phase 2 is included in the cost estimate so the total is comparable to ComAp and DEIF. Timeline and phasing structure remain the client's commercial decision.

---

## 4. Data interface

### Props received by `ElectricalInfraModal`

```javascript
<ElectricalInfraModal
  // Three strategy fleet configurations
  strategies={[
    { id: 'A', label: 'Strategy A', genKWe: number, fleetSize: number },
    { id: 'B', label: 'Strategy B', genKWe: number, fleetSize: number },
    { id: 'C', label: 'Strategy C', genKWe: number, fleetSize: number },
  ]}

  // Hybrid activity flags — drive controller recommendation and phasing narrative
  bessActive={boolean}    // true when bessCapacityKwh > 0
  solarActive={boolean}   // true when solarKwp > 0

  // Callbacks
  onClose={fn}
  onElecCapExChange={fn}  // (stratId: 'A'|'B'|'C', supplyTotal: number, installedTotal: number) => void
/>
```

### What the modal does NOT receive

- `runLengths` — not needed; cable runs are derived from layout mode internally
- `elecOriginPort`, `elecIslandKm` — freight is a qualitative tier output, not a cost calculation
- Any BESS or solar cost data — only the activity flags matter

### State owned by `app.jsx`

```javascript
// Electrical modal open state
const [showElecModal, setShowElecModal] = useState(false)

// Electrical CapEx results — set by the modal via onElecCapExChange callback
// Null until the modal has been opened and results computed
const [elecCapExA, setElecCapExA] = useState(null)  // { supply, installed } | null
const [elecCapExB, setElecCapExB] = useState(null)
const [elecCapExC, setElecCapExC] = useState(null)

// Cleared when any fleet configuration changes — see §12
```

### State owned by `ElectricalInfraModal` (local, not lifted)

```javascript
// User selections — persist while the modal is open, reset on close
const [controllerBrand, setControllerBrand] = useState('comap')   // 'dse' | 'comap' | 'deif'
const [layoutMode, setLayoutMode]           = useState('single')   // 'single' | 'double'
const [witnessedFAT, setWitnessedFAT]       = useState(true)
const [expandedCubicle, setExpandedCubicle] = useState(null)       // strategy id or null
```

---

## 5. Cost model — three tiers

All costs are supply-only. Installed cost = supply × `A-ELEC-23` (1.45 install factor). Both are shown in the modal; only installed feeds the TCO line by default with a toggle to switch to supply.

### 5.1 The governing equation

All sizing starts from rated current:

```
ratedCurrentA(genKWe) = genKWe × 1.804
```

This constant encodes: `1000 / (√3 × 400V × 0.8pf)`. See `A-ELEC-01`, `A-ELEC-02`.

### 5.2 Tier 1 — Per-generator cubicle

One cubicle per generator. Each cubicle contains everything for that generator. Costs scale directly with fleet size.

#### ACB frame selection and cost

```
acbSpecA = ratedCurrentA × 1.25                 // A-ELEC-09: 125% of FLA
acbFrame  = smallest standard frame ≥ acbSpecA
```

| ACB frame A | Generator kWe approx. | Supply cost (mid) | Cliff flag |
|---|---|---|---|
| ≤ 630 | < 280 kWe | $3,200 | — |
| 800 | ~280–350 | $3,800 | — |
| 1,000 | ~350–445 | $4,400 | — |
| 1,250 | ~445–555 | $6,450 | — |
| 1,600 | ~555–710 | $8,200 | — |
| **2,000** | **~710–890** | **$16,500** | **⚑ Frame cliff** |
| 2,500 | ~890–1,110 | $22,000 | ⚑ Frame cliff |
| 3,200 | ~1,110–1,420 | $31,500 | ⚑ Frame cliff |
| 4,000 | ~1,420–1,780 | $39,000 | ⚑ Frame cliff |

Frame cliff flag fires when `acbSpecA > 1,600A`. See `A-ELEC-09`, `A-ELEC-20`.  
All ACBs: 4-pole, draw-out type. See `A-ELEC-10`.

#### Cable class selection

```
singleRunCeilingA = 529                          // A-ELEC-13: derated 400mm² at 35°C, Method F, single circuit
parallelRuns = ratedCurrentA > singleRunCeilingA // true when genKWe ≳ 293 kWe
```

| Condition | Cable class | Description |
|---|---|---|
| `ratedCurrentA ≤ 529` | Single run | One 5-conductor set (3P + N + E) per generator |
| `ratedCurrentA > 529` | Parallel runs | Two or more 5-conductor sets — cost multiplies |

Number of parallel runs: `ceiling(ratedCurrentA / 529)`.

#### Average cable run length by layout mode

```
// Single row: generators distribute uniformly from near-MDB to far end
// Average straight-line distance is half the hall length. Entry zone applies to all.
avgRunM_single = (hallLengthM / 2) + 3.0         // A-ELEC-30

// Double row: front row average same as single row (shorter hall).
// Back row must traverse the full aisle; combined average adds half the aisle width.
avgRunM_double = (hallLengthM / 2) + (aisleWidthM / 2) + 3.0    // A-ELEC-31

// Routing overhead applied on top — accounts for bends, vertical drops, deviations (A-ELEC-29)
effectiveRunM = avgRunM × 1.35                   // A-ELEC-39
```

Hall length is derived from the geometry already computed in `calcPowerhouseGeometry()` for the relevant strategy — the `rH` dimension (engine hall interior length). Aisle width for double row is `p.rowGap` from the clearance preset. These values are available without any new data interface.

#### Cable cost

```
crossSection = selectConductor(ratedCurrentA)    // 95 / 120 / 150 / 185 / 240 / 300 / 400 mm²
cableRatePerM = cableCostTable[crossSection]     // $/m per 5-conductor set, Maldives-landed

cableCostPerGen = parallelRuns × cableRatePerM × effectiveRunM
```

Cable cost table (5-conductor set, Cu XLPE, Maldives-landed):

| mm² | $/m (5-cable set) |
|---|---|
| 95 | 185 |
| 120 | 230 |
| 150 | 285 |
| 185 | 360 |
| 240 | 460 |
| 300 | 575 |
| 400 | 710 |

See `A-ELEC-24` for copper spot price basis.

#### 400mm² practical substitution (A-ELEC-36)

When `selectConductor` returns `crossSectionMm2 === 400 && parallelRuns === 1`, the flag `practicalSplitRecommended` is set. 400mm² Cu XLPE is technically within capacity but impractical on a Maldives island site — minimum bend radius ~1.0–1.2m, weight ~7–8 kg/m per 5-conductor set, and specialist termination tooling typically unavailable.

The cost model substitutes 2× 240mm² (derated capacity 2 × 400A = 800A, exceeds 529A):
- `effectiveParallelRuns = 2`, `effectiveRatePerMSet = 460` (used for cable cost only)
- Returned `crossSectionMm2` and `parallelRuns` still reflect the technically-selected values — trench depth and ACB sizing are unaffected.

The cable class summary row shows `"2× 240mm² (practical)"` with a note. The expanded cubicle detail shows the substitution label and a blue ⓘ callout.

#### Trench class

```
trenchDepthMm = parallelRuns === 1 ? 450 : 600   // 450mm single run, 600mm parallel runs
```

Trench depth is the only spatial output for cable. It feeds the trench shading label in the Powerhouse Modal (§9).

#### Per-cubicle fixed items

| Item | Cost (mid, supply) |
|---|---|
| CTs — 3 × Class 1, FLA-rated primary, 5A secondary | $720 |
| Cubicle enclosure — withdrawable, IP42, tropical finish, width per ACB frame | $3,200 |
| Busbar riser — Cu stub from ACB to main bus | $490 |
| Controller module — see §7 for brand/tier | Variable |
| CT secondary wiring + terminals | $300 |
| kWh metering | $270 |
| Anti-condensation heater | $115 |
| Cable gland plate | $200 |
| Control cable (local AMF panel → MDB), included in trench run | $23/m × effectiveRunM |

Cubicle enclosure width by ACB frame (drives panel length, shipping tier, room sizing):

| ACB frame A | Cubicle width |
|---|---|
| ≤ 1,600 | 600 mm |
| 2,000–2,500 | 700 mm |
| 3,200–4,000 | 800 mm |

#### Per-generator cubicle total (supply)

```
perGenSupply = acbCost
             + ctCost
             + cubicleEnclosure
             + busbarRiser
             + controllerModule        // §7
             + fixedItems              // wiring, metering, heater, gland plate
             + cableCostPerGen
             + controlCableCost
```

### 5.3 Tier 2 — Main distribution (one per switchboard)

#### Main distribution ACB

```
totalFleetFLA = sum(ratedCurrentA(genKWe) for all generators in fleet)
mainACBSpecA  = totalFleetFLA            // no additional spec factor — full fleet is already worst case
mainACBFrame  = smallest standard frame ≥ mainACBSpecA
mainACBCost   = acbCostTable[mainACBFrame]
```

The UAM regulatory minimum is 125% of design load. The tool sizes to the full installed fleet — always equal to or greater than the regulatory minimum, and providing headroom for power creep across the TCO horizon. This is communicated to the user in the main distribution cubicle summary.

#### Bus coupler ACB (split bus only — fleet ≥ 5)

```
splitBus = fleetSize >= 5                        // A-ELEC-15, A-FP-09

if splitBus:
  generatorsPerSection = ceil(fleetSize / 2)
  sectionFLA = generatorsPerSection × ratedCurrentA
  couplerSpecA = sectionFLA × 1.25              // A-ELEC-17
  couplerFrame = smallest standard frame ≥ couplerSpecA
  couplerACBCost = acbCostTable[couplerFrame]
  busTieCubicle  = 5,000                        // enclosure, partial busbar, gland plate (mid)
  couplerProgramming = 3,000                    // controller interlock and sync (mid)
  splitBusTotalCost = couplerACBCost + busTieCubicle + couplerProgramming
```

#### Busbar assembly

```
barsPerPhase = ceiling(totalFleetFLA / 1000)    // A-ELEC-18: 1,000A per 100×10mm Cu bar per phase

panelLengthM = (fleetSize × cubicleWidthM)
             + 0.600                            // main distribution cubicle (fixed 600mm)
             + (splitBus ? 0.700 : 0)           // bus-tie cubicle (700mm mid)

busbarCostPerM = busCostTable[barsPerPhase]
busbarCost = busbarCostPerM × panelLengthM
```

Busbar cost table (fabricated and installed in shop, supply):

| Bars/phase | $/m (mid) |
|---|---|
| 1 | 3,100 |
| 2 | 4,700 |
| 3 | 6,500 |
| 4 | 8,300 |
| 5 | 10,100 |
| 6 | 11,900 |

#### Neutral busbar (A-ELEC-37, A-ELEC-38)

The system is 4-wire TN-S throughout. 4-pole ACBs are mandatory. The neutral bar is a fabrication cost item — sized at 60% of the phase bar cross-section, one bar, full panel length.

```
neutralBarCostPerM = round(3100 × 0.60)  // $1,860/m — 60% of 1-bar/phase rate
neutralBarCost     = neutralBarCostPerM × panelLengthM
```

Displayed as a line item in the Main distribution section: `"Neutral busbar (1 bar, full panel length)"`.

#### Shipping tier (qualitative — not a cost)

Three-tier shipping classification:

| Condition | Tier | Label | Note |
|---|---|---|---|
| `panelLengthM ≤ 5.5` | `20ft` | `1 × 20ft container` | — |
| `5.5 < panelLengthM ≤ 11.5` | `40ft` | `1 × 40ft container` | Panel requires 40ft container — standard option, no site jointing |
| `panelLengthM > 11.5` | `2-sections` | `2 sections + on-site jointing` | Two sections required — 1–2 days on-site busbar jointing |

Thresholds: 20ft ISO usable ~5.9m (5.5m with clearance, A-ELEC-40); 40ft ISO usable ~12.0m (11.5m with clearance, A-ELEC-41).

Displayed as a logistics note. No freight cost is calculated.

#### Tier 2 total (supply)

```
tier2Supply = mainACBCost
            + mainDistribCubicle      // 3,200 enclosure (same as per-gen)
            + busbarCost
            + neutralBarCost          // A-ELEC-37
            + (splitBus ? splitBusTotalCost : 0)
```

### 5.4 Tier 3 — Site-level (one per powerhouse, shared across strategies)

#### Earthing and surge protection

Tiered by fleet size. Values are supply + installation combined (civil work, not separable).

| Fleet size | Earthing + SPDs (mid) |
|---|---|
| 2–3 generators | $6,500 |
| 4–5 generators | $8,000 |
| 6–8 generators | $10,000 |

See `A-ELEC-05` for ambient basis. Coral-fill geology may require additional rods — disclosed as a footnote, not modelled as a variable.

#### Factory acceptance testing (FAT)

| Configuration | Without witness | With witness |
|---|---|---|
| 2–4 generator board | $5,000 | $10,000 |
| 5–8 generator board (split bus) | $7,500 | $14,000 |

Witnessed FAT is strongly recommended for Maldives projects. The cost of defect discovery at an island site — reverse freight, re-fabrication, re-freight — exceeds witness travel cost by a large margin.

#### Site master controller

One per site. Cost by brand and fleet size — see §7.

#### Hybrid integration layer

Fires when `bessActive || solarActive`. Cost by brand — see §7.

#### Tier 3 total (supply)

```
tier3Supply = earthingSPDs
            + fatCost
            + siteMasterCost           // §7
            + hybridIntegrationCost    // §7, 0 if not active
```

### 5.5 Grand total per strategy

```
grandSupply    = (perGenSupply × fleetSize) + tier2Supply + tier3Supply
grandInstalled = grandSupply × 1.45          // A-ELEC-23
```

The installed figure feeds the TCO line (§11). Both figures are displayed in the modal summary.

---

## 6. Flag trigger conditions

Three flags are evaluated per strategy. Each flag renders as a named callout in the strategy column — visible immediately, not hidden in an expand section.

### Flag 1 — ACB frame cliff

**Trigger:** `ratedCurrentA × 1.25 > 1,600A` — i.e., genKWe ≳ 710 kWe

**Display:**

> ⚑ **ACB frame step** — This generator's rated current places its ACB in the high-cost frame tier (above 1,600A specification). Each ACB costs approximately $[acbCost] — compared to $[standardTierMid] at standard frame. This step is a consequence of generator size, not fleet count.

**Rationale:** The ACB frame boundary at 1,600A is the largest cost cliff in the per-generator scope. It is discontinuous — a 720 kWe generator and an 880 kWe generator are not proportionally priced; the latter costs roughly double per ACB. This is the electrical cost counterpressure to the fuel economy argument for larger generators.

### Flag 2 — Split bus

**Trigger:** `fleetSize >= 5`

**Display:**

> ⚑ **Split bus required** — Five or more generators on a single bus produces prospective short-circuit current approaching advisory limits. The bus is divided into two sections with a normally-open coupler ACB. This adds one ACB, one cubicle section, and controller interlock programming — approximately $[splitBusTotalCost] additional.

**Rationale:** The PSCC advisory threshold of 30 kA (`A-ELEC-16`) is reached at 5 generators for typical Maldives resort generator sizes. Split bus is the standard engineering response. The cost step fires once, at fleet size 5, regardless of generator kWe.

### Flag 3 — Two-section panel

**Trigger:** `panelLengthM > 11.5`

**Display:**

> ⚑ **Two-section panel** — Panel length [panelLengthM]m exceeds 40ft container capacity (11.5m usable). The switchboard must be fabricated and shipped in two sections. On-site busbar jointing is required at installation (typically 1–2 additional days).

**Rationale:** Panel section count is a logistics consequence, not a cost variable in this tool. The two-section note prepares the user for the installation implication without inflating the cost estimate with a freight calculation.

---

## 7. Controller brand model

### 7.1 Brand selection

The user selects one brand for all three strategies. Selection is made in the left sidebar. Three options:

| Brand | Positioning for island microgrid |
|---|---|
| **DSE (Deep Sea Electronics)** | Widest global deployment. Strongest for genset-only fleets. Native hybrid dispatch not supported — requires a phasing plan when BESS or solar active dispatch is required. |
| **ComAp** | Native hybrid microgrid stack from day one. InteliNeo 6000 manages gensets, BESS, and solar in a unified control layer. Recommended for projects where BESS/solar timeline is firm. |
| **DEIF** | Premium reliability architecture. Multi-master — no single control point of failure; any controller can assume the master role. Best for large fleets and clients where unplanned downtime is the dominant risk. |

### 7.2 Per-generator controller module cost

| Fleet size | DSE module | ComAp module | DEIF module |
|---|---|---|---|
| 1 generator | DSE 7320 — $550 | InteliGen NTC — $675 | AGC-4 MK II — $900 |
| 2–4 generators | DSE 7420 MKII — $1,100 | InteliGen 1000 — $3,650 (mid) | AGC-4 MK II — $2,200 (mid) |
| 5–8 generators | DSE 8610 MKII — $1,500 | InteliGen 1000 — $3,650 (mid) | AGC-4 MK II — $2,200 (mid) |

Per-generator module cost is included as a line item in the per-gen cubicle (Tier 1).

### 7.3 Site master controller cost

| Fleet size | DSE master | ComAp master | DEIF master |
|---|---|---|---|
| 2–4 generators | DSE 8610 MKII — $4,600 | InteliNeo 5500 — $5,000 (mid) | AGC 150 Master — $4,500 (mid) |
| 5–8 generators | DSE 8920 — $6,400 | InteliNeo 6000 — $7,750 (mid) | AGC 150 Master + EPM — $6,500 (mid) |

Site master is a Tier 3 cost — one per site, shown in each strategy column for correct TCO comparison, flagged as `(shared — one per site)`.

**UI label is tier-aware.** The label shown beneath the site master cost must reflect the actual product for the fleet size, not a single hard-coded name. Product name lookup:

| Fleet size | DSE master label | ComAp master label | DEIF master label |
|---|---|---|---|
| 2–4 | DSE 8610 MKII | ComAp InteliNeo 5500 | DEIF AGC 150 Master |
| 5–8 | DSE 8920 | ComAp InteliNeo 6000 | DEIF AGC 150 Master + EPM |

Resolved in `renderColumn` as: `SITE_MASTER_LABELS[brand][fleetSize >= 5 ? 'high' : 'low']`.

### 7.4 Hybrid integration layer

Fires when `bessActive || solarActive`.

| Brand | What is required | Cost (supply, mid) |
|---|---|---|
| **ComAp** | InteliNeo 530 BESS controller | $3,150 |
| **DEIF** | ASC-4 Battery controller | $4,500 |
| **DSE** | Third-party EMS overlay or future controller upgrade — see phasing narrative below | $20,000 (integration budget, mid) |

ComAp and DEIF costs are discrete hardware items. DSE cost is a planning budget for a Phase 2 EMS integration — included to make the total comparable.

### 7.5 DSE phasing narrative

Rendered as a named section in the left sidebar, below the brand selection, when `controllerBrand === 'dse' && (bessActive || solarActive)`.

---

**DSE — Hybrid phasing**

DSE is selected and BESS or solar is active in this study. DSE controllers do not natively coordinate hybrid dispatch between generators, BESS, and solar. Two paths exist:

**Path 1 — Phase the hybrid installation.**
Commission the powerhouse today with DSE. BESS and solar are installed in a defined second phase, at which point a third-party EMS overlay is added or the DSE controllers are upgraded to a hybrid-capable stack. The powerhouse is operational immediately. The hybrid integration timeline is commercially flexible.

**Path 2 — Upgrade brand now.**
Select ComAp or DEIF — native hybrid from day one.

A hybrid integration budget of $20,000 is included in the cost estimates below. This represents Path 1 (third-party EMS overlay). Path 2 cost is shown in the ComAp and DEIF estimates for direct comparison.

All three stacks are proven. The choice is a matter of timeline, commercial structure, and existing on-island technical familiarity with the product.

---

### 7.6 Capability matrix (rendered in modal)

| Capability | DSE | ComAp | DEIF |
|---|---|---|---|
| Multi-generator paralleling | ✓ | ✓ | ✓ |
| Auto load sharing and sequencing | ✓ | ✓ | ✓ |
| Fault management and protection | ✓ | ✓ | ✓ |
| Native BESS dispatch coordination | ✗ | ✓ | ✓ |
| Native solar active dispatch | ✗ | ✓ | ✓ |
| Single point of failure in control layer | Yes | Yes | No — multi-master |
| Mains / grid-parallel supervision | N/A | N/A | N/A |

Mains supervision is marked N/A for all brands — not applicable to island microgrid operation.

---

## 8. UI specification

### 8.1 Shell

The modal follows the standard shell pattern used by `PowerhouseModal` and `FuelSystemModal`:

```
Fixed backdrop (dark overlay, z-50)
└── White rounded card (max-w-6xl, max-h-[90vh], overflow-hidden)
    ├── Slate-800 header bar
    │   ├── Title: "Electrical Infrastructure"
    │   ├── Subtitle: "Control architecture and infrastructure cost by fleet strategy"
    │   └── Close button (×)
    │
    ├── Left sidebar (w-72, overflow-y-auto, border-r)
    │   └── See §8.2
    │
    └── Right panel (flex-1, overflow-y-auto)
        └── See §8.3
```

### 8.2 Left sidebar contents

**Section 1 — Control Architecture**

Radio group: `DSE | ComAp | DEIF`

Below the radio group: capability matrix (§7.6) — compact table, always visible.

Below the matrix: DSE phasing narrative (§7.5) — visible only when DSE selected and `bessActive || solarActive`. Rendered in an amber-tinted card with no alarmist language — factual, commercial, neutral in tone.

**Section 2 — Layout Mode**

Radio group: `Single Row | Double Row`

Helper text: "Determines cable run length estimate. Select the layout you are costing — this may differ from the layout explored in the Powerhouse Visualiser."

**Section 3 — FAT**

Toggle: `Witnessed FAT` (default: on)

Helper text: "Witnessed FAT strongly recommended for island projects. Discovery of defects on-site costs significantly more than witness travel."

**Section 4 — Note on estimates**

Fixed text block:

> All figures are planning-grade estimates (±25–35%). Prices reflect Maldives-landed supply costs, April 2026. Cable costs track copper spot with approximately 3-week lag. Verify against current distributor quotations before committing to a budget.

### 8.3 Right panel contents

Three columns: Strategy A | Strategy B | Strategy C.

Column header shows strategy label and fleet descriptor: `"4 × 500 kWe"`.

**Block 1 — Infrastructure tier summary (at a glance)**

Four rows, each with a label, value, and brief driver note:

| Row | Value | Driver note |
|---|---|---|
| Bus architecture | Single Bus / Split Bus | "Fleet ≥ 5 triggers split bus" (only shown when split bus) |
| ACB tier | Standard frame / High-cost frame | "Generator size exceeds 1,600A spec threshold" (only shown when cliff) |
| Cable class | Single run / Parallel runs | "Generator FLA exceeds single 400mm² derated capacity" (only shown when parallel) |
| Shipping | 1 container / 2 containers | "Panel [Xm] — requires on-site jointing" (only shown when 2-section) |

**Block 2 — Flags**

Renders only the flags that are triggered for this strategy. Each flag is a named amber callout as specified in §6. If no flags: block is absent (do not render an empty block).

**Block 3 — Cost estimate (collapsed per-cubicle, expandable)**

```
Per-generator cubicle (× fleet size)
  [+] Expand for line items:
      ACB [frame]A             $X,XXX
      Cable ([class], [mm²])   $X,XXX
      Controller module         $X,XXX
      CTs + metering + misc     $X,XXX
      Cubicle enclosure         $X,XXX
  Subtotal per cubicle:         $X,XXX
  × [fleetSize] generators  =  $XX,XXX

Main distribution
  Main ACB [frame]A             $X,XXX
  Busbar ([bars]/phase, [Xm])   $X,XXX
  Main distribution cubicle     $X,XXX
  [Split bus coupler]           $X,XXX   ← only if triggered
  Subtotal:                    $XX,XXX

Site-level  (shared — one per site)
  Site master controller        $X,XXX
  [Hybrid integration]          $X,XXX   ← only if bessActive || solarActive
  Earthing + SPDs               $X,XXX
  FAT ([witnessed/standard])    $X,XXX
  Subtotal:                    $XX,XXX

──────────────────────────────────────
Supply total:               $XXX,XXX
Installed total (×1.45):    $XXX,XXX
```

**Block 4 — TCO integration line**

```
Electrical CapEx added to TCO:  $XXX,XXX  (installed)
                                 $XXX,XXX  (supply)
Using: [Installed ▾]            [toggle]
```

Default: installed cost feeds TCO. Toggle switches to supply. `onElecCapExChange` fires on render and on toggle change.

### 8.4 Interaction behaviour

- Modal computes all three strategies on mount. No "Calculate" button — results are shown immediately.
- Changing `controllerBrand` or `layoutMode` in the sidebar triggers instant recomputation of all three strategy columns.
- Changing `witnessedFAT` triggers instant recomputation of FAT line only.
- Expanding a cubicle detail section (`expandedCubicle` state) does not trigger recomputation — it only reveals pre-computed line items.
- Closing the modal does not persist any state — next open starts fresh.

---

## 9. Powerhouse modal change — trench shading only

This replaces Parts A, B, and C of `POWERHOUSE_LAYOUT_ELECTRICAL_SPEC.md` in their entirety. No changes to `calcPowerhouseGeometry()` return shape. No cable route geometry computation. No `runLengths` array.

### What changes in `PowerhouseModal.jsx`

A single SVG overlay is added to the existing floor plan render: a shaded cable trench zone.

**Single row layout:**
A shaded rectangle running along the full length of the generator row on the alternator side (the end opposite the radiator/exhaust). Width: 0.4m (drawn to scale). Colour: semi-transparent slate (e.g., `rgba(71, 85, 105, 0.25)`). Extends from the first generator to the MDB room wall.

**Double row layout:**
A shaded rectangle running down the central aisle between the two rows, full hall length. Same width and colour treatment.

**Label:**
One callout label attached to the trench zone: `"Cable trench — [depth]mm deep"`, where depth is `parallelRuns === 1 ? 450 : 600`. The parallel runs determination uses the strategy's generator kWe with the same formula as the electrical modal (§5.2).

**Implementation note:**
The trench depth calculation requires `ratedCurrentA(genKWe)` for the strategy being displayed. This is a single multiplication — `genKWe × 1.804` — computable inline in the modal render with no new data interface. No new props are needed.

### What does NOT change

- `calcPowerhouseGeometry()` return shape — unchanged
- MDB room representation — remains as current ("MICROGRID CONTROL" panel strip)
- `PowerhouseFootprint.jsx` — unchanged
- All existing `crW`, `crH`, `crY` fields — unchanged

---

## 10. `app.jsx` integration

### 10.1 New state variables

```javascript
const [showElecModal, setShowElecModal]   = useState(false)
const [elecCapExA, setElecCapExA]         = useState(null)  // { supply, installed } | null
const [elecCapExB, setElecCapExB]         = useState(null)
const [elecCapExC, setElecCapExC]         = useState(null)
```

### 10.2 Entry point button

In the same button cluster as the existing "Powerhouse Layout" and "Fuel System" buttons:

```javascript
<button
  onClick={() => setShowElecModal(true)}
  className="flex items-center gap-1.5 text-xs font-semibold px-3 py-1.5 bg-slate-800 text-white rounded-lg hover:bg-slate-700"
>
  <Zap size={12} />
  Electrical CapEx
</button>
```

### 10.3 Modal render

```javascript
{showElecModal && (
  <ElectricalInfraModal
    strategies={[
      { id: 'A', label: stratA.label, genKWe: specA.maxKw, fleetSize: stratA.fleetSize },
      { id: 'B', label: stratB.label, genKWe: specB.maxKw, fleetSize: stratB.fleetSize },
      { id: 'C', label: stratC.label, genKWe: specC.maxKw, fleetSize: stratC.fleetSize },
    ]}
    bessActive={bessCapacityKwh > 0}
    solarActive={solarKwp > 0}
    onClose={() => setShowElecModal(false)}
    onElecCapExChange={(stratId, supply, installed) => {
      const val = { supply, installed }
      if (stratId === 'A') setElecCapExA(val)
      if (stratId === 'B') setElecCapExB(val)
      if (stratId === 'C') setElecCapExC(val)
    }}
  />
)}
```

### 10.4 Staleness clearing

When any fleet configuration changes (generator model, fleet size, BESS capacity, solar kWp), clear all three electrical CapEx values. This prevents stale numbers from appearing in the TCO.

```javascript
// Call this wherever stratA/B/C fleet parameters are updated
const clearElecCapEx = () => {
  setElecCapExA(null)
  setElecCapExB(null)
  setElecCapExC(null)
}
```

Trigger `clearElecCapEx` on changes to: `stratA.genModel`, `stratA.fleetSize`, `stratB.genModel`, `stratB.fleetSize`, `stratC.genModel`, `stratC.fleetSize`, `bessCapacityKwh`, `solarKwp`.

---

## 11. TCO integration

### 11.1 TCO formula addition

```javascript
const elecVal = (capEx, useSupply) =>
  capEx ? (useSupply ? capEx.supply : capEx.installed) : 0

const tcoA = gensetCapExA + solarCapEx + bessCapEx + tenYrNetOpExA
           + elecVal(elecCapExA, elecUseSupply)

const tcoB = gensetCapExB + solarCapEx + bessCapEx + tenYrNetOpExB
           + elecVal(elecCapExB, elecUseSupply)

const tcoC = gensetCapExC + solarCapEx + bessCapEx + tenYrNetOpExC
           + elecVal(elecCapExC, elecUseSupply)
```

`elecUseSupply` is set by the toggle in the electrical modal (default: false — use installed).

### 11.2 TCO table row

The electrical CapEx row appears in the TCO table in the same format as the BESS and solar CapEx rows. It appears only when at least one strategy has a non-null value. When a strategy value is null (modal not yet opened, or fleet changed after last computation), that column shows `—` with a note: *"Open Electrical CapEx to compute."*

The row label: `"Electrical CapEx"` with a `[estimated]` badge — same visual pattern as the solar CapEx row. No toggle to `[quote-based]` — electrical infrastructure always exists and always falls in the estimated category for planning-grade purposes.

### 11.3 All-three rule

**The TCO comparison is most coherent when all three strategies have been computed.** If only one or two strategies have electrical CapEx values, the TCO comparison is unequal. The TCO table should display a soft warning when electrical CapEx is partially populated: *"Electrical CapEx is only included for [A/B] — open Electrical CapEx to compute all strategies."*

---

## 12. Recomputation behaviour

| Event | What happens |
|---|---|
| User opens electrical modal | Full recomputation for all three strategies on mount. No cached state from previous open. |
| User changes brand in modal sidebar | Instant recomputation — all three strategy cost columns update. |
| User changes layout mode in modal sidebar | Instant recomputation — cable cost and trench depth update for all strategies. |
| User changes FAT witness toggle | FAT line items update. Grand total updates. |
| User closes modal | All local modal state is discarded. `app.jsx` retains the last computed `elecCapEx{A,B,C}` values. |
| User changes fleet configuration in main sidebar | `clearElecCapEx()` fires. All three `elecCapEx` values set to null. TCO table shows `—` for electrical row. |
| User changes BESS capacity or solar kWp in main sidebar | `clearElecCapEx()` fires — hybrid integration costs would change. |

---

## 13. New assumptions ledger entries

The following entries should be added to `docs/ASSUMPTIONS.md`. Electrical constants without `A-ELEC-xx` IDs that were already in `ELECTRICAL_INFRASTRUCTURE.md §18` are not duplicated here — they remain in that document.

### Electrical — new entries for this modal

| ID | Assumption | Value | File | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-ELEC-29 | Cable routing overhead factor | 1.35 × straight-line overspec | `electrical.js` | Overspec for planning grade — accounts for bends, vertical drops, and as-built deviations. Higher than the 1.25× in the original spec to reflect typical Maldives installation experience. | M | 2026-04-20 |
| A-ELEC-30 | Worst-case single-row run length | Hall interior length + 3.0m cable entry | `electrical.js` | 3.0m cable entry zone at MDB end is conservative — allows for gland plate, routing, and dress. | M | 2026-04-20 |
| A-ELEC-31 | Worst-case double-row run length | Hall interior length + aisle width + 3.0m | `electrical.js` | Furthest generator in back row must traverse the aisle before reaching the MDB end. | M | 2026-04-20 |
| A-ELEC-32 | Trench depth — single run | 450 mm | `electrical.js` | Standard minimum for single cable set; allows 75mm sand bed, 75mm marker tape, 300mm burial depth. | L | 2026-04-20 |
| A-ELEC-33 | Trench depth — parallel runs | 600 mm | `electrical.js` | Additional depth for multiple cable sets and separation. | L | 2026-04-20 |
| A-ELEC-34 | Main distribution ACB sizing basis | Full installed fleet FLA (no additional factor) | `electrical.js` | Full fleet simultaneously is the worst case — no separate spec factor needed. Exceeds UAM 125% of design load minimum. Provides headroom for power creep across TCO horizon. | H | 2026-04-20 |
| A-ELEC-35 | Install factor | 1.45 × supply | `electrical.js` | Maldives island premium on labour and logistics. See A-ELEC-23 in ELECTRICAL_INFRASTRUCTURE.md. | H | 2026-04-20 |
| A-CTRL-01 | DSE per-gen module cost — 2–4 gen fleet | $1,100 (mid) | `electrical.js` | DSE 7420 MKII, mid distributor pricing Q1 2026. | M | 2026-04-20 |
| A-CTRL-02 | DSE per-gen module cost — 5–8 gen fleet | $1,500 (mid) | `electrical.js` | DSE 8610 MKII module. | M | 2026-04-20 |
| A-CTRL-03 | ComAp per-gen module cost — 2+ gen fleet | $3,650 (mid) | `electrical.js` | InteliGen 1000, mid distributor pricing Q1 2026. | M | 2026-04-20 |
| A-CTRL-04 | DEIF per-gen module cost — 2+ gen fleet | $2,200 (mid) | `electrical.js` | AGC-4 MK II, mid distributor pricing Q1 2026. | M | 2026-04-20 |
| A-CTRL-05 | ComAp site master — 2–4 gen fleet | $5,000 (mid) | `electrical.js` | InteliNeo 5500. | M | 2026-04-20 |
| A-CTRL-06 | ComAp site master — 5–8 gen fleet | $7,750 (mid) | `electrical.js` | InteliNeo 6000. | M | 2026-04-20 |
| A-CTRL-07 | DEIF site master — 2–4 gen fleet | $4,500 (mid) | `electrical.js` | AGC 150 Master. | M | 2026-04-20 |
| A-CTRL-08 | DEIF site master — 5–8 gen fleet | $6,500 (mid) | `electrical.js` | AGC 150 Master + EPM module. | M | 2026-04-20 |
| A-CTRL-09 | ComAp hybrid integration hardware | $3,150 (mid) | `electrical.js` | InteliNeo 530 BESS controller. | M | 2026-04-20 |
| A-CTRL-10 | DEIF hybrid integration hardware | $4,500 (mid) | `electrical.js` | ASC-4 Battery controller. | M | 2026-04-20 |
| A-CTRL-11 | DSE hybrid integration budget | $20,000 (mid) | `electrical.js` | Third-party EMS overlay planning budget for Phase 2 integration. Included to make DSE total comparable to ComAp and DEIF when hybrid is active. Wide range ($12,000–$35,000) reflects scope variability. | H | 2026-04-20 |

---

## 14. What is explicitly out of scope

The following items were considered during design and deliberately excluded. They do not belong in a future version of this modal — they belong in a separate system or a detailed design deliverable.

| Item | Reason for exclusion |
|---|---|
| Precise cable run lengths from geometry | Planning-grade overspec is sufficient; exact routing cannot be known before civil works |
| Per-circuit grouping factor (Cg) override | Best practice is dedicated tray per generator — Cg = 1.00. Override creates false precision without changing the planning-grade conclusion |
| Freight cost calculation | Barge transport occurs regardless of panel size. Qualitative tier is sufficient for the comparison |
| WebSupervisor / DEIF Cloud OpEx | Optional; on-premise monitoring is achievable at comparable cost — not a required fleet-size consequence |
| Mains parallel / utility sync controller features | Island microgrid is the mains — these features do not apply |
| Harmonic filtering | Requires a site power quality study — not estimable from fleet parameters alone |
| On-site commissioning labour | Site-specific, not parametric |
| Earthing geology override | Coral-fill is the Maldives default. Footnote disclosure is sufficient — adding a geology slider creates false precision |
| Quote overrides for electrical line items | All electrical infrastructure must exist. There is no scenario where it is absent or optional — quote overrides are not meaningful at this stage |
| SCADA above controller level | Optional and client-specific |
| Spare parts inventory | Owner decision, not a fleet consequence |
| Genset local AMF panel | Generator scope — the control cable from that panel to the MDB is included in per-cubicle cable cost |
