# pimon — Raspberry Pi Hardware Monitor Suite

> A family of tools for monitoring Raspberry Pi hardware: temperature, clocks,
> voltages, currents, power, and throttling. Live charts, one-shot snapshots,
> and pipeable KEY=VALUE output for use with [pipechart](./README.md).

---

## Tools at a glance

| Tool | Type | Description |
|------|------|-------------|
| `pistat` | One-shot | Snapshot of all Pi stats in a formatted table |
| `pistat-kv` | One-shot | Same stats in `KEY=VALUE` format (for pipechart) |
| `pimon` | Live | Sparkline dashboard (single-column) |
| `pimon2` | Live | 3-column sparkline dashboard (compact) |
| `pimon3` | Live | Horizontal bar chart dashboard |
| `pipechart` | Live | Generic charting of any `KEY=VALUE` command |

---

## Quick start

```bash
# One-shot snapshot
./pistat

# Live sparkline monitor (updates every second)
./pimon

# Compact 3-column sparklines
./pimon2

# Bar chart dashboard
./pimon3

# Pipe pistat into pipechart for live bar charts
./pipechart -i 1 ./pistat-kv
```

---

## pistat — One-Shot Snapshot

Prints a table of all Pi hardware stats and exits. Ideal for scripts, logging,
or a quick check.

```bash
./pistat
```

Output:

```
ITEM          VALUE
──────────────────────────────
Temp          45.2'C
CPU Clock     1500 MHz
Core Clock    500 MHz
V3D Clock     300 MHz
Core Volt     0.887 V
Core Curr     1.232 A
Core Power    1.09 W
EXT 5V        5.167 V
Throttled     0x0
```

**Metrics:**

| Metric | Source | Description |
|--------|--------|-------------|
| Temp | `vcgencmd measure_temp` | SoC temperature |
| CPU Clock | `vcgencmd measure_clock arm` | ARM core frequency |
| Core Clock | `vcgencmd measure_clock core` | Core frequency |
| V3D Clock | `vcgencmd measure_clock v3d` | GPU/V3D frequency |
| Core Volt | `vcgencmd pmic_read_adc` → VDD_CORE_V | Core voltage |
| Core Curr | `vcgencmd pmic_read_adc` → VDD_CORE_A | Core current |
| Core Power | Computed (V × A) | Instantaneous power draw |
| EXT 5V | `vcgencmd pmic_read_adc` → EXT5V_V | External 5V rail voltage |
| Throttled | `vcgencmd get_throttled` | Throttle status bitmask |

---

## pistat-kv — KEY=VALUE Variant

Same metrics as `pistat`, but outputs `KEY=VALUE` pairs for **pipechart**
auto-detection. No `-e` flags needed.

```bash
./pistat-kv
```

Output:

```
Temp=50.5
CPU=2400
Core=910
V3D=960
CoreV=0.887
CoreA=1.232
Power=1.09
EXT5V=5.167
Thr=0x0
```

### Pipe into pipechart

```bash
# Auto-detect all 9 metrics, single column
./pipechart -i 1 ./pistat-kv

# Two-column layout
./pipechart -i 1 -c 2 ./pistat-kv

# Sparkline mode, 80 history points
./pipechart -i 1 -n 80 -m spark ./pistat-kv

# Bar mode with 60 history points, 3 columns, compact height
./pipechart -i 1 -n 60 -c 3 -h 3 ./pistat-kv

# Log to CSV while charting
./pipechart -i 1 -o pimon.csv -f csv ./pistat-kv
```

---

## pimon — Sparkline Dashboard

Live single-column sparkline dashboard with colored blocks. Updates every
second by default.

```bash
./pimon [interval]
```

**Examples:**

```bash
./pimon           # 1s interval (default)
./pimon 2         # 2s interval
```

**Display:**

```
┌── Pi Monitor ── every 1s ──── Throttle: 0x0 ● OK        ──┐
│ 🌡 45.2°C ⚡CPU 1500 ⚙500 MHz 🔌0.887V ⚡1.23A 1.09W      │
├─────────────────────────────────────────────────────────────┤
│ Temp    45.2°C ▁▂▃▃▄▅▅▆▇█...                          │
│ CPU     1500MHz ▇▇▆▆▅▅▄▄▃▃...                          │
│ Vcore  0.887V  ▁▁▂▂▃▃▄▄▅▅...                          │
│ Amps   1.232A  ▃▃▄▄▅▅▆▆▇▇...                          │
│ Power  1.09W   ▂▂▃▃▄▄▅▅▆▆...                          │
└─────────────────────────────────────────────────────────────┘
  Ctrl+C to exit
```

---

## pimon2 — 3-Column Sparkline Dashboard

Compact layout with three columns of sparklines. Great for narrow terminals or
multi-Pi dashboards.

```bash
./pimon2 [interval] [history_points] [levels]
```

| Argument | Default | Description |
|----------|---------|-------------|
| `interval` | `1` | Update interval in seconds |
| `history` | `24` | History points to keep |
| `levels` | `24` | Sparkline resolution (8, 16, 24, …, 64) |

**Examples:**

```bash
./pimon2                     # Defaults: 1s, 24pt, 24-level
./pimon2 2 32 32             # 2s interval, 32 points, 32-level grayscale
./pimon2 1 16 8              # Fast 1s, 16 points, basic 8-level
```

---

## pimon3 — Bar Chart Dashboard

Full-width horizontal bar charts for each metric. Best for seeing trends at a
glance.

```bash
./pimon3 [interval] [history_points] [bar_height]
```

| Argument | Default | Description |
|----------|---------|-------------|
| `interval` | `1` | Update interval in seconds |
| `history` | `20` | History points (bars) |
| `bar_height` | `6` | Bar height in rows |

**Examples:**

```bash
./pimon3                     # Defaults: 1s, 20pt, 6-row bars
./pimon3 2 30 8              # 2s interval, 30 points, tall 8-row bars
./pimon3 1 15 4              # Fast 1s, compact 4-row bars
```

---

## Throttle status codes

| Value | Meaning |
|-------|---------|
| `0x0` | Normal — no throttling |
| `0x50000` | Under-voltage has occurred |
| `0x50005` | Throttling has occurred |
| `0x80008` | Soft temperature limit active |

---

## Recipes

### Monitor while stress testing

```bash
# Terminal 1: run stress test
stress-ng --cpu 4 --timeout 60s

# Terminal 2: watch live
./pimon3 1 30 6
```

### Log to CSV for later analysis

```bash
./pipechart -i 1 -o stress-test.csv -f csv ./pistat-kv
```

### Side-by-side comparison of two Pis

```bash
# On Pi #1
ssh pi@pi2 'cd /path/to/pimon && ./pistat-kv' | ./pipechart -c 1 -h 5

# Or log both and compare later
./pipechart -i 1 -o pi1.csv -f csv ./pistat-kv
```

### Combine with piping for a full network + hardware dashboard

```bash
# Left: network latency, Right: Pi hardware
# (use tmux split panes)
```

### Monitor over SSH without installing

```bash
ssh pi@raspberrypi 'bash -s' < ./pimon3
```

---

## Dependencies

All tools require only what's already on a standard Raspberry Pi OS:

- **bash** ≥ 4.0
- **vcgencmd** (Raspberry Pi firmware utility)
- **awk** (gawk or mawk)
- **tput** (ncurses, for terminal sizing)
- **bc** (for sub-second sleep precision; falls back gracefully)

Nothing to install. No Python, Node, or compilation needed.

---

## See also

- [pipechart README](./README.md) — full pipechart options
- [PIPING.md](./PIPING.md) — network latency monitor that pipes into pipechart
