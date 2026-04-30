# Heat Transfer Modeling

This document describes the heat transfer subsystem in Marlim3 — from the radial thermal resistance network through convection correlations, formation conduction, 2D/3D soil diffusion, and the coupling to the axial energy equation.

**Key source files:**

| File | Role |
|------|------|
| [`src/TrocaCalor.h`](../../src/TrocaCalor.h) / [`TrocaCalor.cpp`](../../src/TrocaCalor.cpp) | `TransCal` class — radial heat transfer (1D and 2D), convection correlations |
| [`src/Geometria.h`](../../src/Geometria.h) | `DadosGeo` — pipe geometry and wall layer definitions |
| [`src/celula3.h`](../../src/celula3.h) | `Cel` — multiphase cell thermal members (temperature, advection, heat source) |
| [`src/celulaGas.h`](../../src/celulaGas.h) | `CelG` — gas-lift cell thermal members |
| [`src/celulaVapor.h`](../../src/celulaVapor.h) | `CelVap` — steam cell thermal members |
| [`src/Elem2DPoisson.h`](../../src/Elem2DPoisson.h) / [`Elem2DPoisson.cpp`](../../src/Elem2DPoisson.cpp) | 2D FVM element for buried-pipe soil conduction |
| [`src/Malha2DPoisson.h`](../../src/Malha2DPoisson.h) / [`Malha2DPoisson.cpp`](../../src/Malha2DPoisson.cpp) | 2D unstructured triangular mesh |
| [`src/dados1Poisson.h`](../../src/dados1Poisson.h) / [`dados1Poisson.cpp`](../../src/dados1Poisson.cpp) | 2D Poisson solver data and I/O |
| [`src/estruturasPoisson.h`](../../src/estruturasPoisson.h) | Structures for 2D Poisson elements |
| [`src/estruturasPoisson3D.h`](../../src/estruturasPoisson3D.h) | Structures for 3D Poisson elements |
| [`src/dados3DPoisson.h`](../../src/dados3DPoisson.h) / [`dados3DPoisson.cpp`](../../src/dados3DPoisson.cpp) | 3D Poisson solver data |
| [`src/Elem3DPoisson.h`](../../src/Elem3DPoisson.h) / [`Elem3DPoisson.cpp`](../../src/Elem3DPoisson.cpp) | 3D FVM element |
| [`src/SisProd.cpp`](../../src/SisProd.cpp) | Axial energy equation and coupling loop |

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The TransCal Class](#the-transcal-class)
   - [Geometry — DadosGeo](#geometry--dadosgeo)
   - [Temperature and Heat Flux Fields](#temperature-and-heat-flux-fields)
   - [Internal Fluid Properties](#internal-fluid-properties)
   - [External Fluid Properties](#external-fluid-properties)
   - [Control Flags](#control-flags)
   - [Formation / Wellbore Parameters](#formation--wellbore-parameters)
   - [Gas Properties for Annulus](#gas-properties-for-annulus)
3. [Dimensionless Numbers](#dimensionless-numbers)
4. [Convection Correlations](#convection-correlations)
   - [Internal Forced Convection — Petukhov-Gnielinski](#internal-forced-convection--petukhov-gnielinski)
   - [External Cross-Flow — Churchill-Bernstein](#external-cross-flow--churchill-bernstein)
   - [External Natural Convection — Churchill-Chu](#external-natural-convection--churchill-chu)
   - [Confined Natural Convection (Annulus) — Hollands / Catton](#confined-natural-convection-annulus--hollands--catton)
   - [Mixed Convection Regime](#mixed-convection-regime)
5. [Friction Factor — Haaland](#friction-factor--haaland)
6. [Formation Thermal Resistance — Ramey](#formation-thermal-resistance--ramey)
7. [Ambient Fluid Properties](#ambient-fluid-properties)
   - [Seawater](#seawater)
   - [Air](#air)
   - [Gas / Nitrogen (Annulus)](#gas--nitrogen-annulus)
   - [Thermal Expansion Coefficient](#thermal-expansion-coefficient)
8. [Radial Resistance Network](#radial-resistance-network)
   - [Layer Resistance Types](#layer-resistance-types)
   - [Internal Convection Coefficient — hInt()](#internal-convection-coefficient--hint)
   - [External Convection Coefficient — hExt()](#external-convection-coefficient--hext)
   - [Wall Conductance — condParede()](#wall-conductance--condparede)
9. [Steady-State Solver — transperm()](#steady-state-solver--transperm)
10. [Transient Solver — transtrans()](#transient-solver--transtrans)
    - [Radial Discretization](#radial-discretization)
    - [Local Element Matrix — transcel()](#local-element-matrix--transcel)
    - [Global Assembly and Solve](#global-assembly-and-solve)
11. [Axial Energy Equation](#axial-energy-equation)
    - [Energy Balance Terms](#energy-balance-terms)
    - [Coupling: Radial Heat Flux → Axial Solver](#coupling-radial-heat-flux--axial-solver)
12. [Thermal Members in Cell Classes](#thermal-members-in-cell-classes)
13. [2D Buried-Pipe Solver (Poisson)](#2d-buried-pipe-solver-poisson)
    - [Mesh and Elements](#mesh-and-elements)
    - [Boundary Conditions](#boundary-conditions)
    - [Assembly — GeraLocal()](#assembly--geralocal)
    - [Gradient Reconstruction — Green-Gauss](#gradient-reconstruction--green-gauss)
    - [Linear Solve](#linear-solve)
    - [Coupling to 1D Flow](#coupling-to-1d-flow)
14. [3D Soil Diffusion Solver](#3d-soil-diffusion-solver)
15. [Special Configurations](#special-configurations)
    - [Subsea Pipelines](#subsea-pipelines)
    - [Risers and Wellbores](#risers-and-wellbores)
    - [Buried / Trenched Pipelines](#buried--trenched-pipelines)
    - [Lined / Annular Pipes](#lined--annular-pipes)
    - [Annulus with Forced Flow](#annulus-with-forced-flow)
    - [Column Configuration](#column-configuration)
    - [Prescribed Wall Temperature](#prescribed-wall-temperature)
    - [Wax Deposition Layer](#wax-deposition-layer)
16. [Summary of Named Correlations](#summary-of-named-correlations)

---

## Architecture Overview

Each 1D flow cell (`Cel`, `CelG`, `CelVap`) owns a `TransCal calor` object that computes radial heat exchange between the flowing fluid and the environment through a multi-layered pipe wall. The radial heat flux feeds into the axial energy equation as a source term.

```
                        ┌──────────────────────────────────────────────┐
  Axial energy          │  TransCal  (radial heat transfer)            │
  equation in           │                                              │
  SisProd.cpp           │  Fluid ──h_i──[Layer 0]──[Layer 1]──...     │
       │                │    (Tint)      steel    insulation           │
       │ fluxcal        │                  ...──[Layer N-1]──h_e──Env  │
       ◄────────────────┤                                   (Textern) │
       │                │  or: 2D Poisson soil solver (buried pipe)    │
       │                │  or: 3D Poisson soil solver                  │
       ▼                └──────────────────────────────────────────────┘
  T^{n+1} update
```

Three radial heat transfer modes:

| `difus2D` / `modoDifus3D` | Mode | Implementation |
|---------------------------|------|----------------|
| 0 / 0 | **1D Radial** | Concentric cylindrical resistances — `transperm()` / `transtrans()` |
| 1 / 0 | **2D Poisson** | Unstructured FVM for soil around buried pipe — `transperm2D()` / `transtrans2D()` |
| — / 1 | **3D Poisson** | Full 3D FVM soil conduction — `solverP3D` |

---

## The TransCal Class

`TransCal` (defined in [`TrocaCalor.h`](../../src/TrocaCalor.h) / [`TrocaCalor.cpp`](../../src/TrocaCalor.cpp)) is the central radial heat transfer engine.

### Geometry — DadosGeo

The `DadosGeo` structure ([`Geometria.h`](../../src/Geometria.h)) describes the pipe cross-section:

| Member | Type | Units | Description |
|--------|------|-------|-------------|
| `a` | `double` | m | Inner diameter (or inner annulus OD for lined pipe) |
| `b` | `double` | m | Inner annulus ID (for lined pipe, `revest=1`) |
| `dia` | `double` | m | Hydraulic diameter |
| `teta` | `double` | rad | Pipe inclination |
| `rug` | `double` | m | Surface roughness |
| `area` | `double` | m² | Cross-sectional flow area |
| `peri` | `double` | m | Wetted perimeter |
| `ncamadas` | `int` | — | Number of wall layers |
| `cond[i]` | `double*` | W/(m·K) | Thermal conductivity of layer $i$ |
| `diamC[i]` | `double*` | m | Outer diameter of layer $i$ |
| `espessuR[i]` | `double*` | m | Thickness of layer $i$ |
| `cp[i]` | `double*` | J/(kg·K) | Specific heat of layer $i$ |
| `rhoC[i]` | `double*` | kg/m³ | Density of layer $i$ |
| `tipomat[i]` | `int*` | — | **0** = solid (conduction), **2** = liquid (convection), **3** = gas (convection) |
| `revest` | `int` | — | 0 = circular pipe, 1 = annular (lined) pipe |

### Temperature and Heat Flux Fields

| Member | Type | Units | Description |
|--------|------|-------|-------------|
| `Tcamada[i][j]` | `double**` | °C | Temperature at node $j$ of layer $i$ — current step |
| `Tini[i][j]` | `double**` | °C | Temperature — previous time step |
| `Qcamada[i][j]` | `double**` | W/m | Heat flux at node $j$ of layer $i$ — current step |
| `Qini[i][j]` | `double**` | W/m | Heat flux — previous time step |
| `Tint`, `Tint2` | `double` | °C | Internal fluid temperature(s) |
| `Textern1`, `Textern2` | `double` | °C | External environment temperature(s) |
| `fluxIni` | `double` | W/m | Heat flux at inner wall |
| `fluxFim` | `double` | W/m | Heat flux at outer wall |
| `resGlob` | `double` | m·K/W | Overall thermal resistance |

### Internal Fluid Properties

| Member | Type | Units | Description |
|--------|------|-------|-------------|
| `kint` | `double` | W/(m·K) | Thermal conductivity |
| `cpint` | `double` | J/(kg·K) | Specific heat |
| `rhoint` | `double` | kg/m³ | Density |
| `viscint` | `double` | Pa·s | Dynamic viscosity |
| `betint` | `double` | 1/K | Thermal expansion coefficient |
| `Vint` | `double` | m/s | Flow velocity |

### External Fluid Properties

Two sets of external properties are stored (primary and secondary):

| Member | Units | Description |
|--------|-------|-------------|
| `kextern1`, `kextern2` | W/(m·K) | Conductivity |
| `cpextern1`, `cpextern2` | J/(kg·K) | Specific heat |
| `rhoextern1`, `rhoextern2` | kg/m³ | Density |
| `viscextern1`, `viscextern2` | Pa·s | Viscosity |
| `betext` | 1/K | External expansion coefficient |
| `Vextern1` | m/s | External flow velocity |
| `Vconf` | m/s | Annulus flow velocity |

### Control Flags

| Flag | Type | Values |
|------|------|--------|
| `permanente` | `int` | 1 = steady-state, 0 = transient |
| `dirconvExt` | `int` | **0** = cross-flow (Churchill-Bernstein), **1** = parallel flow (Petukhov) |
| `ambext` | `int` | **0** = user-specified, **1** = seawater (auto-properties), **2** = air (auto-properties) |
| `formacPoc` | `int` | **0** = pipe in fluid, **1** = wellbore in rock formation |
| `condiTparede` | `int` | **0** = convective inner BC, **1** = prescribed wall temperature |
| `novoHi` | `double` | User-override for internal $h$ (applied if > 0) |
| `difus2D` | `int` | **0** = 1D radial, **1** = 2D Poisson solver (buried pipe) |
| `npet` | `double` | Petukhov viscosity-ratio exponent (0.25 cooling, 0.11 heating) |

### Formation / Wellbore Parameters

| Member | Type | Units | Description |
|--------|------|-------|-------------|
| `tempprod` | `double` | days | Production time |
| `condform` | `double` | W/(m·K) | Formation thermal conductivity |
| `cpform` | `double` | J/(kg·K) | Formation specific heat |
| `rhoform` | `double` | kg/m³ | Formation density |
| `resFim` | `double` | m·K/W | Computed formation resistance |

### Gas Properties for Annulus

| Member | Value / Type | Description |
|--------|-------------|-------------|
| `airMW` | 28.97 | Air/gas molecular weight |
| `RGas` | 8314.4621 | Universal gas constant [J/(kmol·K)] |
| `pressao` | `double` | System pressure [kgf/cm²] |
| `TCNitro` | `double` | N₂ critical temperature [°R] |
| `PCNitro` | `double` | N₂ critical pressure [psia] |
| `CoefGopalHT[48]` | `constexpr` | Gopal Z-factor coefficients for gas density |

---

## Dimensionless Numbers

All dimensionless numbers are computed as member methods of `TransCal`:

**Grashof** — `Grash(ΔT, β, ν, L)`:

$$\mathrm{Gr} = \frac{g \, \beta \, L^3 \, \Delta T}{\nu^2}, \quad g = 9.82\;\text{m/s}^2$$

**Prandtl** — `Pr(ν, α)`:

$$\mathrm{Pr} = \frac{\nu}{\alpha}$$

**Rayleigh** — `Ra(Gr, Pr)`:

$$\mathrm{Ra} = \mathrm{Gr} \cdot \mathrm{Pr}$$

**Internal Rayleigh** — `RaInt(ΔT, β, ν, α)`:

$$\mathrm{Ra}_i = \frac{g \, \beta \, (0.2854 \, D)^3 \, \Delta T}{\nu \, \alpha}$$

The factor 0.2854 converts pipe diameter to the appropriate length scale.

**External Rayleigh** — `RaExt(ΔT, β, ν, α)`:

$$\mathrm{Ra}_e = \frac{g \, \beta \, (D_{\text{ext}} |\cos\theta|)^3 \, \Delta T}{\nu \, \alpha}$$

where $\theta$ is the pipe inclination. The projected length $D_{\text{ext}} |\cos\theta|$ accounts for inclination effects on buoyancy-driven flow.

**Reynolds** — `ReyIn(V, μ, ρ, D)`:

$$\mathrm{Re} = \frac{D \, \rho \, |V|}{\mu}$$

### Stored Dimensionless Numbers

| Member | Description |
|--------|-------------|
| `reyi`, `reye` | Reynolds — internal, external |
| `grashi`, `grashe` | Grashof — internal, external |
| `nusi`, `nuse` | Nusselt — internal, external |
| `hi`, `he` | Heat transfer coefficient — internal, external [W/(m²·K)] |
| `pri`, `pre` | Prandtl — internal, external |
| `prG`, `prL` | Prandtl — gas, liquid phases |

---

## Convection Correlations

### Internal Forced Convection — Petukhov-Gnielinski

`nussPet(Re, Pr, ε, μ_b, μ_w)` — primary internal convection correlation.

**Turbulent flow** ($\mathrm{Re} > 2400$) with moderate viscosity ratio ($\mu_w / \mu_b < 40$):

$$\mathrm{Nu} = \frac{\mathrm{Re} \cdot \mathrm{Pr}}{1.07 + 12.7\left(\mathrm{Pr}^{2/3} - 1\right)\sqrt{f/8}} \cdot \frac{f}{8} \cdot \left(\frac{\mu_b}{\mu_w}\right)^n$$

where $f$ is the Haaland friction factor and $n$ is the viscosity-ratio exponent:
- $n = 0.25$ for cooling ($T_{\text{wall}} < T_{\text{fluid}}$)
- $n = 0.11$ for heating ($T_{\text{wall}} > T_{\text{fluid}}$)

The exponent $n$ is set by `definePet()`.

**Sleicher-Rouse** for high viscosity ratio ($\mu_w / \mu_b \ge 40$):

$$\mathrm{Nu} = 5 + 0.016 \, \mathrm{Re}^a \, \mathrm{Pr}^b$$

$$a = 0.88 - \frac{0.24}{4 + \mathrm{Pr}}, \quad b = 0.33 + 0.5 \, e^{-0.6 \, \mathrm{Pr}}$$

**Laminar** ($\mathrm{Re} \le 2400$):

$$\mathrm{Nu} = 3.6$$

### External Cross-Flow — Churchill-Bernstein

`nussChuBer(Re, Pr)` — external convection over a cylinder in cross-flow.

**Churchill-Bernstein (1977):**

$$\mathrm{Nu} = 0.3 + \frac{0.62\sqrt{\mathrm{Re}}\;\mathrm{Pr}^{1/3}}{\left[1 + (0.4/\mathrm{Pr})^{2/3}\right]^{1/4}} \cdot \left[1 + \left(\frac{\mathrm{Re}}{282000}\right)^{5/8}\right]^{4/5}$$

A variant exponent of 1/2 is used for $20000 < \mathrm{Re} < 400000$.

### External Natural Convection — Churchill-Chu

`nussNatExt(Ra, Pr)` — free convection from an isothermal cylinder.

**Churchill-Chu (1975):**

$$\mathrm{Nu} = \left[0.6 + \frac{0.387 \, \mathrm{Ra}^{1/6}}{\left(1 + (0.559/\mathrm{Pr})^{9/16}\right)^{8/27}}\right]^2$$

### Confined Natural Convection (Annulus) — Hollands / Catton

`NussConf2(Ra, H, δ, θ)` — natural convection in an inclined annular gap. Three angle regimes:

**For inclination $|\theta| < 60°$** — Hollands et al. (1976):

$$\mathrm{Nu} = 1 + 1.44\left[1 - \frac{1708}{\mathrm{Ra}\cos\theta}\right]^+\!\left[1 - \frac{1708\sin^{1.6}(1.8|\theta|)}{\mathrm{Ra}\cos\theta}\right] + \left[\left(\frac{\mathrm{Ra}\cos\theta}{5830}\right)^{1/3} - 1\right]^+$$

where $[\;]^+$ denotes $\max(0, \cdot)$.

**For $60° \le |\theta| < 90°$** — linear interpolation:

$$\mathrm{Nu} = \frac{(\pi/2 - |\theta|)\,\mathrm{Nu}_{60} + (|\theta| - \pi/3)\,\mathrm{Nu}_{90}}{\pi/6}$$

**For $|\theta| = 90°$ (vertical)** — maximum of three Catton sub-correlations:

$$\mathrm{Nu}_{90} = \max\!\left(0.0605\,\mathrm{Ra}^{1/3},\;\left[1 + \left(\frac{0.104\,\mathrm{Ra}^{0.293}}{1+(6310/\mathrm{Ra})^{1.36}}\right)^3\right]^{1/3},\;0.242\left(\frac{\mathrm{Ra}}{A}\right)^{0.272}\right)$$

**`defineConf(Pr, Ra)`** sets a simpler power-law fallback $\mathrm{Nu} = C \cdot \mathrm{Ra}^n \cdot (H/\delta)^m$:
- $\mathrm{Pr} \in [0.5, 2]$, $\mathrm{Ra} \in [2 \times 10^3, 2 \times 10^5]$: $C = 0.197$, $n = 1/4$, $m = -1/9$
- $\mathrm{Ra} > 2 \times 10^5$: $C = 0.073$, $n = 1/3$, $m = -1/9$
- Otherwise: $\mathrm{Nu} = 1$ (pure conduction)

### Mixed Convection Regime

At low external Reynolds number, if $\mathrm{Gr}/\mathrm{Re}^2 > 0.9$, the natural convection Nusselt is **added** to the forced convection Nusselt:

$$\mathrm{Nu}_{\text{eff}} = \mathrm{Nu}_{\text{forced}} + \mathrm{Nu}_{\text{natural}}$$

---

## Friction Factor — Haaland

`fric(Re, ε/D)` — used inside the Petukhov-Gnielinski correlation.

**Turbulent** ($\mathrm{Re} > 2400$) — Haaland (1983):

$$f = 4 \left[ -1.8 \log_{10}\!\left(\frac{6.9}{\mathrm{Re}} + \left(\frac{\varepsilon/D}{3.7}\right)^{1.11}\right) \right]^{-2}$$

**Laminar** ($\mathrm{Re} \le 2400$):

$$f = \frac{64}{\mathrm{Re}}$$

---

## Formation Thermal Resistance — Ramey

`ResForm()` — wellbore formation resistance based on Ramey (1962).

**Dimensionless time:**

$$\tau_D = \frac{4 \, t_{\text{prod}} \cdot 86400 \cdot \alpha_f}{D_{\text{ext}}^2}, \quad \alpha_f = \frac{k_f}{\rho_f \, c_{p,f}}$$

where $t_{\text{prod}}$ is in days and $\alpha_f$ is the formation thermal diffusivity.

**Time function** (piecewise):

$$f(\tau_D) = \begin{cases}
1.1281\sqrt{\tau_D}\,(1 - 0.3\sqrt{\tau_D}) & \tau_D \le 1 \\[4pt]
0.432 + 0.4792\tau - 0.127\tau^2 + 0.0201\tau^3 - 0.0013\tau^4 & \tau_D \le 4.5 \\[4pt]
(0.4063 + 0.5\ln\tau_D)(1 + 0.6/\tau_D) & \tau_D > 4.5
\end{cases}$$

where $\tau = \ln\tau_D$ in the middle range.

**Formation resistance:**

$$R_{\text{form}} = \frac{f(\tau_D)}{2\pi \, k_f}$$

---

## Ambient Fluid Properties

### Seawater

When `ambext == 1`, external fluid properties are computed automatically at the film temperature $T_f = 0.5(T_{\text{ext}} + T_{\text{wall,outer}})$:

| Property | Correlation |
|----------|-------------|
| Density | McCain brine volume factor: $B_w = 1 + 1.2 \times 10^{-4}(T_F - 60) + 10^{-6}(T_F - 60)^2 - 3.33 \times 10^{-6} P_{\text{psi}}$ |
| Viscosity | $\mu = 2.414 \times 10^{-5} \cdot 10^{247.8/(T+133.15)}$ Pa·s |
| Conductivity | $k = 0.565 + 1.75 \times 10^{-3} T - 6.21 \times 10^{-6} T^2$ W/(m·K) |
| Specific heat | Piecewise polynomial in $T_K$ (split at 410 K) |

### Air

When `ambext == 2`, air properties at 1 atm:

| Property | Correlation |
|----------|-------------|
| Density | Ideal gas: $\rho = \dfrac{101325 \cdot 28.97}{8314.46 \cdot (T + 273.15)}$ kg/m³ |
| Viscosity | $\mu = 1.799 \times 10^{-5} + 1.000 \times 10^{-7} T - 1.370 \times 10^{-9} T^2 + 6.96 \times 10^{-12} T^3$ Pa·s |
| Conductivity | $k = 0.0241 + 7.586 \times 10^{-5} T$ W/(m·K) |
| Specific heat | $c_p = 1000(1.005 + 1.096 \times 10^{-5} T + 4.600 \times 10^{-7} T^2)$ J/(kg·K) |

### Gas / Nitrogen (Annulus)

For gas-filled annular layers (`tipomat == 3`), properties are computed as compressed gas:

| Property | Formula |
|----------|---------|
| Density | $\rho_g = \dfrac{\gamma_g \cdot 28.96 \cdot P \cdot 98066.5}{8046.5 \cdot Z(P_R, T_R) \cdot (T + 273)}$ with $\gamma_g = 0.9669$ |
| Specific heat | $c_p = (1.88 + 1.712\gamma_g)T + 2651 - 897.2\gamma_g$ |
| Conductivity | Pressure-corrected polynomial using Stiel-Thodos form |
| Viscosity | Lee-Gonzalez-Eakin |

The Z-factor is computed by the Gopal correlation (`ZGopal`), using the 48-coefficient piecewise polynomial table `CoefGopalHT[]`:

$$Z = P_R(A \, T_R + B) + C \, T_R + D$$

with $A, B, C, D$ selected from the table based on reduced pressure/temperature regions. High-pressure branch ($P_R > 5.4$):

$$Z = P_R(0.711 + 3.66 \, T_R)^{-1.4667} - \frac{1.637}{0.319 \, T_R + 0.522} + 2.071$$

### Thermal Expansion Coefficient

`beta(T, type)` — numerical differentiation:

$$\beta = -\frac{1}{\rho}\frac{\partial\rho}{\partial T} \approx -\frac{1}{\rho(T)}\frac{\rho(T + \delta T) - \rho(T)}{\delta T}$$

where $\delta T = 0.01 T$ (or 0.1 if $T \approx 0$). Type: 1 = seawater, 2 = gas, 3 = air.

---

## Radial Resistance Network

The pipe wall is modeled as a series of concentric cylindrical thermal resistances:

```
Fluid (Tint) ──h_i──[Layer 0]──[Layer 1]── ... ──[Layer N-1]──h_e── Environment (Textern)
                ↑        ↑          ↑                   ↑         ↑
           convective  solid or   solid or          solid or  convective
              BC       fluid      fluid             fluid        BC
```

### Layer Resistance Types

For each layer, the resistance type depends on `tipomat[j]`:

| `tipomat` | Type | Thermal Resistance (per unit length) |
|-----------|------|--------------------------------------|
| 0 | Solid | $R = \dfrac{\ln(D_{\text{out}} / D_{\text{in}})}{2\pi k}$ |
| 2 | Liquid (stagnant) | Natural convection: $R = \dfrac{\ln(D_o / D_i)}{\pi(D_o - D_i) h_{\text{conv}}}$ |
| 2 | Liquid (flowing, `Vconf > 0`) | Forced convection in annulus (Petukhov with hydraulic diameter) |
| 3 | Gas (stagnant) | Natural convection (same as liquid but gas properties) |
| 3 | Gas (flowing) | Forced convection in annulus |

### Internal Convection Coefficient — hInt()

```
if Re > 2400:
    Nu = nussPet(Re, Pr, ε, μ_b, μ_w)           ← Petukhov-Gnielinski
else:
    compute Ra_internal
    if Ra < 1000 or mixed convection:
        Nu = 3.6                                   ← Laminar pipe flow
    else:
        Nu = 0.22·(Pr·Ra/(0.2+Pr))^0.28 · 3.5^(-0.25)  ← Natural convection in pipe
    For lined pipe (revest = 1):
        Uses confined natural convection (NussConf2)

h_i = Nu · k_int / D_inner
```

If `novoHi > 0`, the user-specified value overrides the computed $h_i$.

### External Convection Coefficient — hExt()

```
if formacPoc == 0 (pipe in fluid):
    if ambext == 1: update seawater properties at film T
    if ambext == 2: update air properties at film T

    if dirconvExt == 0 (cross-flow):
        Nu = nussChuBer(Re, Pr)                    ← Churchill-Bernstein
    else (parallel flow):
        Nu = nussPet(Re, Pr, ε, μ_b, μ_w)         ← Petukhov

    h_e = Nu · k_ext / D_outer

if formacPoc == 1 (wellbore):
    h_e = 1 / (π · D_outer · R_formation)          ← Ramey formation resistance
```

### Wall Conductance — condParede()

Builds the per-unit-length thermal conductance `tec[j]` for each layer:

- **Solid layers:** $\text{tec}_j = \dfrac{2\pi k}{\ln(D_o / D_i)}$

- **Fluid layers (no flow):** compute $\mathrm{Gr}$, $\mathrm{Ra}$, $\mathrm{Nu}_{\text{conf}}$ from the confined convection model, then $\text{tec}_j = \dfrac{\pi(D_o - D_i) h}{\ln(D_o / D_i)}$

- **Fluid layers (flow):** forced convection with Petukhov on annular hydraulic diameter $D_h = 4A/P$, then $\text{tec}_j = \pi D_i h$

Returns the overall wall conductance:

$$k_{\text{wall}} = \left(\sum_j \frac{1}{\text{tec}_j}\right)^{-1}$$

---

## Steady-State Solver — transperm()

`TransCal::transperm()` computes the steady-state radial heat flux.

**Total thermal resistance:**

$$R_{\text{total}} = \frac{1}{\pi D_i h_i} + \sum_{j=0}^{N-1} \frac{1}{\text{tec}_j} + \frac{1}{\pi D_o h_e} + R_{\text{annulus}}$$

**Heat flux per unit length:**

$$q = \frac{T_{\text{ext}} - T_{\text{int}}}{R_{\text{total}}} \quad [\text{W/m}]$$

**Interface temperatures** — walking through the resistance chain:

$$T_{\text{wall,inner}} = T_{\text{int}} + \frac{q}{\text{tec}_0}$$

$$T_k = T_{k-1} + \frac{q}{\text{tec}_k}$$

**Iteration for natural convection:** For low external $\mathrm{Re}$ ($\le 5000$) with cross-flow, the code iterates up to 5 passes to converge the natural convection contribution, since $h_e$ depends on $T_{\text{wall,outer}}$ which depends on $q$ which depends on $h_e$.

If `difus2D == 1`, delegates to `transperm2D()`.

---

## Transient Solver — transtrans()

`TransCal::transtrans()` solves the transient radial heat equation through the pipe wall.

### Radial Discretization

The radial heat equation in cylindrical coordinates:

$$\rho \, c_p \frac{\partial T}{\partial t} = \frac{1}{r} \frac{\partial}{\partial r}\!\left(r \, k \frac{\partial T}{\partial r}\right)$$

Discretized with finite differences at node $r_1$ between neighbors $r_0$ and $r_2$:

$$\rho \, c_p \frac{T^{n+1} - T^n}{\Delta t} = \frac{k}{r_1}\left[\frac{r_1 + r_0}{2} \cdot \frac{T_0 - T_1}{r_1 - r_0} - \frac{r_1 + r_2}{2} \cdot \frac{T_1 - T_2}{r_2 - r_1}\right]$$

Each wall layer is subdivided into `ncamada[i]` radial finite-element nodes with spacing `drcamada[i]`. The total number of global nodes is $1 + \sum_i \text{ncamada}[i]$.

### Local Element Matrix — transcel()

`transcel(icam, idisc)` builds a 2×2 system per radial node pair. The local matrix has 2 DOFs per node (temperature and flux), stored in `localmat[2][6]` and `localvet[2]`:

**Interior nodes** (at radii $r_0$, $r_1$):

$$\text{localmat}[0][1] = \frac{1}{0.5(r_1 + r_0)(r_1 - r_0)}$$

$$\text{localmat}[0][2] = \frac{\rho \, c_p}{\Delta t}$$

$$\text{localvet}[0] = T^n_{i,j} \cdot \frac{\rho \, c_p}{\Delta t}$$

**Inner boundary** ($i_{\text{cam}} = 0$, $i_{\text{disc}} = 0$):

$$\text{localmat}[0][2] = -(h_i \cdot r_a + k_0 \cdot r_a / \delta r)$$

$$\text{localvet}[0] = -h_i \cdot r_a \cdot T_{\text{int}}$$

**Outer boundary** (last layer, last node):

$$\text{localmat}[1][2] = h_e \cdot r_1$$

$$\text{localvet}[1] = h_e \cdot r_1 \cdot T_{\text{ext}}$$

At **fluid-layer interfaces**, natural or forced convection heat transfer coefficients replace conduction terms, treating the fluid layer as a convective resistance rather than a conduction node.

### Global Assembly and Solve

`transtrans()`:

1. Assembles all `transcel()` local matrices into a global banded matrix (`BandMtx` with bandwidth 3,2)
2. Solves via Gauss elimination with partial pivoting
3. Extracts new temperatures $T^{n+1}_{i,j}$ and heat fluxes $Q^{n+1}_{i,j} = 2\pi \cdot q_{i,j}$
4. Updates layer interface continuity conditions

The global system has $2(N_{\text{global}} + 1)$ unknowns (temperature + flux at each node).

**`FeiticoDoTempo()`** — restores all `Tcamada`/`Qcamada` to their `Tini`/`Qini` values. Used for iteration rollback in the coupled flow-thermal solver.

If `difus2D == 1`, delegates to `transtrans2D()`.

---

## Axial Energy Equation

The axial energy balance is evaluated in [`SisProd.cpp`](../../src/SisProd.cpp) for each 1D cell. Temperature is updated explicitly:

$$T^{new} = \frac{(\rho c_v)_{mix} A \cdot T / \Delta t + \text{RHS}}{(\rho c_v)_{mix} A / \Delta t}$$

### Energy Balance Terms

| Term | Expression | Physical meaning |
|------|-----------|-----------------|
| **Pressure work** | $-c_{p,\text{press}} \cdot (P - P^0) \cdot 98066.5 / \Delta t$ | $\partial P/\partial t$ coupling |
| **Advection** | $-(\rho_l u_{ls} c_{p,l} + \rho_g u_{gs} c_{p,g}) A \cdot dT/dx$ | Temperature advection (upwind) |
| **Joule-Thomson** | $+(\rho_l u_{ls} \mu_{JT,l} + \rho_g u_{gs} \mu_{JT,g}) A \cdot dP/dx$ | J-T cooling/heating |
| **Kinetic energy** | $-\text{cinetico}$ | Kinetic energy change |
| **Hydrostatic** | $-\text{hidro}$ | Gravity heating/cooling |
| **Pump power** | $+\text{potTermo} / dx_{\text{med}}$ | BCS/pump dissipation |
| **External source** | $+\text{fonteCal} / dx_{\text{med}}$ | Volumetric heat source |
| **Mass sources** | $+\text{fontemassL} + \text{fontemassG}$ | Enthalpy of injected fluids |
| **Radial heat** | $+\text{fluxcal}$ | **Coupling term from TransCal** |
| **Latent heat** | $-\text{latente}$ | Phase-change enthalpy |

The **advection velocity** for temperature:

$$V_{\text{Temper}} = \frac{\rho_l u_{ls} c_{p,l} + \rho_g u_{gs} c_{p,g}}{(\rho c_v)_{mix}}$$

The spatial derivative $dT/dx$ uses **upwinding** based on the sign of $V_{\text{Temper}}$.

The computed $dT/dt = (T^{new} - T^{old})/\Delta t$ is fed back into the mass conservation equation as a thermal expansion coupling term.

### Coupling: Radial Heat Flux → Axial Solver

The coupling chain:

```
SisProd.cpp energy loop (for each cell i):
  │
  ├─ Update calor properties: calor.Tint, calor.Textern1, calor.rhoint, ...
  │
  ├─ if modoDifus3D == 0:           (standard 1D or 2D)
  │     if modoPerm == 0:
  │         fluxcal = calor.transtrans()     ← transient radial
  │     else:
  │         fluxcal = calor.transperm()      ← steady-state radial
  │
  ├─ elif modoDifus3D == 1:          (3D Poisson for soil)
  │     fluxcal = -arq.celAcop[].FE * poisson3D.dados.qTotal[] / dx
  │
  ├─ celula[i].fluxcalmed = fluxcal  ← stored for diagnostics/output
  │
  └─ Explicit temperature update: T += ... + fluxcal + ...
```

---

## Thermal Members in Cell Classes

### Cel (celula3.h) — Main Multiphase Cell

| Member | Description |
|--------|-------------|
| `TransCal calor` | Radial heat transfer object |
| `temp`, `tempL`, `tempR` | Center, left, right temperatures [°C] |
| `tempini`, `tempLini`, `tempRini` | Previous time step values |
| `dTdt`, `dTdtIni`, `dTdtL` | $\partial T/\partial t$ — for energy equation coupling |
| `VTemper`, `VTemperini` | Advection velocity for energy transport [m/s] |
| `fluxcalmed` | Radial heat flux [W/m] |
| `fluxCal2D` | 2D heat flux (burial) |
| `fonteCal` | Volumetric heat source [W/m] |
| `fluxcalAcopRedeP`, `resAcopRedeP` | Network thermal coupling |
| `potB`, `potBT`, `potTermo` | BCS pump power / thermal power |
| `deltaPar`, `porosoPar`, `MW_wax`, `rhoWaxLiq`, `Sum_dCwaxdT` | Wax deposition thermal parameters |

### CelG (celulaGas.h) — Gas-Lift Service Line Cell

| Member | Description |
|--------|-------------|
| `TransCal calor` | Radial heat transfer object |
| `temp`, `tempL`, `tempR` / `tempini` | Temperatures and previous step |
| `fluxcal` | Heat flux [W/m] |

### CelVap (celulaVapor.h) — Steam/Vapor Cell

| Member | Description |
|--------|-------------|
| `TransCal calor` | Radial heat transfer object |
| `temp`, `tempL`, `tempR`, `tempini` | Temperatures |
| `VTemper` | Advection velocity |

---

## 2D Buried-Pipe Solver (Poisson)

When `difus2D == 1`, the 1D radial model is replaced by a 2D finite-volume solver for the soil surrounding a buried or trenched pipeline. The solver is contained in the `solverP` class.

### Mesh and Elements

The 2D solver uses a **cell-centered Finite Volume Method (FVM)** on **unstructured triangular meshes**.

**Key classes:**

| Class | File | Role |
|-------|------|------|
| `elementoPoisson` | [`estruturasPoisson.h`](../../src/estruturasPoisson.h) | Per-element data: vertex coordinates, centroid, face areas/normals, interpolation factors, temperature field, material properties |
| `elem2dPoisson` | [`Elem2DPoisson.h`](../../src/Elem2DPoisson.h) | Element wrapper with neighbor pointers, boundary conditions, local matrix assembly |
| `malha2d` | [`Malha2DPoisson.h`](../../src/Malha2DPoisson.h) | Mesh container (`vector<elem2dPoisson>`), neighbor linking, face detail computation |
| `solverP` | `solverPoisson.h` | Top-level: `dadosP` data + `malha2d` mesh + sparse matrix (`SparseMtx`) + GMRES solver |

**Mesh input formats:**

- **Triangle format** (`.ele` + `.node` files) when `unv == 0`
- **UNV (Salome/Ideas) format** when `unv == 1` — parses nodes, edges/faces, triangular elements, and named boundary groups

Material properties (`cond`, `cp`, `rho`) and initial conditions are assigned region-by-region via bounding boxes.

### Boundary Conditions

Defined in [`estruturasPoisson.h`](../../src/estruturasPoisson.h):

| BC Type | Structure | Description |
|---------|-----------|-------------|
| **Dirichlet** | `detDiriPoisson` | Prescribed temperature, time-varying series |
| **Von Neumann** | `detVNPoisson` | Prescribed heat flux, time-varying series |
| **Richardson** | `detRicPoisson` | Convective: $h_{\text{amb}}$ and $T_{\text{amb}}$ time series |
| **Coupled** | `rotuloAcop` | Robin condition at the pipe interface: $T_{\text{amb}}$ = pipe wall temperature, $h$ = local wall conductance |

`tipoCC()` classifies each boundary face. `atualizaCC()` interpolates time-varying BC values.

### Assembly — GeraLocal()

In [`Elem2DPoisson.cpp`](../../src/Elem2DPoisson.cpp), for each triangular element:

**Internal faces** (neighbor exists):

1. Harmonic conductivity: $k_h = 1 / (g/k_C + (1-g)/k_{\text{viz}})$
2. Orthogonal diffusion: $k_h \cdot (\vec{e} \cdot \vec{S}_f) / |\vec{e}|$
3. Cross-diffusion correction via Green-Gauss gradient reconstruction

**Dirichlet faces:** diffusion flux using ghost-cell approach.

**Richardson / Coupled faces:** combined conduction + convection resistance, creating effective $\text{coefTHRC}$ and $\text{fonteTHR}$.

**Von Neumann faces:** prescribed flux added directly to RHS.

**Transient term** (when steady-state flag is off):

$$\text{RHS} += V_{\text{elem}} \cdot \rho \cdot c_p \cdot T^0 / \Delta t + \text{FonteT} \cdot V_{\text{elem}}$$

$$\text{diag} += V_{\text{elem}} \cdot (\rho \cdot c_p / \Delta t - \partial\text{FonteT} / \partial T)$$

### Gradient Reconstruction — Green-Gauss

`calcGradGreen()` uses the **Green-Gauss method** with a minimum-correction approach for non-orthogonal meshes:

1. Interpolate face temperature using distance-weighted factor `fatG`
2. Correct face gradient: $\nabla T_f = \nabla T_{\text{interp}} + (dT/dn_{\text{chord}} - \nabla T_{\text{interp}} \cdot \hat{e}) \cdot \hat{e}$
3. **Distortion filter:** if $|\cos(\angle(\vec{E}, \vec{S}))| < 0.9$, the cross-diffusion correction is zeroed

### Linear Solve

The global sparse matrix is assembled in CRS format. Both `permanentePoisson()` and `transientePoisson()` iterate:

1. Copy gradient → `gradGreenTI`
2. `calcGradGreen()` on all elements (OpenMP parallelized)
3. `GeraLocal()` on all elements (OpenMP parallelized)
4. Assemble into CRS
5. Solve with **GMRES** (or FGMRES / BiCGStab), with optional **ILU(k) preconditioning**
6. Update temperatures, check convergence ($\|T - T_{\text{old}}\| / N_{\text{ele}} < 10^{-5}$)
7. If coupled, recompute $q_{\text{acop}}$, $q_{\text{total}}$, $T_{\text{wall}}$, update coupled BC values

### Coupling to 1D Flow

**`transperm2D()`** (steady-state buried pipe):

1. Sets `poisson2D.dados.tInt = Tint`, `tAmb = Textern1`, `hI`, `hE`, `condGlob`, `condLoc`
2. Initializes all 2D elements to $T_{\text{extern}}$
3. Iterates with **pseudo-transient acceleration**: `transientePoissonDummy(deltFic)` with doubling time steps until $|q_{\text{total}} - q_{\text{total,old}}| < 0.1$
4. Switches to `permanentePoisson()`
5. Returns $q = -\text{poisson2D.dados.qTotal}$ [W/m]
6. Updates 1D wall layer temperatures from the computed flux

**`transtrans2D()`** (transient buried pipe):

1. At $t = 0$, switches all 2D elements from permanent to transient mode
2. Updates Richardson BC with current `tAmb`, `hE`
3. Calls `poisson2D.transientePoisson(dt)` with the current flow solver time step
4. Returns $q = -\text{poisson2D.dados.qTotal}$ [W/m]

The **coupling BC on the pipe inner wall** is a Robin condition: for faces labeled `rotuloAcop`:
- $h_{\text{face}} = \text{condLoc}$ (local pipe wall conductance including internal convection)
- $T_{\text{amb,face}} = T_{\text{wall}}$ (pipe wall temperature from flux balance)

Total heat flux: $q_{\text{total}} = q_{\text{acop}} + q_{\text{desacop}}$, where $q_{\text{acop}}$ is integrated over coupled faces and $q_{\text{desacop}}$ accounts for the non-buried arc of the pipe.

---

## 3D Soil Diffusion Solver

The 3D capability mirrors the 2D architecture for full volumetric soil conduction:

| Component | 3D Class | Header |
|-----------|---------|--------|
| Data structures | `detTempoPoisson3D`, `detPropPoisson3D`, `detCCPoisson3D` | [`estruturasPoisson3D.h`](../../src/estruturasPoisson3D.h) |
| Data / parser | `dadosP3D` | [`dados3DPoisson.h`](../../src/dados3DPoisson.h) |
| Element | `Elem3DPoisson` | [`Elem3DPoisson.h`](../../src/Elem3DPoisson.h) |
| Mesh | `malha3d` | `Malha3DPoisson.h` |
| Solver | `solverP3D` | `solver3DPoisson.h` |

**Key differences from 2D:**

- **Material regions by name** (`string *regiao`) instead of bounding boxes
- **Multiple coupled boundaries** (`nAcop` array of named labels, vs. single `rotuloAcop` in 2D) — supporting multiple pipes in the same soil domain
- Per-boundary arrays: `tParede[]`, `tInt[]`, `hI[]`, `hE[]`, `qAcop[]`, `qTotal[]`, `diamRef[]`
- **Dynamic material regions** (`detMudaRegiao3D`) — supports changing material properties during simulation
- Full material catalog (`materialPoisson3D`) including solid/fluid types (water/air) with viscosity and expansivity for natural convection
- Pipe cross-section definitions (`cortedutoPoisson3D`) with multi-layer, annular geometry, discretization per layer
- Reads **UNV format** with named physical groups

**3D activation** in `SisProd.cpp`: when `modoDifus3D == 1`, the energy loop uses:

$$\text{fluxcal} = -\text{arq.celAcop}[i].\text{FE} \cdot \text{poisson3D.dados.qTotal}[i_{\text{acop}}] / dx$$

replacing the 1D/2D heat transfer model entirely. The scaling factor `FE` maps the 3D flux to the 1D cell.

---

## Special Configurations

### Subsea Pipelines

When `ambext == 1`:

- External fluid properties are **automatically computed** as seawater at the film temperature
- Cross-flow external convection (Churchill-Bernstein) with **natural convection augmentation** at low currents
- Typical wall layers: steel + insulation + concrete coating

### Risers and Wellbores

When `formacPoc == 1`:

- External boundary condition replaced by **Ramey formation resistance**
- Time-dependent: $R_{\text{form}}(\tau_D)$ increases with production time
- No external convection coefficient — formation conduction dominates
- Formation diffusivity: $\alpha_f = k_f / (\rho_f \, c_{p,f})$

### Buried / Trenched Pipelines

When `difus2D == 1` or `modoDifus3D == 1`:

- The 1D radial model is replaced by a 2D or 3D Poisson solver
- `transperm2D()`: iterative pseudo-transient solve to reach steady state
- `transtrans2D()`: true transient 2D soil conduction around the pipe
- Wall conductance and convection coefficients are still computed 1D and passed as boundary conditions to the 2D/3D domain

### Lined / Annular Pipes

When `revest == 1`:

- Inner boundary uses **confined natural convection** (`NussConf2`) between the inner tubing and the outer casing
- Hydraulic diameter: $D_h = 4A/P = \dfrac{4\pi(a^2 - b^2)}{4\pi(a + b)}$
- Inner Grashof uses the annular gap $(a - b)/2$ as the characteristic length

### Annulus with Forced Flow

When `Vconf > 0`:

- Any fluid layer in the wall uses **forced convection** (Petukhov with annulus hydraulic diameter) instead of natural convection
- Applicable for gas-lift annulus or completion fluid circulation

### Column Configuration

When `coluna == 1`:

- External convection reference diameter is replaced by `colunaDia`
- Used when the pipe is inside a wellbore casing of known diameter
- Supports both natural and forced convection selection for the outer annulus

### Prescribed Wall Temperature

When `condiTparede == 1`:

- Internal convection coefficient is set to $h_i = 50000 \cdot k_{\text{wall}} \cdot r_a / \delta r$ (effectively infinite — Dirichlet BC)
- Or user-specified via `novoHi`

### Wax Deposition Layer

The `atualiza()` method can add a new innermost layer (wax deposit) by resizing the radial mesh. `atualiza2()` updates only the first layer thickness. Thermal properties of the wax layer affect the overall resistance network. Key parameters from the cell:

- `deltaPar` — wax layer thickness
- `porosoPar` — wax porosity
- `MW_wax` — molecular weight of wax
- `rhoWaxLiq` — wax liquid density
- `Sum_dCwaxdT` — temperature derivative of wax concentration

---

## Summary of Named Correlations

| Correlation | Method | Application |
|-------------|--------|-------------|
| **Petukhov-Gnielinski** | `nussPet()` | Turbulent internal forced convection |
| **Sleicher-Rouse** | `nussPet()` (high $\mu_w/\mu_b$ branch) | High viscosity ratio internal convection |
| **Churchill-Bernstein (1977)** | `nussChuBer()` | External cross-flow over cylinder |
| **Churchill-Chu (1975)** | `nussNatExt()` | External natural convection from cylinder |
| **Hollands et al. (1976)** | `NussConf2()` ($\theta < 60°$) | Inclined cavity natural convection |
| **Catton** | `NussConf2()` ($\theta = 90°$) | Vertical cavity natural convection |
| **Haaland (1983)** | `fric()` | Turbulent friction factor |
| **Ramey (1962)** | `ResForm()` | Wellbore formation thermal resistance |
| **Gopal** | `ZGopal()` | Gas compressibility factor (annulus) |
| **McCain** | seawater density | Brine formation volume factor |
| **Lee-Gonzalez-Eakin** | annulus gas viscosity | Gas viscosity |
| **Stiel-Thodos** | annulus gas conductivity | Gas thermal conductivity |
| **Baker-Swerdloff** | seawater surface tension | (inherited from ProFlu) |
