# Steady-State Solution of a Single Tramo

This document describes the logic and algorithms used by the Marlim3 simulator to compute the **steady-state (permanent) solution** of a single pipeline segment (*tramo*). A tramo is the fundamental simulation unit — a 1D sequence of control volumes (cells) representing a pipe or well, including artificial-lift devices, sources, and sinks.

**Key source files:**

| File | Role |
|------|------|
| [`src/Num4Main.cpp`](../../src/Num4Main.cpp) | Top-level orchestration: `SolveTramoSolteiro()`, `permanenteSimples()` |
| [`src/SisProd.cpp`](../../src/SisProd.cpp) | Core solver methods on the `SProd` class |
| [`src/SisProd.h`](../../src/SisProd.h) | `SProd` class declaration |
| [`src/celula3.h`](../../src/celula3.h) / [`celula3.cpp`](../../src/celula3.cpp) | `Cel` class — per-cell state and low-level physics |
| [`src/PropFlu.h`](../../src/PropFlu.h) / [`PropFlu.cpp`](../../src/PropFlu.cpp) | `ProFlu` — fluid property correlations (black-oil & compositional) |
| [`src/FonteMas.cpp`](../../src/FonteMas.cpp) | Mass source term evaluation |
| [`src/GradientCorrelations.cpp`](../../src/GradientCorrelations.cpp) | Standalone friction-factor and pressure-gradient correlations |
| [`src/Leitura.cpp`](../../src/Leitura.cpp) | JSON input parsing → `SProd` initialization |
| [`src/celulaGas.h`](../../src/celulaGas.h) / [`celulaGas.cpp`](../../src/celulaGas.cpp) | `CelG` class — gas-lift service line cell state and physics |
| [`src/chokegas.h`](../../src/chokegas.h) / [`chokegas.cpp`](../../src/chokegas.cpp) | `ChokeGas` — compressible gas choke model for VGL orifices |

---

## Table of Contents

