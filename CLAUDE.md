# CLAUDE.md — Contribution Standards

Instructions for Claude Code (and any other AI agent or human contributor) working in this repo. Every change must conform to the [Cursor plugin template spec](https://github.com/cursor/plugin-template). Validations below mirror `scripts/validate-template.mjs` from that template.

## Repository Purpose

This repo packages the Harness MCP v2 server + 27 Harness skills + governance hooks as a **single-plugin** Cursor plugin. The reference architecture is [makenotion/cursor-notion-plugin](https://github.com/makenotion/cursor-notion-plugin) — flat layout at the repo root, no marketplace manifest.

## Repository Layout

```
cursor-harness-plugin/
├── .cursor-plugin/plugin.json    # Plugin manifest (REQUIRED)
├── mcp.json                      # MCP server config — EXACT filename
├── hooks/hooks.json              # MCP governance hooks
├── scripts/*.mjs                 # Hook script implementations
├── rules/harness.mdc             # Workspace rule shipped to installs
├── skills/<name>/SKILL.md        # Skill definitions (27 present)
├── assets/logo.png               # Marketplace logo
├── README.md                     # User-facing install + usage
└── .cursor/rules/                # Repo-local rules (contribution-time only)
```

**Do not** introduce `.cursor-plugin/marketplace.json`, a `plugins/` subdirectory, or otherwise convert this to a multi-plugin repo. The `harness-mcp-plugin` is intentionally single-plugin at root.

## Plugin Template Compliance — Hard Rules

1. **Plugin name**: lowercase, matches `^[a-z0-9](?:[a-z0-9.-]*[a-z0-9])?$`. Declared in `.cursor-plugin/plugin.json` under `"name"`.
2. **MCP server definitions**: live exclusively in `mcp.json` at the repo root. Do not rename this file. Do not split into multiple MCP config files.
3. **Hook definitions**: live exclusively in `hooks/hooks.json`. Hook scripts live in `scripts/`. No hook logic in `mcp.json` or `plugin.json`.
4. **Referenced paths** (logo, hook `command`, skill paths in manifest): must be safe **relative** paths — no `..`, no absolute paths, no paths that escape the plugin root.
5. **Logo**: committed to the repo, referenced from `plugin.json` via a relative path.
6. **No secrets** in any committed file. Credentials (`HARNESS_API_KEY`, `HARNESS_ACCOUNT_ID`, etc.) are consumed from environment variables at runtime.

## Frontmatter Requirements

| File type | Location | Required frontmatter keys |
|-----------|----------|---------------------------|
| Rule | `rules/*.mdc`, `*.md`, `*.markdown` | `description` |
| Skill | `skills/<name>/SKILL.md` | `name`, `description` |
| Agent | `agents/*.md` | `name`, `description` |
| Command | `commands/*.(md\|mdc\|markdown\|txt)` | `name`, `description` |

YAML frontmatter must open with `---\n` and close with `\n---\n`. Missing the `name` or `description` key fails the validator.

## Skill Authoring Standards

- **Directory**: `skills/<kebab-case-name>/SKILL.md`. The skill's `name` frontmatter value must match the directory.
- **MCP tools**: use the Harness MCP v2 consolidated surface only — `harness_list`, `harness_get`, `harness_create`, `harness_update`, `harness_delete`, `harness_execute`, `harness_search`, `harness_describe`, `harness_diagnose`, `harness_status`. Do not reference legacy per-resource tool names.
- **Required sections** (match existing skills):
  - `## Instructions` — numbered steps the agent follows
  - `## Examples` — concrete invocation examples
  - `## Performance Notes` — quality/speed guidance
  - `## Troubleshooting` — error recovery
- **References**: skill-specific reference files live in `skills/<name>/references/`. Cross-skill references use relative paths like `create-pipeline/references/native-steps.md`.
- **Frontmatter metadata** (optional but preferred — matches upstream `harness/harness-skills` style):
  ```yaml
  metadata:
    author: Harness
    version: <semver>
    mcp-server: harness-mcp-v2
  license: Apache-2.0
  compatibility: Requires Harness MCP v2 server (harness-mcp-v2)
  ```
- **README sync**: every new skill must appear in the skill table in `README.md`.

## Hook Authoring Standards

- **Events**: use the canonical Cursor event names (`beforeMCPExecution`, `afterMCPExecution`, `beforeShellExecution`, `afterShellExecution`, `afterFileEdit`, `beforeReadFile`, `beforeSubmitPrompt`, `sessionStart`, `sessionEnd`, `stop`). Confirm at [cursor.com/docs/agent/hooks](https://cursor.com/docs/agent/hooks).
- **Matchers** for MCP tools use the format `MCP:<tool_name>` — e.g. `MCP:harness_create`. Multiple tools are separated with `|` (regex alternation).
- **Protocol**: hook scripts read JSON from stdin, write JSON to stdout.
- **Fail open — non-negotiable**: on any missing credentials, bad stdin, network error, or internal exception, the script must emit `{"permission":"allow"}` (for `before*`) or `{}` (for `after*`). A hook must never block the agent because of our own bug. Wrap `main()` in a `try/catch` that prints the allow/noop payload.
- **Script format**: `.mjs` Node modules with `#!/usr/bin/env node` shebang and executable bit (`chmod +x`).
- **Scope narrowly**: use `tool_input.resource_type` (not just the matcher) to decide whether to act. Bail early for unrelated calls.
- **Shared code**: put reusable helpers in `scripts/harness-api.mjs`; individual hook scripts stay small.

## Hook/Script Environment

- Scripts run as child processes of Cursor and inherit its environment.
- Required env: `HARNESS_API_KEY`, `HARNESS_ACCOUNT_ID`. Optional: `HARNESS_BASE_URL`, `HARNESS_ORG`, `HARNESS_PROJECT`.
- Do not hardcode credentials or URLs. Do not log credentials to stdout (Cursor displays stdout to the user).
- Network calls should time out (10s recommended). `harness-api.mjs` already handles this.

## Synchronization Rules

When you change one of these, update all of the others:

- **New MCP tool on the server** → add a row to the tool catalog in `rules/harness.mdc` AND the `README.md` tool table. Update any skills that should reference it.
- **New skill** → add to README skill table AND to this file's tree if notable.
- **New hook** → add to README "Governance Hooks" table AND document in `rules/harness.mdc`.
- **Plugin version bump** → bump `"version"` in `.cursor-plugin/plugin.json`.

## Pre-Commit Checklist

1. **Syntax check** every new/modified `.mjs`:
   ```bash
   node --check scripts/your-script.mjs
   ```
2. **Smoke-test** hook scripts with synthetic input:
   ```bash
   echo '{"tool_name":"MCP:harness_create","tool_input":{"resource_type":"pipeline","body":{}}}' \
     | env -u HARNESS_API_KEY -u HARNESS_ACCOUNT_ID ./scripts/your-hook.mjs
   ```
   Confirm fail-open behavior without credentials.
3. **Structural validation** (same rules as the upstream template validator):
   - `.cursor-plugin/plugin.json` is valid JSON and `name` is kebab-case.
   - `logo` path in `plugin.json` resolves.
   - `mcp.json` is valid JSON.
   - Every file in `rules/` has `description` frontmatter.
   - Every `skills/<name>/SKILL.md` has `name` + `description`.
   - Every hook `command` path exists and is executable.
4. **README** is up to date with any new skills, hooks, or env vars.

## What NOT to Do

- **Do not** rename `mcp.json`, `hooks/hooks.json`, or `.cursor-plugin/plugin.json`. The validator checks exact filenames.
- **Do not** reference paths outside the plugin root or paths containing `..`.
- **Do not** commit `.env` files, API keys, or account IDs.
- **Do not** weaken or remove the fail-open behavior in hook scripts.
- **Do not** add a `plugins/` directory or `.cursor-plugin/marketplace.json` — this plugin is single-plugin.
- **Do not** introduce runtime dependencies in hook scripts beyond Node's standard library. Hook scripts must run with `node` alone, no `npm install`.
- **Do not** invent Cursor hook event names. Only use events documented at [cursor.com/docs/agent/hooks](https://cursor.com/docs/agent/hooks).

## References

- Cursor plugin template — https://github.com/cursor/plugin-template
- Notion plugin (single-plugin reference) — https://github.com/makenotion/cursor-notion-plugin
- Harness Skills (upstream) — https://github.com/harness/harness-skills
- Harness MCP v2 server — https://github.com/harness/mcp-server
- Harness Policy as Code — https://developer.harness.io/docs/platform/policy-as-code/overview
- Cursor hooks documentation — https://cursor.com/docs/agent/hooks
