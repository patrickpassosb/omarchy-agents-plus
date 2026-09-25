# Agents

One bar icon and one panel for every AI coding subscription on the machine.
The panel is strictly a display: it watches the usage records that
`omarchy-agent-usage-update` writes to `~/.local/state/omarchy/agents/usage/`
and draws whatever appears there. `Panel.qml` owns the bar button and the
popup; `Main.qml` discovers and watches the records (and handles the optional
cross-device aggregation); `Agent.qml` is the per-record file watcher.

## Panel

- **Hero** — the mark, the tool, and the plan it runs on ("Max 20x", "Pro").
  Auth and endpoint problems replace the plan line and repeat in a card.
- **Subscription switch** — one chip per enabled agent (`h`/`l` or click).
  It appears only when more than one agent is enabled.
- **Limits** — the percentage of each allowance used, a matching meter, and
  the time until the session or weekly window resets.
- **Models used this week** — request counts per model for the account window,
  heaviest model scaled to full width. Only providers whose collector sources
  request counts show it.
- **Balance** — prepaid agents report a credit ledger instead of limits:
  remaining credit, a fuel-gauge meter that drains toward empty, and
  funded-versus-spent detail.
- **Tokens by day** — one row per day for the last week: day, bar, tokens, with today
  bolded at the bottom. Hover today for its prompt and session count.
- **Tokens by model** — tokens per model with the bar behind each row scaled
  to the heaviest model,
  the same way the weekly chart scales to its busiest day. Hover for the
  input / output / cache split.

A subscription appears only when it is enabled in settings and has actually
recorded usage — on this machine or on a synced one. With one such agent
there is no switch row at all; with none, the module leaves the bar entirely
rather than sitting there with nothing to say. A CLI installed mid-session
shows up at the next refresh, so nothing polls the disk waiting for it.

That self-hiding is why the widget ships in the default bar layout: a machine
that has never run an AI coding agent draws nothing, and the icon arrives on
its own the first time a scan finds usage. Drop it with
`omarchy plugin disable omarchy.agents`.

## Data

Each agent is one JSON record in `~/.local/state/omarchy/agents/usage/`,
written by `omarchy-agent-usage-update`. That command runs one
`omarchy-agent-usage-<agent>` collector per agent; the widget invokes it
on its refresh timer and whenever you ask for a refresh, and picks up any
record that lands in the directory regardless of who wrote it.

Adding an agent therefore never touches this plugin: ship a collector that
prints the record contract (see the `claude` and `codex` collectors in
`bin/`), and the panel gains a tab. An `assets/<id>.svg` mark is optional —
with an `assets/<id>-light.svg` twin if the mark needs a dark variant for
light surfaces — and the bar glyph stands in when there is none.

| Collector | Limits | Local stats |
|---|---|---|
| `claude` | Anthropic's OAuth usage endpoint (5-hour session + 7-day weekly) | `~/.claude/projects` transcripts, opencode sessions on an Anthropic provider, plus `stats-cache.json` and `history.jsonl` as fallback |
| `codex` | The Codex app-server RPC | native Codex CLI session files (plus pi and opencode sessions) |
| `fireworks` | Estimated prepaid balance: configured funding minus rated account costs | Fireworks billing API, grouped by day and model for the last 30 days |

Claude limits need a signed-in CLI; without credentials the panel says so and
falls back to local stats only. A non-default Claude directory is honored via
`CLAUDE_CONFIG_DIR`, Codex via `CODEX_HOME`. Fireworks reads
`FIREWORKS_API_KEY` and `FIREWORKS_ACCOUNT_ID` first, then
`~/.fireworks/auth.ini` (which `firectl set-api-key` creates), then the key
opencode stores in `~/.local/share/opencode/auth.json` when Fireworks is
signed in there.

### Fireworks balance

The collector first asks the account's `:getBalance` endpoint for the real
prepaid ledger. That endpoint exists but is permission-gated, and as of
August 2026 no console-issued API key passes it — Fireworks appears to
reserve it for the dashboard session. The probe stays because it is cheap
and the live figure lights up automatically if Fireworks ever opens it to
keys. Until then the collector falls back to estimating the balance from
configuration in `~/.config/omarchy/agents/fireworks.json`:

```json
{
  "accountId": "",
  "fundedAmount": 20,
  "fundedAt": "2026-07-01"
}
```

Set `fundedAmount` to the credits purchased and optionally `fundedAt` to the
purchase date; with no date, the collector uses the account creation time. It
subtracts rated account costs and the panel labels the result as estimated.
For a later top-up, increase `fundedAmount` by the new credit while keeping
the original `fundedAt`, so both the funding and spend still cover the same
period. `accountId` only matters when one API key can access several
accounts. Without a configured `fundedAmount` the tab still shows token
usage, just no balance. With a live ledger, `fundedAmount` is optional and
only adds the meter and the spent-of-funded line under the real figure.

## Interactions

- Bar icon: left = panel, right = launch agent, middle = next subscription.
- Panel: `h`/`l` switch subscription, `j`/`k` scroll, `r` or Enter refresh,
  Tab moves to the neighboring bar panel, Esc closes.
- IPC: `omarchy-shell omarchy.agents <open|close|toggle|refresh|next>`.

## Settings

