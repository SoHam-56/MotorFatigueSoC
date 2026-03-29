# Real-Time Motor Fatigue Detection on PYNQ Z2

Hardware-accelerated signal processing and machine learning for edge-based induction motor condition monitoring, implemented as a hardware/software co-design on the Xilinx PYNQ Z2 SoC.

**Submitted by:** Soham Pramanik & Soukarya Biswas
**Supervisor:** Prof. Sayan Chatterjee, Jadavpur University  
**Department:** Electronics & Tele-Communication Engineering

---

## Overview

Induction motors are critical to industrial operations, yet susceptible to gradual electrical fatigue — insulation degradation caused by voltage spikes, harmonics, thermal cycling, and mechanical stress. Catastrophic failure is typically preceded by subtle degradation that goes undetected by traditional monitoring approaches.

This project implements a **real-time motor fatigue detection pipeline** that:

1. Acquires 3-phase stator current signals
2. Applies Park's Vector transformation to extract direct-axis (Id) and quadrature-axis (Iq) components
3. Computes the Space Vector Magnitude and isolates one representative current cycle
4. Extracts 11 statistical features (mean, RMS, std. dev, skewness, kurtosis, peak-to-peak, crest factor, shape factor, impulse factor, margin factor, energy)
5. Feeds these features into a trained LDA + classifier model to predict **Healthy** vs **Fatigued** motor state

The same pipeline is implemented twice — once in Python (ARM Cortex-A9) and once as custom FPGA logic via HLS — to benchmark hardware acceleration benefits.

---

## Results

| Implementation | Platform | Execution Time |
|---|---|---|
| Software | Python on ARM Cortex-A9 | 20.46 ms |
| Hardware | FPGA fabric (HLS) | 1.531 ms |
| **Speedup** | | **~13.36×** |

The FPGA implementation achieves over 13× speedup through parallel, pipelined custom datapaths, enabling true real-time edge processing.

---

## Repository Structure

| Directory | Contents |
|---|---|
| `src/` | HLS C++ source files for the hardware IP cores |
| `testbenches/` | HLS testbenches and sample motor current data |
| `PYNQZ2_IPs/` | Packaged Vivado HLS IP cores (.zip) |
| `PYNQZ2_VIVADO/` | Full Vivado block design project |
| `BitStreams/` | Pre-built `.bit` and `.hwh` files for the PYNQ Z2 overlay |
| `Jupyter/` | Notebooks for running HW inference and SW baseline timing |

---

## Signal Processing Pipeline

```
Raw 3-phase currents (Ia, Ib, Ic)
        │
        ▼
Normalize (per-unit conversion)
        │
        ▼
Park's Transformation (Clarke → αβ → dq)
        │
   ┌────┴────┐
   ▼         ▼
  Id (flux)  Iq (torque)
   │         │
   │    Zero-crossing detection on Iq
   │         │
   └────┬────┘
        ▼
Space Vector Magnitude: Pvm = √(Id² + Iq²)
        │
Cycle isolation using Iq zero-crossings
        │
DC offset removal (subtract cycle mean)
        │
Statistical feature extraction (×11)
        │
LDA transformation + classifier
        │
   ┌────┴────┐
   ▼         ▼
Healthy   Fatigued
```

---

## Hardware Architecture

The FPGA pipeline consists of six custom HLS IP cores connected via AXI4-Stream interfaces, coordinated by AXI DMA engines:

| IP Core | Function |
|---|---|
| `pack_stream_to_blk` | Packs AXI stream samples into block format |
| `auto_parkcalc_two_streams` | Clarke + Park transformation, outputs Id and Iq streams |
| `complex_mag_stream` | Computes Space Vector Magnitude (Pvm) |
| `zero_cross` | Detects Iq zero-crossings, determines cycle boundaries |
| `calculate_statistics` | Extracts 11 statistical features from the isolated cycle |
| `unpack_blk_to_stream` | Unpacks result block back to AXI stream for DMA transfer |

The ARM Cortex-A9 (PS) loads the bitstream, orchestrates DMA transfers, and runs the final ML inference (LDA + classifier) in Python.

---

## Getting Started

### Prerequisites

- Xilinx PYNQ Z2 board running PYNQ v2.6 or later
- Python 3 with `numpy`, `pynq` packages (pre-installed on PYNQ image)
- Vivado 2020.2 (to rebuild the project from source)
- Vitis HLS 2020.2 (to modify/re-synthesize IP cores)

### Running the pre-built demo

1. Clone this repository directly onto your PYNQ Z2 board.
2. Navigate to the `Jupyter/` directory.
3. Open `MotorFatigue.ipynb`. This notebook automatically loads the hardware overlay from the `BitStreams/` folder and begins processing data through the FPGA fabric.
4. To benchmark performance against a standard CPU execution, open and run `SW_Time.ipynb`.

### Rebuilding from source

To open and run the full Vivado project:

```bash
# Open the project in Vivado
vivado PYNQZ2_VIVADO/MotorFault_PYNQZ2.xpr
```

To re-synthesize an individual IP core — for the same or a different target FPGA — import the corresponding `.zip` from `PYNQZ2_IPs/` into Vitis HLS, or open the source files in `src/` directly and retarget to your part.

---

## IP Core Details

Each IP in `PYNQZ2_IPs/` was synthesized with Vitis HLS 2020.2 targeting the Zynq-7000 (xc7z020clg400-1). All cores use `ap_fixed<32,16>` fixed-point arithmetic and expose AXI4-Stream slave/master interfaces for streaming throughput.

Key HLS directives used:
- `#pragma HLS PIPELINE II=1` on inner loops for maximum throughput
- `#pragma HLS DATAFLOW` at top level to enable concurrent stage execution
- `#pragma HLS INTERFACE axis` for stream ports
- `#pragma HLS INTERFACE s_axilite` for control registers

---

## Algorithm Background

**Park's Vector (dq transformation):** Converts the 3-phase stator current from the stationary abc frame to a rotating dq reference frame aligned with the rotor's magnetic field. The quadrature axis (Iq) encodes torque-related information; subtle changes in its waveform across cycles serve as a fatigue signature.

**Feature extraction:** 11 statistical descriptors are computed over one isolated Pvm cycle after DC offset removal: mean, standard deviation, RMS, skewness, kurtosis, peak-to-peak amplitude, crest factor, shape factor, impulse factor, margin factor, and signal energy.

**Classifier:** A Linear Discriminant Analysis (LDA) projection followed by a trained classifier (details in `Jupyter/MotorFatigue.ipynb`) distinguishes healthy from fatigued motor states based on these features.

---



## Potential Applications

- Predictive maintenance scheduling in manufacturing
- Industrial automation (CNC machines, robotic arms, conveyor systems)
- HVAC monitoring (fans, pumps, compressors)
- Renewable energy (wind turbine generators)
- Smart embedded sensors for motor health

---

## Documentation

- `Final_UG4_prj.pdf` — Full undergraduate project report (background, methodology, results)
- `Real-Time Motor Fatigue Detection using Hardware-Accelerated Signal Processing.pdf` — Conference/presentation paper
- `Flow.mermaid` — Algorithm flowchart source (render with any Mermaid-compatible tool)

---

## Authors

- **Soham Pramanik** — Jadavpur University
- **Soukarya Biswas** — Jadavpur University

**Supervisors:**
- Prof. Sayan Chatterjee (Jadavpur University)
- Dr. Swagata Mandal (Calcutta University / Jalpaiguri Government Engineering College)

---

## License

Academic project submitted in partial fulfillment of the B.E. E.T.C.E. degree at Jadavpur University, session 2024–25. All rights reserved by the authors.
