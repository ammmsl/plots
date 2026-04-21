# MicroGrid Optimiser — Workflows

Five worked examples. Each shows how to configure the three strategies to answer a specific open-ended question, what to watch in the output, and which assumptions drive the finding. The five workflows cover the questions the tool is actually useful for.

Each workflow uses the same skeleton:

- **Question** — the customer-facing question in plain English
- **Setup** — sidebar values for Strategy A, B, C
- **What to watch** — chart, table, or metric and the threshold that matters
- **Expected finding** — directional, not absolute
- **Inflection / breakpoint** — the knob to sweep to find where the answer flips
- **Dominant assumptions** — link back to [ASSUMPTIONS.md](ASSUMPTIONS.md)
- **When this conclusion inverts** — counter-cases that break the story

---

## Workflow 1 — Many-small vs few-large fleet

> *"We have a 2,400 kWe peak and 800 kWe base. Should we buy six 500 kWe units or two 1,500 kWe units?"*

### Setup

| | Strategy A | Strategy B | Strategy C |
|---|---|---|---|
| Genset model | ~500 kWe class (e.g. Volvo TAD1344GE) | ~1,500 kWe class (e.g. Baudouin 12M33G1400) | ~750 kWe class (midpoint) |
| Fleet size | 6 | 2 | 4 |
| Min online | 0 | 0 | 0 |

Load: base 800, peak 2,400, occupancy 100%, single-peak shape at 18:00. Solar off. BESS off. HW off.

### What to watch

