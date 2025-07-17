---
layout: single
title: "TDR FPGA Progress"
date: 2025-07-17
author: Nathan Zhang
categories: [blog]
tags: [fpga, ml, tdr]
---

# Simulating Time Domain Reflectometry (TDR) on an FPGA

## Project Overview

This project aims to simulate a Time Domain Reflectometry (TDR) system using an FPGA. The core idea is to mimic how a real TDR tool sends out a fast electrical pulse down a transmission line and observes reflections caused by impedance mismatches. These reflections help locate and characterize faults or discontinuities in the line.

Given that actual high-speed transmission behavior is difficult to model directly in an FPGA, the project simulates this digitally by:

* Generating a short pulse when a trigger button is pressed.
* Feeding this pulse into a simulated delay line.
* Introducing reflections after a fixed delay, based on a hardcoded reflection type (e.g., short, open).
* Capturing the resulting waveforms into 8-bit segments.
* Transmitting these segments via UART to a connected USB-UART adapter.
* Using LEDs to indicate the presence and type of reflections.

## What Has Been Done So Far

* Designed a `pulse_gen` module to produce a one-cycle pulse on button press.
* Built a `tdr_line_sim` module with a circular shift register to simulate a transmission line with programmable reflection behavior and delay.
* Added a `waveform_capture` module that converts the digital wave output into 8-bit UART-ready chunks.
* Implemented a `uart_tx_core` module that transmits the captured waveform bytes at 115200 baud.
* Defined four LEDs to indicate reflection types: open, short, multi, and none.
* Assigned pins in Quartus Pin Planner.
* Connected a USB-UART adapter to the `uart_tx` pin for waveform visualization via serial monitor.

## Current Behavior

* When the system powers up and the trigger button is pressed:

  * The third LED (indicating a short reflection) lights up and stays on until reset.
  * The serial monitor connected to the USB-UART adapter continuously prints:

```
00000000
00000000
00000000
...
11111111
```

* The stream of `00000000` bytes continues until power is disconnected. After disconnecting, the last captured byte was `11111111`.

## Current Issues

* The captured UART data appears to be stuck, constantly outputting `00000000` instead of varied waveform values.
* The only noticeable change occurs after disconnecting the board, resulting in a final byte of `11111111`, suggesting that the shift buffer might not be capturing correctly or the waveform is not being simulated as expected.
* It's unclear whether the pulse is being latched into the delay line or if the simulation is not triggering the reflection properly.
* UART transmission is working in terms of electrical communication, but the content being transmitted does not reflect the simulated waveform accurately.

## Next Steps

1. **Debug waveform\_capture module**:

   * Add debug LEDs or a logic analyzer tap to confirm the wave signal is toggling and shifting.
   * Check if the bit counter and data\_valid flags are functioning correctly.

2. **Review tdr\_line\_sim logic**:

   * Confirm that the reflection type and delay are being triggered after the pulse.
   * Add logging via LEDs to show when reflection insertion occurs.

3. **Test with alternate reflection types** to see if any variation appears in the UART output.

4. **Add a UART receive monitor** on the PC side to record and visualize byte streams as a waveform.


## Code Highlights

### Top-Level Module: `tdr_top`

```verilog
// Top-Level Module Declaration
module tdr_top (
    output logic led_reflect_00,
    output logic led_reflect_01,
    output logic led_reflect_10,
    output logic led_reflect_11,
    output logic pulse_led,
    input  logic clk,
    input  logic rst,
    input  logic trigger_btn,
    output logic uart_tx
);
    // Module wiring and instantiation here...
endmodule
```

This is the root module that instantiates all the key components: pulse generator, delay line, waveform capture, and UART transmitter.

### Pulse Generator

```verilog
module pulse_gen (
    input  logic clk,
    input  logic rst,
    input  logic trigger,
    output logic pulse
);
    logic trigger_d;
    always_ff @(posedge clk or posedge rst) begin
        if (rst) begin
            trigger_d <= 0;
            pulse     <= 0;
        end else begin
            trigger_d <= trigger;
            pulse     <= trigger & ~trigger_d;
        end
    end
endmodule
```

This module generates a single-cycle pulse when the trigger button is pressed. It's used to initiate a signal through the simulated transmission line.

### TDR Line Simulation

```verilog
module tdr_line_sim (
    input  logic clk,
    input  logic rst,
    input  logic pulse_in,
    input  logic [1:0] reflect_type,
    input  logic [3:0] reflect_delay,
    output logic wave_out
);
    // Logic for delay line and injecting reflection...
endmodule
```

This module models the delay and the reflection behavior digitally. You can set the type and delay of the reflection manually.

### Waveform Capture

```verilog
module waveform_capture (
    input  logic clk,
    input  logic rst,
    input  logic wave_in,
    output logic [7:0] data_out,
    output logic data_valid
);
    // Sampling logic: 8 samples collected then flagged as valid
endmodule
```

It shifts in one waveform bit per cycle. Once 8 bits are collected, it asserts `data_valid` and sends the result to the UART.

### UART Transmission

```verilog
module uart_tx_core (
    input  logic clk,
    input  logic rst,
    input  logic [7:0] data_in,
    input  logic data_valid,
    output logic tx
);
    // UART state machine: IDLE -> START -> DATA -> STOP
endmodule
```

Implements UART byte-wise transmission. The baud rate divider is tuned for 115200 baud assuming a 50 MHz clock.

For the full codebase, feel free to visit this repository: https://github.com/natjiazhan/ML-TDR-Fault-Detector/tree/main/fpga

This blog post documents the current milestone in building a digital TDR simulation on an FPGA, serving as a checkpoint for debugging and planning future iterations. Once stable, the system can evolve into a full diagnostic tool for cable fault simulation, teaching, or embedded testing environments.
