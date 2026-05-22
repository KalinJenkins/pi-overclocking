
## The File You Edit

All overclock settings live in one place:

```
/boot/firmware/config.txt
```

Relevant settings:

```ini
arm_freq=1375       # CPU clock in MHz
core_freq=500       # GPU core clock
sdram_freq=500      # RAM clock
over_voltage=3      # Voltage offset (~25 mV per step)
gpu_mem=16          # GPU memory split (lower = more RAM for CPU)
```

---

## How over_voltage Works

Each step adds approximately **+25 mV** to the CPU core rail. It provides timing margin at higher clock speeds, it does not guarantee stability, and more voltage stops helping once you hit the silicon's physical limit.

| over_voltage | Approx offset |
|---|---|
| 0 | +0 mV |
| 1 | +25 mV |
| 2 | +50 mV |
| 3 | +75 mV |
| 4 | +100 mV |
| 5 | +125 mV |

---

## Testing Methodology

### 1. Verify clock under load
```bash
stress-ng --cpu 1 --timeout 15s &
sleep 1
vcgencmd measure_clock arm
vcgencmd get_throttled
```

### 2. Check throttle status
```bash
vcgencmd get_throttled
```

| Value | Meaning |
|---|---|
| `0x0` | All clear |
| `0x80000` | Previously throttled (at some point since boot) |
| `0x40000` | Currently throttling |

### 3. Sustained 4-core stress test
```bash
stress-ng --cpu 4 --timeout 120s
watch -n 2 vcgencmd get_throttled
```

### 4. Benchmark
```bash
sysbench cpu --cpu-max-prime=20000 run
```

---

## Findings on Pi 3B

| Frequency | over_voltage | Result |
|---|---|---|
| 1350 MHz | 2 | Rock solid |
| 1375 MHz | 3 | Stable through long soak  |
| 1400 MHz | 4 | Fails under sustained 4-core load  |
| 1390 MHz | 4 | Edge case — marginal at best |
| 1420 MHz | 5 | Hits thermal threshold, `0x80000` throttle flag |

**Sweet spot: `arm_freq=1375`, `over_voltage=3`**

---

## Going Headless (Recommended)

Removing the GUI improves benchmark consistency (not raw score).

### Boot to console
```bash
sudo raspi-config
# System Options → Boot / Auto Login → Console
```

### Lower GPU memory split
In `/boot/firmware/config.txt`:
```ini
gpu_mem=16
```

---