# Embedded 2D/3D Solvers — Poisson Heat Diffusion & Porous Medium Flow

Marlim3 embeds specialised multi-dimensional solvers **inside individual pipeline cells** to resolve phenomena that the 1D pipeline discretisation cannot capture: radial/axial heat conduction through complex pipe walls and near-wellbore reservoir flow with multiphase saturations. These solvers are not standalone simulation modes — they live as member objects of cells or accessories and exchange boundary data with the 1D marching solver at every time step.

**Source files:**

| Solver | Header | Implementation |
|--------|--------|----------------|
| Poisson 2D | [`solverPoisson.h`](../../src/solverPoisson.h) | [`solverPoisson.cpp`](../../src/solverPoisson.cpp) |
| Poisson 3D | [`solver3DPoisson.h`](../../src/solver3DPoisson.h) | [`solver3DPoisson.cpp`](../../src/solver3DPoisson.cpp) |
| Poroso 2D | [`solverPoroso.h`](../../src/solverPoroso.h) | [`solverPoroso.cpp`](../../src/solverPoroso.cpp) |
| Radial porous (1D) | [`PorosoRad-Simples.h`](../../src/PorosoRad-Simples.h) | [`PorosoRad-Simples.cpp`](../../src/PorosoRad-Simples.cpp) |
| Radial porous (full) | [`PorosoRad.h`](../../src/PorosoRad.h) | [`PorosoRad.cpp`](../../src/PorosoRad.cpp) |

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Poisson 2D — Transverse Heat Diffusion](#poisson-2d--transverse-heat-diffusion)
3. [Poisson 3D — Axially-Extended Heat Diffusion](#poisson-3d--axially-extended-heat-diffusion)
4. [Radial Porous Flow — Accessory Type 15](#radial-porous-flow--accessory-type-15)
5. [Poroso 2D — Accessory Type 16](#poroso-2d--accessory-type-16)
6. [Coupling to the 1D Pipeline](#coupling-to-the-1d-pipeline)
7. [Shared Numerical Infrastructure](#shared-numerical-infrastructure)
8. [Output Files](#output-files)

---

## 1. Architecture Overview

The following diagram shows where each embedded solver lives in the object hierarchy:

```
SProd (production system)
 │
 ├── celula[i] : Cel                          ← 1D pipeline cell
 │    ├── calor : TransCal                    ← heat-transfer object
 │    │    ├── difus2D : int                  ← flag (1 = Poisson 2D active)
 │    │    └── poisson2D : solverP            ← 2D heat-diffusion solver
 │    │
 │    └── acsr : acessorio                    ← cell accessory
 │         ├── tipo == 15 → radialPoro : PorosRadSimp   ← 1D radial porous
 │         └── tipo == 16 → poroso2D   : solverPoro     ← 2D porous flow
 │                           └── dados.transfer : PorosRad  ← 1D radial coupling annulus
 │
 └── poisson3D : solverP3D                    ← 3D heat-diffusion (system-level)
```

Key points:

- **Poisson 2D** is embedded per-cell in the `TransCal` heat-transfer object, activated by `difus2D == 1`. It solves transverse heat conduction in the pipe cross-section.
- **Poisson 3D** is a single system-level object on `SProd`, coupling to **multiple** pipeline cells simultaneously. It solves heat conduction in a 3D solid (e.g. a buried pipe section).
- **Accessory type 15** (`PorosRadSimp`) is a simplified 1D radial Darcy-flow solver for near-wellbore inflow.
- **Accessory type 16** (`solverPoro`) is a full 2D unstructured porous-flow solver, which internally contains a 1D radial annulus (`PorosRad`) for the wellbore coupling.

---

## 2. Poisson 2D — Transverse Heat Diffusion

**Class:** `solverP` — [`solverPoisson.h`](../../src/solverPoisson.h)

### Governing Equation

Fourier's transient heat conduction in 2D:

$$\rho \, c_p \frac{\partial T}{\partial t} = \nabla \cdot (k \nabla T)$$

where $\rho$, $c_p$ and $k$ are the local density, specific heat and thermal conductivity of the solid (pipe wall, insulation, soil, etc.). The solver resolves the **cross-sectional temperature field** that the standard 1D radial-resistance model cannot capture (e.g. pipes with asymmetric burial, partial insulation, or irregular geometry).

### Mesh

- **Unstructured 2D triangular** — generated externally (Triangle `.ele`/`.node` format or UNV)
- Represents a single pipe cross-section perpendicular to the flow axis
- Classes: `malha2d` ([`Malha2DPoisson.h`](../../src/Malha2DPoisson.h)), `elementoPoisson` ([`Elem2DPoisson.h`](../../src/Elem2DPoisson.h))
- Each element stores: centroid, 3 face normals (`sFace`), face areas, neighbour indices, interpolation factor (`fatG`), non-orthogonality correction vectors (`vecE`, `vecT`, `angES`)

### Numerical Method

**Cell-centred finite-volume method (FVM)** with:

- **Green-Gauss gradient reconstruction** (`calcGradGreen()`) for computing $\nabla T$ at cell centres
- **Deferred-correction** for non-orthogonal face gradients using the decomposition into orthogonal (`vecE`) and tangential (`vecT`) components
- Local matrix has shape `(1, nvert+1)` — one equation per cell with contributions from the cell itself and its `nvert` neighbours
- Global assembly into a **sparse CSR matrix** (`SparseMtx<double>`)

### Boundary Conditions

| BC type | Struct | Description |
|---------|--------|-------------|
| **Dirichlet** | `detDiriPoisson` | Fixed temperature (time-series capable) |
| **Von Neumann** | `detVNPoisson` | Fixed heat flux (time-series) |
| **Robin** | `detRicPoisson` | Convective: $q = h (T - T_{amb})$, time-varying $h$ and $T_{amb}$ |
| **Coupling** | `rotuloAcop` | Internal face coupled to the 1D pipeline — Robin BC with `condLoc` and `hI` from the pipeline cell |

### Key Methods

| Method | Purpose |
|--------|---------|
| `solverP(vg1dSP, meshFile, condGlob, condLoc, hE, hInt, Tint, Tamb, diamI, diamE, indCel)` | Full constructor: reads mesh, applies material properties, sets coupling parameters |
| `permanentePoisson()` | Iterative steady-state solve |
| `inicializaPermanentePoisson()` / `inicializaTransientePoisson()` | Switch all cells between permanent/transient mode flags |
| `transientePoisson(delt)` | Advance one time step (matrix assembly + linear solve) |
| `transientePoissonDummy(delt)` | Same, but also updates `tempC0` — used during initialisation sub-cycling |
| `defineDeltPoisson()` | Returns $\Delta t$ from a piecewise time schedule |
| `finalizaPassoTransiente(delt, indTramo)` | Updates `tempC0 = tempC`, prints profile at scheduled output times |
| `imprimePermanente(indTramo)` | Writes steady-state temperature field to file |
| `FeiticoDoTempo()` | **Rollback**: restores `tempC = tempC0` (undo a failed step) |

### Data Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `dadosP` | [`dados1Poisson.h`](../../src/dados1Poisson.h) | Input container: mesh file path, material properties, BCs, time control, coupling parameters |
| `elementoPoisson` | [`Elem2DPoisson.h`](../../src/Elem2DPoisson.h) | FV element: vertices, faces, normals, `tempC`/`tempC0`, `rho`, `cp`, `cond`, gradient vectors, local matrix |
| `malha2d` | [`Malha2DPoisson.h`](../../src/Malha2DPoisson.h) | Mesh container: element array `mlh2d[]`, vertex coordinates, face connectivity |
| `detCCPoisson` | [`estruturasPoisson.h`](../../src/estruturasPoisson.h) | BC collection: Dirichlet/Neumann/Robin/Coupling arrays, face label maps |

---

## 3. Poisson 3D — Axially-Extended Heat Diffusion

**Class:** `solverP3D` — [`solver3DPoisson.h`](../../src/solver3DPoisson.h)

### What 3D Adds

While the 2D Poisson solver works on a single cross-section for **one** pipeline cell, the 3D solver discretises a **volumetric solid** (e.g. a buried pipeline segment including surrounding soil) with **tetrahedral elements**. It couples to **multiple** pipeline cells simultaneously, each with its own coupling surface label, fluid temperature, and convection coefficient.

### Mesh

- **Unstructured tetrahedral** — imported from UNV format with named regions
- Classes: `malha3d` ([`Malha3DPoisson.h`](../../src/Malha3DPoisson.h)), `elementoPoisson3D` ([`Elem3DPoisson.h`](../../src/Elem3DPoisson.h))
- Volume computed via the scalar triple product of edge vectors
- 4 triangular faces per element, normals via cross product
- **Region-based material assignment**: element region strings matched against named material properties in the input

### Multiple Coupling Surfaces

| Member | Type | Purpose |
|--------|------|---------|
| `nacop` | `int` | Number of coupling surfaces (one per pipeline cell that touches the 3D domain) |
| `dados.tInt[i]` | `double[]` | Fluid temperature from pipeline cell `i` |
| `dados.hI[i]` | `double[]` | Internal convection coefficient from pipeline cell `i` |
| `dados.qAcop[i]` | `double[]` | Computed heat flux through coupling surface `i` |
| `dados.tParede[i]` | `double[]` | Wall temperature at coupling surface `i` |
| `dados.CC.rotuloAcop[i]` | `string[]` | UNV region label identifying coupling surface `i` |

### Key Methods

| Method | Purpose |
|--------|---------|
| `solverP3D(meshFile, vg1d, nacop, dutoAux, hi, he, ti)` | Constructor: reads UNV mesh, assigns materials by region, sets BCs |
| `permanentePoisson()` | Steady-state solve (returns divergence flag) |
| `transientePoisson(delt)` | Single transient time step |
| `transientePoissonDummy(delt, konta)` | Initialisation sub-cycling variant |
| `renova()` | Commit: `tempC0 = tempC` |
| `FeiticoDoTempo()` | Rollback: `tempC = tempC0` |

### Initialisation in Num4Main.cpp

At lines 13604–13665 of [`Num4Main.cpp`](../../src/Num4Main.cpp):

1. The `solverP3D` object is constructed from the UNV mesh file (`modoDifus3DJson`)
2. Each coupling surface label is matched to a pipeline cell index
3. Initial `tInt[]` and `hI[]` are set from cell temperatures and convection coefficients
4. A sub-cycling loop calls `transientePoissonDummy()` until wall heat flux converges (error < tolerance)

---

## 4. Radial Porous Flow — Accessory Type 15

**Class:** `PorosRadSimp` — [`PorosoRad-Simples.h`](../../src/PorosoRad-Simples.h)

This is a **1D radial Darcy-flow** solver representing the near-wellbore reservoir. It is the simpler of the two porous-flow models and is attached to a pipeline cell as accessory type 15.

### Governing Equation

Multiphase radial Darcy flow in cylindrical coordinates:

$$\frac{1}{r}\frac{\partial}{\partial r}\left(r \, \frac{k_{abs} \, k_{r\alpha}}{\mu_\alpha} \frac{\partial}{\partial r}\left(P - P_{c,\alpha} - \rho_\alpha g z\right)\right) = \phi \, c_t \frac{\partial P}{\partial t}$$

for each phase $\alpha \in \{o, w, g\}$ (oil, water, gas). Three-phase flow with:

- **Relative permeability** tables: `kRelOA` (oil-water), `kRelOG` (oil-gas)
- **Capillary pressure** tables: `pcOA` (oil-water), `pcGO` (gas-oil)
- **Formation volume factors** and **viscosities** computed via the embedded `ProFlu` and `ProFluCol` objects

### Geometry

The `DadosGeoPoro` class ([`GeometriaPoro.h`](../../src/GeometriaPoro.h)) defines a multi-layer radial geometry:

| Member | Purpose |
|--------|---------|
| `a` | Inner diameter (wellbore) |
| `b` | Outer diameter (reservoir boundary) |
| `ncamadas` | Number of material layers |
| `kX[i]` | Radial permeability of layer `i` |
| `kY[i]` | Tangential permeability of layer `i` |
| `poro[i]` | Porosity of layer `i` |
| `compRoc[i]` | Rock compressibility of layer `i` (÷1000 in constructor) |
| `diamC[i]` | Outer diameter of layer `i` |
| `espessuR[i]` | Thickness of layer `i` |

### Radial Cells

`celradSimp` ([`celRad-Simples.h`](../../src/celRad-Simples.h)) stores per-cell state for the 1D radial grid:

- Pressures: `Pcamada`, `PcamadaL`, `PcamadaR`, `Pini`, `Piter`
- Phase flow rates: `QocamadaR/L`, `QwcamadaR/L`, `QgcamadaR/L`
- Saturations: `sL`, `sW`, `sLini`, `sWini`, `sLiter`, `sWiter` (+ left/right)
- Densities and viscosities: `rhoP`, `rhogP`, `rhoPa`, `mio`, `mig`, `mia` (current + previous-step)
- Relative permeabilities: `kmed`, `kmedG`, `kmedA` (+ previous-step)
- Capillary pressures: `pcOG`, `pcAO` (+ previous-step)
- Geometry: `r0`, `r1`, `rm` (inner, outer, mid radius), `zD` (datum elevation)

### Key Methods

| Method | Purpose |
|--------|---------|
| `geraCel(Pcamada, sL, sW, sLRes, sWRes)` | Generate radial cell array with initial pressures and saturations |
| `transperm(mastot)` | Steady-state solve via Ridder's root-finding (`zriddr`) — finds total mass flow that satisfies the radial pressure profile |
| `pseudoTrans()` | Pseudo-transient: pressure from root-finding, used during SS pipeline march |
| `avancoPressao()` | Implicit pressure advance (transient IMPES) |
| `avancoSW(delt)` | Explicit saturation advance (CFL-limited) |
| `avancoSWcorrec()` | Saturation correction step |
| `marchaperm(mastot)` | Single radial sweep: given total mass flow, compute pressure profile cell-by-cell |
| `renovaPres(i, mTot)` | Update pressure and saturations at cell `i` |
| `perfil()` | Return radial profile as `FullMtx<double>` (radius, pressure, flow rates, saturations) |
| `FeiticoDoTempo()` | Rollback all cells to previous step |
| `atualizaIni()` | Commit current state as new initial state |

### Coupling

At [`SisProd.cpp`](../../src/SisProd.cpp) lines 4744–4759, during the 1D pressure march:

```
radialPoro.sWPoc  = in-situ water fraction (from pipeline cell)
radialPoro.Pint   = celula[ind].pres        (wellbore pressure)
radialPoro.avancoPressao()                   (solve radial flow)
celula[ind].fontemassLR = fluxIni + fluxIniA (liquid mass source → pipeline)
celula[ind].fontemassGR = fluxIniG           (gas mass source → pipeline)
```

---

## 5. Poroso 2D — Accessory Type 16

**Class:** `solverPoro` — [`solverPoroso.h`](../../src/solverPoroso.h)

This is a full **2D unstructured finite-volume** solver for multiphase porous-medium flow, representing the near-wellbore reservoir in plan view (areal). It is attached to a pipeline cell as accessory type 16.

### Governing Equations

**IMPES** (Implicit Pressure, Explicit Saturation) for three-phase (oil + water + gas) Darcy flow:

**Pressure equation** (implicit):

$$\nabla \cdot \left[ \lambda_o \nabla \Phi_o + \lambda_w \nabla \Phi_w + \lambda_g \nabla \Phi_g \right] = \phi \, c_t \frac{\partial P}{\partial t}$$

where the phase mobilities and potentials are:

$$\lambda_\alpha = \frac{k_{abs} \, k_{r\alpha}}{\mu_\alpha}, \qquad \Phi_o = P - \rho_o g z, \qquad \Phi_w = P - P_{c,AO} - \rho_w g z, \qquad \Phi_g = P + P_{c,GO} - \rho_g g z$$

**Saturation equations** (explicit, CFL-limited):

$$\phi \frac{\partial S_w}{\partial t} = -\nabla \cdot (\lambda_w \nabla \Phi_w)$$

$$\phi \frac{\partial S_L}{\partial t} = -\nabla \cdot \left[(\lambda_o + \lambda_w) (\nabla P - \bar\rho g \nabla z)\right]$$

### Mesh

- **Unstructured 2D triangular** — same FVM topology as Poisson 2D
- Classes: `malha2dPoro` ([`Malha2DPoroso.h`](../../src/Malha2DPoroso.h)), `elementoPoroso` ([`Elem2DPoroso.h`](../../src/Elem2DPoroso.h))
- Green-Gauss gradient reconstruction with non-orthogonality correction
- **Anisotropic permeability** (`kX`, `kY`) per element
- **Datum elevation** `zD` interpolated per cell (tilted reservoir)
- A regular structured output grid (`malhaH`) for visualisation

### Per-Element State

Each `elementoPoroso` stores (beyond the FVM geometry shared with Poisson):

| Field group | Examples |
|-------------|----------|
| Pressure | `presC`, `presC0`, `presIter` |
| Saturations | `sWC`, `sLC`, `alfC` (gas fraction), `sWC0`, `sLC0` |
| Fluid properties | `rhoL`, `rhoG`, `rhoA`, `mioL`, `mioG`, `mioA`, `bo`, `ba`, `rs`, `bg` |
| Relative permeabilities | `kRelOA`, `kRelOG` (tables), `kmed`, `kmedG`, `kmedA` |
| Capillary pressures | `pcOA`, `pcGO` (tables) |
| Rock properties | `kX`, `kY`, `poro`, `compRoc` |
| Gradient arrays | `gradGreenP`, `gradGreenZdatum`, `gradGreenPcAO`, `gradGreenPcOG` |

### Boundary Conditions

| BC type | Struct | Description |
|---------|--------|-------------|
| **Dirichlet** | `detDiriPoroso` | Fixed pressure + saturation (time-series) |
| **Von Neumann** | `detVNPoroso` | Fixed flux + saturation (time-series) |
| **Robin** | `detRicPoroso` | Productivity-index-like BC |
| **Coupling** | `rotuloAcop` + `satAcop` | Coupling face to the embedded 1D radial solver (`dados.transfer`) |

### Internal 1D Radial Annulus

The 2D solver contains an embedded `PorosRad` object (`dados.transfer`) that represents the **radial annulus** between the wellbore and the 2D reservoir mesh boundary. The coupling chain is:

```
Pipeline cell  ←→  1D radial annulus (PorosRad)  ←→  2D reservoir mesh (solverPoro)
                    │                                     │
                    Pint, sWPoc                      coupling BCs on mesh boundary
                    fluxIni, fluxIniG                pressure/saturation solution
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `solverPoro(vg1dSP, meshFile)` | Constructor: reads mesh, material properties, fluid model, BCs |
| `avancoPressao()` | Implicit pressure solve: assembles global sparse matrix, solves with GMRES/BiCGStab, computes face fluxes, updates the radial `transfer` object |
| `avancoSW(delt)` | Explicit saturation advance from face fluxes (CFL-limited) |
| `avancoSWcorrec()` | Saturation correction step |
| `reavaliaDT(delt)` | Re-evaluate time step after CFL violation; sets `reinicia = -1` |
| `reiniciaEvoluiSW(delt)` | Redo saturation evolution with reduced $\Delta t$ |
| `pseudoTransientePoroso()` | Pseudo-transient pressure solve for the steady-state pipeline coupling |
| `transientePoroso(delt)` | Full transient step: pressure + saturation + CFL check |
| `defineDT(perm)` | Compute $\Delta t$ from CFL conditions and user-specified schedule |
| `preparaTabDin()` | Build dynamic PVT tables for the porous domain |
| `geraMiniTabFlu()` | Generate mini fluid-property tables for fast look-up |
| `atualizaCel2D(i)` | Update fluid properties at element `i` from pressure/saturation |
| `atualizaCelTransfer(i)` | Update fluid properties at radial-annulus cell `i` |
| `imprimePseudo()` | Write pseudo-steady-state results to file |
| `imprimeMalhaRegular(minP)` | Interpolate unstructured solution onto regular grid and write to file |
| `FeiticoDoTempo()` | Rollback pressure + saturation on both the 2D mesh and the radial annulus |
| `FeiticoDoTempoSW()` | Rollback saturation only |
| `FeiticoDoTempoPQ()` | Rollback pressure only |

### Coupling

At [`SisProd.cpp`](../../src/SisProd.cpp) lines 4761–4800, during the 1D pressure march:

```
poroso2D.dados.pW.val[0]      = celula[ind].pres    (wellbore pressure → Dirichlet BC)
poroso2D.sWPoc                = in-situ water fraction
poroso2D.dados.transfer.sWPoc = same
poroso2D.dados.transfer.Pint  = celula[ind].pres
poroso2D.avancoPressao()                             (solve 2D pressure field)
celula[ind].fontemassLR = transfer.fluxIni + transfer.fluxIniA  (liquid mass → pipeline)
celula[ind].fontemassGR = transfer.fluxIniG                     (gas mass → pipeline)
```

---

## 6. Coupling to the 1D Pipeline

### Steady-State Coupling

During the steady-state pressure march (`marchaProdPerm*` methods in `SProd`), when the marching algorithm encounters a cell with accessory type 15 or 16:

1. The pipeline cell pressure and in-situ water fraction are passed to the porous solver
2. The solver computes the reservoir inflow (mass flow rates for liquid, water and gas)
3. These mass sources are written back to `fontemassLR` / `fontemassGR` on the pipeline cell
4. The 1D march continues with the updated source terms

For Poisson 2D, the steady-state coupling is implicit within the `TransCal` heat-balance: the `condLoc` / `hI` coupling BC on the Poisson mesh boundary exchanges heat with the fluid.

### Transient Coupling

During the transient time loop in [`SisProd.cpp`](../../src/SisProd.cpp):

| Phase | What happens | Line refs |
|-------|-------------|-----------|
| **Pressure advance** | `avancoPressao()` on each type-15/16 accessory (implicit pressure, coupled to pipeline cell pressure) | L4747, L4782 |
| **Saturation advance** | `avancoSW(dt)` on each type-15/16 accessory (explicit, after 1D fraction evolution `EvoluiFrac()`) | L12619, L12625 |
| **CFL check** | If any porous solver sets `reinicia == -1`, the pipeline triggers `ReiniEvolFrac0()` (global rollback of 1D fractions), then `reavaliaDT()` + `reiniciaEvoluiSW()` with reduced $\Delta t$ | L12630–L12655 |
| **Saturation correction** | `avancoSWcorrec()` after convergence | L12663, L12665 |
| **Poisson 2D** | `transientePoisson(dt)` called within the `TransCal` heat-balance loop; fluid temperature `tInt` and convection coefficient `hI` updated from the pipeline cell | per-cell in TransCal |
| **Poisson 3D** | `transientePoisson(dt)` called once per time step on the system-level `poisson3D` object; each coupling surface receives its cell's `temp` and `hI` | L12405 in SisProd.cpp |
| **Rollback** | On outer-iteration failure, `FeiticoDoTempo()` restores both the 1D pipeline state and all embedded solver states to the previous time level | all solvers |

### IMPES Time-Step Control

The porous solvers use an IMPES scheme where:

- **Pressure** is solved implicitly (sparse linear system) — unconditionally stable
- **Saturation** is advanced explicitly — CFL-limited

When the saturation advance violates the CFL condition, the porous solver sets `reinicia = -1`. The 1D pipeline detects this, rolls back the global 1D state (`ReiniEvolFrac0`), and restarts the step with a smaller $\Delta t$. This ensures that the 1D pipeline and porous solvers remain synchronised.

---

## 7. Shared Numerical Infrastructure

All three FVM solvers (Poisson 2D, Poisson 3D, Poroso 2D) share the same numerical backbone:

### Gradient Reconstruction

**Green-Gauss method**: the gradient at cell centre $C$ is computed as:

$$(\nabla \phi)_C = \frac{1}{V_C} \sum_{f} \phi_f \, \vec{S}_f$$

where $\phi_f$ is the face value (interpolated from neighbour cells) and $\vec{S}_f$ is the outward face-area vector. Each element stores `gradGreen[]` arrays for the primary variable and auxiliary fields (datum elevation, capillary pressures).

### Non-Orthogonality Correction

Face gradients use a deferred-correction approach. For each face between cells $C$ and $N$:

$$\left(\frac{\partial \phi}{\partial n}\right)_f = \frac{\phi_N - \phi_C}{|\vec{E}|} + \text{correction}(\vec{T}, \nabla\phi)$$

where $\vec{E}$ is the vector connecting cell centres and $\vec{T}$ is the tangential correction. The decomposition is controlled by the angle `angES` between $\vec{E}$ and the face normal $\vec{S}$, with the interpolation factor `fatG`.

### Sparse Linear System

| Component | Implementation |
|-----------|----------------|
| Matrix format | CSR (`SparseMtx<double>`) |
| Assembly | Per-element local matrices assembled into the global system |
| Solvers | GMRES (`solverMat==0`), FGMRES (`solverMat==1`), BiCGStab (`solverMat==2`) |
| Preconditioner | ILU(k) with configurable fill level (`rankLU`), optional graph colouring for parallelism |
| Tolerance | $10^{-5}$ (Poisson), $10^{-6}$ (Poroso) |
| Max iterations | `nele` (number of elements) or 80 (Poroso pressure) |

### Rollback Mechanism — FeiticoDoTempo

All embedded solvers implement `FeiticoDoTempo()` ("time spell" / "time magic"), which restores the solution to the previous time level:

- **Poisson**: `tempC = tempC0` for all elements
- **Poroso 2D**: `presC = presC0`, `sWC = sWC0`, `sLC = sLC0` for all elements + radial annulus
- Variants: `FeiticoDoTempoSW()` (saturation only), `FeiticoDoTempoPQ()` (pressure only)

This mechanism supports the outer coupling iteration: if the 1D pipeline needs to retry a time step, all embedded solvers can be rolled back without re-reading data.

---

## 8. Output Files

| File pattern | Solver | Content |
|--------------|--------|---------|
| `PerfisPocoRadial-{t}-{pos}.dat` | Type 15 (`PorosRadSimp`) | Radial profile: radius, pressure, oil/gas/water flow rates, liquid/water saturation, gas void fraction |
| Poisson 2D steady-state output | `solverP::imprimePermanente()` | Cross-sectional temperature field after steady-state convergence |
| Poisson 2D transient output | `solverP::finalizaPassoTransiente()` | Temperature field at scheduled output times |
| Poroso 2D pseudo-steady output | `solverPoro::imprimePseudo()` | 2D pressure/saturation field |
| Regular-grid visualisation | `solverPoro::imprimeMalhaRegular()` | Interpolated solution on structured grid (pressure, saturation, gas fraction) |
