# @keeb/terraria

[Swamp](https://github.com/systeminit/swamp) extension for Terraria (tModLoader) server control via Docker and tmux over SSH.

## Models

### `terraria/server`

Control a Terraria server running in a Docker container with tmux.

| Method | Description |
|--------|-------------|
| `warnShutdown` | Send in-game countdown warnings before shutdown and wait 30 seconds |
| `status` | Query player count and player names from the tmux pane |
| `collectMetrics` | Collect player count and write Prometheus textfile metrics |

Global arguments: `sshHost`, `sshUser` (default `root`), `containerName` (default `tmodloader`), `serverName` (default `server`).

## Workflows

None — Terraria server start/stop is handled through `docker/compose` workflows. Metrics collection runs via the `collect-game-metrics` workflow from [@keeb/minecraft](https://github.com/keeb/swamp-extensions).

## Dependencies

- [@keeb/ssh](https://github.com/keeb/swamp-extensions) — SSH helpers (`lib/ssh.ts`)

## Install

```bash
swamp extension pull @keeb/terraria
```

## Usage

Warn players before a maintenance window, then read the current player list:

```bash
# Broadcast "SERVER SHUTTING DOWN IN 30 SECONDS" and sleep 30s
swamp model method run my-terraria warnShutdown

# Query current players — writes the "server" resource with online/players
swamp model method run my-terraria status
swamp model get my-terraria --json
```

## License

MIT — see [LICENSE](./LICENSE).
