# 8QAM Transceiver using GNU Radio and ADALM-Pluto SDR

A complete implementation of an 8-QAM (Quadrature Amplitude Modulation) transceiver system built in GNU Radio Companion (GRC), using the ADALM-Pluto SDR for over-the-air transmission and an ideal loopback receiver for signal recovery and bit-stream verification.

---

## Project Overview

This project implements three progressively complex signal processing tasks:

1. **Part 1** — 8QAM transmitter using ADALM-Pluto SDR with RRC pulse shaping
2. **Part 2** — File-based transmitter with binary bitstream input and time/frequency domain analysis
3. **Part 3** — Coherent loopback receiver with symbol timing and phase recovery

---

## Repository Structure

```
8QAM-GNURadio/
│
├── flowgraphs/
│   ├── transmitter1.grc        # Part 1: PlutoSDR transmitter
│   ├── transmitter_2.grc       # Part 2: File source transmitter
│   └── transmitter_3.grc       # Part 3: Loopback receiver
│
├── scripts/
│   ├── generate_abcd_bits.py   # Generates A→B→C→D bit file
│   └── verify_bitstream.py     # Compares TX and RX binary files
│
├── data/
│   ├── abcd_bits.bin           # Input bitstream file (A→B→C→D)
│   └── received_8qam.bin       # Recovered bitstream (generated at runtime)
│
└── README.md
```

---

## System Specifications

| Parameter | Value |
|---|---|
| Modulation | 8-QAM |
| Symbol Rate | 100,000 symbols/sec |
| Samples per Symbol (sps) | 5.12 |
| Sample Rate | 512,000 Hz |
| Pulse Shaping | Root Raised Cosine (RRC) |
| Excess Bandwidth (α) | 0.35 |
| Carrier Frequency | 850 MHz |
| TX Attenuation | 20 dB |
| PlutoSDR URI | ip:192.168.2.1 |

---

## 8QAM Constellation Diagram

The constellation used in this project is a **non-uniform 8QAM** with 4 inner points (radius = √2) and 4 outer points (radius = √10):

```
Q
^
|   F(-3+1j)  B(-1+1j) | A(+1+1j)  E(+3+1j)
|             . - - - . |
|           .           .
|----------.------+------.---------> I
|           .           .
|             . - - - . |
|   G(-3-1j)  C(-1-1j) | D(+1-1j)  H(+3-1j)
```

### Symbol Map (Index → Complex Point)

| Index | Symbol | Complex Value | Bits (Gray) |
|---|---|---|---|
| 0 | G | -3 - 1j | 000 |
| 1 | C | -1 - 1j | 001 |
| 2 | D | +1 - 1j | 010 |
| 3 | H | +3 - 1j | 011 |
| 4 | F | -3 + 1j | 100 |
| 5 | B | -1 + 1j | 101 |
| 6 | A | +1 + 1j | 110 |
| 7 | E | +3 + 1j | 111 |

---

## Part 1 — PlutoSDR Transmitter

### Flowgraph Chain

```
Random Source → Chunks to Symbols → RRC Filter → Rational Resampler → Multiply Const → PlutoSDR Sink
```

### Block Parameters

| Block | Key Parameters |
|---|---|
| Random Source | Output: Byte, Min: 0, Max: 7 |
| Chunks to Symbols | Symbol Table: 8 complex points, Dimension: 1 |
| RRC TX Filter | Interpolation: 5, Gain: 5, SR: 500k, Alpha: 0.35, Taps: 55 |
| Rational Resampler | Interpolation: 128, Decimation: 125 (ratio = 1.024) |
| Multiply Const | Constant: 0.5 |
| PlutoSDR Sink | URI: ip:192.168.2.1, LO: 850 MHz, SR: 512k, Atten: 20 dB |

### Why Rational Resampler (128/125)?

```
Required sample rate = sps × symbol_rate = 5.12 × 100,000 = 512,000
After RRC (integer sps=5): 5 × 100,000 = 500,000
Resampling ratio = 512,000 / 500,000 = 1.024 = 128/125 (exact rational)
```

---

## Part 2 — File Source Transmitter

### Binary File Generation

Run `scripts/generate_abcd_bits.py` to create the input bitstream:

```python
import numpy as np

# A=110, B=101, C=001, D=010 (3 bits each)
bits = np.array([1,1,0, 1,0,1, 0,0,1, 0,1,0], dtype=np.uint8)
data = np.tile(bits, 50000)   # 600,000 bits total
data.tofile('data/abcd_bits.bin')
```

### Flowgraph Chain

```
File Source → Throttle(300k) → Pack K Bits(K=3) → Chunks to Symbols → RRC → Resampler → Multiply Const → Throttle(512k) → Sinks
```

### Sequence Analysis: A→B→C→D

The four inner points rotate 90° counterclockwise each symbol:

```
A(+1+1j) → B(-1+1j) → C(-1-1j) → D(+1-1j) → A...
   45°   →   135°   →   225°   →   315°   → 45°...
```

**Generated signal frequency:**
```
f = symbol_rate / symbols_per_cycle = 100,000 / 4 = 25,000 Hz = 25 kHz
```

**Time domain:** Both I and Q channels produce clean sinusoids at 25 kHz, 90° apart.

**Frequency domain:** Dominant spectral peak at 25 kHz with RRC-shaped rolloff. Comb lines at ±100 kHz spacing due to the periodic nature of the repeating 4-symbol sequence.

### Observations

| Sequence | Time Domain | Frequency |
|---|---|---|
| A→B→C→D | Sinusoids at 25 kHz, 90° apart, constant amplitude | 25 kHz |
| A→C→A→C | Faster sinusoids, 180° phase jumps | 50 kHz |
| A only | Flat DC line | 0 Hz |

