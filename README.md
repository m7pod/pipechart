# pipechart

> Live ASCII bar/sparkline charts from periodic command output.
> Auto-detects `KEY=VALUE` pairs. Just point it at a command.

<p align="center">
  <i>No dependencies beyond bash, awk, and tput. Runs anywhere.</i>
</p>

---

## Quick start

```bash
# Clone
git clone https://github.com/m7pod/pipechart.git
cd pipechart

# Run — charts every KEY=VALUE pair automatically
./pipechart -i 1 vcgencmd pmic_read_adc

# Pipe data in
while true; do some-command; sleep 1; done | ./pipechart

# Sparkline mode for a single metric
./pipechart -i 2 -m spark vcgencmd measure_temp
```

---

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-i SEC` | `1` | Poll interval in seconds |
| `-n N` | `30` | History points to keep |
| `-h N` | `5` | Bar height per metric (auto-reduced for multi-column) |
| `-m MODE` | `bar` | Chart mode: `bar` or `spark` |
| `-c COLS` | `0` (auto) | Column count: `1`, `2`, `3`, or `0` for auto-fit |
| `-e SPEC` | — | Explicit metric extractor (repeatable, see below) |
| `-o FILE` | — | Log metrics to file (appends each sample) |
| `-f FMT` | `txt` | Log format: `csv` or `txt` |

---

## How it works

### Auto-detection

pipechart reads one sample of command output and scans for `KEY=VALUE` patterns:

```
3V3_WL_SW_A=3.36V
3V3_SYS_A=3.33V
1V8_SYS_A=1.80V
DDR_VDD2_A=1.10V
```

→ Each line becomes a charted metric with color-coded bars, the label as the key, and the value + unit extracted automatically.

### Layout adaptation

Column count and bar height adapt to your terminal **automatically**:

| Terminal width | Metrics count | Layout |
|---------------|---------------|--------|
| ≥ 90 cols | ≥ 9 metrics | **3 columns** |
| ≥ 70 cols | ≥ 6 metrics | **3 columns** |
| ≥ 55 cols | ≥ 4 metrics | **2 columns** |
| ≥ 40 cols | ≥ 2 metrics | **2 columns** |
| anything else | any | **1 column** |

Bar height is also capped so the full frame fits within your terminal height.

---

## Explicit extraction (`-e`)

When auto-detection doesn't work (non `KEY=VALUE` output, or you want specific fields):

```bash
# Format: "LABEL,UNIT:AWK_CODE"
# The colon splits ONCE — awk code can contain colons

# Extract from space-separated output like `sensors`
./pipechart -e "CPU,°C:/Core 0/{print \$3}" \
            -e "GPU,°C:/edge/{print \$2}" \
            -i 2 sensors

# From custom script output
./pipechart -e "Load,%:/load average:/{print \$3}" \
            -e "Mem,GB:/Mem:/{print \$2}" \
            -i 1 uptime
```

---

## Examples

### Raspberry Pi system monitoring

```bash
# All PMIC voltages — auto-detected
./pipechart -i 1 vcgencmd pmic_read_adc

# CPU temp — sparkline
./pipechart -i 2 -m spark vcgencmd measure_temp

# Keep 60 data points, 2 columns, compact bars
./pipechart -n 60 -c 2 -h 3 vcgencmd pmic_read_adc
```

### Generic Linux

```bash
# CPU load averages
./pipechart -i 2 -e "1min,:/load average:/{print \$3}" \
            -e "5min,:/load average:/{print \$4}" \
            -e "15min,:/load average:/{print \$5}" \
            uptime

# Memory usage
./pipechart -i 2 -e "Total,MB:/Mem:/{gsub(/[^0-9]/,\"\",\$2);print \$2}" \
            -e "Avail,MB:/Mem:/{gsub(/[^0-9]/,\"\",\$7);print \$7}" \
            free -m

# Disk I/O
./pipechart -i 1 -e "Read,KB/s:/sda/{print \$3}" \
            -e "Write,KB/s:/sda/{print \$7}" \
            iostat -d 1 1
```

### Piping

```bash
# From a custom loop
while true; do
    echo "temp=$(vcgencmd measure_temp | grep -o '[0-9.]*')"
    echo "freq=$(vcgencmd measure_clock arm | grep -o '[0-9]*' | head -1)"
    sleep 1
done | ./pipechart -n 60

# From a log file being written
tail -f /var/log/metrics.log | grep '=' | ./pipechart

# Network latency (ping RTT, jitter, loss) — see PIPING.md
./piping google.com cloudflare.com | ./pipechart -n 60 -c 1 -h 5
```

---

## Logging to file (`-o`, `-f`)

Log every sample to a file while the chart runs. Writes are fast and non-blocking.

```bash
# TXT format (default): human-readable, one line per sample
./pipechart -i 1 -o metrics.log vcgencmd pmic_read_adc
# → [2026-07-13T17:30:00+08:00] 3V3_WL_SW_A=3.36     3V3_SYS_A=3.33     ...

# CSV format: header row + comma-separated, ready for Excel / pandas
./pipechart -i 1 -o metrics.csv -f csv vcgencmd pmic_read_adc
# → timestamp,3V3_WL_SW_A,3V3_SYS_A,...
# → 2026-07-13T17:30:00+08:00,3.36,3.33,...
```

Combine with `-e` for custom labels:

```bash
./pipechart -i 2 -o temps.csv -f csv \
    -e "CPU,°C:/temp/{print \$2}" \
    -e "GPU,°C:/temp/{print \$4}" \
    sensors
```

---

## Dependencies

- **bash** ≥ 4.0
- **awk** (gawk or mawk)
- **tput** (from ncurses, for terminal size detection)
- **bc** (for sub-second sleep precision; falls back gracefully)
- **timeout** (from coreutils; falls back gracefully)

No Python, Node, Rust, or compilation needed. Every dependency is pre-installed on virtually every Linux distribution and macOS.

---

## Controls

| Key | Action |
|-----|--------|
| `Ctrl+C` | Exit (cleans up cursor and temp files) |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "No metrics detected" | Output doesn't contain `KEY=VALUE`. Use `-e` to specify extraction manually. |
| Title row cut off | Your terminal is too narrow. Resize wider, or use `-c 1` for single-column mode. |
| Bars all same height | Normal for the first few frames while history builds up. Wait a few seconds. |
| Flickering | The render buffer should prevent this. If it persists, reduce `-n` (fewer history points). |
| Command not found | Make sure your command is in `$PATH` or use an absolute path. |
| Negative or hex values | Hex (`0x...`) is auto-detected. Negative values should work if the command outputs them with a `-` sign. |

---

## License

MIT — see [LICENSE](LICENSE) file.
