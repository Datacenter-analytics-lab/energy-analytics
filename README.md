# Data Center Capacity & Energy Analytics

**Tier III–aligned simulated architecture for energy efficiency, infrastructure resilience, and capacity planning.**

> **Disclaimer:** All operational data in this repository are synthetic and generated for analytical demonstration. No confidential operational data are used.

## Executive summary

This project simulates 28 days of operation for a Tier III–aligned data center to answer one recurring operational question: **how much additional IT load can this site safely absorb without breaking N+1 redundancy?** Every part of this repository — the star schema, the thermal methodology, the incident simulation — exists to answer that question with a number, not an opinion. The model is intentionally simplified and does not replace a detailed electrical/mechanical engineering study.

The headline result: the site's original cooling design (2 chillers of 450 kWth) looked comfortably sized on paper but **did not actually hold N+1 redundancy** at peak load. Re-splitting the same installed capacity into 3 chillers of 300 kWth restored it **at the same installed capacity** — no increase in kWth. That correction — and its knock-on effects on PUE, growth capacity, and decision-making — is the throughline of everything below.

> **North Star Metric: Binding Constraint N+1 Headroom**
> The lowest N+1 margin among the three critical subsystems (electrical, UPS, cooling) — whichever one it is, that's the number that actually caps growth and defines "safe." It currently sits at cooling (+74 kWth), but the metric isn't tied to cooling specifically: if cooling capacity were increased further, this North Star would simply track whichever subsystem became the new binding constraint.
>
> | | Original design | Corrected design |
> |---|---|---|
> | Binding constraint | Cooling | Cooling |
> | N+1 headroom at peak | **−76 kWth** | **+74 kWth** |

## At a glance

| | Average | Peak |
|---|---|---|
| IT load | 391 kW | 456 kW |
| Facility power | 586 kW | 682 kW |
| Site thermal load | 455 kWth | 526 kWth |
| Cooling electrical load | 131 kW | 157 kW |

