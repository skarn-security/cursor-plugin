# Skarn for Cursor

This Cursor plugin carries the `skarn-audit` skill, the pre-execution guard hooks in audit mode, and a declaration for the local `skarn` MCP server. The recall skill is not part of this plugin. The plugin carries no binary and no detection engine; every part of it calls `skarn` from your PATH.

## Install

Install the `skarn` binary first:

```sh
brew install skarn-security/tap/skarn
```

Or run `npm install -g @skarn-security/skarn`, or put the release download from https://github.com/skarn-security/skarn-dist/releases/latest on your PATH. Confirm it with `skarn --version`.

The Cursor Marketplace listing is pending, so clone the plugin where Cursor reads local plugins:

```sh
mkdir -p ~/.cursor/plugins/local
git clone https://github.com/skarn-security/cursor-plugin ~/.cursor/plugins/local/skarn
```

To install from a checkout you already have, copy it there with `cp -R`. A symlink does not load, because Cursor rejects a local plugin whose symlink target lies outside `~/.cursor/plugins/local`.

Then run `Developer: Reload Window`. Settings > Tools & MCPs lists `skarn` under Plugin MCP Servers with `4 tools enabled`. Cursor starts plugin MCP servers only while you are signed in to a Cursor account.

To wire the guard hooks without the plugin, run `skarn setup --agent cursor`. It merges the hook into `~/.cursor/hooks.json` without touching other hooks, backs the file up first, and ends with the guard self-test. `--scope project` writes `<repo>/.cursor/hooks.json` instead, and `--print` shows the config without writing it. Cursor reloads `hooks.json` on save, so this route needs no reload.

## Local MCP server

The plugin's `mcp.json` declares one stdio server:

```json
{
  "mcpServers": {
    "skarn": {
      "command": "skarn",
      "args": ["mcp"]
    }
  }
}
```

It runs on your machine, makes no network call, and exposes four read-only tools: `scan_sessions` (findings from past sessions), `vet_configs` (findings on your assistant configuration), `list_sessions` and `session_stats` (metadata and aggregates, never message content). Masking covers detected credential values only: `vet_configs` findings quote excerpts of your hook commands and MCP server definitions, and file paths, session ids, timestamps, and a Compliance API export's user and workspace ids come back unmasked.

Without the binary, a pinned launcher works:

```json
{
  "mcpServers": {
    "skarn": {
      "command": "npx",
      "args": ["-y", "@skarn-security/skarn@0.31.0", "mcp"]
    }
  }
}
```

npx downloads the package on the first start, so that start is slower and needs network access. Keep the version pinned; `skarn vet` reports the unpinned form as `vet-mcp-unpinned`.

## Guard hooks

`hooks/hooks.json` runs `skarn guard --agent cursor` before four Cursor events, in audit mode, which logs the would-be verdict and changes nothing:

- `beforeSubmitPrompt` scans the prompt text. Under enforce, a prompt carrying a credential is blocked before it leaves the machine, and the user sees the redacted reason and can edit and resubmit.
- `beforeShellExecution` and `beforeMCPExecution` return deny or ask, and Cursor prompts the user on ask.
- `beforeReadFile` runs with `--strict` and returns deny only, for credential-file reconnaissance. It fires on every file read; if the overhead shows, remove that entry.

Deny means the action leaks a secret into an exfiltration channel, installs a likely malicious or typosquatted package, completes a multi-phase attack chain, or trips a critical rule. Ask means a secret is present with no exfiltration channel in the call, or a high-severity finding fires. Allow emits nothing.

The guard fails open: a malformed event, an out-of-scope action, or a guard error yields allow, unless a licensed enforce runs `--strict`.

The hooks run on macOS and Linux only. Each `command` carries a POSIX `VAR=value cmd` prefix that neither PowerShell nor `cmd.exe` accepts, so on native Windows all four entries fail before `skarn guard` starts, and Cursor lets the action through unscanned. The MCP server and the skill work on every platform; for a guard on native Windows, use the Codex CLI plugin at https://github.com/skarn-security/agent-guard. Cursor has no pre-execution file-write gate, so these hooks do not block a write to disk, and each action is evaluated on its own, with no chain state across actions.

Every event this host fires, and how each one blocks: run `man skarn-guard`, or read https://getskarn.com/manual/.

## Audit first, then enforce

Each flagged action appends one redacted JSONL record to `~/.cache/skarn/guard-cursor-audit.jsonl`; a clean action leaves no record. Review a real window before enforcing:

```sh
skarn guard report --agent cursor --window 30d
```

The report prints a flip command once the window spans at least five days with zero denies. A deny whose top finding carries a secret shows that finding's fingerprint. If the finding is a false positive, accept it, and that finding stops being flagged on the identical call:

```sh
skarn guard accept <fingerprint> --reason 'internal test credential'
```

To enforce a plugin install, change `--guard-mode audit` to `--guard-mode enforce` on all four entries in `~/.cursor/plugins/local/skarn/hooks/hooks.json`, then run `Developer: Reload Window`. The flip command the report prints, `skarn setup --update --agent cursor --mode enforce`, upgrades only the `skarn guard` hooks in `~/.cursor/hooks.json`; on a plugin install it finds no skarn entries there and changes nothing. Enforcement runs on any paid tier. Without one the guard stays in audit, and because Cursor's blocking events have no non-deciding channel, it stays silent apart from the audit log.

## Troubleshooting

If the `skarn` row reads `Error - Show Output`, open Show Output. The last error line in the `MCP: plugin-skarn-skarn` panel names the case:

- `spawn skarn ENOENT`: the binary is not on the PATH Cursor resolved from your login shell. Run `skarn --version` in a terminal; if it works there, put the absolute path in `command`.
- `error: unknown command 'mcp'`: the binary predates the MCP server. Upgrade skarn.
- `[unauthenticated] Error`: you are signed out of Cursor, which stops every plugin MCP server. Sign in.

If the hooks do not fire, confirm the plugin is loaded and that `skarn --version` works. The guard fails open, so a missing binary shows as silence.

## License, privacy, and support

This repository is MIT licensed and carries configuration only; see LICENSE. The `skarn` binary is a separate, closed-source download under the Skarn End User License Agreement at https://getskarn.com/terms/, which running it accepts. What it reads and what stays on your machine: https://getskarn.com/privacy/. Support: hello@getskarn.com. Vulnerability reports: security@getskarn.com; see SECURITY.md.
