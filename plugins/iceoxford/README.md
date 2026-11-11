# iceoxford plugin

Skills for working with data from the ICEOxford cryostat monitor.

- **`iceoxford-data`**: how to find, query and interpret the cloud recording sessions archived
  in MotherDuck (database `iceoxford`): tables, the meaning of every channel column, caveats that
  change conclusions (He gauge unit change, loom pickup, generic sensor curves, running sessions
  still partly in Supabase) and tested DuckDB query recipes. Read only.

Needs the MotherDuck MCP connector, signed in to an account that can read `iceoxford`.

The ICEOxford monitor code itself lives in `inwwin/ICEOxford`; this plugin is kept here and must be
kept in step with it when the schema, channels, wiring or caveats change.

## Install in another repo

```
/plugin marketplace add inwwin/inwwin-claude-marketplace
/plugin install iceoxford@inwwin-claude-marketplace
```

If the repository is private, Claude Code needs git access to it (an SSH key or `gh auth`). A
local clone also works: `/plugin marketplace add C:\path\to\inwwin-claude-marketplace`. Update later with
`/plugin marketplace update inwwin-claude-marketplace`.
