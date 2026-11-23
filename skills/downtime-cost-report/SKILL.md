---
name: downtime-cost-report
description: Build a downtime and cost-of-downtime report from OpexMX for a period, site or asset group: total downtime, the worst assets, MTBF/MTTR, maintenance cost, PM compliance, and downtime converted to money using a cost per hour the user supplies. Use for weekly or monthly maintenance reports, management reviews, "how much did downtime cost us", or picking which assets to fix first.
argument-hint: "<period, e.g. 'last month'> [site or asset group]"
---

# Downtime cost report

**If Anthropic's Claude for Manufacturing plugin is installed** and the user wants a daily production report (output vs plan, OEE, downtime by shift), use its `daily-production-report`, with OpexMX downtime rows supplied as in `cmms-for-manufacturing`. This skill stays the right one for a maintenance view over a period: downtime by asset, bad actors, MTBF/MTTR, PM and the cost of downtime.

## 1. Agree the scope

- **Period:** turn "last month", "this week", "Q3" into exact dates and state them in the report. Default: the last full calendar month.
- **Scope:** whole tenant, one site (`siteId`), or named assets. Resolve names with `search_assets`.
- **Cost of downtime:** OpexMX records how long assets were down, not what an hour of downtime costs the business. Ask the user for a cost per hour of downtime: one figure for all assets, or per asset or line (lost contribution margin, idle labour, scrap, expediting). If they don't have one, produce the report in hours only and leave the money column out. **Never invent or estimate this figure.**

## 2. Gather the data

Make these calls together:

| Data | Call |
|---|---|
| Downtime per asset | `downtime_stats` (`startDate`, `endDate`, optional `assetId`). Durations are in seconds |
| Worst assets | `bad_actors` with `sortBy: "downtime"`, then `sortBy: "cost"`; `period` as days covering the range (e.g. `"30d"`), `siteId` if scoped |
| Reliability | `mtbf_mttr` (`startDate`, `endDate`, `trendInterval: "week"` for a month or longer) |
| Workload | `get_metrics` (`period` closest to the range): tickets opened and closed, average resolution time, overdue PM |
| PM status | `get_pm_overview` |
| Spare-parts risk on the worst assets | `spare_parts_criticality` (`siteId` if scoped) |

`bad_actors.cost` is maintenance cost recorded in OpexMX (labour, parts), not lost production. Keep the two apart in the report. It can be null when no cost was recorded; show "not recorded", not 0.

## 3. Compute

- Downtime hours per asset = seconds ÷ 3600, one decimal.
- Cost of downtime per asset = hours × that asset's cost per hour (only if the user supplied rates).
- Share of total downtime per asset, and the cumulative share (Pareto): the few assets causing most of the downtime are the headline.
- Compare with the previous period of the same length if the user wants a trend: run the same calls for that range.

Show the arithmetic basis once (rate × hours) so a manager can check it.

## 4. Output

```
Downtime report: <scope>, <start> to <end>

Headline: <one or two sentences: total downtime, its cost if rated, and the 2–3 assets behind most of it>

| Asset | Downtime (h) | % of total | Cost of downtime | Maintenance cost | MTBF (h) | MTTR (h) |
|---|---|---|---|---|---|---|

Reliability: MTBF …, MTTR …, trend …
Workload and PM: tickets opened/closed …, overdue PM …
Spare-parts risk: <critical parts low on stock for the worst assets>
Recommended focus: <2–3 actions tied to the numbers above, e.g. root-cause analysis on asset X, clear overdue PM on Y>
Notes: cost per hour basis; data gaps (assets with no downtime segments recorded, null costs)
```

If the user wants a file (spreadsheet, document or slides), build it from the same table. Don't add numbers that aren't in OpexMX or from the user.

Rules:
- If downtime looks implausibly low (e.g. no segments at all), say that asset status may not be tracked for those assets rather than reporting "no downtime".