Settings live in the widget's entry in `~/.config/omarchy/shell.json`. The
top-level keys can be set with
`omarchy bar set omarchy.agents <key> <value>`:

| Key | Default | What it does |
|---|---|---|
| `refreshIntervalSec` | `900` | How often the usage records regenerate |
| `syncMode` | `"Off"` | `"On"` writes this machine's snapshot and merges the others |
| `syncDir` | `""` | A folder synced by Syncthing, Dropbox, rsync, … |
| `syncFileName` | `<hostname>.json` | This machine's snapshot file |
| `syncDeviceId` | hostname | Stable device name inside the snapshot |

Numbers need `--json`, or they land in `shell.json` as strings:

```bash
omarchy bar set omarchy.agents refreshIntervalSec 300 --json
omarchy bar set omarchy.agents syncDir '~/Sync/agent-usage'
```

Per-agent enablement is nested, and `set` writes its key literally rather
than walking a dotted path — so pass the whole `providers` object as JSON (or
edit `shell.json` directly):

```bash
omarchy bar set omarchy.agents providers '{
  "claude": { "enabled": true },
  "codex": { "enabled": false },
  "fireworks": { "enabled": true }
}' --json
```

`enabled` defaults to `true` for every discovered agent; set it to `false` to
hide a subscription that is installed. Disabled agents are also skipped when
the records regenerate.

With `syncMode` on, every `*.json` snapshot in `syncDir` is merged, so today,
the last 7 days, and the all-time totals cover every machine you code on —
active days are unioned by date rather than summed. Rate limits stay
per-account and are never merged. A record may declare `"scope": "account"`
when its stats are account-global rather than machine-local (Fireworks'
billing API); those merge by taking the widest value instead of summing, so
the same account synced from two machines is not counted twice.

One caveat on "all-time": the Codex collector only reads native session files
touched in the last 30 days, and Fireworks requests the last 30 days from its
billing API, so their totals and day counts cover that window. Claude's cover
every transcript still on disk.

## This clone's divergences from `omarchy.agents`

This is a fork (`omarchy plugin clone omarchy.agents` → `patrickpassos.agents`),
kept for these changes. An Omarchy update does **not** reach it, so re-apply
them after a notable stock panel change:

1. **Percentages show one decimal** (`18.9%` instead of `19%`) so they match the
   provider dashboards (`Panel.qml`, the limit value text).
2. **`MODELS USED THIS WEEK`** renders per-model request counts. The stock
   record contract has no field for them, so the adapter carries them on the
   limit row they belong to:

   ```json
   { "label": "Weekly (7-day)", "title": "Weekly", "percent": 0.65,
     "requestModels": [{ "name": "glm-5.3", "requests": 1794 }] }
   ```

   `requestModelRows()` prefers the row titled `Weekly`, falls back to any row
   that carries the key, and renders the top four through the existing
   `ModelRow` (own `valueText` and `tooltip`, because the row counts requests
   and not tokens). The stock panel ignores unknown limit-row keys, so this
   stays compatible with an unforked record.
3. **Chip labels come from the collectors, not the panel.** Twelve providers
   in the switch row made every label a truncated stub, so the row is kept
   usable from the data side instead of with layout surgery: manifests carry
   short names (`Ollama`, `All Ollama`, `Grok`), and an aggregate tab can
   stand in for a family of accounts (one meter per account in a single
   record's limits array — the panel already draws that). Detail tabs stay
   installed and are hidden with `providers.<id>.enabled: false` when the row
   gets crowded.
4. **Derived countdowns are marked.** A limit row may set
   `"resetsEstimated": true` when the reset time was inferred rather than
   given by the provider; that row renders `Resets in ~2h 9m` so an estimate
   never reads as exact.

The Ollama adapter uses both (2) and (3); see
`~/.config/omarchy/agent-collectors/adapters/ollama/` and the vault skill
`Skills/omarchy-agent-panel-usage-chip/`.

Editing any `.qml` here needs a shell restart (`omarchy restart shell`) — record
files hot-reload, QML does not.

## Provider keys

The stock collectors read their keys from the environment, and this plugin does
not change that. One case needs a nudge: the Fireworks collector wants
`FIREWORKS_API_KEY` in its environment, and when that key lives in a `KEY=value`
secrets file instead, opening the panel would collect nothing from Fireworks.
`updateCommand` therefore runs the stock update through a shell that reads that
one variable out of

    ${AGENT_SECRETS_FILE:-~/.config/agent-secrets/.env}

and exports it to that child process only — never printed, never copied
elsewhere — and prints a line to stderr when it finds no key, so a silent
Fireworks never happens quietly. On a machine without that file the wrapper is a
harmless no-op and the stock order still applies (`FIREWORKS_API_KEY` /
`FIREWORKS_ACCOUNT_ID`, then `~/.fireworks/auth.ini`, then opencode's
`auth.json`). Set `AGENT_SECRETS_FILE` or export the key in the session
environment (Hyprland `env =`, `environment.d`) to point it at your own layout.

The collector also needs `accountId` in `~/.config/omarchy/agents/fireworks.json`
for its billing calls to resolve — without it the record says "invalid billing
response". A prepaid meter additionally needs `fundedAmount` (and `fundedAt`)
there, because the real balance endpoint is permission-gated for
console-issued keys.
