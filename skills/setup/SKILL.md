---
name: setup
description: Connect Claude to an OpexMX tenant and check what it can reach. Use when the user says "set up OpexMX", "connect OpexMX", installs this plugin, or when an OpexMX tool call fails with 401/403 or a connection error.
---

# Set up OpexMX

The plugin talks to the site's MCP endpoint, `https://<site>.mx.opexcg.com/api/mcp`, with an OpexMX API key. Every call runs as the key's owner, so the key's permissions decide what Claude can see and change.

## 1. Check the connection

Call `list_available_tools`. If it works, go to step 3.

If the `opexmx` server is missing, didn't start, or the call fails with 401, the plugin's two settings are unset or wrong:

| Setting | Value |
|---|---|
| OpexMX MCP URL (`opexmx_mcp_url`) | `https://<site>.mx.opexcg.com/api/mcp`, or `https://<server>/api/mcp` on-premise |
| OpexMX API key (`opexmx_api_key`) | An API key starting with `mx_` |

The user sets them in the dialog that `/plugin configure opexmx@opexmx` opens (or `/plugin`, the plugin's row, **Configure**). The key is stored in secure storage, not in settings files. Restart the session after changing them.

## 2. Get an API key

Tell the user, step by step:

1. Sign in to OpexMX as the person whose access Claude should have. A planner or reliability engineer account is a good fit; avoid an admin account unless Claude needs admin data.
2. Open **Integrations → API** (`/integrations/api`) and create a key. Copy it: it is shown once.
3. Run `/plugin configure opexmx@opexmx` and paste the URL and key into the dialog, then restart the session.

Never ask the user to paste the key into the chat, and never echo it back.

## 3. Report what works

Make one cheap read per area and report which skills can fully run. Use small limits.

| Check | Call | Needed by |
|---|---|---|
| Assets | `search_assets` with `limit: 1` | all skills |
| Tickets and work orders | `search_tickets` with `limit: 1` | diagnose-breakdown |
| Failure-mode library | `troubleshoot_problems` with a generic symptom such as `"overheat"` | diagnose-breakdown |
| Alarms and condition data | `get_alarm_incidents` with `limit: 1` | diagnose-breakdown |
| Knowledge base | `search_knowledge` with a generic query such as `"lubrication"` | diagnose-breakdown |
| Downtime | `downtime_stats` for the last 30 days | downtime-cost-report |
| Reliability | `mtbf_mttr` for the last 30 days | downtime-cost-report |
| Spare parts | `search_parts` with `limit: 1` (filter argument: `search`) | diagnose-breakdown |

For each, say **works**, **no data yet** (the call succeeded but returned nothing, e.g. no alarms are configured), or **no access** (403: the key's owner lacks the permission; name the area so an admin can grant it). Keep the report short: a table, then one line on what to fix first.

Don't create, update or comment on anything during setup.
