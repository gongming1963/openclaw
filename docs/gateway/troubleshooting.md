---
summary: "Deep troubleshooting runbook for gateway, channels, automation, nodes, and browser"
read_when:
  - The troubleshooting hub pointed you here for deeper diagnosis
  - You need stable symptom based runbook sections with exact commands
title: "Troubleshooting"
---

# Gateway troubleshooting

This page is the deep runbook.
Start at [/help/troubleshooting](/help/troubleshooting) if you want the fast triage flow first.

## Command ladder

Provider-specific shortcuts: [/channels/troubleshooting](/channels/troubleshooting)

## Status & Diagnostics

Quick triage commands (in order):

| Command                            | What it tells you                                                                                      | When to use it                                    |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `openclaw status`                  | Local summary: OS + update, gateway reachability/mode, service, agents/sessions, provider config state | First check, quick overview                       |
| `openclaw status --all`            | Full local diagnosis (read-only, pasteable, safe-ish) incl. log tail                                   | When you need to share a debug report             |
| `openclaw status --deep`           | Runs gateway health checks (incl. provider probes; requires reachable gateway)                         | When “configured” doesn’t mean “working”          |
| `openclaw gateway probe`           | Gateway discovery + reachability (local + remote targets)                                              | When you suspect you’re probing the wrong gateway |
| `openclaw channels status --probe` | Asks the running gateway for channel status (and optionally probes)                                    | When gateway is reachable but channels misbehave  |
| `openclaw gateway status`          | Supervisor state (launchd/systemd/schtasks), runtime PID/exit, last gateway error                      | When the service “looks loaded” but nothing runs  |
| `openclaw logs --follow`           | Live logs (best signal for runtime issues)                                                             | When you need the actual failure reason           |

**Sharing output:** prefer `openclaw status --all` (it redacts tokens). If you paste `openclaw status`, consider setting `OPENCLAW_SHOW_SECRETS=0` first (token previews).

See also: [Health checks](/gateway/health) and [Logging](/logging).

## Common Issues

### "Unauthorized: gateway token mismatch" (Error 1008)

This means the Gateway is running with a configured token (`gateway.auth.token` in `openclaw.json` or `OPENCLAW_GATEWAY_TOKEN`), but the client (Dashboard or CLI) provided a different token (or none).

**Fix:**

1. **Check the configured token on the Gateway host:**

   ```bash
   # Look for gateway.auth.token
   cat ~/.openclaw/openclaw.json
   # Or check env var
   echo $OPENCLAW_GATEWAY_TOKEN
   ```

2. **Connect with the correct token:**
   - **Dashboard:** Open `http://localhost:18789/?token=<YOUR_TOKEN>`
   - **CLI:** `openclaw gateway status --token <YOUR_TOKEN>`
   - **Remote:** Ensure your client/runner uses the matching token.

### No API key found for provider "anthropic"

This means the **agent’s auth store is empty** or missing Anthropic credentials.
Auth is **per agent**, so a new agent won’t inherit the main agent’s keys.

Fix options:

- Re-run onboarding and choose **Anthropic** for that agent.
- Or paste a setup-token on the **gateway host**:
  ```bash
  openclaw models auth setup-token --provider anthropic
  ```
- Or copy `auth-profiles.json` from the main agent dir to the new agent dir.

Verify:

```bash
openclaw models status
```

### OAuth token refresh failed (Anthropic Claude subscription)

This means the stored Anthropic OAuth token expired and the refresh failed.
If you’re on a Claude subscription (no API key), the most reliable fix is to
switch to a **Claude Code setup-token** and paste it on the **gateway host**.

**Recommended (setup-token):**

```bash
# Run on the gateway host (paste the setup-token)
openclaw models auth setup-token --provider anthropic
openclaw models status
```

If you generated the token elsewhere:

```bash
openclaw models auth paste-token --provider anthropic
openclaw models status
```

More detail: [Anthropic](/providers/anthropic) and [OAuth](/concepts/oauth).

### Control UI fails on HTTP ("device identity required" / "connect failed")

If you open the dashboard over plain HTTP (e.g. `http://<lan-ip>:18789/` or
`http://<tailscale-ip>:18789/`), the browser runs in a **non-secure context** and
blocks WebCrypto, so device identity can’t be generated.

**Fix:**

