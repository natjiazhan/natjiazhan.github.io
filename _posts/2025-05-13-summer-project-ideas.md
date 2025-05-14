---
layout: single
title: "Summer Project Ideas"
date: 2025-05-13
author: Nathan Zhang
categories: [blog]
tags: [fpga, ml, tdr, gaussian splitting, summer, ideas]
---

## 2025 Summer Project Ideas
---
The past semester, I have been learning system verilog and applying it for use during labs on FPGAs. I have learned to love this process, and I want to find more ways to apply and expand on these skills. In addition, I have found interest in semiconductor fabrication tools as well as machine learning. For these reasons, I want my summer project(s) centered around FPGA, ML, and the SC industry. Outlined below are three ideas I have been playing around with.
---

### 1. Time Domain Reflectometry (TDR) Emulation on FPGA

**Goal:** Digitally model signal propagation along an interconnect and observe reflections due to faults like opens and shorts.

**Parts Needed:**
- **Pulse Generator**  
  Emits short voltage pulses (step or impulse); user-triggered or clock-driven.
- **Transmission Line Model**  
  Emulated as a shift register or tapped delay line; configurable impedance per segment; faults simulated by reflection logic (e.g., open = full reflection).
- **Reflection Handler**  
  Applies reflection coefficients based on impedance mismatch; can reverse or attenuate the signal.
- **Sampler / Observer**  
  Periodically samples the line to capture returning waveforms; stores samples in a buffer or displays directly.
- **Output Interface**  
  Display waveform via VGA/HDMI monitor.

---

### 2. Point-Based Renderer / Gaussian Splatting Engine

**Goal:** Render 3D point cloud data to a 2D image using transformation and Gaussian-based “splatting” instead of discrete pixels.

**Parts Needed:**
- **Point-Memory / Input Loader**  
  Stores point cloud data (x, y, z, color); can be hardcoded as a testbench.
- **Transformation Unit**  
  Applies 4×4 model-view-projection matrix to each 3D point using fixed-point math.
- **Splat Kernel Generator**  
  For each point, generate a small Gaussian kernel (e.g., 3×3 or 5×5 blur); adds weighted contributions to surrounding pixels.
- **Framebuffer Controller**  
  Stores the rendered image in a 2D buffer; may include Z-buffer for depth visibility.
- **Display Output**  
  Outputs final image via VGA or HDMI.

---

### 3. ML-Accelerated Fault Detection in Hardware

**Goal:** Inject faults into a digital signal or system and classify the behavior (faulty or normal) using ML or heuristic models.

**Modules:**
- **Signal Generator**  
  Creates sequences of digital signals or processor traces; simulates glitches, delays, or missing pulses.
- **Fault Injector**  
  Random or rule-based error injection; controllable rate, type, and position.
- **Feature Extractor**  
  Computes basic signal features such as:
  - Edge count
  - Pulse width
  - Timing variation
  - Mean, variance, high/low duration
- **Classifier**  
  Lightweight ML engine:
  - Threshold logic
  - Decision tree
  - Binarized neural net (BNN)  
  Compares features and produces binary classification.
- **Classification Output**  
  Results displayed via:
  - LEDs
  - On-screen text (VGA/HDMI)
- **Training / Simulation Pipeline** *(optional)*  
  - Train model on synthetic data  
  - Export weights for use in FPGA inference
