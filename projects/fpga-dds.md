---
layout: default
title: FPGA DDS Walkthrough
---

# FPGA DDS Walkthrough

This tutorial describes how to build a DDS using Verilog and a lookup table.

A low cost AMD/Xilinx Zynq 7 board is used for prototyping.  A Digilent DA3 DAC is connected to a PMOD output.

A saleae logic analyzer is used for probing digital signals on a PMOD header from the Zynq 7 board.

## Architecture

- Phase accumulator
- Sine LUT
- DAC interface

