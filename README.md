# linkedin-tracker (Claude Code marketplace)

A one-plugin Claude Code marketplace for the Superside BDR team: the standalone **`/linkedin-tracker`** LinkedIn connection-request + cold-call lead tracker.

Self-contained — it bundles its own read-only LinkedIn MCP and permission deny-list, keeps its own Notion tracking page and databases, and does **not** depend on `convohub-sync` / `linkedin-crm-sync` or any other plugin. One install works for any BDR against their own LinkedIn login and Notion identity, resolved per run.

## Install (each BDR, once)

In an interactive `claude` terminal:

```
/plugin marketplace add thewrathofka/linkedin-tracker
/plugin install linkedin-tracker@linkedin-tracker
```

Then restart Claude Code (or reload plugins) so the bundled `linkedin` MCP server starts. Run it with `/linkedin-tracker`.

### Runtime prerequisites (per BDR)

- **Claude in Chrome**, logged into your own LinkedIn — used for the send path only.
- Your own **Notion** and **Salesforce** access.
- `uv` / `uvx` installed (the LinkedIn MCP runs via `uvx mcp-server-linkedin`).

## Layout

```
linkedin-tracker/                                  ← marketplace repo
├── .claude-plugin/marketplace.json                ← marketplace manifest
└── plugins/
    └── linkedin-tracker/                          ← the plugin
        ├── .claude-plugin/plugin.json
        ├── .mcp.json                              ← read-only LinkedIn MCP
        ├── .claude/settings.json                  ← permission deny-list
        ├── README.md
        └── skills/linkedin-tracker/SKILL.md
```

See [`plugins/linkedin-tracker/README.md`](plugins/linkedin-tracker/README.md) for what the command does and how it stays safe.
