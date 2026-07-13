# pipechart

Live ASCII bar/sparkline charts from periodic command output. Auto-detects `KEY=VALUE` pairs.

## Usage

```bash
# Run a command every 1s and chart all detected metrics
./pipechart -i 1 vcgencmd pmic_read_adc

# Sparkline mode
./pipechart -i 2 -m spark vcgencmd measure_temp

# More history, 2 columns, shorter bars
./pipechart -n 60 -c 2 -h 3 vcgencmd pmic_read_adc

# Pipe data in
while true; do vcgencmd pmic_read_adc; sleep 1; done | ./pipechart
```

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-i SEC` | 1 | Poll interval |
| `-n N` | 30 | History points |
| `-h N` | 5 | Bar height per metric |
| `-m MODE` | bar | `bar` or `spark` |
| `-c COLS` | auto | 1, 2, 3, or 0 for auto-fit |
| `-e SPEC` | — | Explicit extractor: `"LABEL,UNIT:AWK_CODE"` |

## Auto-adapting layout

Column count and bar height automatically adapt to terminal size. Wide terminals get 3 columns, narrow ones get 1 or 2. Bar height is capped to keep the frame within the visible area.
