# Skarn for Cursor

A Cursor plugin that carries three things: the `skarn-audit` skill, the pre-execution guard hooks in audit mode, and a declaration for the local `skarn` MCP server. It carries no binary, no detection rules, and no detection engine. Install the `skarn` binary separately; everything here invokes it from your PATH.

## Install the binary first

Pick one:

```sh
brew install skarn-security/tap/skarn
```

```sh
npm install -g @skarn-security/skarn
```

Or download the release for your platform from https://github.com/skarn-security/skarn-dist/releases/latest and put it on your PATH. Confirm it with `skarn --version`.

## Install the plugin

Once the listing is approved, install it from the Cursor Marketplace. Until then, and for development, put this repository where Cursor reads local plugins:

```sh
mkdir -p ~/.cursor/plugins/local
git clone https://github.com/skarn-security/cursor-plugin ~/.cursor/plugins/local/skarn
```

If you already have a checkout, copy it there instead; a symlink does not work, because Cursor refuses a local plugin whose symlink target lies outside `~/.cursor/plugins/local` (measured on Cursor 3.17.19, which logs `loadUserLocalPlugin skarn rejected: symlink target ... is outside`):

```sh
mkdir -p ~/.cursor/plugins/local
cp -R /path/to/cursor-plugin ~/.cursor/plugins/local/skarn
```

The `local` directory does not exist on a profile that has never installed a local plugin, which is why both forms create it first.

Then run `Developer: Reload Window`. Settings > Tools & MCPs lists `skarn` under Plugin MCP Servers as `4 tools enabled`, and its Configure dialog shows Local `Connected` with `scan_sessions`, `vet_configs`, `list_sessions`, and `session_stats`. Cursor loads plugin MCP servers only while you are signed in to a Cursor account: signed out, every plugin server (not only this one) shows `Error - Show Output` with `[unauthenticated] Error` in the output panel, before the binary is ever looked for.

## The MCP server

The plugin declares one stdio server, which is the whole of `mcp.json`:

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

It runs on your machine, makes no network call, and exposes four read-only tools: `scan_sessions` (findings from past sessions, every previewed value redacted), `vet_configs` (a masked report on your assistant configuration surface), `list_sessions` and `session_stats` (metadata and aggregates, never message content).

The recall skill is not part of this plugin.

If you would rather not install the binary yourself, a version-pinned npx form works:

```json
{
  "mcpServers": {
    "skarn": {
      "command": "npx",
      "args": ["-y", "@skarn-security/skarn@0.26.0", "mcp"]
    }
  }
}
```

npx downloads the package the first time the server starts and caches it afterwards, so the first start is slower and needs network access. Pin the version. The unpinned form, `@skarn-security/skarn` with no `@X.Y.Z`, resolves to whatever the registry serves at start time, and `skarn vet` reports it as `vet-mcp-unpinned`. Do not use it.

## The guard hooks

Before Cursor submits a prompt, runs a shell command, calls an MCP tool, or reads a file, `hooks/hooks.json` runs `skarn guard --agent cursor`, which scans the action with the full Skarn detection engine and returns a verdict:

- **deny** - the action leaks a secret into an exfiltration channel (including a secret pasted into a prompt, which heads to the model provider), installs a likely malicious or typosquatted package, completes a multi-phase attack chain, or trips a critical rule.
- **ask** - a secret is present but the destination is ambiguous (Cursor prompts the user; shell and MCP only).
- **allow** - nothing dangerous detected (emits nothing; the action proceeds).

The headline event is `beforeSubmitPrompt`: Skarn scans the prompt text and blocks it before it leaves the machine if it carries a credential. The prompt block is recoverable; the user sees the redacted reason and can edit and resubmit.

The shipped hooks register `beforeSubmitPrompt`, `beforeShellExecution`, `beforeMCPExecution`, and `beforeReadFile` in **audit mode**, which logs the would-be verdict and never changes what Cursor does.

### Without the plugin

`skarn setup --agent cursor` is the other route. It merges the same guard hook into an existing `~/.cursor/hooks.json` without touching foreign hooks, backs the file up first, writes the absolute binary path, and ends with the guard self-test. `skarn setup --agent cursor --scope project` writes `<repo>/.cursor/hooks.json` instead, and `--print` shows the config without writing anything. Cursor reloads `hooks.json` on save, so no restart is needed.

### Audit log

Each flagged action appends one redacted JSONL record to `~/.cache/skarn/guard-cursor-audit.jsonl`: `ts`, `mode`, `agent`, `event`, `verdict`, `tool`, `rule`, `severity`, `fingerprint`, `incidents`, `chain`, `enforceable`, redacted `reason`, `session`, `cwd`, `latency_ms`. Clean actions are not logged, and the reason is redacted rather than carrying the raw secret. Review a real window before flipping any machine to enforce:

```sh
skarn guard report --agent cursor --window 30d
```

It summarizes the flagged actions and prints the exact flip command once the window spans at least five days with zero denies. The counts are counts of flagged actions, not rates over all traffic, because a clean action leaves no record. For the raw records behind a finding:

```sh
jq -c 'select(.verdict=="deny")' ~/.cache/skarn/guard-cursor-audit.jsonl
```

