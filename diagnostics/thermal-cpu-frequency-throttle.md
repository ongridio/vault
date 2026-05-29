---
title: Thermal / Frequency Throttling (Hardware Slowdown)
kind: howto
tags: [cpu, thermal, frequency, throttle, cpufreq, power, performance]
applies_to: [edge]
---

# Thermal / Frequency Throttling (Hardware Slowdown)

Use on bare-metal / edge hardware when performance degrades under
sustained load with no software cause — CPU usage looks normal but work
is slow, because the CPU clocked *down*. Causes: thermal throttling
(overheating), a power/`governor` capping frequency, or a failing PSU/
power limit. **Mostly an edge/bare-metal concern — VMs hide this.**

| Symptom | Probable cause |
|---|---|
| Slow under sustained load, cool when idle | thermal throttling (heat/dust/fan) |
| Frequency stuck low always | `powersave` governor / power cap |
| Throttle flags set in dmesg | hardware thermal/power limit hit |
| Worse over months | dust buildup / degraded thermal paste / aging fan |

## Step 1 — Read actual vs rated frequency

```bash
# Current per-core MHz vs base/max
grep MHz /proc/cpuinfo | head
cpupower frequency-info 2>/dev/null | grep -E 'current|limits|governor'
# Or:
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq 2>/dev/null | head
```

Cores running well below their rated/turbo frequency under load = clocked
down. Compare to `cpufreq` `cpuinfo_max_freq`.

## Step 2 — Thermal state

```bash
sensors 2>/dev/null | grep -iE 'core|package|temp'   # lm-sensors
cat /sys/class/thermal/thermal_zone*/temp 2>/dev/null  # millidegrees C
dmesg -T | grep -iE 'thermal|throttl|clock throttled|temperature above'
```

Package temp near the junction max (often ~90–100°C) + `dmesg` throttle
messages = thermal throttling. Cooling problem: dust, failed/slow fan,
dried thermal paste, blocked airflow, ambient too hot.

## Step 3 — Governor / power cap

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor   # powersave vs performance
# Intel RAPL / power cap (if present)
cat /sys/class/powercap/intel-rapl*/constraint_0_power_limit_uw 2>/dev/null
```

A `powersave` governor or an aggressive RAPL power cap holds frequency
down regardless of temperature. Set `performance` governor for
latency-critical edge nodes.

## Step 4 — Fix

- **Thermal**: clean dust, fix/replace fans, reseat heatsink/paste,
  improve airflow/ambient cooling. Throttling protects the silicon —
  don't disable it; fix the heat.
- **Governor/power**: set `performance` governor; review RAPL/BIOS power
  limits if they cap below rated.
- Monitor temp as a metric so this is caught before it degrades service.

## Decision tree

| Signal | Action |
|---|---|
| high package temp + dmesg throttle | cooling problem — clean/fix fans, paste, airflow |
| freq low, temp fine, `powersave` | set `performance` governor |
| RAPL/power cap below rated | raise/review power limit (BIOS/RAPL) |
| VM/cloud guest | you can't see host thermals — likely steal, not thermal |
| degrades over months | dust/aging cooling — physical maintenance |

## References

- [CPUFreq governors — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html)
- [lm-sensors](https://github.com/lm-sensors/lm-sensors)
- vault: `diagnostics/cpu-throttling.md`, `diagnostics/cpu-steal-noisy-neighbor.md`, `diagnostics/high-load-low-cpu.md`
