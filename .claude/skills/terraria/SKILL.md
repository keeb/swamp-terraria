---
name: terraria
description: Control a Terraria (tModLoader) server running in a Docker container with tmux on a remote host over SSH via the @keeb/terraria swamp extension. Use when working with the `@user/terraria/server` model type, broadcasting in-game shutdown warnings, querying Terraria player count and player names, or collecting Prometheus textfile metrics for game-server dashboards. Triggers on "terraria", "tmodloader", "terraria server", "terraria players", "warnShutdown", "collectMetrics", "@user/terraria/server", "terrariaServer", or editing YAML definitions that reference `@user/terraria/server`.
---

# terraria

`@keeb/terraria` is a swamp extension that controls a Terraria (tModLoader)
server running inside a Docker container with a tmux session attached to the
server console, all over SSH. There is no native Terraria RCON or API — every
operation drives the server by sending keystrokes to tmux inside the container.

## Models

### `@user/terraria/server`

Drive a tModLoader server via `docker exec <container> tmux send-keys ...` on a
remote host.

- **Global arguments**
  - `sshHost` (string, nullable, required) — SSH hostname/IP. Wire from a
    fleet/VM lookup with CEL; if `null`/`"null"`/`"undefined"`/empty, methods
    short-circuit and write a `skipped` / `serverRunning: false` result instead
    of failing.
  - `sshUser` (string, default `root`) — SSH user.
  - `containerName` (string, default `tmodloader`) — Docker container running
    tModLoader.
  - `serverName` (string, default `server`) — instance name used for
    `writeResource` and Prometheus `server="..."` label.
- **Resources**
  - `server` — operation result for `warnShutdown` / `status`. Schema:
    `success`, `skipped`, `serverRunning`, `online`, `max`, `players[]`,
    `timestamp`. `lifetime: infinite`, `garbageCollection: 10`.
  - `metrics` — `collectMetrics` result, same schema as `server`.
- **Methods**
  - `warnShutdown` — Broadcasts
    `say == SERVER SHUTTING DOWN IN 30
    SECONDS ==` three times into the tmux
    pane and then sleeps 30s. No arguments. Skips (with
    `success: true, skipped: true`) if `sshHost` is invalid, the container isn't
    running, or no tmux session exists. Writes a `server` resource named
    `server`.
  - `status` — Sends `playing` to the tmux console, waits 2s, captures the pane
    with `tmux capture-pane -p`, and parses tModLoader's output
    (`": No players connected."` or `"N player(s) connected."` preceded by
    `: name (ip:port)` lines) to populate `online` and `players[]`. No
    arguments. Writes a `server` resource named `server`. `max` is always `null`
    — tModLoader doesn't expose it via this command.
  - `collectMetrics` — Same player-parsing logic as `status`, then calls
    `writeMetricsFiles` (from `lib/metrics.ts`) to write a Prometheus textfile
    at `/var/lib/node_exporter/textfile_collector/game_terraria.prom` and append
    a JSON line to `/var/log/game-players.log` on the **remote host**. No
    arguments. Writes a `metrics` resource named `metrics`. Game type is
    hard-coded to `terraria`; the Prometheus `server` label comes from
    `globalArguments.serverName`.

## Dependencies

Requires `@keeb/ssh` (listed in `manifest.yaml`). SSH helpers (`sshExec`,
`sshExecRaw`, `isValidSshHost`) live in `extensions/models/lib/ssh.ts` and shell
out to the system `ssh` binary with `StrictHostKeyChecking=no`,
`UserKnownHostsFile=/dev/null`, and a 10s connect timeout. The host running
swamp needs an SSH key the target accepts. There is no vault integration — auth
is whatever ssh-agent / key files swamp's host already has.

`collectMetrics` is typically scheduled from the `collect-game-metrics` workflow
shipped by `@keeb/minecraft` — the extension itself ships no workflows. Server
start/stop is handled through `@keeb/docker`'s `compose` model.

## Common patterns

### Creating a terraria server model

```yaml
name: calamityTerraria
type: "@user/terraria/server"
globalArguments:
  sshHost: ${{ data.latest("fleet", "calamity").attributes.ip }}
  sshUser: root
  containerName: tmodloader
  serverName: calamity
```

