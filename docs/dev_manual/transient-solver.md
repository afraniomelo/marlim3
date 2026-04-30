# Transient Solver for a Single Tramo

This document describes the logic and algorithms used by the Marlim3 simulator to compute the **transient (time-dependent) solution** of a single pipeline segment (*tramo*). While the [steady-state solver](steady-state-tramo.md) uses a shooting method with cell-by-cell spatial marching, the transient solver advances the entire domain in time, coupling mass, momentum, and energy equations through a **drift-flux two-fluid model** on a staggered grid.

**Key source files:**

| File | Role |
|------|------|
| [`src/SisProd.cpp`](../../src/SisProd.cpp) | `SolveTrans()` main loop and all system-level transient methods |
| [`src/SisProd.h`](../../src/SisProd.h) | `SProd` class declaration |
| [`src/celula3.cpp`](../../src/celula3.cpp) | `Cel` class — cell-level transport, matrix assembly, state rewind |
| [`src/celula3.h`](../../src/celula3.h) | `Cel` class declaration and member variables |
| [`src/PropFlu.h`](../../src/PropFlu.h) / [`PropFlu.cpp`](../../src/PropFlu.cpp) | Fluid property correlations (black-oil & compositional) |
| [`src/estrat.cpp`](../../src/estrat.cpp) | Stratified flow model (Taitel-Dukler map) |
| [`src/FonteMas.cpp`](../../src/FonteMas.cpp) | Mass source term evaluation |
| [`src/TrocaCalor.cpp`](../../src/TrocaCalor.cpp) | Radial heat-transfer (pipe wall, insulation, seabed) |
| [`src/celulaGas.h`](../../src/celulaGas.h) / [`celulaGas.cpp`](../../src/celulaGas.cpp) | `CelG` class — gas-lift service line cell state, matrix assembly, rewind |
| [`src/chokegas.h`](../../src/chokegas.h) / [`chokegas.cpp`](../../src/chokegas.cpp) | `ChokeGas` — compressible gas choke model for VGL orifices |

---

## Table of Contents

