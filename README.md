# @keeb/terraria

[Swamp](https://github.com/systeminit/swamp) extension for Terraria server control via Docker and tmux.

## Models

### `terraria/server`

Control a Terraria server running in a Docker container with tmux.

| Method | Description |
|--------|-------------|
| `warnShutdown` | Send in-game countdown warnings before shutdown |
| `status` | Query player count and server status |
| `collectMetrics` | Collect player count and write Prometheus textfile metrics |

## Workflows

None — Terraria server start/stop is handled through `docker/compose` workflows. Metrics collection runs via the `collect-game-metrics` workflow from [@keeb/minecraft](https://github.com/keeb/swamp-minecraft).

## Dependencies

- [@keeb/ssh](https://github.com/keeb/swamp-ssh) — SSH helpers (`lib/ssh.ts`)

## Install

```bash
swamp extension pull @keeb/terraria
```

## License

MIT
