# Marlim3 Developer Documentation

Welcome to the **Marlim3** developer documentation. This guide covers the internal architecture of the Marlim3 1D multiphase flow simulator developed by Petrobras.

## Contents

- [Num4Main.cpp — Main Entry Point](num4main.md)
- [Core Domain Classes](core-domain-classes.md)
- [Input Parsing and SProd Construction](input-parsing.md)
- [Steady-State Solution of a Single Tramo](steady-state-tramo.md)
- [Transient Solver for a Single Tramo](transient-solver.md)
- [Network Simulation](network-simulation.md)
- [Fluid Thermodynamic Modeling](fluid-thermodynamics.md)
- [Heat Transfer Modeling](heat-transfer.md)
- [Injection Well Simulation](injection-well.md)
- [2D Natural Convection Solver](natural-convection.md)
- [Sensitivity Analysis](sensitivity-analysis.md)

## Overview

Marlim3 is a C++/Fortran simulator for 1D multiphase flow in oil & gas production systems. It supports:

- **Steady-state and transient** simulations of single pipeline segments (*tramos*)
- **Production and injection networks** composed of multiple interconnected segments
- **Gas-lift loop networks** and **parallel network** topologies
- **Natural convection** analysis in confined cross-sections (2D finite-volume solver)
- **Compositional and black-oil** fluid models
- **Artificial lift**: gas-lift valves, ESP pumps, volumetric pumps
- **Thermal diffusion**: coupled 2D and 3D Poisson solvers
- **Sensitivity analysis**: automated parameter sweeps

## Build System

The project uses **CMake** with presets defined in `CMakePresets.json`. See [README.md](../../README.md) for build instructions.
