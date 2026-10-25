# akrida-demo

An example RTL repository verified by [Akrida](https://tryakrida.com), the AI
verification engineer.

`blocks/axil_ram/` is an AXI4-Lite RAM (RTL from Alex Forencich's
verilog-axi, MIT licensed) with its block spec. Akrida reads only the spec,
writes a UVM testbench, runs it on every commit, and files the design bugs
it finds as issues in this repo. When a later commit fixes a bug, Akrida
re-runs the failing test and closes the issue. When a bug appears, Akrida
bisects the history to the commit that introduced it.