Wire `sshHost` from a fleet/VM lookup with CEL — never hard-code IPs. Set
`serverName` to the human label you want on Prometheus / Grafana (it becomes
`server="calamity"` in the textfile).

### Graceful restart pattern

```bash
swamp model exec calamityTerraria warnShutdown
# wait for the 30s broadcast to finish, then bounce compose:
swamp model exec calamityCompose stop
swamp model exec calamityCompose start
```

`warnShutdown` blocks for ~30s — chain it before any `docker compose down` so
connected players get an in-game heads-up.

### Reading player state via CEL

```yaml
running: ${{ data.latest("calamityTerraria", "server").attributes.serverRunning }}
online: ${{ data.latest("calamityTerraria", "server").attributes.online }}
players: ${{ data.latest("calamityTerraria", "server").attributes.players }}
```

The `dataName` is always `server` for `warnShutdown` / `status` and `metrics`
for `collectMetrics`, regardless of `serverName`.

### Wiring into `collect-game-metrics`

Add a step to the workflow shipped by `@keeb/minecraft`:

```yaml
- name: collect-calamity
  description: Collect Terraria player metrics from calamity
  task:
    type: model_method
    modelIdOrName: calamityTerraria
    methodName: collectMetrics
  dependsOn:
    - step: sync-fleet
      condition:
        type: succeeded
```

`sync-fleet` populates the VM IP that `calamityTerraria.sshHost` resolves
through CEL.

## Gotchas

- **No RCON, no API — everything is keystrokes.** All commands run as
  `docker exec <container> tmux send-keys "<cmd>" Enter`. If the tmux session
  name isn't the default, or the server console isn't the active pane, the model
  silently does nothing useful. Don't add methods that assume a network
  protocol.
- **`status` and `collectMetrics` sleep 2s after sending `playing`.** This is
  the wait for tModLoader to print its response into the pane before
  `tmux capture-pane -p`. Don't shorten it without testing on a slow server.
- **`max` is always `null`.** tModLoader's `playing` command doesn't expose max
  slots. The Prometheus exporter only emits `game_players_max` when
  `max !== null`, so the metric will be missing — that's by design, not a bug.
- **Methods short-circuit instead of failing.** Invalid `sshHost`, a stopped
  container, or a missing tmux session all log a message and write a `skipped` /
  `serverRunning: false` resource with `success: true`. This is intentional for
  scheduled metrics jobs that must not page when a VM is off. If you need a hard
  failure, check the resource attributes downstream — don't rely on the method
  exit code.
- **Pane parsing walks backwards.** `status` searches `tmux capture-pane` output
  from the bottom for the most recent count line, then collects preceding
  `: name (ip:port)` lines until it has `online` players or hits a non-matching
  line. If a player name contains whitespace or the console is noisy, parsing
  may under-count. Don't "improve" the regex without checking real tModLoader
  output.
- **`collectMetrics` writes files on the remote host, not locally.** The
  Prometheus textfile lands in
  `/var/lib/node_exporter/textfile_collector/game_terraria.prom` on `sshHost`,
  and node_exporter on that host is what scrapes it. The `metrics` resource only
  stores the parsed counts, not the file contents. The directory is created with
  `mkdir -p` and the textfile is written via `tmp → mv` so node_exporter never
  sees a partial file — preserve that pattern.
- **Container name is global, not per-method.** To manage multiple tModLoader
  containers on one host, create one model per container rather than overriding
  `containerName` per call.
- **`sshExec` throws on non-zero exit; `sshExecRaw` does not.** Every
  reachability / container / tmux probe uses `sshExecRaw` so a missing container
  doesn't abort the method. Preserve this distinction when editing.
- **`writeMetricsFiles` uses `sshExec`, which throws.** A failed Prometheus
  write _will_ fail `collectMetrics` (unlike the parsing short-circuits). That's
  deliberate — silently dropping metrics is worse than a paged failure.

## Verification before destructive ops

`warnShutdown` is the only "destructive" method in the sense that it takes ~30
seconds and bothers connected players. Before broadcasting:

```bash
swamp model exec calamityTerraria status
swamp model get calamityTerraria --json
```

Confirm `serverRunning: true` and the player list looks right before firing
`warnShutdown` followed by a compose bounce.
