# Reflex CPUFreq Governor

A Linux kernel CPUFreq governor that combines the instant responsiveness of the `interactive` governor with the efficiency of modern `schedutil` PELT-based scaling.

## Overview

Reflex is a hybrid CPUFreq governor designed to deliver the best of both worlds:

- **Instant high-speed jumps** on idle-to-busy transitions, based on real CPU busy% measured via `kcpustat` idle-time counters.
- **PELT-proportional scaling** for sustained workloads, identical to `schedutil` with 1.25x DVFS headroom.
- **Smooth blending** via a PELT-complementary exponential decay algorithm (32 ms half-life), ensuring high-speed + PELT coverage stays at ~100% at all times.

After approximately 10 half-lives (~320 ms), PELT takes full control with no residual frequency floor lock-in.

## How It Works

### Blending Algorithm

When a CPU wakes from idle:

1. An immediate frequency floor is set based on measured CPU busy%.
2. This high-speed contribution decays exponentially: `blended = pelt + (hispeed - pelt) >> half_lives`.
3. As high-speed decays, PELT's utilization signal naturally rises to fill the gap.

### Asymmetric EWMA Filtering

The busy% measurement uses asymmetric filtering:

- **Ramp-up:** Instant tracking (`filtered = measured`).
- **Ramp-down:** Exponential decay (`filtered -= (filtered - measured) >> shift`), tunable via `hispeed_filter_shift`.

### Two-Phase Idle-Time Measurement

Uses `kcpustat` kernel idle-time counters (not PELT) for raw CPU busy ratio, with a two-phase measurement protocol that prevents transient `busy_pct=0` readings from being mistakenly treated as genuine idle signals.

### Grace Period for Decay Timer

The high-speed decay timer is only reset after two consecutive idle windows, preventing rapid idle/busy cycling from defeating the decay mechanism.

### I/O Wait Boost

Schedutil-compatible I/O wait boosting: starts at 1/8 of max capacity, doubles on each consecutive I/O wait event, and decays by half each period when not waiting.

## Tunables

Exposed under `/sys/devices/system/cpu/cpufreq/reflex/`:

| Tunable | Default | Description |
|---|---|---|
| `rate_limit_us` | Driver default | Minimum interval between frequency updates (typically 1000-2000 us) |
| `hispeed_window_us` | 4000 | Observation window duration for busy% calculation (us) |
| `hispeed_filter_shift` | 3 | EWMA down-ramp shift for filtered busy% (0 = off, higher = slower decay) |
| `version` | *(read-only)* | Reports the governor version string |

## Installation

Reflex is delivered as a kernel patch. Apply it to your Linux kernel source tree:

### 1. Apply the patch

```sh
cd /path/to/linux
patch -p1 < /path/to/reflex/patches/0001-Reflex-CPUFreq-Governor-v0.2.1.patch
```

### 2. Enable in kernel config

```
CONFIG_CPU_FREQ_GOV_REFLEX=m
```

Or to set as the default governor:

```
CONFIG_CPU_FREQ_DEFAULT_GOV_REFLEX=y
```

### 3. Build

```sh
make modules
```

### 4. Load the module

```sh
sudo modprobe cpufreq_reflex
```

### 5. Activate

**Note:** If your system uses `intel_pstate` or `amd_pstate` in **active** mode, external governors like Reflex cannot be activated. Check the current mode:

```sh
cat /sys/devices/system/cpu/intel_pstate/status
# or
cat /sys/devices/system/cpu/amd_pstate/status
```

If it reports `active`, switch to **passive** or **guided** mode:

```sh
echo passive | sudo tee /sys/devices/system/cpu/intel_pstate/status
# or
echo passive | sudo tee /sys/devices/system/cpu/amd_pstate/status
```

To make this persistent across reboots, add a kernel boot parameter:

```
intel_pstate=passive
# or
amd_pstate=passive
```

Once the driver is in passive/guided mode, activate Reflex:

```sh
echo reflex | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## Dependencies

- Linux kernel with `CPU_FREQ`, `SMP`, and `CPU_FREQ_GOV_SCHEDUTIL` enabled
- x86 or other architecture with a CPUFreq driver supporting the `update_util` callback (`intel_pstate`/`amd_pstate` in passive or guided mode, `acpi-cpufreq`, etc.)
- Standard kernel build toolchain (gcc or clang)