- Prefer HTTPS via [Tailscale Serve](/gateway/tailscale).
- Or open locally on the gateway host: `http://127.0.0.1:18789/`.
- If you must stay on HTTP, enable `gateway.controlUi.allowInsecureAuth: true` and
  use a gateway token (token-only; no device identity/pairing). See
  [Control UI](/web/control-ui#insecure-http).

### CI Secrets Scan Failed

This means `detect-secrets` found new candidates not yet in the baseline.
Follow [Secret scanning](/gateway/security#secret-scanning-detect-secrets).

### Service Installed but Nothing is Running

If the gateway service is installed but the process exits immediately, the service
can appear “loaded” while nothing is running.

**Check:**

```bash
openclaw gateway status
openclaw doctor
```

Doctor/service will show runtime state (PID/last exit) and log hints.

**Logs:**

- Preferred: `openclaw logs --follow`
- File logs (always): `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (or your configured `logging.file`)
- macOS LaunchAgent (if installed): `$OPENCLAW_STATE_DIR/logs/gateway.log` and `gateway.err.log`
- Linux systemd (if installed): `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`
- Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`

**Enable more logging:**

- Bump file log detail (persisted JSONL):
  ```json
  { "logging": { "level": "debug" } }
  ```
- Bump console verbosity (TTY output only):
  ```json
  { "logging": { "consoleLevel": "debug", "consoleStyle": "pretty" } }
  ```
- Quick tip: `--verbose` affects **console** output only. File logs remain controlled by `logging.level`.

See [/logging](/logging) for a full overview of formats, config, and access.

### "Gateway start blocked: set gateway.mode=local"

This means the config exists but `gateway.mode` is unset (or not `local`), so the
Gateway refuses to start.

**Fix (recommended):**

- Run the wizard and set the Gateway run mode to **Local**:
  ```bash
  openclaw configure
  ```
- Or set it directly:
  ```bash
  openclaw config set gateway.mode local
  ```

**If you meant to run a remote Gateway instead:**

- Set a remote URL and keep `gateway.mode=remote`:
  ```bash
  openclaw config set gateway.mode remote
  openclaw config set gateway.remote.url "wss://gateway.example.com"
  ```

**Ad-hoc/dev only:** pass `--allow-unconfigured` to start the gateway without
`gateway.mode=local`.

**No config file yet?** Run `openclaw setup` to create a starter config, then rerun
the gateway.

### Service Environment (PATH + runtime)

The gateway service runs with a **minimal PATH** to avoid shell/manager cruft:

- macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
- Linux: `/usr/local/bin`, `/usr/bin`, `/bin`

This intentionally excludes version managers (nvm/fnm/volta/asdf) and package
managers (pnpm/npm) because the service does not load your shell init. Runtime
variables like `DISPLAY` should live in `~/.openclaw/.env` (loaded early by the
gateway).
Exec runs on `host=gateway` merge your login-shell `PATH` into the exec environment,
so missing tools usually mean your shell init isn’t exporting them (or set
`tools.exec.pathPrepend`). See [/tools/exec](/tools/exec).

WhatsApp + Telegram channels require **Node**; Bun is unsupported. If your
service was installed with Bun or a version-managed Node path, run `openclaw doctor`
to migrate to a system Node install.

### Skill missing API key in sandbox

**Symptom:** Skill works on host but fails in sandbox with missing API key.

**Why:** sandboxed exec runs inside Docker and does **not** inherit host `process.env`.

**Fix:**

- set `agents.defaults.sandbox.docker.env` (or per-agent `agents.list[].sandbox.docker.env`)
- or bake the key into your custom sandbox image
- then run `openclaw sandbox recreate --agent <id>` (or `--all`)

### Service Running but Port Not Listening

If the service reports **running** but nothing is listening on the gateway port,
the Gateway likely refused to bind.

**What "running" means here**

- `Runtime: running` means your supervisor (launchd/systemd/schtasks) thinks the process is alive.
- `RPC probe` means the CLI could actually connect to the gateway WebSocket and call `status`.
- Always trust `Probe target:` + `Config (service):` as the “what did we actually try?” lines.

**Check:**

- `gateway.mode` must be `local` for `openclaw gateway` and the service.
- If you set `gateway.mode=remote`, the **CLI defaults** to a remote URL. The service can still be running locally, but your CLI may be probing the wrong place. Use `openclaw gateway status` to see the service’s resolved port + probe target (or pass `--url`).
- `openclaw gateway status` and `openclaw doctor` surface the **last gateway error** from logs when the service looks running but the port is closed.
- Non-loopback binds (`lan`/`tailnet`/`custom`, or `auto` when loopback is unavailable) require auth:
  `gateway.auth.token` (or `OPENCLAW_GATEWAY_TOKEN`).
- `gateway.remote.token` is for remote CLI calls only; it does **not** enable local auth.
- `gateway.token` is ignored; use `gateway.auth.token`.

**If `openclaw gateway status` shows a config mismatch**

- `Config (cli): ...` and `Config (service): ...` should normally match.
- If they don’t, you’re almost certainly editing one config while the service is running another.
- Fix: rerun `openclaw gateway install --force` from the same `--profile` / `OPENCLAW_STATE_DIR` you want the service to use.

**If `openclaw gateway status` reports service config issues**

- The supervisor config (launchd/systemd/schtasks) is missing current defaults.
- Fix: run `openclaw doctor` to update it (or `openclaw gateway install --force` for a full rewrite).

**If `Last gateway error:` mentions “refusing to bind … without auth”**

- You set `gateway.bind` to a non-loopback mode (`lan`/`tailnet`/`custom`, or `auto` when loopback is unavailable) but didn’t configure auth.
- Fix: set `gateway.auth.mode` + `gateway.auth.token` (or export `OPENCLAW_GATEWAY_TOKEN`) and restart the service.

**If `openclaw gateway status` says `bind=tailnet` but no tailnet interface was found**

- The gateway tried to bind to a Tailscale IP (100.64.0.0/10) but none were detected on the host.
- Fix: bring up Tailscale on that machine (or change `gateway.bind` to `loopback`/`lan`).

**If `Probe note:` says the probe uses loopback**

- That’s expected for `bind=lan`: the gateway listens on `0.0.0.0` (all interfaces), and loopback should still connect locally.
- For remote clients, use a real LAN IP (not `0.0.0.0`) plus the port, and ensure auth is configured.

### Address Already in Use (Port 18789)

This means something is already listening on the gateway port.

**Check:**

```bash
openclaw gateway status
```

It will show the listener(s) and likely causes (gateway already running, SSH tunnel).
If needed, stop the service or pick a different port.

### Extra Workspace Folders Detected

If you upgraded from older installs, you might still have `~/openclaw` on disk.
Multiple workspace directories can cause confusing auth or state drift because
only one workspace is active.

**Fix:** keep a single active workspace and archive/remove the rest. See
[Agent workspace](/concepts/agent-workspace#extra-workspace-folders).

### Main chat running in a sandbox workspace

Symptoms: `pwd` or file tools show `~/.openclaw/sandboxes/...` even though you
expected the host workspace.

**Why:** `agents.defaults.sandbox.mode: "non-main"` keys off `session.mainKey` (default `"main"`).
Group/channel sessions use their own keys, so they are treated as non-main and
get sandbox workspaces.

**Fix options:**

- If you want host workspaces for an agent: set `agents.list[].sandbox.mode: "off"`.
- If you want host workspace access inside sandbox: set `workspaceAccess: "rw"` for that agent.

### "Agent was aborted"

The agent was interrupted mid-response.

**Causes:**

- User sent `stop`, `abort`, `esc`, `wait`, or `exit`
- Timeout exceeded
- Process crashed

**Fix:** Just send another message. The session continues.

### "Agent failed before reply: Unknown model: anthropic/claude-haiku-3-5"

OpenClaw intentionally rejects **older/insecure models** (especially those more
vulnerable to prompt injection). If you see this error, the model name is no
longer supported.

**Fix:**

- Pick a **latest** model for the provider and update your config or model alias.
- If you’re unsure which models are available, run `openclaw models list` or
  `openclaw models scan` and choose a supported one.
- Check gateway logs for the detailed failure reason.

See also: [Models CLI](/cli/models) and [Model providers](/concepts/model-providers).

### Messages Not Triggering

**Check 1:** Is the sender allowlisted?

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Expected healthy signals:

- `openclaw gateway status` shows `Runtime: running` and `RPC probe: ok`.
- `openclaw doctor` reports no blocking config/service issues.
- `openclaw channels status --probe` shows connected/ready channels.

## No replies

If channels are up but nothing answers, check routing and policy before reconnecting anything.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list <channel>
openclaw config get channels
openclaw logs --follow
```

Look for:

- Pairing pending for DM senders.
- Group mention gating (`requireMention`, `mentionPatterns`).
- Channel/group allowlist mismatches.

Common signatures:

- `drop guild message (mention required` → group message ignored until mention.
- `pairing request` → sender needs approval.
- `blocked` / `allowlist` → sender/channel was filtered by policy.

Related:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/pairing](/channels/pairing)
- [/channels/groups](/channels/groups)

## Dashboard control ui connectivity

When dashboard/control UI will not connect, validate URL, auth mode, and secure context assumptions.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Look for:

- Correct probe URL and dashboard URL.
- Auth mode/token mismatch between client and gateway.
- HTTP usage where device identity is required.

Common signatures:

- `device identity required` → non-secure context or missing device auth.
- `unauthorized` / reconnect loop → token/password mismatch.
- `gateway connect failed:` → wrong host/port/url target.

Related:

- [/web/control-ui](/web/control-ui)
- [/gateway/authentication](/gateway/authentication)
- [/gateway/remote](/gateway/remote)

## Gateway service not running

Use this when service is installed but process does not stay up.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep
```

Look for:

- `Runtime: stopped` with exit hints.
- Service config mismatch (`Config (cli)` vs `Config (service)`).
- Port/listener conflicts.

Common signatures:

- `Gateway start blocked: set gateway.mode=local` → local gateway mode is not enabled.
- `refusing to bind gateway ... without auth` → non-loopback bind without token/password.
- `another gateway instance is already listening` / `EADDRINUSE` → port conflict.

Related:

- [/gateway/background-process](/gateway/background-process)
- [/gateway/configuration](/gateway/configuration)
- [/gateway/doctor](/gateway/doctor)

## Channel connected messages not flowing

If channel state is connected but message flow is dead, focus on policy, permissions, and channel specific delivery rules.

```bash
openclaw channels status --probe
openclaw pairing list <channel>
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Look for:

- DM policy (`pairing`, `allowlist`, `open`, `disabled`).
- Group allowlist and mention requirements.
- Missing channel API permissions/scopes.

Common signatures:

- `mention required` → message ignored by group mention policy.
- `pairing` / pending approval traces → sender is not approved.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → channel auth/permissions issue.

Related:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/whatsapp](/channels/whatsapp)
- [/channels/telegram](/channels/telegram)
- [/channels/discord](/channels/discord)

## Cron and heartbeat delivery

If cron or heartbeat did not run or did not deliver, verify scheduler state first, then delivery target.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Look for:

- Cron enabled and next wake present.
- Job run history status (`ok`, `skipped`, `error`).
- Heartbeat skip reasons (`quiet-hours`, `requests-in-flight`, `alerts-disabled`).

Common signatures:

- `cron: scheduler disabled; jobs will not run automatically` → cron disabled.
- `cron: timer tick failed` → scheduler tick failed; check file/log/runtime errors.
- `heartbeat skipped` with `reason=quiet-hours` → outside active hours window.
- `heartbeat: unknown accountId` → invalid account id for heartbeat delivery target.

Related:

- [/automation/troubleshooting](/automation/troubleshooting)
- [/automation/cron-jobs](/automation/cron-jobs)
- [/gateway/heartbeat](/gateway/heartbeat)

## Node paired tool fails

If a node is paired but tools fail, isolate foreground, permission, and approval state.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Look for:

- Node online with expected capabilities.
- OS permission grants for camera/mic/location/screen.
- Exec approvals and allowlist state.

Common signatures:

- `NODE_BACKGROUND_UNAVAILABLE` → node app must be in foreground.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → missing OS permission.
- `SYSTEM_RUN_DENIED: approval required` → exec approval pending.
- `SYSTEM_RUN_DENIED: allowlist miss` → command blocked by allowlist.

Related:

- [/nodes/troubleshooting](/nodes/troubleshooting)
- [/nodes/index](/nodes/index)
- [/tools/exec-approvals](/tools/exec-approvals)

## Browser tool fails

Use this when browser tool actions fail even though the gateway itself is healthy.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Look for:

- Valid browser executable path.
- CDP profile reachability.
- Extension relay tab attachment for `profile="chrome"`.

Common signatures:

- `Failed to start Chrome CDP on port` → browser process failed to launch.
- `browser.executablePath not found` → configured path is invalid.
- `Chrome extension relay is running, but no tab is connected` → extension relay not attached.
- `Browser attachOnly is enabled ... not reachable` → attach-only profile has no reachable target.

Related:

- [/tools/browser-linux-troubleshooting](/tools/browser-linux-troubleshooting)
- [/tools/chrome-extension](/tools/chrome-extension)
- [/tools/browser](/tools/browser)

## If you upgraded and something suddenly broke

Most post-upgrade breakage is config drift or stricter defaults now being enforced.

### 1) Auth and URL override behavior changed

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

What to check:

- If `gateway.mode=remote`, CLI calls may be targeting remote while your local service is fine.
- Explicit `--url` calls do not fall back to stored credentials.

Common signatures:

- `gateway connect failed:` → wrong URL target.
- `unauthorized` → endpoint reachable but wrong auth.

### 2) Bind and auth guardrails are stricter

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

What to check:

- Non-loopback binds (`lan`, `tailnet`, `custom`) need auth configured.
- Old keys like `gateway.token` do not replace `gateway.auth.token`.

Common signatures:

- `refusing to bind gateway ... without auth` → bind+auth mismatch.
- `RPC probe: failed` while runtime is running → gateway alive but inaccessible with current auth/url.

### 3) Pairing and device identity state changed

```bash
openclaw devices list
openclaw pairing list <channel>
openclaw logs --follow
openclaw doctor
```

What to check:

- Pending device approvals for dashboard/nodes.
- Pending DM pairing approvals after policy or identity changes.

Common signatures:

- `device identity required` → device auth not satisfied.
- `pairing required` → sender/device must be approved.

If the service config and runtime still disagree after checks, reinstall service metadata from the same profile/state directory:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Related:

- [/gateway/pairing](/gateway/pairing)
- [/gateway/authentication](/gateway/authentication)
- [/gateway/background-process](/gateway/background-process)
