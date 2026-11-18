# Methodology

Every calculation in the model, in dependency order. DAX is quoted from `Pricing Power.SemanticModel/model.bim`, shortened where noted.

- [1. Grain and current vs. previous month](#1-grain-and-current-vs-previous-month)
- [2. Price / volume decomposition](#2-price--volume-decomposition)
- [3. Quadrants](#3-quadrants)
- [4. The Pricing KPI](#4-the-pricing-kpi)
- [5. KPI variants](#5-kpi-variants)
- [6. Shipper buckets](#6-shipper-buckets)
- [7. Quadrant dynamics](#7-quadrant-dynamics)
- [8. Lane Score](#8-lane-score)
- [9. Supporting metrics](#9-supporting-metrics)
- [10. Notes & caveats](#10-notes--caveats)

---

## 1. Grain and current vs. previous month

`Main` holds one row per **year × month × shipper × KMA lane × equipment × quoting method**. Most measures only make sense when a single month is selected. `Month Filter = ISFILTERED('Main'[month])` gates them, and the "current month" measures return blank otherwise.

The previous month is found with plain arithmetic, so no date table is needed:

```dax
VAR CM = SELECTEDVALUE('Main'[month])
VAR CY = SELECTEDVALUE('Main'[year])
VAR PM = IF(CM = 1, 12, CM - 1)
VAR PY = IF(CM = 1, CY - 1, CY)
RETURN CALCULATE(<metric>, 'Main'[year] = PY, 'Main'[month] = PM)
```

The same pattern produces `Bookings $`, `Mileage`, `RPM` and `SSB %`, each in a *Cur. Month* and a *Prev. Month* version.

## 2. Price / volume decomposition

| Measure | Formula |
|---|---|
| `MoM` | Bookings cur. ÷ Bookings prev. − 1 |
| `DPr` (Δ price) | RPM cur. ÷ RPM prev. − 1 |
| `DVr` (Δ volume) | Mileage cur. ÷ Mileage prev. − 1 |
| `DPm`, `DVm` | `DPr`, `DVr` with \|x\| < 0.2% set to 0 |

A result of exactly −100% (nothing in the current month) is returned as blank.

Since bookings = RPM × mileage, the decomposition is exact and multiplicative:

```
(1 + MoM) = (1 + DP) · (1 + DV)
```

`sanity_check_r = (1+DPr)(1+DVr) − 1 − MoM` should always be ≈ 0. `sanity_check_m` is the same check using the de-noised `DPm`/`DVm`, and it shows how much the 0.2% dead-band moves the result. Both are shown on *Big Figures*.

The dead-band matters for classification: without it, a lane whose rate moved by $0.001/mile would flip between quadrants on rounding noise.

## 3. Quadrants

```dax
Quadrant =
IF(NOT [Month Filter] || ISBLANK([MoM]) || [# Unique Lanes] > 1, BLANK(),
    IF([DPm] > 0,
        IF([DVm] > 0, "1. Upgrade", "2. Strategic Price"),
        IF([DVm] < 0, "3. Downgrade", "4. Strategic Vol.")))
```

A quadrant only exists for **one KMA lane in one month**. Higher levels of the report count lanes and loads by quadrant:

- `# Lanes (n. …)`: lanes in the quadrant (`COUNTROWS(FILTER(VALUES(lane_kma), [Quadrant] = …))`)
- `# Loads (n. …)`: loads on those lanes
- `# Loads (1a. Upgrade & Winning)` / `(1b. … Losing)`: Upgrade loads split by row-level sourcing spread (`ss_base > 0` vs. `≤ 0`)
- `Total # Loads/Lanes (…)`: the monthly counts summed over several months, for period totals

## 4. The Pricing KPI

```dax
KPI =
( K1a·[Weight .1a] + K1b·[Weight .1b] + K2·[Weight .2] + K3·[Weight .3] + K4·[Weight .4] )
/ ( K1a + K1b + K2 + K3 + K4 )
```

`Kx` is the number of loads in each quadrant. The weights are measures in the *Parameters* folder:

| Parameter | Value | Meaning |
|---|---:|---|
| `Weight .1a` | 2 | Upgrade, winning: best case, growing with rate and margin |
| `Weight .1b` | 1.5 | Upgrade, losing: growing with rate, but negative spread |
| `Weight .2` | 1 | Strategic Price: rate up, volume down, a deliberate trade |
| `Weight .4` | 1 | Strategic Volume: volume up, rate down, a deliberate trade |
| `Weight .3` | −1 | Downgrade: losing both, the only negative score |

The KPI ranges from −100% (all loads on Downgrade lanes) to 200% (all loads on winning Upgrade lanes). **100% is the neutral line**: the business is, on average, making deliberate price/volume trades and not losing ground.

## 5. KPI variants

| Measure | Filter context it keeps | Use |
|---|---|---|
| `KPI` | everything | lane and shipper detail |
| `KPI (Shipper Specific)` | everything, but only when `# Unique Shippers = 1` | bucketing shippers |
| `Pricing KPI` | `ALLEXCEPT(year, month, equip_type, sales_team, quoting_method)` | trend lines that stay stable when the user clicks a lane or shipper |
| `KPI (Company-Wide)` | `ALLEXCEPT(year, month)` plus `sales_team ∈ {Account Sales, Enterprise Sales}` | the company benchmark on every chart |

## 6. Shipper buckets

Each shipper's `KPI (Shipper Specific)` is compared with three thresholds (*Parameters*):

| Bucket | Rule | Threshold |
|---|---|---|
| 0. KPI is blank | no classifiable lane that month | |
| 1. KPI less than 0% | KPI < `Threshold .1` | 0 |
| 2. KPI between 0% and 100% | `Threshold .1` ≤ KPI < `Threshold .2` | 1 |
| 3. KPI at 100% | KPI = `Threshold .2` | |
| 4. KPI between 100% and max. | `Threshold .2` < KPI ≤ `Threshold .3` | 5/3 |
| 5. KPI at max. | KPI > `Threshold .3` | |

For each bucket, the report shows the number of shippers (`# Shippers (…)`) and the amount in it (`1. … / 5. …`). The amount comes from `Total Amount $`:

```dax
Total Amount $ = SWITCH(SUM(Amounts[index]), 1, SUM([bookings]), 2, SUM([ss_base]))
```

`Amounts` is a disconnected two-row table (*Bookings $*, *Sourcing Spread Base $*). One slicer switches every bucket visual between revenue and margin. `title_chart1`, `title_chart2` and `title_table` build the matching dynamic titles.

**Coverage indicators:**
- `CI (Lanes) %` = (unique lanes − classified lanes) ÷ unique lanes: the share of lanes that couldn't be put in a quadrant, for example because they had no activity last month.
- `CI (Amount $) %` = bucketed amount ÷ (bucketed + blank-KPI amount): the share of revenue or spread the KPI actually covers.

## 7. Quadrant dynamics

| Measure | Definition |
|---|---|
| `Quadrant Prev. Month` / `Quadrant Next Month` | the lane's quadrant one month earlier / later |
| `Quadrant Changed From Previous Month? (Y/N)` | `Y` when the two differ |
| `# Months` | months in context where the lane has a quadrant (only when no single month is selected) |
| `# Changes` | months where the quadrant changed |
| `Quadrant Volatility %` | `# Changes ÷ (# Months − 1)` per lane × equipment × quoting method |
| `Span (# Months)` | `# Months` per lane × equipment, ignoring other filters |

High volatility means the lane's position is noise-driven, so a pricing action shouldn't be read into a single month. Low volatility with a good quadrant shows a durable pricing position.

## 8. Lane Score

A letter grade for **sourcing-spread health** that compares this month's `SSB %` with last month's:

| Current month | Previous month | Score |
|---|---|---|
| > +5% | > +5% | **A**: consistently profitable |
| > +5% | ≤ +5% | **B**: improving |
| between −3% and +5% | any | **C**: neutral |
| < −3% | ≥ −3% | **D**: deteriorating |
| < −3% | < −3% | **F**: consistently losing |

## 9. Supporting metrics

| Measure | Definition |
|---|---|
| `RPM` | bookings ÷ mileage |
| `Sourcing Spread Base %` | gross sourcing spread ÷ bookings |
| `Markup Over ECOH %` | markup ÷ estimated cost of haul, loads with ECOH ≥ $300 |
| `Conversion %` | MIN(loads ÷ quotes, 100%), matched on month : KMA lane : equipment : quoting method |
| `Conversion (last 12 mo.) %` | conversion over the 12 months up to the latest month in `Periods` |
| `# Unique Carriers (last 12 mo.)` | distinct carriers on the lane (single lane only) |
| `%GT # Loads`, `(Q. Method)`, `(S. Team)` | share of classified loads by equipment, quoting method or sales team |
| `Bookings (M$)` | current-month bookings in millions |

**Calculated tables and columns:**
- `Mileages`: average miles per load for each KMA lane. It feeds `Main[mileage_group]`, a distance band: 100-mile steps below 200 mi, 200-mile steps below 2,000 mi, then 400-mile steps.
- `Periods`: monthly bookings of Account + Enterprise sales. It drives `Main[Display? (Y/N)]`, which flags months under $1 M (partial or ramp-up months). A **report-level filter** on that flag removes those months from every page. `Periods` also anchors the trailing-12-month window.
- `List`: the lane universe. KMA lanes are kept only if their RPM since Dec 2022 falls within `[P25 − 1.5·IQR, P75 + 2·IQR]` for their equipment type, which removes lanes with implausible rates.

## 10. Notes & caveats

- **The KPI measures pricing behaviour, not profit.** A Strategic Volume month scores +1 whether or not the extra volume was profitable. Read it next to `Sourcing Spread Base %` and the Lane Score.
- The **0.2% dead-band** is applied to classification only. `MoM`, `DPr` and `DVr` stay exact.
- **"At 100%"** and **"at max."** test equality and "above" respectively. A shipper whose loads all sit on Strategic Price or Strategic Volume lanes scores exactly 100%, which is why that bucket exists. `Threshold .3` (5/3) sits below the theoretical maximum of 2, so "at max." really means "above 167%".
- **Label history:** the *Instructions* page still uses the quadrants' earlier names: *Growth* = Upgrade, *Profitable* = Strategic Price, *Loss Leader* = Downgrade, *Strategic* = Strategic Volume.
