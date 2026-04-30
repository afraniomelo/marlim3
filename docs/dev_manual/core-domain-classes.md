# Core Domain Classes

This document provides a high-level overview of the main classes that compose a simulation in Marlim3. Detailed internals are covered in the dedicated documents; this document describes each class's role, responsibilities, and relationships.

**Key source files:**

| File | Role |
|------|------|
| [`src/SisProd.h`](../../src/SisProd.h) / [`SisProd.cpp`](../../src/SisProd.cpp) | `SProd` — production system (tramo) |
| [`src/celula3.h`](../../src/celula3.h) / [`celula3.cpp`](../../src/celula3.cpp) | `Cel` — cell (control volume) |
| [`src/Geometria.h`](../../src/Geometria.h) | `DadosGeo` — pipe geometry |
| [`src/acessorios.h`](../../src/acessorios.h) | `acessorio` — cell accessories |
| [`src/FonteMas.h`](../../src/FonteMas.h) | `AbsFonte`, `IPR`, `InjGas`, `InjLiq`, `InjMult` — mass sources |
| [`src/Acidentes2.h`](../../src/Acidentes2.h) | `MudaArea`, `choke` — area changes and choke valves |
| [`src/estrat.h`](../../src/estrat.h) | `estratificado` — stratified-flow parameters |
| [`src/celulaGas.h`](../../src/celulaGas.h) | `CelG` — gas-lift service-line cell |
| [`src/chokegas.h`](../../src/chokegas.h) | `ChokeGas` — compressible-gas choke valve |

---

## Table of Contents

