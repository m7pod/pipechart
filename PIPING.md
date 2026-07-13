# piping — Network Latency Monitor

> Live ASCII bar & sparkline charts for ping RTT, jitter, and packet loss.
> Pipes into [pipechart](./README.md). No dependencies beyond bash + ping.

---

## Quick start

```bash
# Single host — bar chart, 5-line bars
./piping google.com | ./pipechart -n 60 -c 1 -h 5

# Two hosts side by side
./piping google.com cloudflare.com | ./pipechart -n 60 -c 2 -h 5

# Sparkline mode (compact)
./piping google.com | ./pipechart -n 80 -m spark
```

---

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-j` | off | Emit jitter — absolute delta from previous RTT (ms) |
| `-l` | off | Emit cumulative loss counter (running total of lost packets) |
| `-L LABELS` | auto | Comma-separated custom metric labels |
| `-w SEC` | `1` | Per-host ping deadline / timeout in seconds |
| `-d MS` | `200` | Delay between sample batches, in milliseconds |
| `-h` | — | Show help |

### Labeling

Labels are derived automatically for readability:

| Host | Label |
|------|-------|
| `google.com` | `google` |
| `cloudflare.com` | `cloudflare` |
| `8.8.8.8` | `8_8_8_8` |
| `one.one.one.one` | `one` |

Override with `-L`:

```bash
./piping -L "Google,Cloudflare,Quad9" 8.8.8.8 1.1.1.1 9.9.9.9 | ./pipechart ...
```

---

## pipechart layout reference

piping emits `KEY=VALUE` lines — any pipechart layout works. Common presets:

### One row per host (recommended)

Full-width bars, 5 lines high, labels never truncate:

```bash
./piping [hosts...] | ./pipechart -n 60 -c 1 -h 5
```

```
┌google          165ms ────────────────────────────────────────────────────────┐
│       ██                                                                      │
│      ███                                                                      │
│     ████                                                                      │
│    █████                                                                      │
│   ██████                                                                      │
└──────────────────────────────────────────────────────────────────────────────┘
┌cloudflare      163ms ────────────────────────────────────────────────────────┐
│        █                                                                      │
│       ██                                                                      │
│      ███                                                                      │
│     ████                                                                      │
│    █████                                                                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Side by side (wide terminal, ≥ 100 cols)

```bash
./piping [hosts...] | ./pipechart -n 60 -c 2 -h 5
```

> Note: bar height auto-reduces to ~3 rows on 80-col terminals. Use `-c 1` if you
> see `2x3` instead of `2x5` in the pipechart header.

### Sparkline (ultra-compact)

```bash
./piping [hosts...] | ./pipechart -n 80 -m spark
```

---

## Recipes

### Monitor two DNS servers with loss + jitter

```bash
./piping -j -l -L "Google,Cloudflare" 8.8.8.8 1.1.1.1 | ./pipechart -n 60 -c 1 -h 5
```

Emits six metrics: `Google`, `Google_L`, `Google_J`, `Cloudflare`, `Cloudflare_L`, `Cloudflare_J`.

- **Google** — RTT bar
- **Google_L** — cumulative loss count (0, 1, 2, …)
- **Google_J** — jitter (absolute Δ from last RTT)

### Monitor a local gateway with fast polling

```bash
./piping -w 1 -d 100 192.168.1.1 | ./pipechart -n 80 -m spark
```

`-d 100` → 100ms between batches (10 samples/sec). Good for catching micro-bursts.

### Log to CSV while watching

```bash
./piping -j google.com cloudflare.com \
  | ./pipechart -n 60 -c 1 -h 5 -o ping.csv -f csv
```

Chart runs live; every sample is also appended to `ping.csv` with timestamps.

### Four hosts on a wide monitor

```bash
./piping -L "A,B,C,D" host1 host2 host3 host4 \
  | ./pipechart -n 60 -c 4 -h 5
```

### Embedded / BusyBox (no `grep -P`)

piping auto-detects missing `grep -P` and falls back to `sed`. No flags needed.

---

## How it works

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ ping -c1 │    │ ping -c1 │    │ ping -c1 │    ← all hosts in parallel
│ google   │    │ cloudfl  │    │ quad9    │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
                     │ wait
                     ▼
         ┌─────────────────────┐
         │ google=165          │
         │ google_L=0          │   ← KEY=VALUE lines
         │ google_J=6          │
         │ cloudflare=163      │
         │ ...                 │
         └──────────┬──────────┘
                    │ stdout
                    ▼
            ┌─────────────┐
            │  pipechart  │   ← live ASCII bars
            └─────────────┘
```

1. **Parallel pings** — all hosts pinged simultaneously. A 1s timeout on one host
   never blocks the others.
2. **KEY=VALUE output** — each sample writes one line per metric.
   pipechart auto-detects metrics on the first read.
3. **Loss counter** — cumulative, stored per-host in temp files. Bar grows with
   each lost packet.
4. **Jitter** — absolute delta `|current_rtt - previous_rtt|`, computed with
   `bc` (or `awk` fallback).

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| "ERROR: no output from command" | Pass `-i 2` (or higher) to pipechart — piping needs ~1s startup |
| Bars stuck at 0 | Host unreachable or ping needs root. Try `ping -c 1 127.0.0.1` first |
| Labels truncated (`google_lo…`) | Use `-c 1` for full-width rows, or shorter `-L` labels |
| Bar height lower than `-h 5` | Multi-column auto-reduces height. Use `-c 1` or widen terminal |
| `-j` emits no jitter | Normal for the first sample (no previous RTT to compare) |
| `grep -P` not found | piping auto-falls-back to `sed` |

---

## See also

- [pipechart README](./README.md) — full pipechart options
- `./piping -h` — built-in help
- `./pipechart -i 1 vcgencmd pmic_read_adc` — Raspberry Pi PMIC monitoring
