# MicroGrid Optimiser — Documentation

This folder is the specification reference for the MicroGrid Optimiser. It describes what the tool does, how it does it, and every modelling assumption behind the numbers. It is intended for engineers, sales engineers, and future maintainers who need to plan studies, challenge a constant, or branch the tool for a new use case.

## Documents

| File | What it answers |
|---|---|
| [SPEC.md](SPEC.md) | What the tool does, how the simulation works, how subsystems fit together |
| [ASSUMPTIONS.md](ASSUMPTIONS.md) | Every hard-coded constant and formula with `file:line` back-references |
| [PARAMETERS.md](PARAMETERS.md) | How input parameters interact, with a directional matrix and priority-driven decision trees |
| [WORKFLOWS.md](WORKFLOWS.md) | Five worked examples — how to configure the tool to answer specific customer questions |
| [ELECTRICAL_INFRA_MODAL_SPEC.md](ELECTRICAL_INFRA_MODAL_SPEC.md) | Full specification for the Electrical Infrastructure Modal — cost model, UI, TCO integration, assumptions |

## Reading order

- **New to the tool** → `SPEC.md` → `WORKFLOWS.md`
- **Running a study** → `WORKFLOWS.md` → `PARAMETERS.md`
- **Challenging a number** → `ASSUMPTIONS.md` → cited `file:line`
- **Planning a new branch or feature** → `SPEC.md` §Non-goals → `ASSUMPTIONS.md` §Environmental → `PARAMETERS.md` §Decision trees
- **Understanding electrical infrastructure costs** → `ELECTRICAL_INFRA_MODAL_SPEC.md` → `ELECTRICAL_INFRASTRUCTURE.md`

## Maintenance rule

If you change a numeric constant in `/src` or `app.jsx`, update the matching `A-ID` row in `ASSUMPTIONS.md` in the same PR. See the rule at the top of that file.

