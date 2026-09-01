# 8-QAM Transceiver using GNU Radio and ADALM-Pluto SDR

A complete 8-QAM transceiver built in GNU Radio Companion (GRC) — a PlutoSDR-based
transmitter with RRC pulse shaping, and a coherent loopback receiver with symbol
timing and phase recovery, verified via bit-error-rate (BER) analysis.

## Overview

| Part | Description |
|---|---|
| **1 — Transmitter** | Random-bit 8-QAM transmitter with RRC pulse shaping, transmitted over PlutoSDR |
| **2 — File TX** | File-based transmitter driven by a known bitstream, for time/frequency-domain analysis |
| **3 — Receiver** | Coherent loopback receiver with Gardner timing recovery and Costas phase recovery, verified bit-exact against the TX bitstream |

## Repository Structure
```
8QAM-GNURadio/
├── flowgraphs/
│   ├── transmitter1.grc      # Part 1: PlutoSDR transmitter
│   ├── transmitter_2.grc     # Part 2: File source transmitter
│   └── transmitter_3.grc     # Part 3: TX + loopback receiver
├── scripts/
│   ├── generate_abcd_bits.py # Generates A→B→C→D input bitstream
│   └── verify_bitstream.py   # Compares TX vs RX bitstreams, computes BER
├── data/
│   ├── abcd_bits.bin         # Input bitstream (A→B→C→D)
│   └── received_8qam.bin     # Recovered bitstream (generated at runtime)
└── README.md
```

## System Specifications

| Parameter | Value |
|---|---|
| Modulation | 8-QAM (4 inner points r=√2, 4 outer points r=√10) |
| Symbol Rate | 100,000 sym/s |
| Samples/Symbol | 5.12 |
| Sample Rate | 512,000 Hz |
| Pulse Shaping | Root Raised Cosine, α = 0.35, 55 taps |
| Carrier Frequency | 850 MHz |
| TX Attenuation | 20 dB |
| Resampling | 128/125 (exact rational, 500k → 512k) |

## Transmitter Chain
```
Random Source → Chunks to Symbols → RRC Filter → Rational Resampler → Multiply Const → PlutoSDR Sink
```
File-based variant (Part 2) adds `File Source → Throttle → Pack K Bits(K=3)` before symbol mapping.

## Receiver Chain (Ideal Loopback)
```
RRC Matched Filter → Symbol Sync (Gardner TED) → Costas Loop (order 4) → Constellation Decoder → Unpack K Bits → File Sink
```
- **Symbol Sync:** Gardner TED, loop BW 0.01, damping 1.0, out sps = 1
- **Costas Loop:** loop BW 0.0628, 4th-order (matches 90° rotational symmetry of inner points)

## Verification
`scripts/verify_bitstream.py` aligns TX/RX files for filter group delay and computes BER.

```
Best delay: 51 bits (17 symbols)
Bit errors: 0
BER: 0.00000000 → PERFECT MATCH
```

## Setup & Run
```bash
sudo apt install gnuradio gr-iio
pip install numpy

# Part 1 — PlutoSDR TX (connect Pluto @ ip:192.168.2.1)
gnuradio-companion flowgraphs/transmitter1.grc

# Part 2 — File TX
python scripts/generate_abcd_bits.py
gnuradio-companion flowgraphs/transmitter_2.grc

# Part 3 — TX + Loopback RX, then verify
gnuradio-companion flowgraphs/transmitter_3.grc
python scripts/verify_bitstream.py
```

## Key Concepts
- **RRC filtering** at TX and RX together form a Raised Cosine filter, eliminating ISI at the optimal sampling instant.
- **Rational resampling (128/125)** converts integer sps=5 to fractional sps=5.12 with exact integer arithmetic.
- **Gardner TED** locks symbol timing; **Costas loop** removes residual carrier phase rotation.