1. [Overview](#overview)
2. [Call Hierarchy](#call-hierarchy)
3. [Step 1 — Object Construction](#step-1--object-construction)
4. [Step 2 — SolveTramoSolteiro: Fluid Model Strategy](#step-2--solvetramosolteirofluid-model-strategy)
5. [Step 3 — permanenteSimples: Boundary Condition Dispatch](#step-3--permanentesimples-boundary-condition-dispatch)
6. [Step 4 — buscaProdPfundoPerm: Bracket Search & Root Finding](#step-4--buscaprodpfundoperm-bracket-search--root-finding)
7. [Step 5 — marchaProdPerm1: The Cell-by-Cell March](#step-5--marchaprodperm1-the-cell-by-cell-march)
8. [The Marching Equations](#the-marching-equations)
9. [Convergence and Root Finding](#convergence-and-root-finding)
10. [Boundary Condition Variants](#boundary-condition-variants)
11. [Gas-Lift Service Line](#gas-lift-service-line)
12. [Compositional Model Acceleration](#compositional-model-acceleration)
13. [Output and Post-Processing](#output-and-post-processing)
14. [Summary of Key Methods](#summary-of-key-methods)

---

## Overview

The steady-state solver computes the **pressure, temperature, void fraction, and flow-rate profiles** along a tramo under time-invariant conditions. The fundamental approach is:

1. **Guess** a boundary value (bottom-hole pressure or inlet mass flow rate)
2. **March** cell-by-cell from the upstream end to the downstream end, computing pressure, temperature, and mass flow at each cell
3. **Evaluate the residual** — the mismatch between the computed downstream state and the imposed downstream boundary condition (e.g., separator pressure or choke flow)
4. **Iterate** using Ridder's root-finding method until the residual is zero

This is a **shooting method**: the unknown boundary value is the "shot", and the marching equations propagate that shot through the domain.

```
    Upstream (cell 0)                                  Downstream (cell ncel)
    ┌────────────────────────────────────────────────────────────┐
    │  Source/IPR    →  cell 1  →  cell 2  → ... →  cell ncel   │
    │  (BC: P or Q)    march     march              (BC: P_sep) │
    └────────────────────────────────────────────────────────────┘
           ↑ GUESS                                    ↑ RESIDUAL
           └──────────── Ridder's method ─────────────┘
```

---

## Call Hierarchy

The steady-state solution for a single tramo follows this call chain:

```
main()
 └─ SolveTramoSolteiro(sistem1)          [Num4Main.cpp]
     ├─ (optional) Switch to black-oil mode for initial convergence
     ├─ permanenteSimples(sistem1)        [Num4Main.cpp]
     │   ├─ Detect BC type and flow direction
     │   └─ buscaProdPfundoPerm(chute)    [SisProd.cpp]
     │       ├─ Estimate initial pressure guess (hydrostatic walk)
     │       ├─ marchaProdPerm1(pchute)   [SisProd.cpp]  ← first trial
     │       │   ├─ Initialize cell 0 from source/IPR
     │       │   └─ for i = 1 to ncel:
     │       │       ├─ RenovaPresPermMon(i)    — P: cell center → left boundary
     │       │       ├─ atualizaPeriPmonProd(i)  — Apply BCS/pump ΔP
     │       │       ├─ RenovaMassPerm(i)        — Mass flow + source terms
     │       │       ├─ RenovaTempPerm(i)        — Energy equation
     │       │       ├─ RenovaPresPermJus(i)     — P: left boundary → cell center
     │       │       ├─ atualizaPeriPjusProd(i)  — Apply BCS/pump ΔP (jusante)
     │       │       └─ RenovaTransMassPerm(i-1) — Interphase mass transfer
     │       ├─ Bracket search: find pchute_neg, pchute_pos with opposite residual signs
     │       └─ zriddr(pchute_neg, pchute_pos)  [SisProd.cpp]
     │           └─ multMarcha(x) → marchaProdPerm1(x) (repeatedly)
     ├─ (optional) Switch back to compositional, use black-oil result as initial guess
     ├─ permanenteSimples(sistem1)        ← second pass (compositional)
     └─ Print profiles, trends, log file
```

---

## Step 1 — Object Construction

When `main()` creates the `SProd` object:

```cpp
SProd sistem1(nomeArquivoEntrada, nomeArquivoLog, validacaoJson, tipoSimulacao, &vg1dTramo);
```

The constructor ([`SisProd.cpp`](../../src/SisProd.cpp)) performs:

1. **Constructs the `arq` member** (type `Leitura`) — reads the JSON input file, parses geometry, fluid properties, boundary conditions, accessories, and mesh parameters
2. **Allocates the cell array** — `celula[0..ncel]`, each a `Cel` object containing geometry (`DadosGeo duto`), fluid properties (`ProFlu flui`, `ProFluCol fluicol`), and solver state (pressure, temperature, void fraction, mass flows)
3. **Initializes solver state** — zeroes counters, flags, pointers, and intermediate variables
4. **Allocates profile output tables** — `flut` for production line, `flutG` for gas-lift line
5. **(If gas-lift)** Allocates the gas-line cell array `celulaG[]` and sets up valve positions

The `Cel` class ([`celula3.h`](../../src/celula3.h)) holds per-cell state organized into:

| Category | Key Fields |
|----------|------------|
| Geometry | `dutoL`, `duto`, `dutoR` — diameter, area, perimeter, inclination, roughness |
| Pressure | `presL`, `presaux` (left boundary), `pres` (center), `presR` |
| Temperature | `tempL`, `temp`, `tempR` |
| Mass flow | `MC` (total), `Mliqini` (liquid), `QG` (gas volume), `QL` (liquid volume) |
| Void fraction | `alf` (gas), `bet` (complementary fluid fraction) |
| Source terms | `fontemassLR`, `fontemassCR`, `fontemassGR` |
| Accessories | `acsr` — BCS pump, valve, source, IPR, etc. |
| Drift-flux | `c0` (distribution parameter), `ud` (drift velocity), `term1`, `term2` |
| Heat transfer | `calor` — `TrocaCalor` object with pipe wall, insulation, environment model |
| Fluid models | `flui` (owned `ProFlu`), `fluiL`/`fluiR` (pointers to neighbor fluids), `fluicol` (`ProFluCol`) |

Each cell stores copies of its neighbors' state (`presL`/`presR`, `tempL`/`tempR`, `dutoL`/`dutoR`) and pointers to neighbor fluid objects (`fluiL`, `fluiR`), making it self-sufficient for computing local balances. Each cell can hold exactly **one accessory** (`acsr`); if two devices occupy the same position, the cell must be split. Each cell is associated with a specific fluid model (`flui`), enabling multi-fluid simulations with different properties per segment.

---

## Step 2 — SolveTramoSolteiro: Fluid Model Strategy

`SolveTramoSolteiro()` ([`Num4Main.cpp`](../../src/Num4Main.cpp)) is a wrapper that handles the **two-pass strategy** for compositional simulations:

### Black-Oil or Flash Table Modes (`flashCompleto != 2`)

For black-oil or flash table fluid models, a single call to `permanenteSimples()` suffices:

```
SolveTramoSolteiro
 └─ permanenteSimples(sistem1, chute0)  →  converged profiles
```

### Compositional Mode (`flashCompleto == 2`)

Compositional flash calculations are expensive. To accelerate convergence, the solver uses a **two-pass approach**:

```
SolveTramoSolteiro
 ├─ Pass 1: Switch to black-oil mode
 │   ├─ Set flashCompleto = 0 on all cells and accessories
 │   └─ permanenteSimples(sistem1, chute0)  →  approximate profiles
 │
 ├─ Extract initial guess from black-oil result:
 │   └─ inichute = cell[0].pres  (or derived flow rate)
 │
 ├─ Restore compositional mode (flashCompleto = 2)
 ├─ (optional) preparaTabDin() — build dynamic PVT tables from black-oil pressure range
 │
 └─ Pass 2: permanenteSimples(sistem1, inichute)  →  final compositional profiles
```

This avoids the expensive flash calculations during the initial bracket search, when pressure guesses may be far from the solution.

---

## Step 3 — permanenteSimples: Boundary Condition Dispatch

`permanenteSimples()` ([`Num4Main.cpp`](../../src/Num4Main.cpp)) selects the solver variant based on the **upstream and downstream boundary conditions**.

### Decision Tree

```
permanenteSimples
 │
 ├─ Production system (pocinjec == 0)?
 │   ├─ ConContEntrada == 0  (source BC at inlet: flow rate, IPR, or mass source)
 │   │   ├─ Flow direction?
 │   │   │   ├─ Normal (upstream → downstream):
 │   │   │   │   ├─ Choke wide open (abertura > 0.6):
 │   │   │   │   │   ├─ Gas-lift with pressure BC? → iterate on gas injection rate
 │   │   │   │   │   └─ Otherwise → buscaProdPfundoPerm()
 │   │   │   │   └─ Choke active (abertura ≤ 0.6):
 │   │   │   │       ├─ Gas-lift with pressure BC? → iterate (same logic)
 │   │   │   │       └─ Otherwise → buscaProdPfundoPerm2()
 │   │   │   └─ Reverse flow → buscaProdPfundoPermRev()
 │   │   │
 │   ├─ ConContEntrada == 1  (pressure BC at inlet)
 │   │   ├─ Choke wide open → buscaProdPresPresPerm()
 │   │   ├─ Choke active    → buscaProdPresPresPerm2()
 │   │   └─ Choke closed    → buscaProdPresPresPerm3()
 │   │
 │   └─ ConContEntrada == 2  (pressure + flow rate at inlet)
 │       └─ buscaProdPfundoPerm3()
 │
 └─ Injection system (pocinjec == 1)?
     ├─ CC == 0 → buscaInjPfundoPerm2()
     ├─ CC == 1 or 3 → buscaInjPfundoPerm1()
     ├─ CC == 2 → buscaInjPfundoPerm3()
     ├─ CC == 4 → buscaInjPfundoPerm4()
     └─ otherwise → buscaInjPfundoPerm5()
```

### Key Boundary Condition Types

| Upstream BC (`ConContEntrada`) | Unknown | March Function | Root-Finding Target |
|-------------------------------|---------|----------------|---------------------|
| `0` — Flow rate / IPR | Bottom-hole pressure | `marchaProdPerm1` | $P_{\text{sep}} - P_{\text{last cell}} = 0$ |
| `0` — Flow rate + active choke | Bottom-hole pressure | `marchaProdPerm2` | $\dot{m}_{\text{last cell}} - \dot{m}_{\text{choke}} = 0$ |
| `1` — Pressure at inlet | Mass flow rate | `marchaProdPresPres1` | $P_{\text{sep}} - P_{\text{last cell}} = 0$ |

---

## Step 4 — buscaProdPfundoPerm: Bracket Search & Root Finding

`buscaProdPfundoPerm()` ([`SisProd.cpp`](../../src/SisProd.cpp)) finds the bottom-hole pressure that satisfies the downstream boundary condition. It has four phases:

### Phase 1: Initial Pressure Estimate

If no initial guess is provided (`chute < 0`), the method walks **backwards** from the downstream end (cell `ncel`) to the upstream end (cell 0), accumulating hydrostatic and friction contributions:

```
pchute = P_sep  (separator pressure)
for i = ncel down to 1:
    pchute += (ρ_mix · g · sin(θ) · Δx + f_friction · Δx) / 98066.5
```

Where:
- $\rho_\text{mix}$ is estimated from a guessed void fraction
- Friction uses the Fanning equation with Haaland/Colebrook correlation
- Pressure units are **kgf/cm²** (the factor 98066.5 converts from Pa)
- BCS pump heads are subtracted (`dpB`)
- IPR/reservoir pressure caps are respected

### Phase 2: First March

Runs `marchaProdPerm1(pchute)` and checks the residual sign:
- **Residual < 0** (`pchute` too high): the march overshoots — separator pressure is lower than computed
- **Residual > 0** (`pchute` too low): the march falls short
- **Residual = ±1e10**: march failed (negative pressures, PVT table out of bounds)

### Phase 3: Bracket Search

The method needs two pressure guesses with **opposite residual signs** to start the root finder. Starting from the initial guess, it:

- If residual < 0: holds `chuteNeg = pchute`, then **reduces** `pchute` by factor `(1 - buscaFC)` until the residual flips positive
- If residual > 0: holds `chutePos = pchute`, then **increases** `pchute` by factor `(1 + buscaFC)` until the residual flips negative

The parameter `buscaFC` (default ~0.1) controls the bracket expansion step size.

Special handling:
- If a march returns `−1e10` (pressure collapsed), the method tries a new estimate with higher liquid holdup
- If a march returns `+1e10` (pressure exceeded IPR limit), it tries with a lower-density estimate

### Phase 4: Root Finding (Ridder's Method)

Once a valid bracket `[chuteNeg, chutePos]` is found:

```cpp
return zriddr(chuteNeg, chutePos, 1, 0);
```

`zriddr()` ([`SisProd.cpp`](../../src/SisProd.cpp)) implements **Ridder's method**, which is superlinearly convergent (order ~1.84) and guaranteed to converge within a bracket. Each iteration:

1. Evaluates the residual at the bracket midpoint: $f_m = f\left(\frac{x_1 + x_2}{2}\right)$
2. Computes the Ridder update: $x_\text{new} = x_m + (x_m - x_1) \cdot \text{sign}(f_1 - f_2) \cdot \frac{f_m}{\sqrt{f_m^2 - f_1 \cdot f_2}}$
3. Evaluates $f(x_\text{new})$ and narrows the bracket

Each evaluation calls `multMarcha(x, prod=1, tipoCC=0)` which dispatches to `marchaProdPerm1(x)` — a full cell-by-cell march.

Convergence tolerance: $|f| < 10^{-5}$ kgf/cm². Maximum iterations: 100.

---

## Step 5 — marchaProdPerm1: The Cell-by-Cell March

`marchaProdPerm1()` ([`SisProd.cpp`](../../src/SisProd.cpp)) is the **core marching routine**. Given a pressure guess `pchute` at cell 0, it sweeps from cell 1 to cell `ncel`, computing all profiles.

### Initialization (Cell 0)

Based on the source type at cell 0 (`acsr.tipo`):

| `tipo` | Source Type | Initialization |
|--------|------------|----------------|
| `0` | None | Set void fraction $\alpha = 1$, $\beta = 0$ |
| `1` | Gas injection | Compute gas/liquid split from flash at `pchute`, `T_source` |
| `2` | Liquid injection | Compute gas/liquid split from RGO, Bo, Ba at `pchute` |
| `3` | IPR | Compute inflow from IPR curve at `pchute` |
| `10` | Mass source | Compute gas/liquid/comp split from specified mass rates |
| `15` | Radial porous | Solve near-wellbore model, get inflow |
| `16` | 2D porous | Solve 2D near-wellbore model, get inflow |

Sets `celula[0].pres = pchute`, `celula[0].alf = α_ini`, `celula[0].bet = β_ini`.

### Main March Loop

```
for i = 1 to ncel:
    1. RenovaPresPermMon(i)     — Pressure: cell[i-1].center → cell[i].left boundary
    2. atualizaPeriPmonProd(i)   — Apply artificial-lift ΔP at cell[i].left boundary
    3. RenovaMassPerm(i)         — Mass flow rates at cell i
    4. RenovaTempPerm(i)         — Temperature at cell i
    5. RenovaPresPermJus(i)      — Pressure: cell[i].left boundary → cell[i].center
    6. atualizaPeriPjusProd(i)   — Apply artificial-lift ΔP at cell[i].center
    7. RenovaTransMassPerm(i-1)  — Interphase mass transfer at cell i-1
```

Each step is described in detail in [The Marching Equations](#the-marching-equations).

### Early Termination

The march returns sentinel values if:
- `−1e10`: Pressure dropped below 0.1 kgf/cm² (vacuum) — `pchute` was too low
- `+1e10`: Pressure exceeded IPR static pressure, or PVT table bounds — `pchute` was too high, or physically impossible
- `NaN` detected in pressure, temperature, or void fraction

### Return Value

```cpp
return pGSup - (celula[ncel].pres + corrigePresF);
```

The residual is the difference between the **imposed downstream pressure** (`pGSup`, typically the separator pressure) and the **computed pressure at the last cell**. When this equals zero, the steady state is found.

### Inner Iteration (Gas-Lift Coupling)

If the tramo has gas-lift valves (`arq.lingas > 0`), after the production-line march, the method also:

1. **Marches the gas-lift service line** (`marchaGasPerm1` / `buscaGasPresPerm2` / `buscaGasPresPerm3`), solving for gas-injection rate given injection pressure or vice versa
2. **Connects columns** — gas-lift valve flow rates depend on the differential pressure between the production and service lines at each valve position

The entire march (production + gas line) may repeat (`iterperm` loop) until the gas-lift coupling converges.

---

## The Marching Equations

### Pressure March: `RenovaPresPermMon`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Marches pressure over the **left half-cell** from cell `i-1` center to cell `i` left boundary:

$$P_{i}^{\text{aux}} = P_{i-1} - \frac{1}{98066.5} \left( \Delta P_{\text{fric}} + \Delta P_{\text{hydro}} \right)$$

Where:

$$\Delta P_{\text{fric}} = \frac{f \cdot \rho_\text{mix} \cdot |j| \cdot j \cdot S}{2A} \cdot \Delta x$$

$$\Delta P_{\text{hydro}} = 9.82 \cdot \sin(\theta) \cdot \rho_\text{mix} \cdot \Delta x$$

- $j = u_{gs} + u_{ls}$ is the mixture superficial velocity
- $f$ is the Fanning friction factor from Haaland/Colebrook iteration
- $S$ is the wetted perimeter, $A$ is the cross-sectional area
- $\theta$ is the pipe inclination angle
- $\Delta x = \frac{1}{2} \Delta x_L$ (half the left cell length)
- The factor 98066.5 converts Pa to kgf/cm²

The friction factor is computed by the `Cel::fric()` method ([`celula3.cpp`](../../src/celula3.cpp)):
- **Laminar** ($Re \leq 2400$): $f = 16 / Re$ (Fanning)
- **Turbulent** ($Re > 2400$): Haaland initial estimate + 2 Colebrook iterations

Reynolds number via `Cel::Rey()`:

$$Re = \frac{D \cdot |v| \cdot \rho}{\mu \times 10^{-3}}$$

where viscosity $\mu$ is in centipoise (cP).

### Pressure March: `RenovaPresPermJus`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Same equations, but marches over the **right half-cell** from cell `i` left boundary to cell `i` center, using $\Delta x = \frac{1}{2} \Delta x_i$. Also handles area-change pressure losses when the pipe diameter changes between cells.

### Artificial Lift: `atualizaPeriPmonProd`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

After computing `presaux` at the cell boundary, this method adds the **pressure gain** from artificial-lift devices located at cell `i-1`:

| `acsr.tipo` | Device | Pressure Contribution |
|-------------|--------|-----------------------|
| `4` | BCS (ESP pump) | $\Delta P_B = \text{sgn}(Q) \cdot 0.3048 \cdot H_\text{vis} \cdot \rho_\text{mix} \cdot 9.82$ (converted from head in ft to Pa) |
| `7` | Generic ΔP | $\Delta P_B = \text{sgn}(Q) \cdot \Delta P_\text{user} \cdot 98066.5$ |
| `17` | MultiBCS | Calls `marchaMultiBcs()` — multi-stage pump model |

The BCS model uses `NovaVis()` to apply **viscosity corrections** to the pump curve before computing head.

### Mass Flow: `RenovaMassPerm`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Updates mass flow rates at cell `i`:

1. Calls `renovaFonte(i-1)` — evaluates source terms at cell `i-1` at the local pressure and temperature
2. **Propagates source terms** from cell `i-1` right side to cell `i` left side:
   - `fontemassLL` (liquid), `fontemassCL` (complementary), `fontemassGL` (gas)
3. **Computes flash** at cell `i-1`: $R_s$, $B_o$, $B_a$ (solution gas ratio, oil/water formation volume factors)
4. **In-situ water fraction**: $f_w = \frac{\text{BSW} \cdot B_a}{B_o + B_a \cdot \text{BSW} - \text{BSW} \cdot B_o}$
5. **Updates mass flow** at cell `i`:
   - Total mass: $\dot{m}_C = \dot{m}_{C,i-1} + \text{source terms}$
   - Liquid mass: $\dot{m}_{L} = \dot{m}_{L,i-1} + \text{liquid source}$
   - Gas volume / liquid volume updated from flash split

### Source Terms: `renovaFonte`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Evaluates mass source terms at a cell based on the accessory type:

| `acsr.tipo` | Source | Evaluation |
|-------------|--------|------------|
| `1` | Gas injection | `VMas(P, T)` → split into gas + liquid via flash |
| `2` | Liquid injection | `QLiq` → split via RGO, BSW, Bo, Ba |
| `3` | IPR | `MasG(P,T)`, `MasL(P,T)` from the inflow performance curve |
| `5` | Choke (internal) | Choke flow equation |
| `10` | Mass source | Direct `MassG`, `MassP`, `MassC` specification |
| `15` | Radial porous | Near-wellbore radial model |
| `16` | 2D porous | Near-wellbore 2D model |

Results stored in `fontemassGR` (gas), `fontemassLR` (liquid), `fontemassCR` (complementary fluid) — these are the right-side source contributions of the cell.

### Temperature: `RenovaTempPerm`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Marches the energy equation from cell `i-1` to cell `i`. The temperature change combines four contributions:

$$\Delta T = \left( \mu_{JT,l} \cdot \frac{dP}{dx}\bigg|_l + \mu_{JT,g} \cdot \frac{dP}{dx}\bigg|_g - \frac{\rho g \sin\theta}{C_p} - \frac{\dot{Q}_\text{wall}}{\dot{m} \cdot C_p} \right) \cdot \Delta x$$

Where:
- $\mu_{JT}$ = Joule-Thomson coefficient (K·cm²/kgf) for liquid and gas phases
- $\dot{Q}_\text{wall}$ = heat transfer rate to/from the environment, computed by the `TrocaCalor` object (overall heat transfer coefficient accounting for pipe wall, insulation, burial, and ambient conditions)
- $C_p$ = mixture heat capacity (mass-weighted average of gas and liquid)

The heat-transfer model at each cell (`celula[i].calor`) receives:
- Internal fluid temperature, velocity, and properties
- External temperature (sea water, soil, ambient air)
- Pipe geometry (wall layers, insulation thickness)

And returns the heat flux through the pipe wall.

### Interphase Mass Transfer: `RenovaTransMassPerm`

**Source:** [`SisProd.cpp`](../../src/SisProd.cpp)

Computes the **gas liberation/absorption** at cell `i` due to changes in pressure and temperature along the pipe:

$$\dot{m}_\text{transfer} \propto \frac{\partial (R_s / B_o)}{\partial x}$$

Evaluates $R_s$ (solution gas-oil ratio) and $B_o$ (oil formation volume factor) at both the left and right boundaries of cell `i`, and differences them to obtain the spatial derivative. This term represents the flash — gas coming out of solution (or being absorbed) as P and T change.

---

## Convergence and Root Finding

### multMarcha — March Dispatcher

`multMarcha()` ([`SisProd.cpp`](../../src/SisProd.cpp)) is the dispatch function called by the root finder:

| `prod` | `tipoCC` | Function Called |
|--------|----------|-----------------|
| `0` (gas) | `0` | `marchaGasPerm2` |
| `0` (gas) | `1` | `marchaGasPerm3` |
| `1` (production) | `0` | `marchaProdPerm1` (or `Rev` variant) |
| `1` (production) | `1` | `marchaProdPerm2` |
| `2` (P-P) | `0` | `marchaProdPresPres1` (or `Rev` variant) |
| `2` (P-P) | `1` | `marchaProdPresPres2` / `marchaProdPresPres3` |
| injection | any | `marchaInjPerm1` |

### zriddr — Ridder's Method

`zriddr()` ([`SisProd.cpp`](../../src/SisProd.cpp)) implements **Ridder's root-finding algorithm**:

1. Ensures the bracket has opposite signs; if not, nudges the boundaries
2. Iterates up to 100 times with the Ridder formula:

$$x_\text{new} = x_m + (x_m - x_1) \cdot \frac{\text{sgn}(f_1) \cdot f_m}{\sqrt{f_m^2 - f_1 \cdot f_2}}$$

3. Evaluates $f(x_\text{new})$ via `multMarcha(x_new, ...)`
4. Stops when $|f| < 10^{-5}$

Ridder's method was chosen for its:
- **Guaranteed convergence** (as long as the bracket is valid)
- **No derivative requirement** (unlike Newton-Raphson)
- **Superlinear convergence** (order ~1.84, faster than bisection)

### Inner Convergence (marchaProdPerm1)

Within each march, the main `while` loop repeats the full sweep until **both** pressure and mass flow converge between consecutive iterations:

$$\frac{|m_\text{end}^{(k)} - m_\text{end}^{(k-1)}|}{|m_\text{end}^{(k)}|} < \epsilon_\text{perm}$$

$$\frac{|P_\text{end}^{(k)} - P_\text{end}^{(k-1)}|}{|P_\text{end}^{(k)}|} < \epsilon_\text{perm}$$

The tolerance `CriterioConvergPerm` is read from the input JSON. Typically 1–2 iterations suffice (controlled by `limIter`). The option `AceleraConvergPerm` reduces to a single sweep when gas-lift coupling is not present.

---

## Boundary Condition Variants

### `marchaProdPerm1` — Pressure at Downstream End

- **Unknown**: bottom-hole pressure $P_0$
- **March direction**: cell 0 → cell ncel
- **Residual**: $r = P_\text{sep} - P_{\text{ncel}}$
- **Called by**: `buscaProdPfundoPerm`

### `marchaProdPerm2` — Choke at Downstream End

- **Unknown**: bottom-hole pressure $P_0$
- **March direction**: cell 0 → cell ncel
- **Residual**: $r = \dot{m}_{\text{ncel}} - \dot{m}_\text{choke}(P_{\text{ncel}}, P_\text{sep})$
- **Called by**: `buscaProdPfundoPerm2`

When the choke is active (small opening), the downstream BC is no longer a simple pressure match. Instead, the choke equation relates the flow through the restriction to the pressure drop across it. The residual becomes the mismatch between the computed flow arriving at the choke and the flow the choke would pass at the computed upstream pressure.

### `marchaProdPresPres1` — Pressure at Both Ends

- **Unknown**: mass flow rate $\dot{m}_0$
- **March direction**: cell 0 → cell ncel
- **Residual**: $r = P_\text{sep} - P_{\text{ncel}}$
- **Called by**: `buscaProdPresPresPerm`

When both upstream and downstream pressures are known (e.g., a subsea pipeline between two platforms), the flow rate is the unknown.

### Reverse Flow Variants

When the source at cell 0 has negative flow rate, the solver switches to `marchaProdPerm1Rev` / `marchaProdPresPres1Rev`, which handle the reversed flow direction in the friction and mass-transfer terms.

---

## Gas-Lift Service Line

When a gas-lift service line is present (`arq.lingas > 0`), the steady-state solver must find a self-consistent solution between the production line and the gas-lift annulus.

### The Gas-Lift Service Line Model

The gas-lift service line is discretized into its own 1D cell array `celulaG[]` of type `CelG` (defined in [`celulaGas.h`](../../src/celulaGas.h) / [`celulaGas.cpp`](../../src/celulaGas.cpp)). Each `CelG` cell stores:

| Category | Key Members |
|----------|-------------|
| Geometry | `dutoL`, `duto`, `dutoR` — diameter, area, perimeter, inclination |
| Pressure | `pres`, `presL`, `presR` — cell center and face pressures |
| Temperature | `temp`, `tempL`, `tempR` |
| Mass flow | `VGasL`, `VGasR`, `VGasRR` — gas mass flow at left/right/far-right faces |
| Density | `u1L`, `u1R` ($\rho \cdot A$ products), `rg` |
| VGL source | `massfonteCH` — mass extracted through gas-lift valve |
| Discharge | `razInter`, `celInter` — gas/liquid interface tracking (for liquid-loaded annuli) |
| Stagnation state | `pEstag`, `tEstag`, `pGarg`, `tGarg`, `qGarg`, `areaGarg` — VGL choke conditions |
| Fluid model | `flui` (`ProFlu`), `calor` (`TransCal`), `chkcell` (`ChokeGas`) |

Unlike the production-line `Cel` class which tracks multiphase flow (gas + oil + water), the `CelG` class models **single-phase compressible gas** (or, below the liquid interface `celInter`, completion fluid).

### Gas-Line Marching Functions

The steady-state gas-line solver marches cell-by-cell from the injection point (wellhead annulus) down to the deepest gas-lift valve. Three variants exist:

| Function | BC Mode | Approach |
|----------|---------|----------|
| `marchaGasPerm1()` | Pressure BC | Iterative: guesses total injection mass flow, marches, sums VGL extractions, relaxes estimate until mass balance converges (tolerance < 1e-5) |
| `marchaGasPerm2()` | Flow-rate BC | Single march with given injection pressure `pchute` and prescribed flow rate. Returns residual $\Sigma(\dot{m}_{\text{VGL}}) - \dot{m}_{\text{inj}}$. Used by `buscaGasPresPerm2()` with Ridder root finding |
| `marchaGasPerm3()` | Injection choke BC | Single march with injection choke (`chokeInj.massica()`). Returns residual $\Sigma(\dot{m}_{\text{VGL}}) - \dot{m}_{\text{choke}}$. Used by `buscaGasPresPerm3()` |

Each march step calls three helper methods per cell:

1. **`RenovaPresGasPerm(i)`** — two half-cell pressure advances (center → face → center), accounting for friction (Fanning/Colebrook via `CelG::fric()`), hydrostatic head ($\rho g \sin\theta \Delta x$), and area-change dynamic pressure loss
2. **`RenovaTempGasPerm(i)`** — energy equation march using Joule-Thomson cooling, radial heat transfer (`calor.transperm()`), friction heating, and hydrostatic work
3. **`calcVazGasPerm(i)`** — at each cell containing a gas-lift valve, computes:
   - Stagnation pressure `presEstag` from gas-cell pressure
   - Throat pressure `presGarg` from production-column pressure with recovery fraction `frec`
   - Mass flow through the orifice via `chokeVGL[j].massica()` (compressible gas choke model in [`chokegas.cpp`](../../src/chokegas.cpp))
   - For IPO valves (`tipo == 1`): effective area modulated by `areaValvCali()` (calibrated orifice area vs. pressure differential)
   - Gas injection temperature via `TempDescGL()` (isenthalpic expansion, see below)
   - Subtracts valve flow from the running line flow total

### Gas-Lift Valve Choke Model

Each VGL is represented by a `ChokeGas` object (`chokeVGL[]` array). The method `chokeVGL[i].massica()` computes mass flow through the orifice using a **compressible gas choke model** (implemented in [`chokegas.cpp`](../../src/chokegas.cpp)):

- **Subcritical flow**: mass flow depends on the pressure ratio $P_{\text{throat}} / P_{\text{stagnation}}$ and discharge coefficient $C_d$
- **Critical flow**: when the pressure ratio falls below the critical value, the flow is choked (mass flow is limited by sonic conditions at the throat)
- **Liquid mode**: when the cell is below the gas/liquid interface (`celInter`), `massica(1, salinidade)` uses a liquid-phase flow model with completion-fluid properties

### Temperature Drop Across VGL (`TempDescGL`)

`SProd::TempDescGL()` computes the gas temperature after isenthalpic expansion through a gas-lift valve:

- Performs a **stepwise pressure march** from `presEstag` to `presGarg` in steps of 50 kgf/cm²
- Each step uses the real-gas expansion formula:
  $$
  T_1 = \frac{1}{\frac{1}{T_0} - \frac{286.998}{\rho_g} \cdot \frac{\partial Z / \partial T}{c_{p,g}} \cdot \ln\frac{P_1}{P_0}}
  $$
- If the step is too small, falls back to the Joule-Thomson approximation: $\Delta T = \mu_{JT} \cdot \Delta P$

### Coupling Mechanism

Gas-lift valves (VGLs) are placed at specific positions along the production and service lines. Each valve has a flow rate that depends on the **pressure difference** between the service line (annulus) and the production line at that position.

### Solution Strategy

1. On the **first iteration** (`iterperm == 0`), `IniciaVazValvGasPerm()` estimates initial valve flow rates
2. After the production-line march, the gas-lift service line is marched:
   - `marchaGasPerm1()` — direct march (for valve-flow BC)
   - `buscaGasPresPerm2()` — root finding for injection-flow BC
   - `buscaGasPresPerm3()` — root finding for choke-restricted injection
3. Gas-lift valve rates are updated based on the new pressure differential
4. If the gas-lift BC is injection pressure (not flow rate), an **outer loop** in `permanenteSimples` iterates on the gas injection rate:
   - Start with an initial estimate: $Q_g \approx 150{,}000 \times A_\text{annulus} / A_\text{ref}$
   - Solve with this rate, obtain service-line pressure
   - Compare with target injection pressure
   - Adjust rate by ±5% and retry (up to 40 attempts)

### Strong Thermal Coupling

When `acopColAnulPermForte > 0`, the solver performs **pseudo-transient thermal iterations** to couple the temperature fields of the production and service lines:

```
for kontaPseudo = 0 to acopColAnulPermForte:
    1. March gas-line temperature
    2. March production-line temperature
    3. Update fluid properties
    4. Re-connect columns (heat exchange between concentric pipes)
```

---

## Compositional Model Acceleration

For compositional fluid models (`flashCompleto == 2`), `SolveTramoSolteiro` employs the two-pass approach described in [Step 2](#step-2--solvetramosolteirofluid-model-strategy).

Additionally, the `preparaTabDin()` function ([`Num4Main.cpp`](../../src/Num4Main.cpp)) can build **dynamic PVT tables** from the black-oil result. These pre-tabulated flash results cover the pressure and temperature range discovered during the first pass, making the compositional pass much faster by avoiding repeated flash calculations.

The approach within `preparaTabDin`:
1. Identifies P,T range from the black-oil solution
2. Performs flash calculations at a grid of (P,T) points
3. Stores results in look-up tables on each cell's fluid object
4. During the compositional march, `atualizaPropComp()` interpolates from the table instead of solving the full flash

---

## Output and Post-Processing

After `permanenteSimples` returns successfully, `SolveTramoSolteiro` writes:

1. **Profiles** via `arq.imprimeProfile()` — pressure, temperature, void fraction, velocities, flow rates vs. position
2. **Summary** via `arq.resumoPermanente()` — key values at inlet and outlet
3. **2D Poisson** results if thermal diffusion was coupled
4. **Near-wellbore profiles** (radial/2D porous models)
5. **Trend files** for each monitored variable at specified positions
6. **Gas-lift line profiles** if applicable
7. **Log file** with convergence status, timing, and version information

---

## Summary of Key Methods

| Method | File | Purpose |
|--------|------|---------|
| `SolveTramoSolteiro` | Num4Main.cpp | Top-level: handles compositional two-pass, calls `permanenteSimples`, writes output |
| `permanenteSimples` | Num4Main.cpp | BC dispatch: selects solver variant based on boundary conditions |
| `buscaProdPfundoPerm` | SisProd.cpp | Bracket search + Ridder root finding for $P_0$ |
| `buscaProdPfundoPerm2` | SisProd.cpp | Same but for active choke (calls `marchaProdPerm2`) |
| `buscaProdPresPresPerm` | SisProd.cpp | Bracket search + Ridder for mass flow rate $\dot{m}_0$ |
| `marchaProdPerm1` | SisProd.cpp | Cell-by-cell march, returns $P_\text{sep} - P_\text{ncel}$ |
| `marchaProdPerm2` | SisProd.cpp | Same march, returns $\dot{m} - \dot{m}_\text{choke}$ |
| `marchaProdPresPres1` | SisProd.cpp | March with known inlet P, returns $P_\text{sep} - P_\text{ncel}$ |
| `multMarcha` | SisProd.cpp | Dispatch: routes `(chute, prod, tipoCC)` to the correct march function |
| `zriddr` | SisProd.cpp | Ridder's root-finding method |
| `RenovaPresPermMon` | SisProd.cpp | Pressure: cell center → left boundary (half-step) |
| `RenovaPresPermJus` | SisProd.cpp | Pressure: left boundary → cell center (half-step) |
| `atualizaPeriPmonProd` | SisProd.cpp | Apply BCS/pump ΔP at cell boundary |
| `RenovaMassPerm` | SisProd.cpp | Mass flow update + source evaluation |
| `RenovaTempPerm` | SisProd.cpp | Energy equation march (Joule-Thomson + heat transfer) |
| `RenovaTransMassPerm` | SisProd.cpp | Interphase mass transfer (gas liberation/absorption) |
| `renovaFonte` | SisProd.cpp | Evaluate source terms (gas/liquid/mass injection, IPR, porous) |
| `Cel::fric` | celula3.cpp | Fanning friction factor (Haaland + Colebrook) |
| `Cel::Rey` | celula3.cpp | Reynolds number |
| `hidroreverso` | SisProd.cpp | Backward hydrostatic walk for initial pressure estimate |
| `preparaTabDin` | Num4Main.cpp | Build dynamic PVT tables for compositional acceleration |
| `marchaGasPerm1` | SisProd.cpp | Gas-line steady-state march (pressure BC, iterative) |
| `marchaGasPerm2` | SisProd.cpp | Gas-line march (flow-rate BC), returns residual for Ridder |
| `marchaGasPerm3` | SisProd.cpp | Gas-line march (injection choke BC), returns residual for Ridder |
| `RenovaPresGasPerm` | SisProd.cpp | Gas-line pressure march: friction + hydrostatic per half-cell |
| `RenovaTempGasPerm` | SisProd.cpp | Gas-line temperature march: J-T + heat transfer |
| `calcVazGasPerm` | SisProd.cpp | Gas-line valve flow: choke model, IPO correction, temperature drop |
| `TempDescGL` | SisProd.cpp | Isenthalpic gas temperature drop across VGL |
| `CelG::fric` | celulaGas.cpp | Friction factor for gas-line cells (Haaland + Colebrook) |
| `CelG::Rey` | celulaGas.cpp | Reynolds number for gas-line cells |
| `chokeVGL[].massica` | chokegas.cpp | Compressible gas choke mass-flow model |
