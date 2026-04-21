# MicroGrid Optimiser — Assumptions Ledger

This is the authoritative ledger of every modelling constant and formula. Every row has an ID, a value, a code back-reference, and a sensitivity rating. Studies that draw conclusions from the tool should cite the IDs of the assumptions that drove them — see [WORKFLOWS.md](WORKFLOWS.md).

## Maintenance rule

> **If you change a numeric constant or a modelling formula in `/src` or `app.jsx`, update the matching `A-ID` row in this file in the same PR.** Update the `Value` and `Last challenged` columns. If the rationale is changing, update that too. Without this rule, the ledger rots in a quarter.

## Sensitivity legend

- **H** — changes the ranking of strategies in typical studies
- **M** — changes magnitudes but not usually rankings
- **L** — marginal in normal operating ranges; may matter at extremes

## Dispatch & fuel model

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-DISP-01 | Below 30% load, fuel is clamped to the 30% SFC point | 30% | [dispatch.js:42](../src/utils/dispatch.js#L42) | Protects the model from unphysical extrapolation below the lowest measured SFC point. Drives most "fleet too big" findings. | H | 2026-04-19 |
| A-DISP-02 | Per-unit load cap unless full fleet at 100% | 90% | [dispatch.js:47-51](../src/utils/dispatch.js#L47-L51) | Safety margin and drives N+1 sizing decisions. | H | 2026-04-19 |
| A-DISP-03 | SFC between 25/50/75/100% points is linearly interpolated | Linear | [dispatch.js:14-29](../src/utils/dispatch.js#L14-L29) | Simplification of real SFC curves, which are gently convex. Under-estimates fuel at 40–60% marginally. | M | 2026-04-19 |
| A-DISP-04 | Below 25% load, SFC is linearly scaled from zero | Linear | [dispatch.js:15-17](../src/utils/dispatch.js#L15-L17) | Needed because no data exists below 25%. The 30% floor (A-DISP-01) masks most of this range. | L | 2026-04-19 |
| A-DISP-05 | Dispatch selection criterion is minimum total fuel | Min-fuel | [dispatch.js:57-58](../src/utils/dispatch.js#L57-L58) | Not "fewest units". When the two disagree, fuel wins. | M | 2026-04-19 |
| A-DISP-06 | Fuel colour bands: off / <30 / <50 / ≤85 / ≤95 / >95 | Fixed | [dispatch.js:5-12](../src/utils/dispatch.js#L5-L12) | Visual bands used in the dispatch matrix — "sweet zone" is 50–85%. | L | 2026-04-19 |

## Load shape & occupancy

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-LOAD-01 | Base load scales `0.4 + 0.6 × occupancy` | Linear floor 40% | [app.jsx:149](../app.jsx#L149) | Fixed resort ops (cold storage, lighting, BMS) never drop below 40% even at zero occupancy. | H | 2026-04-19 |
| A-LOAD-02 | Peak load scales `0.2 + 0.8 × occupancy^1.5` | Convex floor 20% | [app.jsx:151](../app.jsx#L151) | Peaks drop disproportionately with occupancy because guest-driven spikes disappear. | H | 2026-04-19 |
| A-LOAD-03 | HW demand scales `max(0.1, occupancy)` | Linear floor 10% | [app.jsx:155](../app.jsx#L155) | Kitchens and staff quarters maintain a floor. | M | 2026-04-19 |
| A-LOAD-04 | Peak is always ≥ base + 50 kW | 50 kW | [app.jsx:152-153](../app.jsx#L152) | Prevents degenerate flat profiles at very low occupancy. | L | 2026-04-19 |
| A-LOAD-05 | Single-peak shape is cosine over 24 h | Cosine | [useSimulation.js:41-43](../src/hooks/useSimulation.js#L41-L43) | Smooth single-mode load; `peakHour` sets position. | M | 2026-04-19 |
| A-LOAD-06 | Dual-peak shape is cosine over 12 h | Cosine | [useSimulation.js:38-39](../src/hooks/useSimulation.js#L38-L39) | Breakfast + dinner resort pattern. | M | 2026-04-19 |
| A-LOAD-07 | Flat shape is constant at 0.5 × range | 0.5 | [useSimulation.js:35](../src/hooks/useSimulation.js#L35) | 24/7 industrial comparator. | L | 2026-04-19 |

## BESS model

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-BESS-01 | LFP: DoD 90%, RTE 94%, $750/kWh, 2.5%/yr degradation | Fixed | [generators.js:749-754](../src/data/generators.js#L749-L754) | Typical container-scale LFP as of 2025. | H | 2026-04-19 |
| A-BESS-02 | NMC: DoD 85%, RTE 92%, $600/kWh, 4.0%/yr degradation | Fixed | [generators.js:755-760](../src/data/generators.js#L755-L760) | Higher energy density, shorter life, lower $/kWh. | H | 2026-04-19 |
| A-BESS-03 | SoC bounds 10%–95% of usable capacity | 10% / 95% | [app.jsx:173-174](../app.jsx#L173-L174) | Protects against deep-discharge damage and overcharge. | M | 2026-04-19 |
| A-BESS-04 | RTE applied symmetrically: `oneWayEff = √rte` | Symmetric | [dispatch.js:85](../src/utils/dispatch.js#L85) | Modelling simplification — real asymmetries folded into one number. | M | 2026-04-19 |
| A-BESS-05 | Load Levelling targets genset band 65–75% | 65% / 75% | [dispatch.js:109-118](../src/utils/dispatch.js#L109-L118) | Sweet zone midpoint; leaves headroom to the 85% trigger. | H | 2026-04-19 |
| A-BESS-06 | Load Levelling charge trigger < 50%, discharge trigger > 85% | 50% / 85% | [dispatch.js:109,115](../src/utils/dispatch.js#L109) | Dead-band wide enough to avoid cycling around transitions. | M | 2026-04-19 |
| A-BESS-07 | Peak Shave charge trigger is 70% of threshold | 70% | [dispatch.js:133](../src/utils/dispatch.js#L133) | Gives enough runway for the next peak event. | M | 2026-04-19 |
| A-BESS-08 | Universal safety clamp reduces charge to keep gensets ≤ 88% | 88% | [dispatch.js:155](../src/utils/dispatch.js#L155) | 2-point margin below 90% cap (A-DISP-02). | M | 2026-04-19 |
| A-BESS-09 | BESS dispatch uses Strategy B as dispatch preview | Strategy B | [useSimulation.js:68-74](../src/hooks/useSimulation.js#L68-L74) | Single BESS profile across the three strategies for comparability. Distorts when A/B/C shapes diverge. | H | 2026-04-19 |
| A-BESS-10 | Initial SoC by mode: Solar Buffer min, Load Levelling 50%, Peak Shave 80%, Reserve max | Mode-specific | [useSimulation.js:22-27](../src/hooks/useSimulation.js#L22-L27) | Aligns opening state to the mode's intent — first hour is not a transient. | L | 2026-04-19 |
| A-BESS-11 | O&M default $7.50/kWh/yr | $7.50 | [defaults.js:48](../src/data/defaults.js#L48) | Mid-range container BESS O&M. | M | 2026-04-19 |

## Heat recovery

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-HEAT-01 | Recoverable fraction of genset heat | 70% | [generators.js:763](../src/data/generators.js#L763) | Practical upper bound for jacket+exhaust HX with good design. | H | 2026-04-19 |
| A-HEAT-02 | Water ΔT for hot-water sizing | 40°C | [generators.js:764](../src/data/generators.js#L764) | From ~25°C inlet to ~65°C service. | M | 2026-04-19 |
| A-HEAT-03 | Energy-to-water conversion constant | 860 | [useSimulation.js:50,91](../src/hooks/useSimulation.js#L50) | From specific heat of water: 1 kWh ≈ 860 °C·L. | M | 2026-04-19 |
| A-HEAT-04 | Heat recovery is linear in load | Linear | [dispatch.js:70](../src/utils/dispatch.js#L70) | Real recoverable heat is flatter than fuel; this over-estimates at low load. | M | 2026-04-19 |
| A-HEAT-05 | HX unit CapEx | $4,000 per genset | [app.jsx:319](../app.jsx#L319) | Mid-range jacket HX package. Scales with fleet size — many-small fleets pay more. | H | 2026-04-19 |
| A-HEAT-06 | Recovered heat is capped at period HW demand | Demand-capped | [app.jsx:258-260](../app.jsx#L258-L260) | Surplus heat is wasted (no thermal storage modelled). | M | 2026-04-19 |
| A-HEAT-07 | Electric-offset savings use the 75% SFC point | 75% | [app.jsx:265](../app.jsx#L265) | Approximates the average genset operating point when HW is the driver. | M | 2026-04-19 |
| A-HEAT-08 | Default HW demand 25,000 L/day | 25,000 L | [defaults.js:18](../src/data/defaults.js#L18) | 200-key resort reference. | L | 2026-04-19 |
| A-HEAT-09 | HW hourly profile — normalised 24-h vector peaking 06–09 and 18–23 | Fixed array | [generators.js:768-774](../src/data/generators.js#L768-L774) | Hospitality pattern; sums to 1.0. | M | 2026-04-19 |

## Solar PV

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-SOLAR-01 | Hourly generation is a sine from 06:00 to 18:00 | sin(phase) × kWp | [useSimulation.js:54-60](../src/hooks/useSimulation.js#L54-L60) | Idealised clear-sky tropical profile. No seasonality. | H | 2026-04-19 |
| A-SOLAR-02 | Cloud variance uses `\|sin(hour × 13.37)\|` as pseudo-random noise | Deterministic | [useSimulation.js:57-58](../src/hooks/useSimulation.js#L57) | Reproducible, not a weather model. Scaled by the `solarDrop %` slider. | M | 2026-04-19 |
| A-SOLAR-03 | Peak-sun-hours per year | 1,825 | [app.jsx:531](../app.jsx#L531) | Maldives horizontal irradiance approximation. Needs parametrisation for other climates. | H | 2026-04-19 |
| A-SOLAR-04 | Performance ratio | 0.80 | [app.jsx:529](../app.jsx#L529) | Tropical climate, accounts for soiling, thermal, inverter losses. | M | 2026-04-19 |
| A-SOLAR-05 | Panel life | 25 years | [app.jsx:527](../app.jsx#L527) | Standard mono-c-Si warranty horizon. | L | 2026-04-19 |
| A-SOLAR-06 | Annual degradation | 0.5%/yr linear | [app.jsx:528](../app.jsx#L528) | Standard monocrystalline degradation. | L | 2026-04-19 |
| A-SOLAR-07 | Land footprint per kWp | 5.5 m²/kWp | [app.jsx:453](../app.jsx#L453) | High-efficiency panel assumption; rises for lower-efficiency or ground-mount with spacing. | M | 2026-04-19 |
| A-SOLAR-08 | Default installed cost | $1.80/Wp | [defaults.js:37](../src/data/defaults.js#L37) | 2025 tropical-install reference. User-editable. | M | 2026-04-19 |
| A-SOLAR-09 | Default penetration | 60% of peak load | [defaults.js:34](../src/data/defaults.js#L34) | Compromise default between "no solar" and "oversized". | L | 2026-04-19 |

## Footprint

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-FP-01 | Three clearance presets (Regulatory / Practical / Preferred) | See table | [dispatch.js:175-177](../src/utils/dispatch.js#L175-L177) | Presets encode typical ranges; real sites fall in Practical. | H | 2026-04-19 |
| A-FP-02 | Double-row layout when `fleetSize ≥ 6` or (`≥ 4` and width ≥ 2.5 m) | Rule | [dispatch.js:184-187](../src/utils/dispatch.js#L184-L187) | Otherwise single row. Drives the m² step-change at fleet size 6. | H | 2026-04-19 |
| A-FP-03 | Wall thickness 0.2 m | 0.2 m | [dispatch.js:182](../src/utils/dispatch.js#L182) | Structural minimum. | L | 2026-04-19 |
| A-FP-04 | Control room width 2.5 m, height `max(4 m, l × fleet/5)` | Formula | [dispatch.js:211-212](../src/utils/dispatch.js#L211-L212) | Grows with fleet to fit switchgear. | M | 2026-04-19 |
| A-FP-05 | Footprint is genset-hall only | Scope | [dispatch.js:173-219](../src/utils/dispatch.js#L173-L219) | Does not include fuel tanks, bulk storage, BESS. Those are separate modals. | M | 2026-04-19 |

## TCO & economics

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-TCO-01 | TCO horizon | 10 years | [app.jsx:242](../app.jsx#L242) | Rule-of-thumb hospitality equipment horizon. | H | 2026-04-19 |
| A-TCO-02 | Period multipliers: Day 1, Week 7, Month 30, Year 365 | Fixed | [app.jsx:236-243](../app.jsx#L236-L243) | No seasonality — day is repeated. | M | 2026-04-19 |
| A-TCO-03 | No discounting (NPV not applied) | None | [app.jsx:325-327](../app.jsx#L325-L327) | Comparison number, not lifecycle cost. Ratios between strategies are unaffected by a flat discount. | M | 2026-04-19 |
| A-TCO-04 | No fuel-price escalation | Flat | — | User can re-run with a higher fuel price to test sensitivity. | M | 2026-04-19 |
| A-TCO-05 | Default fuel cost | $1.10/L | [defaults.js:17](../src/data/defaults.js#L17) | Maldives reference. | M | 2026-04-19 |
| A-TCO-06 | Genset O&M not in TCO | Excluded | [app.jsx:325-327](../app.jsx#L325-L327) | Only fuel and HW-savings in OpEx. Favour of fleets with cheaper CapEx/higher O&M. | H | 2026-04-19 |
| A-TCO-07 | Genset replacement not modelled | None | — | Some gensets will not last 10 years — not reflected. | M | 2026-04-19 |

## Environmental

| ID | Assumption | Value | File:Line | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-ENV-01 | No altitude derating | None | — | Gensets use nameplate maxKw at all altitudes. | L for Maldives, H for highland sites |
| A-ENV-02 | No ambient-temperature derating | None | — | Hot climates reduce real output — not reflected. | M |
| A-ENV-03 | No air-quality / salt-mist derating | None | — | Coastal / polluted sites not modelled. | L |
| A-ENV-04 | Single day repeated 365 times | No seasonality | [app.jsx:236-243](../app.jsx#L236-L243) | Annual total = daily × 365. Hides seasonal solar and load swings. | H |

## Electrical infrastructure

| ID | Assumption | Value | File | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
| A-ELEC-29 | Cable routing overhead factor applied to average run length | 1.35 × average straight-line run | `electrical.js` | Overspec for planning grade — accounts for bends, vertical drops, and as-built deviations. Higher than the 1.25× in the original spec to reflect typical Maldives installation experience. | M | 2026-04-21 |
| A-ELEC-30 | Average single-row run length basis | `(hallLengthM / 2) + 3.0m entry zone` | `electrical.js` | Generators distribute uniformly along the row. Average run is half the hall length plus the cable entry zone at the MDB end which all generators share. | M | 2026-04-21 |
| A-ELEC-31 | Average double-row run length basis | `(hallLengthM / 2) + (aisleWidthM / 2) + 3.0m entry zone` | `electrical.js` | Front row average is half the hall. Back row must traverse the full aisle; combined average adds half the aisle width. Cable entry zone applies to all. | M | 2026-04-21 |
| A-ELEC-32 | Trench depth — single run | 450 mm | `electrical.js` | Standard minimum for single cable set; allows 75mm sand bed, 75mm marker tape, 300mm burial depth. | L | 2026-04-20 |
| A-ELEC-33 | Trench depth — parallel runs | 600 mm | `electrical.js` | Additional depth for multiple cable sets and separation. | L | 2026-04-20 |
| A-ELEC-34 | Main distribution ACB sizing basis | Full installed fleet FLA (no additional factor) | `electrical.js` | Full fleet simultaneously is the worst case — no separate spec factor needed. Exceeds UAM 125% of design load minimum. Provides headroom for power creep across TCO horizon. | H | 2026-04-20 |
| A-ELEC-35 | Install factor | 1.45 × supply | `electrical.js` | Maldives island premium on labour and logistics. See A-ELEC-23 in ELECTRICAL_INFRASTRUCTURE.md. | H | 2026-04-20 |
| A-ELEC-36 | 400mm² practical substitution | 2× 240mm² substituted for single 400mm² when capacity-selected | `electrical.js` | 400mm² Cu XLPE bend radius (~1.0–1.2m), weight (~7–8 kg/m per 5-conductor set), and specialist termination requirement make single-run 400mm² impractical on Maldives island sites. 2× 240mm² derated capacity (2 × 400A = 800A) exceeds 400mm² capacity (529A). Cost increases ~30% but installation risk is substantially reduced. | M | 2026-04-21 |
| A-ELEC-37 | Neutral busbar sizing basis | 60% of phase bar cross-section, 1 bar, full panel length | `electrical.js` | Island microgrid with significant single-phase load. Neutral can approach full phase current under unbalanced load. 60% is a conservative planning-grade basis — detailed design may reduce this after load analysis. | L | 2026-04-21 |
| A-ELEC-38 | Neutral bar cost per metre | $1,860/m (mid) — 60% of 1-bar/phase rate ($3,100/m) | `electrical.js` | Proportional to phase bar rate. Single neutral bar for all fleet sizes within the model range. | L | 2026-04-21 |
| A-ELEC-39 | Cable run length basis | Average run × fleet size, with 1.35× routing overhead | `electrical.js` | Worst-case × N overbills total cable quantity by approaching 2×. Generators distribute from near-MDB to far end; average is the correct cost basis for fleet total. Entry zone (3.0m) applies to all generators. | M | 2026-04-21 |
| A-ELEC-40 | 20ft container usable length threshold | 5.5m | `electrical.js` | Standard 20ft ISO container internal length ~5.9m. 5.5m allows for end-door clearance and panel blocking. | L | 2026-04-21 |
| A-ELEC-41 | 40ft container usable length threshold | 11.5m | `electrical.js` | Standard 40ft ISO container internal length ~12.0m. 11.5m allows for end-door clearance and panel blocking. Two-section flag fires only above this threshold. | L | 2026-04-21 |

## Controller

| ID | Assumption | Value | File | Rationale | Sens | Last challenged |
|---|---|---|---|---|---|---|
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
