# Reflex CPUFreq Governor

A Linux CPUFreq governor that is `schedutil` plus a measured fast path for rising load: it gets bursts that follow an idle up to speed within one scheduler tick, instead of waiting for PELT to ramp up.

## Why

`schedutil` ramps up slowly for a reason that is easy to miss. On a CPU that is saturated (100% busy), frequency-invariant utilization can only reach the CPU's *current* capacity: the demand above it is censored. `schedutil` then raises the frequency by its 1.25x headroom at PELT's pace (a 32-period half-life of about 33.5 ms), so going from a quarter of the maximum to the maximum takes on the order of a few hundred milliseconds. Short interactive bursts, especially after an idle, finish before the frequency arrives.

Reflex keeps everything `schedutil` does right and adds only what PELT cannot see.

## How It Works

### Measured demand

Each CPU closes, on itself, intervals of at least `max(rate_limit_us, 1 tick)` and measures over each one

```
D = busy * freq_scale * capacity        (PELT's frequency-invariant units)
```

from `kcpustat` idle time and `arch_scale_freq_capacity()`. An interval with any idle in it shows the demand exactly. One with no idle is *censored*: the CPU is falling behind, and its demand is only known to be at least `D`.

### State per CPU

- **H**, the envelope of `D`: it follows a rise at once and decays exactly like PELT (`decay_load()`).
- **N**, the running time seen lately, also decayed like PELT.

From `N`, the probability that the CPU is behind is

```
p = 1                     after a censored interval
p = alpha / (N + alpha)   otherwise, alpha = tau_min of running
```

The less a CPU has been seen running lately, the more likely new work on it is behind. After a long idle `N` has faded and `p` is near 1; ticks and tiny wakeups add next to nothing to `N`, so they cannot make a long idle look short. There is no threshold: `p` moves continuously with the running time observed.

### The request

```
util = max(1.25 * PELT, 1.25 * H), capped by uclamp_max
freq >= s*(p * lambda),   s*(l) = argmin_s (P(s) + l) / s
```

Behind with probability `p`, a unit of work costs `(P(s) + p * lambda) / s` in energy plus latency, and `s*(p * lambda)` is the speed that minimizes it. `s*(0)` is the most efficient speed; work that is behind (`p = 1`) runs at `s*(lambda)`. The floor is tabulated over `p` (65 points), so a frequency update only looks it up.

### Power model

| Source | Where | s*(0) | s*(lambda) |
|---|---|---|---|
| Energy Model | ARM (DT, SCMI, qcom, mediatek, apple), `cppc_cpufreq`, and any driver that registers one | cheapest state per unit of work | exact, from the state table |
| ACPI CPPC | `amd-pstate`, Intel with `_CPC`, ARM servers | lowest nonlinear performance | cube-law model |
| none | anything else | 0 (as `schedutil`) | cube-law model |

The floors are rebuilt only at start, on a limits change, on a `latency_weight` write and when the Energy Model's table is replaced; nothing is added to the update path but a table lookup.

### What stays with PELT

PELT is read, never simulated. Wake-up prediction (`util_est`), task migration, uclamp, RT, DL and I/O wait boost all work exactly as in `schedutil`.

## Tunables

Exposed under `/sys/devices/system/cpu/cpufreq/reflex/`:

| Tunable | Default | Description |
|---|---|---|
| `rate_limit_us` | Driver default | Minimum interval between frequency updates, as in `schedutil`. Measurement intervals are at least `max(rate_limit_us, 1 tick)`. |
| `latency_weight` | 1000 | The latency weight `lambda`, per mille of `lambda_max`: the least weight at which work that is behind runs at the maximum frequency. Lower trades latency for energy; 0 keeps only the efficiency floor `s*(0)`. Up to 100000. |
| `version` | *(read-only)* | The governor version. |

0.4.0 removes `hispeed_window_us` and `hispeed_filter_shift`.

## Results

5 ms of work (at the top speed) after an idle, on three isolated Zen 4 cores (AMD Ryzen 7 7840HS, `amd-pstate` passive), 24 independent trials per cell, with the governor-to-core mapping rotated every round. Median completion time; the share of runs over 7 ms in brackets.

| Idle before the burst | Reflex 0.4.0 | Reflex 0.3.3 | schedutil |
|---|---|---|---|
| 1 s | **5.53 ms** (2%) | 5.54 ms (21%) | 12.93 ms (59%) |
| 100 ms | **5.52 ms** (1%) | 5.62 ms (40%) | 14.50 ms (88%) |
| 16.7 ms (one 60 Hz frame) | **5.57 ms** (4%) | 5.69 ms (29%) | 10.05 ms (78%) |

The ideal is about 5.2 ms. 0.3.3 and `schedutil` are bimodal: some runs happen to start at a stale high frequency and finish fast, the rest run slow. In an earlier, smaller run with 50 ms bursts, all three governors were within 1-2% of the ideal: the difference is in short bursts. Energy has not been measured yet.

## Supported Kernels

| Patch | Base |
|---|---|
| `patches/0001-linux6.12.74-reflex-v0.4.0.patch` | Linux 6.12.74 (LTS) |
| `patches/0001-linux6.18.3-reflex-v0.4.0.patch` | Linux 6.18.3 (LTS) |
| `patches/0001-linux7.2.8-reflex-v0.4.0.patch` | Linux 7.2.8 |
| `patches/0001-linux7.3-rc1-reflex-v0.4.0.patch` | Linux 7.3-rc1 |

`cpufreq_reflex.c` is byte-identical in all four; the version differences live in `cpufreq_reflex_compat.h` and in the kernel-side exports.

## Installation

### 1. Apply the patch

```sh
cd /path/to/linux
patch -p1 < /path/to/reflex/patches/0001-linux6.12.74-reflex-v0.4.0.patch
```

The patch also touches the scheduler and the cpufreq core, to export what the governor needs (`cpufreq_get_effective_util()`, `cpufreq_pelt_decay()`, `idle_cpu()` and a few cpufreq helpers).

### 2. Enable in kernel config

```
CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y
CONFIG_CPU_FREQ_GOV_REFLEX=m
```

`CONFIG_ENERGY_MODEL` and `CONFIG_ACPI_CPPC_LIB` are used when present; Reflex works without them.

### 3. Build and boot

Build and install the whole kernel, not just the module: the exports above are part of the kernel image. Then boot it.

### 4. Load the module

```sh
sudo modprobe cpufreq_reflex
```

### 5. Activate

If your system uses `intel_pstate` or `amd_pstate` in **active** mode, external governors like Reflex cannot be activated. Check the current mode:

```sh
cat /sys/devices/system/cpu/intel_pstate/status
# or
cat /sys/devices/system/cpu/amd_pstate/status
```

If it reports `active`, switch to **passive** (or, for `amd_pstate`, **guided**):

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

Then activate Reflex and check it:

```sh
echo reflex | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpufreq/reflex/version          # 0.4.0
cat /sys/devices/system/cpu/cpufreq/reflex/latency_weight   # 1000
```

## Dependencies

- Linux kernel with `CPU_FREQ`, `SMP` and `CPU_FREQ_GOV_SCHEDUTIL` enabled
- A CPUFreq driver that uses governors: `intel_pstate` / `amd_pstate` in passive or guided mode, `acpi-cpufreq`, `cppc_cpufreq`, `cpufreq-dt`, `scmi-cpufreq`, and so on
- Standard kernel build toolchain (gcc or clang); x86_64 and arm64 are build-tested
