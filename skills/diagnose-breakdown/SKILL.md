---
name: diagnose-breakdown
description: Rank the likely causes of an equipment breakdown or abnormal symptom using the asset's OpexMX history (past tickets, failure modes, alarms, condition data, manuals) plus anything the user provides (photos, sensor exports, notes), and give the technician an ordered list of checks. Use when a machine has stopped, tripped, is overheating, vibrating, leaking or behaving abnormally, or when someone asks "what could be wrong with X".
argument-hint: "<asset name or code> — <symptom>"
---

# Diagnose a breakdown

**If Anthropic's Claude for Manufacturing plugin is installed**, its `breakdown-diagnosis` (and `breakdown-to-next-report`) is the main workflow: use it, with OpexMX as its CMMS source as described in `cmms-for-manufacturing`. Use this skill only when that plugin isn't installed, or when the user asks for the OpexMX version by name.

The output is a ranked list of likely causes and the checks that confirm or rule each one out, for a technician to act on. It is not a verdict: the technician on site decides.

## 1. Pin down the asset and the symptom

- Find the asset with `search_assets` (`search` = name or code). If several match, ask which one; don't guess. Then `get_asset` for its details and `get_asset_status` for its current state.
- Get the symptom in the user's words: what happened, when, and what was running at the time. If they haven't said, ask once, briefly. Work with what they give you.
- If the user attached photos, sensor or PLC exports, or repair notes, read them now; they are evidence too.

## 2. Gather the evidence

Make these calls together; skip any the setup check showed as unavailable:

| Evidence | Call |
|---|---|
| Known failure modes and repair procedures for this symptom | `troubleshoot_problems` (`query` = symptom, `assetId`) |
| Repair history | `search_tickets` (`assetId`, recent first). Open the 3–5 most relevant with `get_ticket` |
| Active and recent alarms | `get_alarm_incidents` (`assetId`, `status: "all"`); `explain_alert` for any that line up with the symptom |
| Condition monitoring | `vibration_alerts`, `predictive_anomalies`, `get_health_scores`, `get_rul`, `failure_predictions` (all with `assetId`) |
| Manuals, SOPs, OEM notes | `search_knowledge` (symptom + asset model or type) |
| PM status | `get_pm_overview` (`assetId`): an overdue lubrication or inspection is often the answer |

No condition data is normal on many assets: say so and rely on history and failure modes.

## 3. Rank the causes

For each candidate cause (aim for 3–5):

- **Why it's likely:** cite the evidence: ticket numbers and dates, the failure mode's likelihood from `troubleshoot_problems`, the alarm, the reading, the overdue PM. A cause seen on this exact asset before ranks above a generic one.
- **What argues against it**, if anything.
- **How to check:** the first, cheapest, safest check that confirms or rules it out. Use the repair procedure's sub-steps when `troubleshoot_problems` returns one.
- **Parts likely needed:** look them up with `search_parts` (the argument is `search`, not `query`: part name, code or size such as "6305") and `get_part_stock`, and say whether they're in stock and where.

Order the checks so the technician starts with what is quick, safe and most likely, not strictly by likelihood alone.

Rules:
- Never state a cause as confirmed. Use "likely", "consistent with", "rule out".
- Don't invent readings, ticket numbers, parts or procedures. If the evidence is thin, say so and say what data would help (e.g. "a vibration reading on the drive-end bearing").
- Any check that opens guards, panels or hydraulic/pneumatic/electrical circuits starts with isolation and lock-out/tag-out per site procedure. Say it every time; don't assume.

## 4. Output

```
<Asset name> (<code>): <symptom>
Status: <current status, since when>

Likely causes
1. <cause>: <likelihood: high/medium/low>
   Evidence: …
   Check first: …
   Parts: <part, stock on hand, location> / none expected
2. …

Also worth knowing: <overdue PM, recurring pattern, warranty, related assets>
Gaps: <data that was missing and would sharpen this>
```

## 5. Record it (only when asked)

Offer, don't do:
- If a ticket already exists for this breakdown, offer to add the diagnosis as a comment (`add_ticket_comment`).
- Otherwise offer to open a corrective ticket (`create_ticket` with `assetId`, a clear title, the symptom, the ranked causes and `type: "corrective"`).

Show the exact text first and write only after the user confirms.