*Average and peak values are each the highest/mean observed independently over the 28-day simulation — the facility power peak and the IT load peak occur at different moments (see [§8](#the-main-finding-cooling-redundancy-before-and-after) and the Peak Energy Balance table in [§7](#key-metrics)), driven by a lower cooling COP at that specific interval, not by higher IT demand. The site thermal load peak (525.6 kWth, rounded to 526) occurs at yet another interval, distinct from both — it is this value, not the ones in the Peak Energy Balance table, that sets the +74.4 kWth cooling N+1 headroom used throughout this document.*

## Table of contents

1. [Executive summary](#executive-summary)
2. [At a glance](#at-a-glance)
3. [Business questions](#business-questions)
4. [Architecture](#architecture)
5. [Assumptions & planning parameters](#assumptions--planning-parameters)
6. [Simulation methodology](#simulation-methodology)
7. [Key metrics](#key-metrics)
8. [The main finding: cooling redundancy, before and after](#the-main-finding-cooling-redundancy-before-and-after)
9. [Capacity planning & decisions](#capacity-planning--decisions)
10. [Recommendations](#recommendations)
11. [Reliability & data governance](#reliability--data-governance)
12. [Incident deep dive](#incident-deep-dive)
13. [Model validation](#model-validation)
14. [What this model does NOT prove](#what-this-model-does-not-prove)
15. [Sample data & sources](#sample-data--sources)

## Business questions

| Category | Question | Answer |
|---|---|---|
| Energy | How efficient is the facility? | PUE **1.50** |
| Capacity | How much capacity remains? | Cooling is limiting, **~74 kWth** headroom at peak |
| Resilience | Can we lose one critical component? | Yes — N+1 maintained on all three pillars |
| Reliability | Did the simulated incidents affect IT service? | No — **100% IT service availability** |
| Growth | Can we add more IT load? | **+50 kW: recommended planning range**. +70 kW: upper simulated limit. +80 kW+: not recommended |
| Optimization | Where should we optimize? | Cooling setpoint, but only *after* protecting the thermal N+1 margin |
| Data quality | Can we trust the data? | Duplicates are detected and quantified; sources reconcile within noise |

## Architecture

Strict star schema. Each fact table represents an independent measurement or simulated telemetry stream — derived engineering quantities (like `FACT_Thermal_Load`, built from IT load + losses + lighting + occupants) are explicitly identified as model outputs, not raw sensor readings. No fact-to-fact relationships: everything correlates through shared dimensions.

**Infrastructure at a glance:**
- **Power:** 2 transformers of 800 kW each, N+1 (either one alone covers the full site load)
- **UPS:** 3 units of 300 kW each, N+1 (2 required, 1 redundant)
- **Cooling:** 3 chillers of 300 kWth each, N+1 (2 required, 1 redundant) — the corrected design, see [§8](#the-main-finding-cooling-redundancy-before-and-after). Cooling medium (chilled water vs. direct expansion) and physical loop topology are **not specified** in the model — it tests capacity, not piping design, which would require a real engineering drawing.

![Star schema](figures/01_star_schema.png)

| Table | Grain | Rows | Role |
|---|---|---|---|
| `FACT_IT_Load` | Timestamp × Room | 5,376 | Single source of truth for IT load |
| `FACT_UPS` | Timestamp × UPS | 8,064 | 3 UPS units, states, losses |
| `FACT_Thermal_Load` | Timestamp | 2,688 | Thermal load broken down by source |
| `FACT_Cooling` | Timestamp × Chiller | 8,064 | 3 chillers, thermal + electrical measurements |
| `FACT_Electrical_Meter` | Timestamp | 2,688 | Facility power at the meter |
| `FACT_Alarms` | Event | 14 | Incidents, planned maintenance, downtime |

**Column-level detail** (analytically relevant columns only — each table carries a few more housekeeping fields):

`FACT_IT_Load`
| Column | Type | Meaning |
|---|---|---|
| Timestamp | datetime | 15-min interval |
| Room_ID | text | Room A / Room B |
| IT_Load_kW | decimal | IT power draw |
| Status | text | Active / etc. |

`FACT_UPS`
| Column | Type | Meaning |
|---|---|---|
| Timestamp | datetime | 15-min interval |
| UPS_ID | text | Equipment identifier |
| Output_Load_kW | decimal | Power delivered to IT |
| UPS_Loss_kW | decimal | No-load + proportional loss |
| Load_Percent | decimal | Output ÷ rated capacity |
| Status | text | Online / Maintenance / Fault / Bypass |

`FACT_Thermal_Load`
| Column | Type | Meaning |
|---|---|---|
| Timestamp | datetime | 15-min interval |
| IT_Thermal_Load_kWth | decimal | IT heat (1:1 from IT load) |
| UPS_Thermal_Load_kWth | decimal | UPS loss as heat |
| Power_Distribution_Thermal_Load_kWth | decimal | Distribution loss as heat |
| Total_Thermal_Load_kWth | decimal | Sum of all thermal sources |

`FACT_Cooling`
| Column | Type | Meaning |
|---|---|---|
| Timestamp | datetime | 15-min interval |
| Cooling_Equipment_ID | text | Chiller identifier |
| Cooling_Power_kW | decimal | Electrical input |
| Cooling_Output_kWth | decimal | Thermal output |
| Efficiency_COP | decimal | Output ÷ electrical input |
| Status | text | Active / Degraded |
| Alarm_Code | text | Linked alarm, if any |

`FACT_Electrical_Meter`
| Column | Type | Meaning |
|---|---|---|
| Timestamp | datetime | 15-min interval |
| Power_kW | decimal | Total facility power |
| PowerFactor | decimal | Active ÷ apparent power |

`FACT_Alarms`
| Column | Type | Meaning |
|---|---|---|
| Alarm_ID | text | Event identifier (1 deliberate duplicate) |
| Start_Timestamp / End_Timestamp | datetime | Event window |
| Equipment_ID | text | Linked equipment |
| Service_Impacting | boolean | Component-level impact flag |
| IT_Outage | boolean | Actual IT service interruption (always False in this run) |

## Assumptions & planning parameters

`DIM_Scenario` is the planning layer used for capacity projections in [§9](#capacity-planning--decisions):

```
Scenario_ID   Additional_IT_kW
S00           0
S01           20
S02           50
S03           70
S04           80
S05           100
```

Every number elsewhere in this document is either measured by the simulation, fixed by design, or an explicit assumption — none are presented as manufacturer data.

| Parameter | Value | Type |
|---|---|---|
| Simulation period | 28 days, 15-min intervals | Model design — fit for capacity/trend analysis, coarser than a real-time monitoring (DCIM/BMS) feed, which typically samples every 1–5 minutes for incident detection |
| Peak IT load | 456 kW | Simulated |
| Floor area / headcount | 400 m² / 6 people | Assumption |
| UPS no-load loss | 8 kW/unit | Assumption |
| COP (nominal / variation) | 3.5 ± 0.3 | Simulation assumption — a deliberate day-to-day drift model, not a real chiller's datasheet curve |
| IT heat conversion | 1:1 | Schneider WP25 methodology |
| Redundancy topology | N+1 | Architecture |
| Generator telemetry | Not modelled | Limitation |

## Simulation methodology

Thermal load follows the Schneider Electric White Paper 25 methodology: IT electrical power converts 1:1 to heat, plus UPS losses, distribution losses, lighting, and occupants. The Schneider method was used as an **additional sizing reference** for validating cooling capacity (§8) — not as a claim about any specific equipment's real-world behaviour.

![Thermal load breakdown](figures/02_thermal_breakdown.png)

Data generation uses no non-deterministic random function: noise is a trigonometric hash indexed on timestamp and equipment ID, so every model refresh produces identical results — a reproducibility requirement, not a cosmetic detail.

## Key metrics

### Energy (28-day totals)

| KPI | Value |
|---|---|
| IT Energy | 262.8 MWh |
| Facility Energy | 393.8 MWh |
| Cooling Energy | 88.0 MWh |
| PUE | **1.50** |
| Cooling Energy / IT Energy | 0.335 |

![PUE breakdown](figures/03_pue_breakdown.png)

Raising the cooling supply setpoint from 18°C to 22°C would bring PUE to 1.46 — a **simulated annualized saving under the model's COP sensitivity assumption (+3% per °C)**, not a measured result, and only worth pursuing *after* the capacity fix in §8 (a higher setpoint eats into the thermal margin needed during a failure). The tested range sits within ASHRAE TC 9.9's recommended envelope (~18–27°C for Class A1 data centers) — the model doesn't test beyond it.

### Capacity

| KPI | Value | Equivalent additional IT load |
|---|---|---|
| IT Peak | 456 kW | — |
| Electrical N+1 Headroom | 117.7 kW (facility) | ~84.7 kW |
| UPS N+1 Headroom | 144.4 kW (UPS output) | 144.4 kW |
| Cooling N+1 Headroom | 74.4 kWth (thermal) | ~69.5 kW |
| Binding Constraint | **Cooling** | — |

Where each margin is measured. The three headroom figures are not directly comparable, because each subsystem sits at a different point in the power chain. Electrical headroom (117.7 kW) is the gap between transformer capacity and total facility power at the meter, which includes cooling, UPS losses and auxiliaries. UPS headroom (144.4 kW) is measured at the UPS output, which carries IT load only. Cooling headroom (74.4 kWth) is thermal, not electrical. Expressed as the additional IT load each subsystem can still absorb, the ranking is cooling ~69.5 kW, electrical ~84.7 kW, UPS ~144 kW. That conversion, not the raw figures, is what makes cooling the binding constraint. See §9
![Capacity constraint](figures/05_capacity_constraint.png)

### Peak Energy Balance

"Peak IT load" and "peak facility power" don't occur at the same 15-minute interval — the model was queried directly for both moments, not reconstructed from averages.

| | Peak IT moment (14 Aug, 15:00) | True facility power peak (12 Aug, 15:15) |
|---|---|---|
| IT load | 455.6 kW | 454.7 kW |
| Thermal load | 523.8 kWth | 524.9 kWth |
| **COP at that moment** | 3.58 | **3.34** |
| Cooling electrical load | 146.3 kW | **157.3 kW** |
| UPS losses | 46.8 kW | 48.8 kW |
| Distribution losses | 15.1 kW | 15.1 kW |
| Lighting / people | 6.3 kW | 6.3 kW |
| **Facility power** | 670.9 kW | **682.3 kW** |
| **PUE at that moment** | 1.47 | **1.50** |

Balance check: each column sums to within 1 kW of the independently metered value (0.8 kW at the peak IT interval, 0.1 kW at the facility peak) — the residual is the meter's own noise term, not a modelling error. The facility-power peak doesn't occur when IT is highest; it occurs when a below-average COP (3.34, within the model's ±0.3 COP noise) pushes cooling electrical draw to its own peak, independent of IT.

## The main finding: cooling redundancy, before and after

**Design challenge.** With 2 chillers of 450 kWth, installed capacity (900 kWth) looked comfortably above peak thermal load (526 kWth).

**Analysis.** Redundancy doesn't depend on total capacity — it depends on what's left when one unit fails. Split across only two units, losing one costs 50% of capacity, leaving 450 kWth against a 526 kWth requirement.

**Correction.** The same 900 kWth, split into 3 units of 300 kWth instead of 2 of 450 kWth. A single-unit failure now costs 33%, not 50%.

![Before / after](figures/04_before_after_cooling.png)

| Option | Installed | Available N+1 | Headroom at peak |
|---|---|---|---|
| Original: 2 × 450 kWth | 900 kWth | 450 kWth | **−76 kWth** |
| **Corrected: 3 × 300 kWth** | **900 kWth (unchanged)** | **600 kWth** | **+74 kWth** |

Zero additional installed capacity. This is the architecture applied in the final model — every metric elsewhere in this document reflects the corrected state.

## Capacity planning & decisions

Using the `DIM_Scenario` table (§5), remaining headroom at each subsystem is queried directly from the model for every scenario — not interpolated:

| Additional IT | Cooling headroom | UPS headroom | Electrical headroom | Binding constraint | Verdict |
|---|---|---|---|---|---|
| 0 kW | 74.4 kWth | 144.4 kW | 117.7 kW | Cooling | Baseline (ACCEPTABLE) |
| +20 kW | 53.0 kWth | 124.4 kW | 89.9 kW | Cooling | ACCEPTABLE |
| +50 kW | 20.9 kWth | 94.4 kW | 48.2 kW | Cooling | Recommended planning range |
| +70 kW | −0.5 kWth | 74.4 kW | 20.4 kW | Cooling | Upper simulated limit |
| +80 kW | −11.2 kWth | 64.4 kW | 6.5 kW | Cooling | Not recommended |
| +100 kW | −32.6 kWth | 44.4 kW | **−21.3 kW** | Cooling | Not recommended |

The exact tipping point for cooling sits at **~69.5 kW** — 70 kW itself already crosses it, by a margin so thin (−0.5 kWth) it's effectively the boundary, not a comfortable cutoff.

*Why each added kW of IT load costs more than 1 kW of headroom, and why the cost differs per subsystem:*
- *UPS headroom tracks IT load 1:1 — the UPS carries IT load only.*
- *Cooling headroom falls by ~1.07 kWth per additional IT kW: the added load also raises UPS and distribution losses, which become heat too (per §6's thermal formulas).*
- *Electrical headroom falls by ~1.39 kW per additional IT kW: the same 1.07 kW of non-cooling load, plus the chiller electrical draw needed to remove the extra 1.07 kWth of heat (1.07 ÷ COP 3.34 at the peak interval).*

Queried from the model: cooling crosses zero at **~69.5 kW**, electrical at **~84.7 kW**, UPS stays positive throughout the tested range. Cooling is therefore the binding constraint across the whole recommended envelope, but the electrical margin is tighter than the raw kW figures in [§7](#key-metrics) suggest — comparing 117.7 kW (facility) against 144.4 kW (UPS output) directly would be misleading, since they aren't the same quantity (see the note in §7).

![IT growth scenarios](figures/06_it_growth_scenarios.png)

Cooling stays the binding constraint across the whole tested range. UPS headroom remains positive throughout. Electrical headroom stays positive up to ~84.7 kW of added IT load and turns negative at +100 kW — it is not an unlimited reserve, it is simply reached later than cooling.

| Decision | Verdict | Basis |
|---|---|---|
| Maintain N+1 UPS | ACCEPT | 144 kW margin |
| Maintain N+1 cooling | ACCEPT | 74 kW margin |
| Add 50 kW IT | Recommended planning range | All margins comfortably positive |
| Add 70 kW IT | Upper simulated limit | Cooling headroom −0.5 kWth — the model's own decision measure already flips to NOT RECOMMENDED here |
| Add 80 kW IT | Not recommended | Cooling constraint breached |
| Remove one UPS (2 instead of 3) | REJECT | Resilience lost for a 0.02 PUE gain |
| Raise cooling setpoint (18°C→22°C) | CONDITIONAL | Energy gain vs. thermal margin — sequence after the capacity fix above |

## Recommendations

Each recommendation below is tied to a modelled finding, not a generic best practice — and sequenced deliberately, since applying them out of order would undo the capacity fix in §8.

1. **Potential mitigation: power-factor correction, subject to harmonic and electrical-system assessment.** Power factor drops below 0.95 at peak load (`ALM-0008`), consistent with increased inductive loading from cooling equipment — the model does not explicitly simulate motor reactive components, so this is a correlation the data supports, not a modelled causal mechanism. Correcting power factor would primarily improve **kVA headroom / transformer utilization** for the same active (kW) load, and avoid utility power-factor penalties — not reduce the site's real active power draw.
2. **Consider modular UPS for the next capacity expansion.** UPS units run at 43.4% utilization on average — a direct, unavoidable consequence of sizing 3×300 kW for N+1 on ~390 kW of average load, not inefficiency to "fix" by removing a unit (see the Decision matrix: that trade loses resilience for a 0.02 PUE gain, rejected). Modular UPS would let inactive power modules idle in rotation, pulling active modules closer to their efficiency sweet spot without touching the N+1 topology.
3. **Evaluate hot/cold aisle containment before assuming the humidification and building-envelope limitations ([§14](#what-this-model-does-not-prove)) are negligible.** This model doesn't quantify recirculation losses or humidification overhead — containment is the standard mitigation for both, and is worth costing out precisely because the model can't tell you how much margin it would recover.

## Reliability & data governance

100% IT service availability over 28 days is a **simulation result**, not proof the site meets the Uptime Institute's Tier III annual target (99.982%, 1.6h/year) — the window is too short to extrapolate, and Tier III status can only be conferred by an Uptime Institute audit of a real facility. This project does not claim certification.

| Reliability | Value | Note |
|---|---|---|
| Equipment Availability | 99.335% | Frequency of alarms affecting a component |
| **IT Service Availability** | **100.000%** | No actual interruption over the window |
| Downtime (merged) | 4.47 h | Overlapping alarms merged, not double-counted |
| MTTR / MTBF | ~2.8 h / ~14 days | Based on 2 observed failures — illustrative, not statistically robust |
| Incidents / Service-impacting | 13 unique / 2 | Both resolved, no IT outage |

**Alarm log** (13 unique events — `ALM-0007` is the 14th raw record, a deliberate duplicate discussed below):

| ID | Equipment | Type | Service-impacting |
|---|---|---|---|
| `ALM-0001` | UPS-01 | High load | No |
| `ALM-0002` | Generator | Scheduled test | No |
| `ALM-0003` | CHILLER-02 | Degraded output | **Yes** |
| `ALM-0004` | Room A | Temperature excursion | **Yes** |
| `ALM-0005` | Environmental sensor | Communication loss | No |
| `ALM-0006` | UPS-02 | Planned maintenance | No |
| `ALM-0007` | CHILLER-01 | Sensor glitch | No |
| `ALM-0008` | Utility meter | Low power factor | No |
| `ALM-0009` | UPS-01 | Battery self-test | No |
| `ALM-0010` | CHILLER-02 | Efficiency drift (open) | No |
| `ALM-0011` | UPS-01 | Fault | No |
| `ALM-0012` | CHILLER-03 | Planned maintenance | No |
| `ALM-0013` | TRANSFORMER-01 | Planned maintenance | No |

Concurrent maintainability is demonstrated on **all three infrastructure pillars**, each with zero IT impact: `ALM-0006` (UPS maintenance), `ALM-0012` (chiller maintenance), `ALM-0013` (transformer maintenance).

**Data quality:** the `ALM-0007` duplicate is kept intentionally rather than silently removed, to demonstrate a deduplication calculation (7.1%) instead of hiding the defect. The independent electrical meter reading reconciles with the sum of its component sub-systems (IT + cooling + UPS losses + auxiliary) within noise (<1%) — a deliberate design choice that lets the model surface this kind of cross-source check. `ALM-0008` (power factor below 0.95 during peak load) is addressed in [Recommendations](#recommendations).

## Incident deep dive

A chiller degradation incident (`ALM-0003`) is embedded in the data, with effects consistent across every table.

![Incident](figures/07_incident_chiller02.png)

With three chillers, compensation is shared between the two healthy units (178 and 183 kWth each) instead of falling entirely on one, as it would under the original two-chiller design.

**Thermal grace period:** the degradation alarm fires at 03:22; the room temperature alarm doesn't trigger until 04:05 — a 43-minute window driven by thermal inertia and redundant compensation, during which an operations team has a simulated intervention window before the defined room-temperature threshold is reached.

## Model validation

Every claim above rests on an explicit check, not an assumption that the model "should" be consistent.

| Check | Expected | Result |
|---|---|---|
| IT load reconciliation | Room A + Room B = site IT load | PASS |
| Thermal balance | IT + UPS + distribution + lighting + people = total thermal load | PASS |
| PUE decomposition | Base + cooling + UPS + auxiliary contributions sum to measured PUE | PASS |
| N+1 cooling | Available capacity > peak demand | PASS (+74 kWth, after correction) |
| N+1 UPS | Available capacity > peak demand | PASS (+144 kW) |
| N+1 electrical | Available capacity > peak demand | PASS (+118 kW) |
| Meter reconciliation | Independent meter reading vs. sum of components | PASS (<1% variance) |
| Duplicate detection | `ALM-0007` identified and quantified | PASS (7.1% duplicate rate) |
| Overlapping downtime | Concurrent alarms merged, not double-counted | PASS (8.18h → 4.47h) |

## What this model does NOT prove

This simulation does **not**:

- Certify Tier III compliance
- Predict real equipment performance
- Replace a detailed mechanical/electrical engineering study
- Represent the configuration of any existing data center
- Quantify building-envelope heat gain or humidification overhead — these are not modelled and are treated as out of scope, not as negligible
- Support real-time incident detection — the 15-minute interval is fit for capacity and trend analysis, not for the faster sampling (typically 1–5 minutes) a real monitoring system would use to catch a developing fault as it happens
- Model ambient-dependent chiller derating — the 300 kWth rating is treated as constant, whereas a real air-cooled chiller loses capacity as outdoor temperature rises. In a hot climate this would erode the N+1 margin reported here precisely when the thermal load is highest

Every figure here is either a direct simulation output or an explicitly labelled assumption — where a result depends on one (COP sensitivity, floor area, headcount), that dependency is stated next to the number, not buried in a footnote.

**Note on transformer sizing and power factor.** This model expresses transformer capacity as 800 kW of active power, derived from a 1,000 kVA nameplate at a 0.8 power factor (Schneider Electric White Paper 3). That 0.8 is a conservative planning assumption, not the site's measured behaviour: `FACT_Electrical_Meter` shows a power factor of 0.96 on average (0.95–0.97 range across the period), including at the facility power peak itself (0.961). Sizing at 0.8 therefore understates the transformer's usable active-power capacity, deliberately, so that the electrical headroom reported here is a floor rather than a best case. A kVA-based view, queried at the facility peak (682.3 kW, PF 0.961), gives an apparent power of ~710 kVA against the 1,000 kVA nameplate — roughly 290 kVA of apparent-power margin. That framing is what makes the power-factor correction in Recommendation 1 worth costing: it recovers kVA headroom, not active power. `ALM-0008` records a brief dip below 0.95 during one peak-load interval — the dataset's measured minimum is exactly 0.95 — so the model's normal operating PF sits comfortably above the 0.8 planning assumption even during that event.

## Sample data & sources

Full 28-day tables aren't included here (5,000–8,000 rows each). `data/` contains a **2-day illustrative extract** (9–10 August: one normal day, one incident day) plus the complete 14-row alarm log — full dataset and the Power BI model available on request.

| File | Rows | Content |
|---|---|---|
| `fact_it_load_sample.csv` | 384 | 2 rooms, 2 days |
| `fact_ups_sample.csv` | 576 | 3 UPS units, 2 days |
| `fact_cooling_sample.csv` | 576 | 3 chillers, includes the live incident |
| `fact_thermal_load_sample.csv` | 192 | Full thermal breakdown, 2 days |
| `fact_electrical_meter_sample.csv` | 192 | Site meter, 2 days |
| `fact_alarms_full.csv` | 14 | Complete alarm log |

**Sources:**
- Schneider Electric, Data Center Science Center — *White Paper 25: Calculating Total Cooling Requirements for Data Centers* (Neil Rasmussen)
- Schneider Electric, Data Center Science Center — *White Paper 3: Calculating Total Power Requirements for Data Centers* (Richard L. Sawyer)
- Uptime Institute — Tier III criteria (N+1 redundancy, concurrent maintainability, 99.982% availability target)
- ASHRAE TC 9.9 — recommended thermal envelope for data center equipment (Class A1)

All operational data, capacities, and incidents in this project are simulated and calibrated against these public references. They do not represent any real data center's actual configuration.