1. **FuelChart** — daily fuel L/day for A, B, C.
2. **DispatchMatrix** — for Strategy B (few-large), count hours where units sit below the 30% band (orange cells, [A-DISP-01](ASSUMPTIONS.md#dispatch--fuel-model)).
3. **TCOTable** — 10-year TCO spread.
4. **PowerhouseFootprint modal** — m² at the `Practical` clearance preset.

### Expected finding

At full occupancy on a peaky profile, Strategy B (few-large) typically wins on fuel at peak hours (sweet zone), but A (many-small) wins at low-occupancy hours and minimises time below the 30% floor. TCO rankings depend on whether peaks or troughs dominate the day.

### Inflection

Sweep **occupancy** from 100% → 20%. There is an occupancy threshold — typically 40–60% — at which the 30% floor pushes B's fuel above A's. Find it, and you've found the site profile at which the answer flips.

### Dominant assumptions

- [A-DISP-01](ASSUMPTIONS.md#dispatch--fuel-model) 30% fuel floor
- [A-DISP-02](ASSUMPTIONS.md#dispatch--fuel-model) 90% per-unit cap
- [A-LOAD-01](ASSUMPTIONS.md#load-shape--occupancy) base-load 40% floor
- [A-FP-02](ASSUMPTIONS.md#footprint) double-row layout at ≥6 units

### When this conclusion inverts

- If occupancy sits at 30% year-round, many-small dominates.
- If the site runs 24/7 flat at peak, few-large dominates.
- If footprint is binding, few-large wins regardless of fuel — a single-row small-fleet hall is often smaller than a six-unit double-row hall.

---

## Workflow 2 — Fleet-size breakpoint

> *"At what point does adding another unit stop helping?"*

### Setup

Hold genset model constant. Sweep fleet size across the three strategies:

| | Strategy A | Strategy B | Strategy C |
|---|---|---|---|
| Genset model | same mid-range model (e.g. Volvo TWD1645GE ~400 kWe) | same | same |
| Fleet size | 3 | 5 | 8 |
| Min online | 0 | 0 | 0 |

Run the same load; note metrics. Then rerun with fleet sizes 4 / 6 / 7 to fill in the curve.

### What to watch

1. **Daily fuel L/day** as a function of fleet size — plot mentally.
2. **PowerhouseFootprint modal** — m² jumps at fleet size 6 when layout switches to double-row ([A-FP-02](ASSUMPTIONS.md#footprint)).
3. **TCOTable** — marginal 10-year TCO change per added unit.
4. **MainHeader** — redundancy label (N, N+1, N+2…).

### Expected finding

Fuel saved per added unit decreases fast. There is typically a **knee** — usually at the fleet size that first reaches N+1 on the peak load — after which each unit saves <1% fuel but adds ~30–80 m². At fleet size 6 the layout shifts to double-row and the footprint steps up visibly.

### Inflection

Find the smallest fleet at which all these hold simultaneously:

- N+1 redundancy is satisfied
- Each unit's peak-hour load is in the sweet zone (50–85%)
- Each unit's off-peak load does not fall below 30% (check the DispatchMatrix for orange cells)

That fleet size is the "stop adding units" breakpoint.

### Dominant assumptions

- [A-DISP-01](ASSUMPTIONS.md#dispatch--fuel-model) 30% fuel floor
- [A-DISP-02](ASSUMPTIONS.md#dispatch--fuel-model) 90% per-unit cap
- [A-FP-01](ASSUMPTIONS.md#footprint) clearance preset
- [A-FP-02](ASSUMPTIONS.md#footprint) double-row trigger at ≥6 units
- [A-TCO-06](ASSUMPTIONS.md#tco--economics) genset O&M excluded from TCO

### When this conclusion inverts

- If maintenance windows matter (genset O&M is excluded from the TCO — [A-TCO-06](ASSUMPTIONS.md#tco--economics)), larger fleets with one unit always available for service can beat smaller ones in reality even if the TCO table disagrees.
- If the site's operator will not run above 70%, the effective peak-hour band tightens and the breakpoint moves up.

---

## Workflow 3 — Solar + large gensets friction

> *"We're adding 2 MW of solar. Does our existing fleet of two 1.5 MW gensets work, or do we need to change the fleet?"*

### Setup

| | Strategy A | Strategy B | Strategy C |
|---|---|---|---|
| Genset model | 1,500 kWe class | 1,500 kWe class | 500 kWe class |
| Fleet size | 2 | 2 | 6 |
| Min online | 0 | 0 | 0 |
| BESS | off | 1,000 kWh Solar Buffer | off |

Solar 2,000 kWp, solar drop 20%. Load base 800, peak 2,400, single-peak at 18:00.

### What to watch

1. **DemandChart** — the mid-day dip where solar pushes net load low. For A, count hours where both units would run below 30% (they will — that's the 30% floor at work, [A-DISP-01](ASSUMPTIONS.md#dispatch--fuel-model)).
2. **Solar curtailment** — kWh of solar spilled mid-day in A vs B vs C.
3. **DispatchMatrix** — orange cells clustered around noon for A; fewer for B (BESS absorbs) and C (smaller units step down gracefully).
4. **FuelChart** and **TCOTable** — does the solar payback actually materialise for A?

### Expected finding

A (large gensets + solar, no BESS) shows heavy mid-day curtailment and the 30% floor dominates noon fuel — solar savings are smaller than the nameplate kWh would suggest. B (same gensets + BESS) recovers most of the curtailed solar into evening discharge. C (small gensets + solar) stages units down gracefully and minimises both curtailment and fuel.

### Inflection

Sweep **solar kWp** from 500 → 3,000. The inflection for A is around the point where `solar midday peak ≈ base load`. Beyond that, either the BESS (path B) or smaller gensets (path C) becomes mandatory.

### Dominant assumptions

- [A-DISP-01](ASSUMPTIONS.md#dispatch--fuel-model) 30% fuel floor — this is the whole story
- [A-SOLAR-01](ASSUMPTIONS.md#solar-pv) sine profile — in reality cloudy days give more smooth overlap; the deterministic sine exaggerates the noon cliff
- [A-BESS-09](ASSUMPTIONS.md#bess-model) BESS dispatch uses Strategy B — comparisons here are fair because A and C have similar gensets to B's 1.5 MW
- [A-SOLAR-03](ASSUMPTIONS.md#solar-pv) PSH 1825 — scales the annual savings number

### When this conclusion inverts

- If the site has a large flat daytime load (laundry, desalination), the solar fills genuine demand and the floor is not triggered.
- If fuel is very cheap, the curtailment loss is not worth the BESS CapEx to recover.

---

## Workflow 4 — BESS mode comparison

> *"Which BESS mode earns its keep for our site?"*

BESS modes are mutually exclusive and the tool can only show one at a time, so this workflow requires **two runs** of the same strategy set with different modes.

### Setup (Run 1)

| | Strategy A | Strategy B | Strategy C |
|---|---|---|---|
| Genset model | same for all (mid-range) | same | same |
| Fleet size | same for all | same | same |
| BESS | 1,000 kWh **Solar Buffer** | 1,000 kWh **Load Levelling** | 1,000 kWh **Peak Shave** @ 1,500 kW threshold |

Solar 1,500 kWp, load base 800, peak 2,400, single-peak 18:00.

### Setup (Run 2)

Repeat with A = **Reserve** mode (keep B and C as before or cycle others).

### What to watch

1. **DemandChart** — where the BESS charges and discharges for each mode.
2. **Solar curtailment** — Solar Buffer and Reserve minimise it; Load Levelling and Peak Shave may leave some on the table.
3. **BESS metrics card** — cycles/day and fuel saved/day.
4. **TCOTable** — 10-year TCO difference attributable to the BESS.
5. **DispatchMatrix — Baseline toggle** — shows the same hourly dispatch without BESS for comparison.

### Expected finding (general pattern)

- **Solar Buffer** wins on sites with large solar arrays and predictable evening peaks.
- **Load Levelling** wins on sites with peaky loads near the 85% band — keeps gensets in the sweet zone without needing a user-specified threshold.
- **Peak Shave** wins when a single known peak dominates (e.g. dinner rush) and the rest of the day is quiet. Requires the user to know the threshold.
- **Reserve** is the safest bet for resilience-first sites — lowest cycling, highest availability, but lowest fuel saving.

### Inflection

- **Solar kWp** — below a threshold, Solar Buffer has nothing to buffer and collapses to idle.
- **Peak-to-base ratio** — above ~3:1 Peak Shave starts winning; below it Load Levelling is better.
- **BESS C-rate** — Peak Shave needs enough power to cover the peak; undersized C-rate silently neuters this mode.

### Dominant assumptions

- [A-BESS-01](ASSUMPTIONS.md#bess-model) / [A-BESS-02](ASSUMPTIONS.md#bess-model) chemistry RTE and DoD
- [A-BESS-04](ASSUMPTIONS.md#bess-model) symmetric RTE
- [A-BESS-05](ASSUMPTIONS.md#bess-model) Load Levelling targets 65–75% sweet band
- [A-BESS-09](ASSUMPTIONS.md#bess-model) BESS dispatch against Strategy B — so if A/C fleets differ significantly, the BESS behaviour under those strategies is indicative, not exact

### When this conclusion inverts

- If the fleet is already perfectly sized for the profile, Load Levelling has nothing to level and Solar Buffer wins.
- If the site has regulatory minimum running (min online > 0), Load Levelling can actively hurt by charging while the floor is already active.

---

## Workflow 5 — Heat recovery: large vs many

> *"Hot-water demand is 30,000 L/day. Do fewer large gensets recover more usable heat than many smaller ones?"*

### Setup

| | Strategy A | Strategy B | Strategy C |
|---|---|---|---|
| Genset model | 1,500 kWe class | 500 kWe class | 750 kWe class |
| Fleet size | 2 | 6 | 4 |
| HW recovery | **on** | **on** | **on** |
| Daily HW demand | 30,000 L | 30,000 L | 30,000 L |

Load base 800, peak 2,400, single-peak at 18:00. Solar off to isolate the HW effect.

### What to watch

1. **HotWaterChart** — daily L recovered and daily L utilised (capped at demand, [A-HEAT-06](ASSUMPTIONS.md#heat-recovery)).
2. **FinancialTable** — HW savings column.
3. **TCOTable** — 10-year TCO after the HX unit CapEx ($4,000 × fleetSize, [A-HEAT-05](ASSUMPTIONS.md#heat-recovery)).
4. **DispatchMatrix** — how many hours of the day each fleet is running (heat only comes from running units).

### Expected finding

A (few-large) captures more heat per running hour (higher `heatKwAt100`) but runs fewer units; its HX CapEx is $8,000. B (many-small) runs more HX units ($24,000 total) and captures more total kWh of heat but each unit's HX is under-utilised because the HW demand cap ([A-HEAT-06](ASSUMPTIONS.md#heat-recovery)) binds. Net savings often favour A at typical hospitality HW demands.

### Inflection

Sweep **daily HW demand** from 5,000 → 100,000 L/day. At low demand, A wins (the cap limits both fleets equally, so CapEx wins). At high demand (above A's recovery capacity), B starts winning because its extra HX units earn their CapEx. Find the crossover point for the customer's real HW demand.

### Dominant assumptions

- [A-HEAT-01](ASSUMPTIONS.md#heat-recovery) 70% recovery efficiency
- [A-HEAT-04](ASSUMPTIONS.md#heat-recovery) linear heat-vs-load — overstates heat at low load, so B looks slightly better than reality
- [A-HEAT-05](ASSUMPTIONS.md#heat-recovery) $4,000 HX CapEx per unit — the lever that punishes large fleets
- [A-HEAT-06](ASSUMPTIONS.md#heat-recovery) recovered heat capped at HW demand
- [A-HEAT-07](ASSUMPTIONS.md#heat-recovery) electric-offset savings at 75% SFC

### When this conclusion inverts

- If the site has genuine 24/7 HW demand far above what A's two units can provide when running normally, B's higher total recovery capacity is needed.
- If the gensets in B run much more of the day (e.g. flat profile), B's higher utilisation closes the gap even at modest HW demand.
- If the HX CapEx drops (volume pricing, different HX class), the break-even moves in B's favour.

---

## Running workflows in practice

1. **Snapshot the baseline** — before changing anything, open the Snapshot Modal and save the current configuration. You will re-run and want to compare.
2. **Change one thing** — do not move multiple sliders at once; the whole point of a three-strategy comparison is to isolate the variable under test.
3. **Check the DispatchMatrix colours** — orange cells below 30% are *not* the tool telling you a genset is running inefficiently. They are the tool telling you the 30% fuel floor is clamping the model. The real genset may be running much worse.
4. **Note the dominant assumptions from [ASSUMPTIONS.md](ASSUMPTIONS.md)** — if the customer questions the result, you should be able to cite which three assumptions carried the conclusion and what would happen if they changed.
5. **For resilience studies, toggle Failure Scenario** at the end to check the fleet's behaviour with one unit down.
6. **Open Electrical CapEx after finalising fleet shape** — once the fuel and footprint comparison has identified the leading strategy, open the Electrical CapEx modal to see the infrastructure cost dimension. Select the controller brand that matches the project's BESS and solar timeline. The three-strategy electrical comparison often surfaces an ACB frame cliff or split bus step cost that materially changes the TCO ranking — particularly when comparing few-large vs many-small fleets where the fuel story and the electrical story point in opposite directions.

If a workflow produces a surprising answer, re-read the four forcing functions at the top of [PARAMETERS.md](PARAMETERS.md) — eight times out of ten the surprise traces back to the 30% floor or the 90% cap.