1. [SProd — Production System](#sprod--production-system)
2. [Cel — The Cell (Control Volume)](#cel--the-cell-control-volume)
3. [DadosGeo — Pipe Geometry](#dadosgeo--pipe-geometry)
4. [acessorio — Cell Accessories](#acessorio--cell-accessories)
5. [estratificado — Stratified Flow Parameters](#estratificado--stratified-flow-parameters)
6. [CelG — Gas-Lift Service Line Cell](#celg--gas-lift-service-line-cell)
7. [ChokeGas — Gas Injection Choke](#chokegas--gas-injection-choke)

---

## SProd — Production System

`SProd` ([`SisProd.h`](../../src/SisProd.h) / [`SisProd.cpp`](../../src/SisProd.cpp)) represents a single **tramo** (pipeline segment) and contains all the methods needed to simulate it. A network is represented as a vector of `SProd` objects.

In `main()`, the objects are named:

- **`sistem1`** — single tramo (a single `SProd` object)
- **`malha`** — interconnected tramos, represented as a pointer of pointers (matrix) of `SProd`. The matrix structure allows multiple independent sub-networks

The `SProd` constructor performs two critical actions:

1. Constructs the **`arq`** member (type `Ler`) in the initializer list — triggers the full JSON parse
2. Calls **`montasistema()`** at the end of the constructor body — builds the cell array and attaches accessories

The steady-state solver methods on `SProd` are organized into two categories:

- **"Marcha" methods** — objective functions that march cell-by-cell and return a residual:
  - `marchaProdPerm1(pchute)`: BCs are downstream pressure + upstream mass source (e.g. IPR). Choke is fully open. Returns the mismatch between the computed and specified downstream pressure.
  - `marchaProdPerm2(pchute)`: Same march but with a restricted choke valve. Since the post-choke pressure always exceeds separator pressure, the objective function is the mismatch between the computed flow rate and the choke-predicted flow rate.
- **"Busca" methods** — root-finding algorithms that drive the marcha residual to zero:
  - `buscaProdPfundoPerm()`: paired with `marchaProdPerm1`. Starts from a user-supplied initial guess or a rough hydrostatic estimate of upstream pressure, then uses Ridder's method to converge. Used when `ConContEntrada == 0` and choke is inactive.
  - `buscaProdPfundoPerm2()`: paired with `marchaProdPerm2`. Same algorithm, different objective. Used when `ConContEntrada == 0` and choke is active.
  - `buscaProdPfundoPerm3()`: variant for `ConContEntrada == 2` (pressure + injection flow-rate BC).
  - `buscaProdPfundoPermRev()`: handles reverse flow conditions (negative source rates).
  - `buscaProdPresPresPerm()`: used when `ConContEntrada == 1` (imposed inlet pressure). The unknown is the mass flow rate at the inlet. Choke inactive.
  - `buscaProdPresPresPerm2()` / `buscaProdPresPresPerm3()`: same, with active / closed choke.
  - `buscaInjPfundoPerm1..5()`: five variants for injection wells, selected by `condpocinj.CC` (0–4 and default).

---

## Cel — The Cell (Control Volume)

`Cel` ([`celula3.h`](../../src/celula3.h) / [`celula3.cpp`](../../src/celula3.cpp)) is the fundamental discretised unit of the pipeline. An `SProd` object is composed of an array of `Cel` objects (`celula[0..ncel]`), plus boundary conditions and events stored in `arq`.

Key design principles:

- **Self-sufficiency**: each cell stores copies of adjacent-cell state (left: `presL`, `tempL`, `dutoL`, `fluiL`; right: `presR`, `tempR`, `dutoR`, `fluiR`) plus pointers to neighbor fluids (`ProFlu *fluiL`, `ProFlu *fluiR`). This allows the cell to compute its own mass, energy, and momentum balances without traversing the array.
- **One accessory per cell**: each cell has exactly one `acessorio acsr` member. If two devices must exist at the same axial position, the cell must be split in two.
- **Fluid association**: each cell owns a `ProFlu flui` (hydrocarbon fluid) and a `ProFluCol fluicol` (complementary fluid), enabling multi-fluid simulations with distinct property models per cell.
- **Heat transfer**: each cell owns a `TransCal calor` object for radial heat exchange (see [Heat Transfer Modeling](heat-transfer.md)).

---

## DadosGeo — Pipe Geometry

`DadosGeo` ([`Geometria.h`](../../src/Geometria.h)) stores the cross-sectional dimensions of the pipe (inner diameter, wall layers, roughness, area, perimeter, inclination), as well as the thermal properties of each wall layer (conductivity, density, Cp). It also supports confined fluid layers within the wall (e.g. gas-filled or liquid-filled annuli), identified by the `tipomat` flag (`0` = solid, `2` = liquid, `3` = gas). See [Heat Transfer Modeling](heat-transfer.md) for the full member list.

---

## acessorio — Cell Accessories

The `acessorio` class ([`acessorios.h`](../../src/acessorios.h)) represents any device inserted inside a cell that modifies the flow conditions beyond what the boundary balances would produce. Each accessory must be associated with exactly one cell.

Accessories fall into two broad categories: **mass sources** and **localized pressure variations**.

The accessory type is identified by the integer flag `tipo`:

| `tipo` | Device | Category |
|--------|--------|----------|
| 0 | No accessory | — |
| 1 | Gas injection source | Mass source |
| 2 | Liquid injection source | Mass source |
| 3 | IPR (reservoir inflow) | Mass source |
| 4 | BCS (ESP pump) | Pressure variation |
| 5 | Choke valve | Pressure variation |
| 6 | Diameter change | Pressure variation |
| 7 | Generic ΔP | Pressure variation |
| 8 | Volumetric pump | Pressure variation |
| 9 | Fonte choke (leak through tubing wall, defined by leak area and external pressure) | Mass source |
| 10 | Multi-phase mass source (per-phase mass flow rates) | Mass source |
| 11 | Steam IPR | Mass source |
| 12 | Steam mass source | Mass source |
| 14 | Volumetric pump for steam | Pressure variation |
| 15 | Radial porous medium inflow | Mass source |
| 16 | 2D porous medium inflow | Mass source |
| 17 | Multi-stage BCS | Pressure variation |

> **Note:** Type 13 is unused (skipped in the enumeration). Steam-related types (11, 12, 14) are implemented in [`SisProdVap.cpp`](../../src/SisProdVap.cpp) for vapor line simulations.

The accessory classes that implement mass-source functionality are defined in [`FonteMas.h`](../../src/FonteMas.h):

| Class | Accessory types | Description |
|-------|-----------------|-------------|
| `AbsFonte` | — | Abstract base: `virtual double VMas(...)` |
| `IPR` | 3, 15, 16 | Reservoir inflow performance relationship |
| `InjGas` | 1 | Gas injection at specified rate/pressure |
| `InjLiq` | 2 | Liquid injection at specified rate/pressure |
| `InjMult` | 10 | Multi-phase mass injection (per-phase rates) |

Pipe area changes and multiphase choke valves are handled by the `Acidentes` module ([`Acidentes2.h`](../../src/Acidentes2.h)), which provides:
- `MudaArea` — models contraction/expansion with upstream/downstream area and pressure-drop / temperature-drop methods
- `choke` — multiphase choke valve model with subcritical/critical flow, area interpolation, and maximum mass flow calculation

---

## estratificado — Stratified Flow Parameters

The `estratificado` class ([`estrat.h`](../../src/estrat.h)) stores the detailed stratified-flow calculation results for a cell: film height (`hfilm`), liquid holdup (`holliq`), gas/liquid wall shear stresses (`twg`, `twl`), interfacial friction factor (`fi`), phase Reynolds numbers (`reyL`, `reyG`), superficial velocities (`uls`, `ugs`), pipe inclination (`ang`), drift-flux parameters (`coefC0`, `valUd`), and flow regime indicator (`arr`).

---

## CelG — Gas-Lift Service Line Cell

`CelG` ([`celulaGas.h`](../../src/celulaGas.h)) provides the cell type and marching methods for gas injection service lines (gas-lift annulus). It uses `ChokeGas` ([`chokegas.h`](../../src/chokegas.h)) for modeling compressible gas flow through injection orifices and valves.

---

## ChokeGas — Gas Injection Choke

`ChokeGas` ([`chokegas.h`](../../src/chokegas.h)) models compressible-gas choke valves used in gas-lift injection, including subcritical, critical, and liquid flow regimes. It contains a pre-computed table (`ventCR`) for critical flow calculations.