1. [Overview](#overview)
2. [Conservation Equations](#conservation-equations)
3. [The Drift-Flux Closure Model](#the-drift-flux-closure-model)
4. [Time Stepping and CFL Condition](#time-stepping-and-cfl-condition)
5. [SolveTrans — Top-Level Walkthrough](#solvetrans--top-level-walkthrough)
6. [EvoluiFrac — Void Fraction Evolution](#evoluifrac--void-fraction-evolution)
7. [SolveAcopPV — Pressure-Velocity Coupling](#solveacoppv--pressure-velocity-coupling)
8. [marchaEnergTrans — Transient Energy Equation](#marchaenergtrans--transient-energy-equation)
9. [Boundary Conditions](#boundary-conditions)
10. [State Finalization (renova)](#state-finalization-renova)
11. [Closure-Law Update (renovaterm)](#closure-law-update-renovaterm)
12. [Gas-Lift Service Line Coupling](#gas-lift-service-line-coupling)
13. [PIG Tracking](#pig-tracking)
14. [Adaptive Model Complexity](#adaptive-model-complexity)
15. [Time Step Restart Logic](#time-step-restart-logic)
16. [Compositional and Scalar Transport](#compositional-and-scalar-transport)
17. [Output and Post-Processing](#output-and-post-processing)
18. [Summary of Key Methods](#summary-of-key-methods)

---

## Overview

The transient solver computes the **time evolution of pressure, velocity, void fraction, water cut, and temperature** along a tramo. The algorithm can be summarized as:

1. **Determine the time step** $\Delta t$ from CFL and operational constraints
2. **Advance the void fraction** $\alpha$ and water cut $\beta$ explicitly in time (hyperbolic transport)
3. **Solve the coupled pressure-velocity system** implicitly via a banded linear system
4. **March the energy equation** (temperature) semi-implicitly
5. **Finalize** state variables, update fluid properties, write output, and advance time

Unlike the steady-state solver which marches spatially from one end to the other, the transient solver marches **in time**, updating all cells simultaneously at each time step.

The drift-flux model is **always** used in transient mode (regardless of the `tipoModeloDrift` setting, which only affects the steady-state solver). This means the phase velocities are not solved independently; instead, a kinematic relation links gas velocity to mixture velocity via the distribution parameter $C_0$ and drift velocity $u_d$.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SolveTrans — Main Time Loop                      │
│                                                                      │
│  for each time step:                                                 │
│    ┌─────────────────────────────────────────────────────────────┐   │
│    │ 1. determinaDT()       — compute Δt (CFL + constraints)    │   │
│    │ 2. solveLinGas()       — gas-lift service line              │   │
│    │ 3. atualizaCC1()       — update boundary conditions        │   │
│    │ 4. avaliaVariaDpDt()   — adaptive model complexity         │   │
│    │ 5. EvoluiFrac()        — advance α, β explicitly           │   │
│    │ 6. AtualizaPig()       — PIG tracking                      │   │
│    │ 7. calcCCpres()        — outlet pressure BC                │   │
│    │ 8. renovaterm()        — update drift-flux closure (c₀,ud) │   │
│    │ 9. SolveAcopPV()       — pressure-velocity coupling        │   │
│    │10. renova()            — extract P, M from solution vector  │   │
│    │11. marchaEnergTrans()  — energy equation (temperature)      │   │
│    │12. renovaTemp()        — update T-dependent properties      │   │
│    │13. renovaMasEsp()      — update densities                   │   │
│    │14. renovaRGOdgYco2()   — advect RGO, API, BSW, CO₂         │   │
│    │15. Output              — profiles, trends, snapshots        │   │
│    └─────────────────────────────────────────────────────────────┘   │
│    advance time: t += Δt                                             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Conservation Equations

The transient solver solves four conservation equations for a 1D multiphase mixture flowing through a pipe of cross-sectional area $A$ and perimeter $S$:

### Mass conservation — gas phase

$$
\frac{\partial}{\partial t}\left( A \, \alpha \, \rho_g \right) + \frac{\partial}{\partial x}\left( \dot{m}_g \right) = \dot{m}_{g,\text{source}} + \dot{m}_{\text{transfer}}
$$

where $\alpha$ is the gas void fraction, $\rho_g$ is the gas density, $\dot{m}_g$ is the gas mass flow rate, and $\dot{m}_{\text{transfer}}$ accounts for inter-phase mass transfer (flashing/condensation).

### Mass conservation — liquid phase

$$
\frac{\partial}{\partial t}\left( A \, (1-\alpha) \, \rho_l \right) + \frac{\partial}{\partial x}\left( \dot{m}_l \right) = \dot{m}_{l,\text{source}} - \dot{m}_{\text{transfer}}
$$

The liquid density $\rho_l$ is a weighted average over oil and water:

$$
\rho_l = (1-\beta)\,\rho_o + \beta\,\rho_w
$$

where $\beta$ is the water cut (volume fraction of water in the liquid phase).

### Mixture momentum conservation

$$
\frac{\partial \dot{m}}{\partial t} + \frac{\partial}{\partial x}\left(\frac{\dot{m}_g^2}{A\,\alpha\,\rho_g} + \frac{\dot{m}_l^2}{A\,(1-\alpha)\,\rho_l}\right) = -A\frac{\partial P}{\partial x} - \frac{f\,\rho_m\,|j|\,j\,S}{8} - A\,\rho_m\,g\sin\theta
$$

where $\dot{m} = \dot{m}_g + \dot{m}_l$ is the total mixture mass flow, $j$ is the mixture superficial velocity, $f$ is the Darcy friction factor, and $\theta$ is the pipe inclination.

### Energy conservation

$$
\frac{\partial T}{\partial t} + \frac{\dot{m}}{A\,\rho_m\,c_{p,m}} \frac{\partial T}{\partial x} = \frac{1}{A\,\rho_m\,c_{p,m}}\left[\mu_{JT}\frac{\partial P}{\partial t} + \dot{q}_{\text{friction}} + \dot{q}_{\text{hydrostatic}} - \dot{q}_{\text{wall}}\right]
$$

where $\mu_{JT}$ is the effective Joule-Thomson coefficient (weighted by quality), and $\dot{q}_{\text{wall}}$ represents radial heat loss through pipe layers to the environment.

### Discretization approach

- **Void fraction** ($\alpha$) and **water cut** ($\beta$) are advanced **explicitly** in time using upwind finite differences (`avancalf`, `avancbet`)
- **Pressure** ($P$) and **mixture mass flow** ($\dot{m}$) are solved **implicitly** via a coupled banded linear system (`GeraLocal` → `SolveAcopPV`)
- **Temperature** ($T$) is advanced **semi-implicitly** — the advection is explicit, but heat loss is evaluated at the new time level (`calctemp` inside `marchaEnergTrans`)

---

## The Drift-Flux Closure Model

The drift-flux model provides the **kinematic closure** that relates gas velocity to mixture velocity. Instead of solving separate momentum equations for each phase, the gas velocity is expressed as:

$$
u_g = C_0 \, j + u_d
$$

where:
- $j = u_{sg} + u_{sl}$ is the mixture superficial velocity
- $C_0$ is the **distribution parameter** (accounts for non-uniform void fraction and velocity profiles across the pipe cross-section; $C_0 > 1$ means gas concentrates in the faster-flowing core)
- $u_d$ is the **drift velocity** (accounts for buoyancy-driven relative motion between phases)

### CalcC0Ud — Flow-pattern-dependent closure

The method `SProd::CalcC0Ud()` (`SisProd.cpp`) computes $C_0$ and $u_d$ for each cell face. The calculation depends on the **flow pattern**, which is determined by flow-pattern maps:

1. **Near-horizontal pipes** ($|\theta| < 45°$): the **Taitel-Dukler** map (`estrat::mapaTD()`) classifies the flow as:
   - **Stratified** → blended `C0UdEstratificado` / `C0UdDisperso`
   - **Intermittent (slug)** → `C0UdDisperso` (dispersed-bubble/slug closure)
   - **Annular** → `C0UdAnularChurn`

2. **Inclined/vertical pipes**: the **Barnea-type** map (`arranjo::verificaArr()`) classifies the flow and dispatches to the appropriate closure.

3. **PIG present**: forces homogeneous flow ($C_0 = 1$, $u_d = 0$) around the PIG location.

4. **Smooth transitions**: when the flow pattern changes between time steps, the code applies a relaxation (blending over a `transic` counter) to avoid abrupt jumps in $C_0$ and $u_d$.

5. **Optional override**: if `escorregaTran == 0` in the input, slip is disabled globally ($C_0 = 1$, $u_d = 0$), reducing the model to a homogeneous mixture.

The closure is called during the `renovaterm()` phase (step 8 in the main loop), which propagates void fractions to cell faces and then evaluates the drift-flux parameters at each interior face.

---

## Time Stepping and CFL Condition

The time step $\Delta t$ is determined by `SProd::determinaDT()` and must satisfy multiple constraints:

### CFL condition (explicit mode)

When the explicit solver is active (`vExpli == 1`), the time step is limited by the Courant-Friedrichs-Lewy condition:

$$
\Delta t \leq \min_i \frac{\Delta x_i}{\max\left(|u_{\text{adv},i} + c_i|, \; |u_{\text{adv},i} - c_i|\right)}
$$

where:
- $u_{\text{adv},i}$ = advection velocity at cell $i$ (from `Cel::termAdSomVel()`)
- $c_i$ = sound speed in the mixture (from `Cel::somVel()`)

This is computed by `determinaDTExpli()`, which loops over all cells and takes the global minimum.

### Additional constraints

- **User-specified maximum**: `arq.dtmax` provides an upper bound
- **Oscillation penalty**: if a closed valve/choke is detected and spatial oscillation of $\alpha$ is found in 3+ consecutive cells, $\Delta t$ is divided by 10
- **Porous media**: if radial or 2D porous sub-models are active, their sub-timestep constraints are honoured
- **Valve events**: `restringeDTporValv()` limits $\Delta t$ during valve opening/closing transients
- **Moving average attenuation**: `atenuaDtMax()` smoothly adjusts the maximum $\Delta t$ based on recent history

### Time step restart

If the explicit advance of $\alpha$ produces out-of-bounds values ($\alpha < 0$ or $\alpha > 1$), the cell computes the maximum stable sub-step `dt1` (or `dt2` for $\beta$, `dtPig` for PIG), sets `reinicia = -1`, and the main loop **undoes** the entire step, halves $\Delta t$, and retries. See [Time Step Restart Logic](#time-step-restart-logic).

---

## SolveTrans — Top-Level Walkthrough

`SProd::SolveTrans()` (`SisProd.cpp`) orchestrates the entire transient computation. Here is the step-by-step logic:

### Initialization (before the time loop)

1. **Compositional mini-table**: if `flashCompleto == 2`, initializes the compositional PVT interpolation tables
2. **Initial output**: writes the profile and trend data at $t = 0$
3. **Initial property update**: `renovaTemp()` evaluates temperature-dependent fluid properties

### Time loop body

Each iteration advances the solution by one time step:

```
while (t < t_final):

    1. atualizaSonico()           — update sound speed per cell
    2. determinaDT(vExpli)        — compute Δt
    3. dtauxCFL = dt              — save CFL-limited Δt for diagnostics

    4. aberturaVal0/Val1          — valve opening/closing schedule
    5. BuscaPresInjDesc()         — descent control (if controDesc == 1)

    6. solveLinGas()              — advance gas-lift service line
    7. arq.atualiza(...)          — update time-dependent BCs from schedule
    8. atualizaCC1()              — set inlet conditions (P, T, tit, β)

    9. avaliaVariaDpDt(...)       — decide model complexity for this step

   10. COUPLING LOOP (kontaAcop = 0 to 1*modeloCompleto):

       a. Set m2d flags per cell (adaptive complexity control)
       b. EvoluiFrac(αrev, βrev, kontaAcop)      — advance α, β
       c. Near-wellbore advance (if porous BCs)
       d. restringeDTporValv()                    — valve Δt limits

       e. IF reinicia == -1:
            ReiniEvolFrac0()      — find min stable sub-dt
            dt /= 2              — halve time step
            ReiniEvolFrac()      — revert α, β, PIG to old state
            EvoluiFrac(...)      — retry with smaller Δt
            (repeat if still unstable)

       f. AtualizaPig()          — advance PIG position
       g. atenuaDtMax()          — smooth Δt limiting
       h. calcCCpres(...)        — outlet BC (choke model)
       i. renovaterm()           — update c₀, ud (drift-flux closure)
       j. SolveAcopPV(vExpli)    — solve for P, ṁ
       k. IF not last iteration:
            compute dpdt per cell
            FeiticoDoTempo2()    — rewind fractions for re-coupling
       l. IF last iteration:
            renova()             — finalize P, ṁ into cell state
       m. marchaEnergTrans()     — energy equation (T)

   11. salvaFonte()              — store source terms
   12. renovaTemp()              — T-dependent property update
   13. renovaalbetini()          — save α, β as initial for next step
   14. renovaRGOdgYco2() / renovaFracMol2()  — advect scalars
   15. renovaMasEsp()            — density update

   16. Moving averages (P, v, α)
   17. Output: profiles, trends, logs, progress
   18. t += Δt
```

### The coupling loop (`kontaAcop`)

When `modeloCompleto == 1`, the coupling loop runs **twice** (`kontaAcop = 0, 1`). The first pass:

- Advances $\alpha, \beta$ and solves pressure-velocity
- Computes `dpdt` for each cell (pressure rate of change)
- Calls `FeiticoDoTempo2()` to **rewind** the fraction state back to the beginning of the step

The second pass:

- Re-advances $\alpha, \beta$ with the updated `dpdt` (which feeds back into the mass transfer terms)
- Solves pressure-velocity again
- Calls `renova()` to finalize

This two-pass scheme provides an **implicit-like coupling** between pressure changes and void fraction evolution, improving stability during rapid transients (e.g., blowdown, slug arrival at surface).

When `modeloCompleto == 0` (quasi-steady periods), only one pass is executed, saving roughly half the computational cost.

---

## EvoluiFrac — Void Fraction Evolution

`SProd::EvoluiFrac()` advances the void fraction $\alpha$ and water cut $\beta$ by one time step. It consists of four OpenMP-parallel loops:

### Loop 1 — Advance $\alpha$ (`avancalf`)

For each interior cell ($i < n_{\text{cel}}$), calls `celula[i].avancalf()` to advance $\alpha$ explicitly. The last cell ($i = n_{\text{cel}}$) uses boundary-dependent logic — either copies from the upstream neighbor or calls `avancalf()` depending on network topology and choke configuration.

### Cell-level `avancalf` (celula3.cpp)

The void fraction transport equation is:

$$
\alpha^{n+1}_i = \alpha^n_i + \Delta t \cdot \frac{F_{\alpha}}{1 + K_{\alpha}}
$$

where $F_{\alpha}$ (`term1Mass`) gathers:

- **Convective flux**: upwinded liquid mass flow difference $(\dot{m}_{l,i+1/2} - \dot{m}_{l,i-1/2}) / \Delta x$, divided by liquid density
- **Mass sources**: gas injection, liquid injection, condensate source
- **Inter-phase transfer**: $\dot{m}_{\text{transfer}} / \rho_l$ (flashing/condensation)
- **Pressure transient** correction: $-(\partial T_{\text{trans}} / \partial P) \cdot (dP/dt) / \rho_l$ — accounts for density changes due to pressure variation
- **Temperature transient** correction: $-(\partial T_{\text{trans}} / \partial T) \cdot (dT/dt) / \rho_l$

The divisor $K_{\alpha}$ (`CoefDTR / (rpC * area)`) provides implicit stabilization.

**Bounds enforcement**: if $\alpha^{n+1} < 0$ or $\alpha^{n+1} > 1$, the cell:
1. Computes the maximum stable sub-step: $\Delta t_{\text{sub}} = -\alpha^n / (F_{\alpha} / (1 + K_{\alpha}))$
2. Sets `reiniciaAlf = -1` (flags restart needed)
3. Clamps $\alpha$ to $[0, 1]$

### Loop 2 — Advance $\beta$ (`avancbet`)

Same structure as $\alpha$, but for the water volume fraction in the liquid phase. The transport equation is:

$$
\beta^{n+1}_i = \beta^n_i + \Delta t \cdot \frac{-\text{convec}_{\beta} + \text{source}_{\beta} + \rho_w A \beta^n (\alpha^{n+1} - \alpha^n) / \Delta t}{\rho_w A (1 - \alpha^n)}
$$

If $\alpha \approx 1$ (all-gas cell), $\beta$ is not updated.

### Loop 3 — Advance PIG fractions

If a PIG is present in cell $i$ (`estadoPig == 1`), calls `avancPig()` to advance the PIG position ratio within the cell, then `avancalfPig()` / `avancbetPig()` to split $\alpha, \beta$ into upstream/downstream sub-cell values (`alfPigE`, `alfPigD`, `betPigE`, `betPigD`).

### Loop 4 — Restart check

Serial loop that checks if any cell flagged `reiniciaAlf < 0`, `reiniciaBet < 0`, or `reiniciaPig < 0`. If so, sets the global `reinicia = -1`, triggering the time step restart logic in `SolveTrans`.

---

## SolveAcopPV — Pressure-Velocity Coupling

`SProd::SolveAcopPV()` is the implicit core of the transient solver. It assembles and solves the linearized pressure-velocity system.

### Assembly: `Cel::GeraLocal`

For each cell $i$ (OpenMP-parallel), `GeraLocal()` (celula3.cpp) builds a $2 \times 6$ local matrix `local[2][6]` and right-hand side `TL[2]`:

**Row 0 — Mass conservation** (combines gas and liquid mass equations, eliminating void fraction):

The six columns correspond to the stencil $[\dot{m}_{i-2},\; \dot{m}_{i-1},\; P_{i-1},\; \dot{m}_i,\; P_i,\; \dot{m}_{i+1}]$:

- **Pressure derivative terms**: gas compressibility $(\alpha / \rho_g) / (\partial P / \partial \rho_g)$ and liquid compressibility $((1-\alpha)(1-\beta) \, \partial\rho_l/\partial P) / \rho_l$
- **Mass flow terms**: use `term1` / `term2` (the linear split $\dot{m}_l = \text{term1} \cdot \dot{m} + \text{term2}$, derived from the drift-flux closure)
- **Inter-phase transfer**: `DTransDx*` and `DTransDt*` discretize $\partial \dot{m}_{\text{transfer}} / \partial x$ and $\partial \dot{m}_{\text{transfer}} / \partial t$
- **Sources**: gas injection (`fontemassG`), liquid injection (`fontemassL`), condensate (`fontemassC`), IPR inflow

The right-hand side `TL[0]` contains the known terms evaluated at the current state plus the temporal derivative contribution $A \Delta x (\ldots) P^n / \Delta t$.

**Row 1 — Momentum conservation**:

$$
\frac{\dot{m}_i^{n+1} - \dot{m}_i^n}{\Delta t} + \frac{u_g \dot{m}_{g,i+1/2} - u_g \dot{m}_{g,i-1/2}}{\Delta x} + \frac{u_l \dot{m}_{l,i+1/2} - u_l \dot{m}_{l,i-1/2}}{\Delta x} = -A\frac{P_{i+1/2} - P_{i-1/2}}{\Delta x_{\text{med}}} - F_{\text{fric}} - F_{\text{grav}}
$$

The pressure gradient is treated implicitly (columns 3 and 5 carry $\pm A \cdot 98066.5 / \Delta x_{\text{med}}$). Friction and hydrostatic forces appear in `TL[1]`.

### Global system assembly

The local $2 \times 6$ matrices are assembled into a **banded global matrix** `matglobP` with bandwidth ≈ 6. Each cell contributes 2 rows:
- Row $2i$: mass conservation
- Row $2i+1$: momentum conservation

### Solution

`matglobP.GaussElimPP(termolivreP)` solves the banded system using **Gauss elimination with partial pivoting** adapted for the banded structure. The solution vector `termolivreP` contains alternating $(\dot{m}_i, P_i)$ values.

---

## marchaEnergTrans — Transient Energy Equation

`SProd::marchaEnergTrans()` advances the temperature field. The strategy is:

### Boundary fix

At the outlet cell ($i = n_{\text{cel}}$), if the liquid mass flow is negative (reverse flow), clamp it to zero to prevent unphysical energy transport.

### Inlet temperature

$$
T_0^{n+1} = \begin{cases} T_E & \text{if inlet BC active (ConContEntrada > 0)} \\ T_{\text{reservoir}} & \text{otherwise} \end{cases}
$$

The inlet temperature rate $dT/dt$ is computed for use in subsequent cells.

### Interior cells — `calctemp()`

For each interior cell, `SProd::calctemp()` solves the 1D energy balance:

$$
T_i^{n+1} = T_i^n + \frac{\Delta t}{A \, \rho_m \, c_{p,m}} \left[ \mu_{JT,\text{eff}} \frac{\Delta P}{\Delta t} + \dot{q}_{\text{fric}} + \dot{q}_{\text{hydro}} - \dot{q}_{\text{wall}} - \dot{m} c_{p,m} \frac{T_i - T_{i-1}}{\Delta x} \right]
$$

where:
- $\mu_{JT,\text{eff}}$ is the quality-weighted effective Joule-Thomson coefficient: $(1-x)\,\mu_{JT,l}/c_{p,l} + x\,\mu_{JT,g}/c_{p,g}$
- $\dot{q}_{\text{fric}}$ accounts for viscous dissipation
- $\dot{q}_{\text{hydro}}$ is the hydrostatic work term: $(\rho_l u_{sl} + \rho_g u_{sg}) A g \sin\theta$
- $\dot{q}_{\text{wall}}$ is the radial heat loss computed by the `TrocaCalor` model (multi-layer: fluid → steel → insulation → concrete → soil/sea)

### 2D/3D heat diffusion

If enabled (`modoDifus3D`), cells with active 2D Poisson sub-grids solve the radial temperature distribution first (OpenMP-parallel), providing the wall heat flux that feeds back into the 1D energy balance.

---

## Boundary Conditions

### Inlet boundary (`atualizaCC1`)

The upstream boundary sets:
- **Pressure** $P_E$ (or mass flow rate, depending on `ConContEntrada` mode)
- **Temperature** $T_E$
- **Gas quality** $\text{tit}_E$ (gas mass fraction)
- **Water cut** $\beta_E$

These may be time-varying according to a schedule read from the JSON input (`arq.atualiza()`).

### Outlet boundary (`calcCCpres`)

`SProd::calcCCpres()` evaluates the downstream pressure condition through a **choke model**:

1. **Gas quality** at the outlet: $x = |\dot{m}_g / \dot{m}|$ from the last cell face
2. **Mixture density**: harmonic mean weighted by quality:
   $$
   \rho_m = \left[ \frac{x}{\rho_g} + \frac{1-x}{\rho_l} \right]^{-1}
   $$
3. **Choke mass flow** via the critical/subcritical flow model:
   - Pressure ratio $y = P_{\text{sep}} / P_{\text{upstream}}$
   - If $y < 1$: forward flow through choke, mass flow computed from the choke area and $\Delta P$
   - If $y > 1$: reverse flow (sign flip)
4. **Joule-Thomson cooling** across the choke:
   $$
   T_{\text{downstream}} = T_{\text{upstream}} + \left[(1-x)\frac{\mu_{JT,l}}{c_{p,l}} + x\frac{\mu_{JT,g}}{c_{p,g}}\right] (P_{\text{sep}} - P_{\text{upstream}}) \cdot 98066.5
   $$

The resulting mass flow and pressure are imposed as boundary conditions for the last cell.

---

## State Finalization (renova)

`SProd::renova()` extracts the solution from the global system and updates cell state:

1. **Pressure and mass flow**: for each interior cell:
   $$
   P_i = \texttt{termolivreP}[2i+1], \qquad \dot{m}_i = \texttt{termolivreP}[2i]
   $$
2. **NaN check**: aborts with `NumError` if any value is NaN
3. **Pressure rate**: $dP/dt = (P^{n+1} - P^n) / \Delta t$ (stored for the next step's coupling)
4. **Liquid split**: $\dot{m}_{l,i} = \text{term1}_i \cdot \dot{m}_i + \text{term2}_i$
5. **Propagate to faces**: left/right neighbor face values (`presL`, `presR`, `ML`, `MR`, etc.)
6. **Auxiliary pressure** `presaux`: a friction+hydrostatic-corrected pressure used for the momentum equation stencil:
   $$
   P_{\text{aux},i} = P_i + \frac{F_{\text{fric}} + F_{\text{grav}} - \Delta P_B}{98066.5}
   $$
   where friction is $\frac{1}{2} f \rho_m |j| j \cdot S \cdot \Delta x / A$ and gravity is $9.82 \sin\theta \, \rho_m \, \Delta x$.

---

## Closure-Law Update (renovaterm)

`SProd::renovaterm()` runs between `EvoluiFrac` and `SolveAcopPV` to refresh the drift-flux closure parameters:

1. **Save old values**: $C_0^n, u_d^n$ for relaxation
2. **Propagate fractions**: copy $\alpha_i, \beta_i$ to left/right faces of neighboring cells
3. **Detect closed accessories**: if a valve (tipo 5) or BCS pump (tipo 8) at the cell face is closed, skip that face
4. **Compute upwind velocities**: superficial gas velocity $u_{sg}$ and liquid velocity $u_{sl}$ using upwinded densities
5. **Call `CalcC0Ud()`**: evaluates the flow-pattern-dependent drift-flux parameters
6. **Derive `term1`, `term2`**: the linear mass split coefficients used by `GeraLocal`:
   $$
   \dot{m}_l = \text{term1} \cdot \dot{m} + \text{term2}
   $$
   where `term1` and `term2` encode the drift-flux relation.

---

## Gas-Lift Service Line Coupling

`SProd::solveLinGas()` couples the gas-lift service line to the production line. The gas-lift line has its own 1D cell array `celulaG[]` of type `CelG` (defined in [`celulaGas.h`](../../src/celulaGas.h) / [`celulaGas.cpp`](../../src/celulaGas.cpp)), modelling **single-phase compressible gas** above the liquid interface and completion fluid below it.

### solveLinGas — Orchestration

The top-level coupling method performs three steps:

1. **`ValvGasTrans()`**: solves the transient gas-lift valve model. For each VGL:
   - If the valve cell is in the **gas zone** (above `celInter`): computes `presEstag` from the gas-cell pressure, `presGarg` from the production-column pressure with recovery fraction `frec`, then calls `chokeVGL[i].massica()` for the compressible gas choke mass flow (in [`chokegas.cpp`](../../src/chokegas.cpp))
   - For **IPO valves** (`tipo == 1`): the effective orifice area is modulated by `areaValvCali()`, which models the calibrated area vs. differential pressure characteristic
   - If the valve cell is in the **liquid zone** (below `celInter`): uses `massica(1, salinidade)` for liquid-phase flow with completion-fluid properties
   - A **master-2 choke** at `posicM2` handles bidirectional flow between production and service lines, with hydrostatic correction

2. **Map gas-line cells to production cells**: for each valve position, transfers:
   - Injection gas flow rate `QGas` to the production cell's gas-source accessory (`celula[posGLP].acsr.injg`)
   - Injection temperature computed by `TempDescGL()` — isenthalpic expansion from stagnation to throat using real-gas thermodynamics (stepwise pressure march in 50 kgf/cm² increments with the formula $T_1 = 1/(1/T_0 - (286.998/\rho_g) \cdot (\partial Z/\partial T)/c_{p,g} \cdot \ln(P_1/P_0))$, falling back to Joule-Thomson for small steps)
   - Stagnation/throat conditions (`pEstag`, `tEstag`, `pGarg`, `tGarg`) are stored on the gas cell for trending

3. **`subtempoGas()`**: advances the gas-line transient solver for one production-line time step, potentially using **sub-timestepping** (multiple smaller steps).

### subtempoGas — Gas-Line Transient Solver

The gas-line solver is structurally similar to the production-line solver but operates on `CelG` cells with 3 DOFs per cell (pressure, mass flow, temperature):

```
subtempoGas():
  1. Save state          — DeVoltaParaoFuturo() for each gas cell
  2. Interface advance   — track gas/liquid boundary (if discharge model)
  3. Coupling loop (ciclomax iterations):
     a. Compute injection choke source (if tipoCC == 0)
     b. CelG::GeraLocal() for each cell (OMP-parallel)
        → assemble 3×9 local matrix into banded global system matglobG
     c. Gauss elimination: matglobG.GaussElimPP(termolivreG)
     d. renovaGas() — scatter solution to cell P, VGasR
     e. calctempGas() — temperature march for gas cells
     f. FeiticoDoTempo() — rewind if more coupling iterations remain
  4. Update densities: rg, u1L, u1R, u1LL for all cells
  5. resolveDescarga() — if liquid-discharge model is active
```

### CelG::GeraLocal — Gas-Line Matrix Assembly

Each `CelG` cell assembles a $3 \times 9$ local matrix and 3-element RHS vector (`celulaGas.cpp`):

| Row | Equation | Physics |
|-----|----------|---------|
| 0 | **Continuity** | $\partial(\rho A)/\partial t + \partial \dot{m}/\partial x = \text{source}$ — uses gas compressibility $\partial\rho/\partial P$, temperature derivative $\partial\rho/\partial T$, VGL extraction `massfonteCH`, master-2 source `fonteM2` |
| 1 | **Momentum** | $\partial \dot{m}/\partial t + v \cdot \partial \dot{m}/\partial x + A \cdot \partial P/\partial x + F_{\text{fric}} + F_{\text{grav}} = 0$ — Darcy friction via `CelG::fric()` (Haaland + Colebrook), hydrostatic via $\sin\theta$ |
| 2 | **Temperature** | Identity equation $T = T_{\text{current}}$ — temperature is solved separately by `calctempGas()` after the pressure-velocity solve |

The 9-column stencil spans 3 cells (left, center, right) with 3 DOFs each. Boundary conditions: cell 0 applies either pressure or flow BC; the last cell applies zero-flow. Cells below the liquid interface (`celInter`) receive identity equations (frozen state).

### CelG State Management

- `CelG::DeVoltaParaoFuturo()` — saves current state into `*ini` copies (commits the time level)
- `CelG::FeiticoDoTempo()` — restores all state from `*ini` copies (rolls back to the start of the step)
- `renovaGas()` — scatters the global solution vector `termolivreG[]` back into each cell's pressure and mass flow

---

## PIG Tracking

`SProd::AtualizaPig()` tracks pipeline inspection gauge (PIG) movement:

### PIG position advance (`avancPig`)

Each cell has a `razPig` ∈ [0, 1] representing where the PIG is within the cell:

$$
\text{razPig}^{n+1} = \text{razPig}^n + \frac{v_{\text{PIG}} \cdot \Delta t}{\Delta x}
$$

If $\text{razPig} \geq 1$: PIG moves to the next cell downstream.
If $\text{razPig} \leq 0$: PIG moves upstream (reverse flow).

Out-of-bounds triggers a time step restart, same as for $\alpha$.

### Cell-to-cell transfer

When the PIG crosses a cell boundary:
- The `estadoPig` flag is transferred from cell $i$ to cell $i \pm 1$
- The PIG velocity and index are propagated
- Upstream/downstream sub-cell fractions (`alfPigE`, `alfPigD`, `betPigE`, `betPigD`) are updated

### Water-cut interface

After PIG update, the upwinded water cut at each face (`betI`) is set accounting for the PIG-partitioned sub-cell values, ensuring that the liquid composition on each side of the PIG is tracked correctly.

---

## Adaptive Model Complexity

`SProd::avaliaVariaDpDt()` dynamically adjusts the solver complexity:

### Algorithm

1. Scan all cells in blocks of 10
2. For each block, compute the average pressure rate:
   $$
   \overline{|dP/dt|} = \frac{1}{N_{\text{block}}} \sum_{j \in \text{block}} \frac{|P_j^n - P_j^{n-1}|}{\Delta t}
   $$
3. If any block exceeds $2 \times \text{taxaDespre}$ (the user-defined depressurization rate threshold), set `modeloCompleto = 1`
4. Maintain a rolling window (last 10 steps) of the maximum rates `taxaDpMax` and `taxaDTMax` for smoothed decision-making

### Effect on solver

| `modeloCompleto` | Coupling iterations | Cost |
|---|---|---|
| 0 | 1 pass | ~50% cheaper |
| 1 | 2 passes (with rewind) | Full accuracy |

During quasi-steady periods (e.g., stable production), the solver automatically drops to single-pass mode. During transient events (valve closure, slug arrival, blowdown), it switches to two-pass coupled mode.

Additionally, `modeloCompleto` controls per-cell flags (`m2d`, `mudaDT`, `ativaDeri`) that enable/disable:
- Pressure-dependent inter-phase mass transfer corrections
- Gas compressibility derivatives in the mass equation
- Liquid compressibility terms

---

## Time Step Restart Logic

When the explicit advance produces out-of-bounds fractions, the solver must retry with a smaller time step. The restart mechanism involves three methods:

### `ReiniEvolFrac0()` — Find minimum stable sub-step

Scans all cells and finds the minimum of `dt`, `dt1`, `dt2`, `dtPig` — these are the per-cell maximum stable sub-steps computed during `avancalf`, `avancbet`, and `avancPig`.

### `ReiniEvolFrac()` — Revert state

1. Sets all cells to the new (smaller) global $\Delta t$
2. Reverts gas-line cells via `FeiticoDoTempo()` (cell-level time-rewind)
3. Calls `SubReiniEvolFrac()` to restore $\alpha$ to $\alpha^n$ (initial)
4. Restores $\beta$, `razPig`, and PIG sub-cell fractions to their initial values

### `FeiticoDoTempo` / `FeiticoDoTempo2` (celula3.cpp)

These are the **cell-level rewind** methods:

- `FeiticoDoTempo()`: restores pressure, mass flow, temperature, flow rates, and heat-transfer state to their values at the start of the time step
- `FeiticoDoTempo2()`: restores fractions ($\alpha$, $\beta$), PIG state, drift-flux parameters ($C_0$, $u_d$), and flow pattern transition counters

The naming comes from the Portuguese *feitiço do tempo* ("time spell" — literally rewinding time).

### Restart sequence in SolveTrans

```
IF reinicia == -1:
    ReiniEvolFrac0()          // find min stable dt
    dt = min(dt, dt/2)        // halve global dt
    ReiniEvolFrac()           // revert α, β, PIG to old values
    EvoluiFrac(...)           // re-advance with smaller dt
    IF still reinicia == -1:
        repeat halving...
```

---

## Compositional and Scalar Transport

### Scalar advection (`renovaRGOdgYco2`)

`SProd::renovaRGOdgYco2()` advects fluid property scalars along the pipeline using the mass-flow-weighted transport:

- **RGO** (gas-oil ratio)
- **dg** (gas specific gravity)
- **yCO₂** (CO₂ mole fraction)
- **API** (oil gravity)
- **BSW** (basic sediment & water)
- **Liquid and water viscosity**

Each scalar is transported using upwind differencing based on the cell-face mass flow direction.

### Compositional updates (`renovaFracMol2`)

When `flashCompleto == 2` (compositional mode), `renovaFracMol2()` updates the mole fractions of each component in the oil and gas phases. The compositional model uses a **mini-table** (`miniTab`) approach: a local interpolation table is built around the current (P, T) state to avoid calling the full flash calculation at every cell and time step.

### Density update (`renovaMasEsp`)

`SProd::renovaMasEsp()` re-evaluates gas, oil, and water densities at the updated (P, T) state for each cell, ensuring consistency between the thermodynamic state and the flow equations for the next time step.

---

## Output and Post-Processing

The transient solver produces several types of output at user-specified intervals:

| Output type | Method | Content |
|---|---|---|
| **Spatial profiles** | `imprimeProfile()` | P, T, α, β, velocities along the pipeline |
| **Production trends** | `imprimeTrend()` | Time series of flow rates, pressures, temperatures at key points |
| **Gas trends** | `imprimeTrendGas()` | Gas-lift line pressures and injection rates |
| **Thermal layer trends** | `imprimeTrendTermico()` | Pipe wall and insulation temperatures |
| **Log events** | `log.registra()` | Valve events, PIG passage, alarms |
| **Snapshots** | `salvaArquivoSnap()` | Full state dump for restart |
| **Progress** | Console output | Time, dt, CFL fraction, iteration count |

Moving averages of pressure, velocity, and void fraction are maintained for trend output smoothing, using a configurable window.

---

## Summary of Key Methods

| Method | File | Purpose |
|---|---|---|
| `SolveTrans()` | SisProd.cpp | Main transient time-loop orchestrator |
| `determinaDT()` | SisProd.cpp | Time step calculation (CFL + constraints) |
| `determinaDTExpli()` | SisProd.cpp | CFL-limited Δt from sound speed + advection |
| `EvoluiFrac()` | SisProd.cpp | Explicit advance of α, β |
| `avancalf()` | celula3.cpp | Cell-level void fraction transport |
| `avancbet()` | celula3.cpp | Cell-level water cut transport |
| `avancPig()` | celula3.cpp | Cell-level PIG position advance |
| `SolveAcopPV()` | SisProd.cpp | Pressure-velocity coupling (banded system) |
| `GeraLocal()` | celula3.cpp | Local 2×6 matrix assembly (mass + momentum) |
| `marchaEnergTrans()` | SisProd.cpp | Transient energy equation |
| `calctemp()` | SisProd.cpp | Per-cell temperature advance |
| `renova()` | SisProd.cpp | Extract solution, update cell state |
| `renovaterm()` | SisProd.cpp | Update drift-flux closure (c₀, ud) |
| `CalcC0Ud()` | SisProd.cpp | Flow-pattern-dependent c₀, ud |
| `calcCCpres()` | SisProd.cpp | Outlet choke boundary condition |
| `solveLinGas()` | SisProd.cpp | Gas-lift service line coupling |
| `AtualizaPig()` | SisProd.cpp | PIG tracking and cell transfer |
| `avaliaVariaDpDt()` | SisProd.cpp | Adaptive model complexity |
| `ReiniEvolFrac0()` | SisProd.cpp | Find min stable sub-dt |
| `ReiniEvolFrac()` | SisProd.cpp | Revert state for dt restart |
| `FeiticoDoTempo()` | celula3.cpp | Cell-level state rewind (P, ṁ, T) |
| `FeiticoDoTempo2()` | celula3.cpp | Cell-level state rewind (α, β, PIG, c₀) |
| `renovaRGOdgYco2()` | SisProd.cpp | Scalar transport (RGO, API, BSW, CO₂) |
| `renovaMasEsp()` | SisProd.cpp | Density re-evaluation |
| `renovaTemp()` | SisProd.cpp | T-dependent property refresh |
| `ValvGasTrans()` | SisProd.cpp | Transient gas-lift valve model (choke flow + IPO) |
| `subtempoGas()` | SisProd.cpp | Gas-line transient sub-timestepping |
| `renovaGas()` | SisProd.cpp | Scatter gas-line solution to CelG cells |
| `TempDescGL()` | SisProd.cpp | Isenthalpic gas temperature drop across VGL |
| `CelG::GeraLocal()` | celulaGas.cpp | Gas-line local 3×9 matrix assembly (continuity + momentum + T) |
| `CelG::FeiticoDoTempo()` | celulaGas.cpp | Gas-line cell-level state rewind |
| `CelG::DeVoltaParaoFuturo()` | celulaGas.cpp | Gas-line cell-level state commit |
| `chokeVGL[].massica()` | chokegas.cpp | Compressible gas choke mass-flow model |