---

## Part 3 — Coherent Loopback Receiver

### Assumptions (Ideal Loopback)

- No channel noise
- No carrier frequency offset
- No timing offset
- No phase offset

### Receiver Flowgraph Chain

```
[Loopback from Multiply Const]
         ↓
   RRC RX Filter (matched filter)
         ↓
   Symbol Sync (Gardner TED, timing recovery)
         ↓
   Costas Loop (phase recovery, order=4)
         ↓
   Constellation Decoder
         ↓
   Unpack K Bits (K=3)
         ↓
   File Sink → received_8qam.bin
```

### Receiver Block Parameters

**RRC RX Filter (Matched Filter):**
```
Decimation: 1
Gain: 1
Sample Rate: 512000
Symbol Rate: 100000
Alpha: 0.35
Num Taps: 55
```

**Symbol Sync:**
```
Timing Error Detector: Gardner
Samples per Symbol: 5.12
Loop Bandwidth: 0.01
Damping Factor: 1.0
Maximum Deviation: 1.5
Output Samples/Symbol: 1
```

**Costas Loop:**
```
Loop Bandwidth: 0.0628
Order: 4
```

**Constellation Decoder:**
```
Constellation Object: Variable Constellation
Symbol Map: [0, 1, 2, 3, 4, 5, 6, 7]
Points: [(-3-1j),(-1-1j),(1-1j),(3-1j),(-3+1j),(-1+1j),(1+1j),(3+1j)]
```

### Null Sinks Required

| Block | Port | Null Sink Type |
|---|---|---|
| Symbol Sync | error | Float |
| Symbol Sync | T_inst | Float |
| Symbol Sync | T_avg | Float |
| Costas Loop | frequency | Float |
| Costas Loop | phase | Float |
| Costas Loop | error | Float |

---

## Verifying TX and RX File Match

Run `scripts/verify_bitstream.py`:

```python
import numpy as np

tx = np.fromfile('data/abcd_bits.bin', dtype=np.uint8)
rx = np.fromfile('data/received_8qam.bin', dtype=np.uint8)

# Find best delay offset (filter group delay)
best_errors = len(tx)
best_delay = 0
for delay in range(0, 500, 3):
    n = min(len(tx), len(rx) - delay)
    errors = np.sum(tx[:n] != rx[delay:delay+n])
    if errors < best_errors:
        best_errors = errors
        best_delay = delay

N = min(len(tx), len(rx) - best_delay)
errors = np.sum(tx[:N] != rx[best_delay:best_delay+N])
ber = errors / N

print(f"Best delay: {best_delay} bits ({best_delay//3} symbols)")
print(f"Bit errors: {errors}")
print(f"BER: {ber:.8f}")
print("PERFECT MATCH!" if errors == 0 else f"MISMATCH: {errors} errors")
```

**Expected output:**
```
Best delay: 51 bits (17 symbols)
Bit errors: 0
BER: 0.00000000
PERFECT MATCH!
```

---

## Dependencies and Installation

### GNU Radio
```bash
# Ubuntu/Debian
sudo apt install gnuradio

# Verify installation
gnuradio-companion --version
```

### gr-iio (PlutoSDR support)
```bash
sudo apt install gr-iio
```

### Python dependencies
```bash
pip install numpy
```

### PlutoSDR Firmware
- Connect PlutoSDR via USB or Ethernet
- Set static IP: `192.168.2.1`
- Verify: `ping 192.168.2.1`

---

## Running the Project

### Part 1 — PlutoSDR Transmitter
```bash
# Connect PlutoSDR, then open GRC
gnuradio-companion flowgraphs/transmitter1.grc
# Press F6 to run
```

### Part 2 — File Source Transmitter
```bash
# Generate input file first
python scripts/generate_abcd_bits.py

# Open and run flowgraph
gnuradio-companion flowgraphs/transmitter_2.grc
```

### Part 3 — Loopback Receiver
```bash
# Run transmitter_3 which includes both TX and RX
gnuradio-companion flowgraphs/transmitter_3.grc

# After running, verify files match
python scripts/verify_bitstream.py
```

---

## Expected Results

### Part 2 Outputs

| Sink | Expected |
|---|---|
| Constellation Sink | 4 tight dots at (±1, ±1) — only A,B,C,D visible |
| Time Sink | I and Q sinusoids at 25 kHz, 90° phase apart |
| Frequency Sink | Dominant peak at 25 kHz, comb pattern at ±100 kHz spacing |

### Part 3 Outputs

| Sink | Expected |
|---|---|
| TX Constellation Sink | 4 dots at (±1, ±1) |
| RX Constellation Sink | Identical 4 dots — confirms perfect recovery |
| BER | 0.0 — zero bit errors under ideal loopback |

---

## Key Signal Processing Concepts

**Root Raised Cosine (RRC) Filtering:**
TX and RX each apply one RRC filter. Together they form a full Raised Cosine filter which eliminates Inter-Symbol Interference (ISI) at the optimal sampling instant.

**Rational Resampling (128/125):**
Converts integer sps=5 (500 kHz) to fractional sps=5.12 (512 kHz) required by PlutoSDR using exact integer arithmetic — no floating point approximation errors.

**Symbol Sync (Gardner TED):**
Finds the optimal sampling instant within each symbol period by minimizing the timing error signal, outputting exactly one sample per symbol.

**Costas Loop (Order 4):**
Corrects residual phase rotation using a 4th-order phase-locked loop, matching the 4-fold rotational symmetry of the inner constellation points A, B, C, D.

---
