---
name: cmms-for-manufacturing
description: Use OpexMX as the CMMS connector for Anthropic's Claude for Manufacturing plugin. Maps its CMMS inputs (assets, maintenance work orders and history, PM plans, downtime rows) to OpexMX MCP tools, and gives custom-connector the setup steps. Use when Claude for Manufacturing skills (onboarding, custom-connector, breakdown-diagnosis, breakdown-to-next-report, daily-production-report, data-consolidate) look for a CMMS, a maintenance export or downtime events and OpexMX is connected.
---

# OpexMX as the CMMS for Claude for Manufacturing

Claude for Manufacturing's skills read "a CMMS connector" for assets, maintenance history and PM plans, and fall back to a maintenance export when none is live. Its worked example is Limble, added through `custom-connector`. OpexMX is the same case: a vendor-published MCP, nothing to build. When the `opexmx` MCP server is connected, treat it as the live CMMS connector, and don't ask for a maintenance export.

## For custom-connector: setup, in its format

| | |
|---|---|
| System | OpexMX (CMMS / EAM) |
| MCP | `<tenant URL>/api/mcp`, Streamable HTTP, e.g. `https://acme.mx.opexcg.com/api/mcp`; on-prem installs use their own address |
| Credentials | An OpexMX user opens **Integrations → API** and creates an API key (starts with `mx_`, shown once). It is sent as `Authorization: Bearer <key>`; with this plugin, the user enters the MCP URL and the key in `/plugin configure opexmx@opexmx` (the key goes to secure storage). Never paste it into the chat |
| Scope | The key acts as the user who made it, with that user's role permissions. For read-only, create it from a user whose role has no create/update rights on tickets |
| Write scope | Confirmed: `create_ticket`, `update_ticket`, `add_ticket_comment`, `complete_pm_occurrence`, `reschedule_pm_occurrence`. Still gate every write behind the user's approval (the manufacturing plugin's approval-gates rule) |
| Test call | `search_assets` with `limit: 5`, then show the asset names and codes back |

Run the `setup` skill in this plugin for the full per-area check.

## CMMS inputs → OpexMX tools

| What the manufacturing skill needs | OpexMX call |
|---|---|
| Asset record and specs | `search_assets` (`search` = name or code), then `get_asset` |
| Current asset state | `get_asset_status` (`assetId`) |
| Maintenance work orders, open and in progress | `search_tickets` (`assetId`, `status`), `get_ticket` |
| Maintenance history and close-out notes | `search_tickets` (`assetId`), then `get_ticket` for the notes and comments |
| PM plan and interval for an asset | `search_schedules` / `preventive_maintenance_schedules` (`assetId`), `get_pm_overview` for overdue |
| Known failure modes and repair procedures | `troubleshoot_problems` (`query` = symptom, `assetId`) |
| Manuals and SOPs | `search_knowledge` |
| Alarms and condition monitoring (in place of part of a historian export) | `get_alarm_incidents`, `vibration_alerts`, `predictive_anomalies`, `get_health_scores`, `get_rul` |
| Bad actors | `bad_actors` (`sortBy`: `downtime`, `ticketCount`, `mtbf`, `cost`) |
| Spare parts on hand | `search_parts` (`search`, not `query`), `get_part_stock` |
| Downtime events, row by row | `api_call` `GET /asset-status-segments` with `params` `{"startDate", "endDate", "assetId"?}` (below) |

## Downtime rows in the `downtime-record` shape

`GET /asset-status-segments` returns, per asset, the status segments overlapping the window: `startTime`, `endTime` (null = still in that state), `durationSeconds`, `statusName`, `statusIsFailure`, `statusIsRunning`, `notes`. Keep the segments where `statusIsFailure` is truthy (it can be `1` or `true`). Segment starts before the window are clamped to the window start; say so when a stoppage straddles it.

Map each kept segment to the contract's stoppage fields:

| Contract field | From OpexMX |
|---|---|
| `site` | `get_asset`'s `siteId` when set; otherwise follow `parentAssetId` with `get_asset` up to the asset whose `assetType` is `site`; otherwise the plant context |
| `asset_id` / `asset_id_verbatim` | Normalized `assetCode` / `assetCode` as returned |
| `started_at` / `ended_at` | `startTime` / `endTime`, both UTC (`Z`). Null `endTime` = still down; never zero |
| `duration_minutes` | `durationSeconds ÷ 60`, rounded; null while `endTime` is null |
| `reason_code` | `statusName` (the plant's own status, e.g. "Breakdown"). Not remapped |
| `reason_text` | `notes` |
| `production_wo` | Null: OpexMX segments don't carry the interrupted production order |
| `shift` / `production_day` | Resolve from the plant's shift pattern and day boundary in the manufacturing plugin's plant context; OpexMX's MCP doesn't expose shifts. If unresolved, follow the contract's missing-field rules |
| `source_system` | `cmms` |
| `source_ref` | `opexmx:asset-status-segment:<segment id>` |

For the diagnosis fields, `maintenance_wo` and `cited_work_orders` take OpexMX ticket numbers from `search_tickets`. Because these come from the live CMMS rather than a nightly export, the export-freshness disclosure doesn't apply; name OpexMX as the source and the time of the read instead.

## Writing back

The manufacturing skills assume a CMMS's write scope is unconfirmed and produce a draft for the technician. With OpexMX the write exists, so after the user approves the exact text:
- a diagnosis or close-out note → `add_ticket_comment` on the maintenance ticket;
- a new corrective job → `create_ticket` (`assetId`, `type: "corrective"`);
- a proposed PM interval change → keep it as a draft for the planner; don't edit schedules.

Never write without the approval step, and never write on a scheduled or unattended run.
