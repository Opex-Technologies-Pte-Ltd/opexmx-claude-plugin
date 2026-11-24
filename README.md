# OpexMX for Claude

A Claude plugin that connects Claude to [OpexMX](https://mx.opexcg.com), the maintenance system of record: assets and their repair history, work orders, downtime, alarms and condition data, preventive maintenance (PM) schedules, spare parts and purchasing.

A general assistant can rank the causes of a breakdown or report on downtime only from the data it can reach. This plugin gives Claude your OpexMX data, and skills that know how a maintenance team uses it.

![OpexMX](.claude-plugin/icon.png)

## Skills

| Skill | What it does |
|---|---|
| `setup` | Checks the connection and reports which data your API key can reach. Say "set up OpexMX". |
| `diagnose-breakdown` | Ranks the likely causes of a breakdown from the asset's tickets, known failure modes, alarms, condition data and manuals, and lists the checks and spare parts for the technician. It can record the diagnosis on a ticket, after you confirm the text. |
| `downtime-cost-report` | Downtime per asset, the worst assets, MTBF/MTTR, maintenance cost and PM status for a period, converted to money with the cost per hour you give it. |
| `cmms-for-manufacturing` | Makes OpexMX the CMMS connector for Anthropic's Claude for Manufacturing plugin: maps its CMMS inputs (assets, work orders, history, PM plans) to OpexMX, supplies downtime rows in its `downtime-record` shape, and lets its breakdown work write back to OpexMX tickets after you approve. |

Claude also gets every OpexMX MCP tool (about 85, each one either read-only or a write, covering assets, tickets, PM, parts, purchase orders, reliability analytics, safety and compliance), so it can answer other maintenance questions directly.

## With Claude for Manufacturing

Anthropic's Claude for Manufacturing plugin reads "a CMMS connector" for assets, maintenance history and PM plans, and falls back to file exports when none is connected. Install both plugins and OpexMX fills that slot. Its breakdown diagnosis, daily production report and data consolidation then read live OpexMX data instead of a nightly export, and a diagnosis can be written back to the OpexMX ticket once you approve it. When both plugins are installed, this plugin's `diagnose-breakdown` steps aside for theirs.

## Install (Claude Code)

1. In OpexMX, open **Integrations → API** and create an API key. It acts as the user who created it, so make it from a planner or reliability-engineer account rather than an admin.
2. In Claude Code:

   ```
   /plugin marketplace add Opex-Technologies-Pte-Ltd/opexmx-claude-plugin
   /plugin install opexmx@opexmx
   ```

3. When the plugin asks, enter your OpexMX MCP URL, `https://<your-site>.mx.opexcg.com/api/mcp` (on-premise: `https://<your-server>/api/mcp`), and the API key. The key is stored in Claude Code's secure storage. To change either later, run `/plugin configure opexmx@opexmx`.
4. Ask Claude to "set up OpexMX".

## What the plugin sends, and where

- The plugin contains no code: only skills (instructions for Claude) and one MCP server entry.
- Claude talks to one server, the OpexMX site at the URL you enter, over HTTPS. It sends your API key as a Bearer token on each request, and the tool calls Claude decides to make, such as searches, asset lookups and reports. Nothing is sent anywhere else.
- What Claude can read or change is limited to what the key's user may do in OpexMX, and records it creates or edits show that user as their author.
- The skills only create or update OpexMX records (ticket comments, new tickets) after you confirm the exact text.
- OpexMX's privacy policy: <https://mx.opexcg.com/privacy>.

## Layout

```
.claude-plugin/plugin.json        plugin manifest and the two settings it asks for
.claude-plugin/marketplace.json   single-plugin marketplace, so this repository can be added directly
.mcp.json                         the OpexMX MCP server entry
skills/<name>/SKILL.md            the skills
```

## Support

Questions and issues: open an issue on this repository, or contact us through <https://mx.opexcg.com>.

## License

Apache-2.0. See [LICENSE](LICENSE).