A deny whose top finding carries a secret shows that finding's fingerprint and the command that accepts it. A secret-less deny (a structural egress detector, a typosquat, a chain) carries no fingerprint and no accept line. When a fingerprint is shown:

```sh
skarn guard accept <fingerprint> --reason 'internal test credential'
```

That records it as a false positive in the personal accepted-findings baseline (`~/.config/skarn/baseline.json`, the same file `skarn check` honors), and the identical call stops being flagged. A confirmed real leak is a different label: `skarn baseline audit` records it as a true positive, which is never suppressed, so the guard keeps blocking it until the credential is rotated.

### Flip one machine to enforce

When a machine's audit window looks clean, run `skarn setup --agent cursor --update --mode enforce`, or edit `--guard-mode audit` to `--guard-mode enforce` in that machine's `~/.cursor/hooks.json` for all four entries. Cursor reloads on save. An update replaces the skarn-owned entries whole, with one exception: a `timeout` you tuned yourself on such an entry is carried across when it pairs unambiguously with its replacement and the value is sane, so a per-machine budget survives the refresh. Any other field you added to a skarn-owned entry is dropped, and a field a newer template no longer writes stays gone.

Enforce applies `deny` and `ask`, so the prompt, shell, MCP, and read actions above are blocked rather than logged. Enforce needs a registered license; without one the guard is forced to audit-only. Cursor's blocking events have no non-deciding advisory channel, so an unlicensed Cursor guard stays silent and relies on the audit log.

### Verdict schema per event

Cursor's verdict schema is not uniform, so Skarn emits the right shape per event:

- `beforeShellExecution` and `beforeMCPExecution` produce `{"permission":"deny"|"ask","user_message":...,"agent_message":...}`
- `beforeReadFile` produces `{"permission":"deny",...}`, with no `ask` channel; it is evaluated only under the `--strict` flag already set on that entry, for credential-file recon
- `beforeSubmitPrompt` produces `{"continue":false,"user_message":...}`, with no `permission` and no `ask`; it blocks submission

### Safety

- **Fail-open**: a malformed event, an out-of-scope action, or any error yields allow, so the guard never bricks Cursor. The only fail-closed path is `--strict`, and Cursor's own `failClosed: true`, which you can add per entry for high-assurance machines.
- **Redacted**: verdict reasons name the rule and a redacted preview, never the raw secret.
- **Local**: the guard makes no network call.

### Limitations

The hooks are macOS and Linux only. Each `command` carries a POSIX `VAR=value cmd` environment prefix, which neither PowerShell nor `cmd.exe` accepts, so on native Windows all four entries fail before `skarn guard` starts and Cursor's fail-open behavior lets the action through unscanned. Cursor is not a validated native-Windows guard target; Codex CLI is the verified one, through the Codex plugin at https://github.com/skarn-security/agent-guard. The MCP server and the skill in this plugin are unaffected and work on every platform.

Each action is evaluated independently; cross-action chain state is the planned daemon. Cursor's pre-execution blocking set has no file-write gate, since `afterFileEdit` is post-execution and advisory, so write-to-disk persistence is not blocked through these hooks. `beforeReadFile` is high-volume, because it fires on every file read; if the overhead is noticeable, remove that entry, since the prompt, shell, and MCP gates carry most of the leak-prevention value.

Every event this host fires, what it scans, how it blocks, and whether it is wired by default: run `man skarn-guard`, or read https://getskarn.com/manual/.

## Troubleshooting

**The MCP server does not start, or the `skarn` row reads `Error - Show Output`.** Show Output opens the `MCP: plugin-skarn-skarn` panel, and its last error line says which case you are in (all three measured on Cursor 3.17.19, macOS):

- `spawn skarn ENOENT`: the binary is not on the PATH Cursor resolved. Cursor reads your login shell's PATH even when launched from the Dock, so `/opt/homebrew/bin` is normally visible; run `skarn --version` in a terminal, and if that fails install the binary. If it works but Cursor still cannot find it, put the absolute path in `command`.
- `error: unknown command 'mcp'` followed by `Run 'skarn --help' for usage.`: the binary Cursor found predates the MCP server. Upgrade skarn.
- `[unauthenticated] Error`: you are signed out of Cursor. Sign in; this is Cursor's gate on every plugin MCP server, not a skarn condition.

**The hooks do not fire.** Confirm the plugin is loaded, then check that `skarn --version` works from the same PATH, as above. The guard fails open, so a missing binary shows up as silence rather than an error.

## Reporting a vulnerability

See SECURITY.md in this repository.

## License, privacy, and support

This repository is MIT licensed; see LICENSE. It carries configuration only, so that covers the two manifests, the hooks, the skill, and the MCP server declaration in it.

The `skarn` binary those files invoke is a separate download and is not open source. It is licensed under the Skarn End User License Agreement at https://getskarn.com/terms/, and running it accepts that agreement. What it reads, what it keeps, and what it never sends anywhere: https://getskarn.com/privacy/.

Support: hello@getskarn.com. Vulnerability reports go to security@getskarn.com; see SECURITY.md in this repository.
