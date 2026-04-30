# Sensitivity Analysis

This document describes the **sensitivity analysis** (Análise de Sensibilidade — AS) framework in Marlim3. Sensitivity analysis automates the execution of multiple steady-state simulations over a user-defined parameter space, producing pressure/temperature profiles, VFP tables, and summary reports for each combination.

**Key source files:**

| File | Role |
|------|------|
| [`src/LerAS.h`](../../src/LerAS.h) | `ASens` class definition, parameter structs, `variaveis` activation flags |
| [`src/LerAS.cpp`](../../src/LerAS.cpp) | JSON parsing, sequence generation, parameter selection, output formatting |
| [`src/Num4Main.cpp`](../../src/Num4Main.cpp) | `leituraAS()`, `leituraASparalelo()`, `leituraASparaleloReserva()` |

---

## Table of Contents

1. [Overview](#overview)
2. [Activation from Main](#activation-from-main)
3. [JSON Input Format](#json-input-format)
4. [Sweepable Parameters](#sweepable-parameters)
5. [The ASens Class](#the-asens-class)
6. [Sequence Generation — Full Factorial](#sequence-generation--full-factorial)
7. [Parameter Selection — selecaoAS](#parameter-selection--selecaoas)
8. [Serial Execution — leituraAS](#serial-execution--leituraas)
9. [Parallel Execution — leituraASparalelo](#parallel-execution--leituraasparalelo)
10. [VFP Table Generation](#vfp-table-generation)
11. [Output Files](#output-files)
12. [Worked Example](#worked-example)

---

## Overview

Sensitivity analysis performs a **full factorial sweep** (Cartesian product) over user-specified parameter values. For each combination, the simulator:

1. Substitutes the parameter values into the `SProd` system
2. Solves the steady-state problem via `SolveTramoSolteiro()` → `permanenteSimples()`
3. Collects results (bottom-hole pressure, temperature, profiles, trends)
4. Reports success or failure

**Scope:** Single tramo (pipeline segment) only — sensitivity analysis does not operate on networks.

**Output modes:**
- `tipoAS == 0` — Standard: writes profiles, trends, and summary per case
- `tipoAS == 1` — VFP table: builds bottom-hole pressure tables for reservoir simulation coupling

---

## Activation from Main

Sensitivity analysis is triggered when the JSON input sets `AS == 1` (field `"analiseSensibilidade"` in the tramo JSON). The dispatch in `main()` ([`Num4Main.cpp`](../../src/Num4Main.cpp)):

```
main()
 └─ tipoSimulacao != rede AND != convecNatural:
     └─ SProd sistem1(...)                          // build system
         └─ arq.perm == 1:                          // steady-state enabled
             ├─ arq.AS == 0 → SolveTramoSolteiro()  // single solve
             └─ arq.AS == 1:
                 ├─ arq.paralelAS == 0 → leituraAS("leituraAS.json", sistem1)
                 └─ arq.paralelAS == 1 → leituraASparalelo("leituraAS.json", ...)
```

The input file is always named `leituraAS.json`, placed alongside the tramo JSON.

---

## JSON Input Format

The sensitivity analysis JSON file defines which parameters to sweep and their explicit value lists. Example ([`demos/leituraAS.json`](../../demos/leituraAS.json)):

```json
{
  "nthread": 4,
  "tipoAS": 0,
  "vfp": 1,

  "IPR": [
    {
      "indiceIPR": 0,
      "presEstatica": [240, 250, 260, 255]
    },
    {
      "indiceIPR": 1,
      "IP": [20, 30, 15]
    }
  ],

  "psep": {
    "pressao": [25, 10, 30]
  },

  "GasLift": {
    "vazGas": [150000.0, 100000, 80000, 200000]
  },

  "choke": {
    "abertura": [0.2, 0.8, 0.1]
  }
}
```

### Global settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `nthread` | int | 1 | Number of OpenMP threads for parallel mode |
| `tipoAS` | int | 0 | 0 = standard output, 1 = build VFP tables |
| `vfp` | int | 1 | VFP format: 1 = Eclipse standard, 0 = IMEX pressure curve |

### Parameter sections

Each section corresponds to a component type. Only sections present in the JSON are activated. Values are given as **explicit arrays** — not min/max/step ranges. Each array element defines one level for that sub-parameter.

The schema is validated against `schema_AS_1_0_0.json` at parse time via RapidJSON schema validation.

---

## Sweepable Parameters

The following table lists all parameter types that can be swept. Each type maps to a detail struct in `ASens` and a `variaveis` activation flag.

| JSON Section | Activation Flag | Detail Struct | Sweepable Sub-Parameters |
|-------------|----------------|---------------|-------------------------|
| `"IPR"` | `vipr` | `detIPRAS` | `presEstatica` (reservoir pressure), `tempRes` (reservoir temperature), `IP` (productivity index), `II` (injectivity index), `qMax` (max rate), `fluido` (fluid index) |
| `"GasLift"` | `vgasinj` | `detGASINJAS` | `temperatura`, `presInj` (injection pressure), `vazGas` (gas rate, Sm³/d) |
| `"psep"` | `vpsep` | `detPSEPAS` | `pressao` (separator / outlet pressure) |
| `"RGO-fluido0"` | `vRGO` | `detRGOAS` | `RGO` (gas-oil ratio of fluid 0) |
| `"BSW-fluido0"` | `vBSW` | `detBSWAS` | `BSW` (basic sediment & water of fluid 0) |
| `"pentrada"` | `vpresent` | `detPresEntAS` | `pressao`, `temperatura`, `titulo` (quality), `beta` |
| `"vazpentrada"` | `vvazpresent` | `detVazPresEntAS` | `pressao`, `temperatura`, `vazMass` (mass flow), `beta` |
| `"FonteLiquido"` | `vfonliq` | `detFONLIQAS` | `temperatura`, `beta`, `vazLiq` (liquid rate), `fluido` (fluid index) |
| `"FonteGas"` | `vfongas` | `detFONGASAS` | `temperatura`, `vazGas`, `vazComp` (complementary rate) |
| `"FonteMassa"` | `vfonmas` | `detFONMASSAS` | `temperatura`, `vazaoProd`, `VazaoComp`, `VazaoGas`, `fluido` |
| `"BCS"` | `vbcs` | `detBCSAS` | `frequencia` (pump frequency), `estagio` (number of stages) |
| `"BVol"` | `vbvol` | `detBVOLAS` | `frequencia`, `capacidade` (pump capacity), `fatorPoli` (polytropic factor) |
| `"Valvula"` | `vvalv` | `detValvAS` | `abertura` (opening fraction), `CD` (discharge coefficient) |
| `"Vazamento"` | `vfuro` | `detFUROAS` | `pressao`, `temperatura`, `beta`, `CD`, `abertura`, `fluido` |
| `"DP"` | `vdp` | `detDPREQAS` | `deltapressao` (localized ΔP) |
| `"dPdLHidro"` | `vdpH` | `detDPHidro` | `fator` (hydrostatic correction factor) |
| `"dPdLFric"` | `vdpF` | `detDPFric` | `fator` (friction correction factor) |
| `"dTdL"` | `vdt` | `detDT` | `fator` (temperature gradient correction) |
| `"choke"` | `vchk` | `detCHOKESUPAS` | `abertura` (opening fraction), `CD` |
| `"secaoTransversal"` | `diam` | `detDiamRug` | `DiamIntMaior` (major ID), `DiamIntMenor` (minor ID), `Rugosidade` (roughness) |
| `"Condutividade"` | `kequiv` | `detCondEquiv` | `condutividade` (thermal conductivity of a wall layer) |
| `"pocoInjetor"` | `vpocinj` | `detCondConInjecAS` | `pressaoInj`, `pressaoFinal` (BHP), `temperatura`, `vazao` |

**Multi-instance parameters:** `IPR`, `FonteLiquido`, `FonteGas`, `FonteMassa`, `BCS`, `BVol`, `DP`, `dPdLHidro`, `dPdLFric`, `dTdL`, `Valvula`, `Vazamento`, `secaoTransversal`, and `Condutividade` are JSON arrays — each element targets a specific accessory by its `indice*` field (zero-based index). Multiple accessories of the same type can be swept independently.

**Single-instance parameters:** `psep`, `RGO-fluido0`, `BSW-fluido0`, `GasLift`, `choke`, `pentrada`, `vazpentrada`, and `pocoInjetor` are single objects — only one set of values per type.

---

## The ASens Class

The `ASens` class ([`LerAS.h`](../../src/LerAS.h)) manages the entire sensitivity analysis lifecycle.

### Construction

```cpp
ASens analiseSens(vg1dSP, "leituraAS.json", ncel, celp, flup, bcs, fonteg);
```

The constructor:

1. Calls `parseEntrada()` — loads and validates the JSON against `schema_AS_1_0_0.json`
2. Calls `parse_variaveis()` — detects which sections are present, sets `listaV` flags
3. Calls individual `parse_*()` methods for each active section (e.g., `parse_IPR()`, `parse_Psep()`, etc.)
4. Calls `lerArq()` which:
   - Computes `dim` (total number of independent sub-parameter dimensions)
   - Computes `nVariaveis` (total cases = product of all `parserie*` counts)
   - Calls `constroiVecParSerie()` — builds the `vecParSerie` array
   - Calls `inicializaSequen()` — generates all case index tuples
   - Calls `traduzSeq()` — translates generic indices to parameter-specific indices

### Key members

| Member | Type | Description |
|--------|------|-------------|
| `listaV` | `variaveis` | Activation flags — one `int` per parameter type (0 = inactive, 1 = active) |
| `dim` | `int` | Total number of independent dimensions in the sweep |
| `nVariaveis` | `int` | Total number of cases (full factorial product) |
| `sequenciaAS` | `casoVEC*` | Array of size `nVariaveis` — each element stores per-parameter indices for one case |
| `genericoAS` | `genericoVEC*` | Array of size `nVariaveis` — flat index tuples before translation |
| `vecParSerie` | `int*` | Array of size `dim` — number of levels per dimension |
| `varSeq[22]` | `int[22]` | Maps the 22 parameter types to their position in the enumeration sequence |
| `tipoAS` | `int` | Output mode: 0 = standard, 1 = VFP tables |
| `vfp` | `int` | VFP format: 1 = Eclipse, 0 = IMEX |
| `nthrdAS` | `int` | Thread count for parallel execution |
| `saidaBHP` | `double**` | BHP output table (when `tipoAS == 1`) |
| `saidaVazLiq` | `double**` | Liquid rate output table (when `tipoAS == 1`) |

---

## Sequence Generation — Full Factorial

The sweep generates all combinations using a Cartesian product. The total number of cases:

$$N_{\text{cases}} = \prod_{k=1}^{\text{dim}} n_k$$

where $n_k$ is the number of values for dimension $k$ (stored in `vecParSerie[k]`).

### Step 1 — constroiVecParSerie

Builds the `vecParSerie` array by iterating through all active parameter types in a fixed order (defined by `varSeq[22]`). For each active sub-parameter with `parserie > 0`, appends its count to `vecParSerie`.

### Step 2 — inicializaSequen

Generates all index tuples using a **nested-loop counter** (odometer pattern):

```
iVaria[0..dim-1] = {0, 0, ..., 0}        // start at all zeros

for seq = 0 to nVariaveis-1:
    genericoAS[seq].generico = copy of iVaria[]
    
    // increment rightmost dimension, carry left on overflow
    iVaria[dim-1]++
    for d = dim-1 down to 1:
        if iVaria[d] >= vecParSerie[d]:
            iVaria[d] = 0
            iVaria[d-1]++
```

**Example:** If `vecParSerie = [4, 3, 3]` (IPR pressure × IPR IP × separator pressure), the 36 cases enumerate as:
`(0,0,0), (0,0,1), (0,0,2), (0,1,0), (0,1,1), ..., (3,2,2)`.

### Step 3 — traduzSeq

Translates each `genericoAS[seq].generico` tuple into the parameter-specific indices stored in `sequenciaAS[seq]`. The mapping follows the fixed order of parameter types, distributing indices to the correct `casoVEC` vectors (e.g., `IPRpres`, `PSEPpres`, `BCSfreq`, etc.).

---

## Parameter Selection — selecaoAS

`selecaoAS()` (and its variant `selecaoASImex()`) applies a specific case's parameter values to the simulation system. For case `seq`:

1. **For each active parameter type:**
   - Look up the index from `sequenciaAS[seq]` (e.g., `sequenciaAS[seq].PSEPpres`)
   - Fetch the actual value from the detail struct (e.g., `ASPsep.pres[index]`)
   - Update the corresponding system variable (e.g., `pGSup = value`)

2. **Parameter-specific post-processing** (back in the calling function):
   - Gas injection: convert Sm³/d to mass flow rate using gas density
   - Choke opening: copy to `chokep.abertura[0]`
   - Pressure inlet BC: set `presE`, `betaE`, `tempE`, `titE`
   - Flow-pressure inlet BC: compute phase mass flows from total mass rate
   - Injection well: set `condpocinj.presfundo`, injection temperature and rate

3. **Geometry updates** (`atualizaGeom()`): when diameter or roughness is swept, updates all cells in the target pipe section

4. **Fluid property updates** (`atualizaCompRGO()`, `atualizaBSW()`): when GOR or BSW is swept, recomputes fluid composition for fluid index 0

Two variants exist:
- `selecaoAS()` — writes parameter selections to `relacaoAS.dat` (serial mode)
- `selecaoASsemImpre()` — silent version used in parallel mode (no file I/O during parallel loop)

---

## Serial Execution — leituraAS

`leituraAS()` ([`Num4Main.cpp`](../../src/Num4Main.cpp)) runs cases sequentially:

```
leituraAS(nomeArquivoAS, sistem1):
│
├─ Construct ASens → parse JSON, generate sequences
├─ Create "sucessoAS.dat" file
│
└─ FOR iSeq = 0 to nVariaveis-1:
    │
    ├─ selecaoAS(seq=iSeq) → apply parameters to sistem1
    ├─ Post-process special parameters (gas injection, choke, inlet BC)
    │
    ├─ Determine initial guess (chute):
    │   ├─ iSeq == 0: chute = -1 (no guess)
    │   └─ iSeq > 0: use previous solution as guess
    │       ├─ CC=0: chute = celula[0].pres
    │       └─ CC≠0: chute = flow rate from previous solution
    │
    ├─ SolveTramoSolteiro(sistem1, chute)
    │   └─ If failed and chute > 0: retry without guess
    │
    ├─ Append success/failure to "sucessoAS.dat"
    │
    └─ Output (depends on tipoAS):
        ├─ tipoAS == 0: imprimeProfile, resumoPermanente, trends, Poisson 2D
        └─ tipoAS == 1: tabelaGenerica → fill BHP/VazLiq tables
```

**Key optimization:** For `iSeq > 0`, the previous solution is used as the initial guess (`chute`), which typically accelerates convergence when parameters change incrementally. If the guess leads to failure, the solver retries without it.

---

## Parallel Execution — leituraASparalelo

`leituraASparalelo()` ([`Num4Main.cpp`](../../src/Num4Main.cpp)) uses **OpenMP** to solve cases concurrently:

### Pre-parallel phase (single-threaded)

1. Construct `ASens` object
2. Write output headers (`cabecalhoAS()` / `cabecalhoASImex()`)
3. Write trend file headers
4. **Create per-case copies** of the input data (`Ler` objects, state variables):
   ```cpp
   Ler *vecArq = new Ler[nVariaveis];
   varGlob1D *vg1dTramo = new varGlob1D[nVariaveis];
   for (iSeq = 0; iSeq < nVariaveis; iSeq++)
       vecArq[iSeq].copiaSemJson(sistem1.arq);
   ```

### Parallel phase

```cpp
#pragma omp parallel for num_threads(analiseSens.nthrdAS)
for (int iSeq = 0; iSeq < analiseSens.nVariaveis; iSeq++) {
    SProd sistem2;
    sistem2.copiaSemJson(vecArq[iSeq], ...);   // thread-local SProd copy
    selecaoASsemImpre(sistem2, iSeq);           // apply parameters (no I/O)
    // ... post-process special parameters ...
    SolveTramoSolteiro(sistem2);                // solve (no initial guess)
    // ... collect results, write profiles/trends ...
}
```

Each thread:
- Creates an **independent `SProd` object** (`sistem2`) from the pre-copied input data
- Applies parameters via `selecaoASsemImpre()` (no file writes — thread-safe)
- Solves independently — no initial guess from prior cases (unlike serial mode)
- Writes per-case profiles and trends (each case has a unique file suffix)

### Post-parallel phase (single-threaded)

1. Sort results by case index (parallel execution order is non-deterministic)
2. Re-run `selecaoAS()` with `imprime=1` to write `relacaoAS.dat` parameter log
3. Write `variaveisInteresseAS.dat` with collected BHP and temperature per case
4. Write `sucessoAS.dat` with success/failure report and timing

### Backup variant — leituraASparaleloReserva

A more memory-intensive but safer variant (`leituraASparaleloReserva()`) that constructs **fully independent system objects** per thread (re-parsing JSON for each case). Used when shared-state copies prove unreliable for complex simulations.

---

## VFP Table Generation

When `tipoAS == 1`, the framework builds **Vertical Flow Performance (VFP) tables** — lookup tables mapping operating conditions to bottom-hole pressure, used by reservoir simulators (Eclipse, IMEX).

### Standard VFP (vfp == 1)

For each case:
1. Solve steady-state
2. Record BHP in `saidaBHP[iSeq]`
3. Record liquid rate in `saidaVazLiq[iSeq]`
4. Call `tabelaGenerica()` to store results in the multi-dimensional output arrays

The resulting table dimensions correspond to the swept parameter axes (e.g., liquid rate × GOR × water cut × gas-lift rate × wellhead pressure).

### IMEX VFP (vfp == 0)

Same structure but uses the IMEX format for the pressure curve output. Parameter selection uses `selecaoASImex()` and output uses `imprimeVarInteresseASImex()`.

---

## Output Files

### sucessoAS.dat

Reports success/failure for each case with execution timing:

```
relatório de falhas da Analise de Sensibilidade para um Tramo
0 :  Resultado = sucesso
1 :  Resultado = sucesso
2 :  Resultado = falha
...

datahora = 29/4/2026 14:32:05
     DURACAO    127 segundos
     Versao    3.5.0
```

### variaveisInteresseAS.dat

Semicolon-separated table with one row per case. Written in parallel mode only:

```
indice AS ; Sucesso ; Pressao Fundo ; Temperatura Plataforma ; <param1> ; <param2> ; ...
0 ; 1 ; 312.45 ; 42.3 ; 240 ; 25 ; ...
1 ; 1 ; 308.12 ; 41.8 ; 250 ; 25 ; ...
2 ; -1 ; -1e10 ; -1e10 ; 260 ; 25 ; ...
```

The header is dynamically generated by `cabecalhoAS()` based on which parameters are active.

### relacaoAS.dat

Detailed parameter log per case (appended during `selecaoAS()` with `imprime=1`):

```
0 : indice Pressao Separador = 0 Pressao Separador = 25
0 : indice presEstaticaIPR0 = 0 presEstaticaIPR0 = 240
1 : indice Pressao Separador = 0 Pressao Separador = 25
1 : indice presEstaticaIPR0 = 1 presEstaticaIPR0 = 250
...
```

### Per-case output files

For `tipoAS == 0`, each case generates the standard simulation outputs:

| Output | Content |
|--------|---------|
| Pressure/temperature profile | Cell-by-cell P, T, holdup along pipeline |
| Steady-state summary | BHP, rates, GOR, water cut |
| Trend data | Variables at specific monitoring cells |
| Gas line profile | (if `lingas == 1`) Gas system P, T, velocity |
| Poisson 2D | (if `difus2D == 1`) Thermal field in buried cells |

File naming includes the AS sequence index (e.g., `_AS_0.log`, `_AS_1.log`, etc.).

---

## Worked Example

Given the demo [`leituraAS.json`](../../demos/leituraAS.json):

```json
{
  "IPR": [
    {"indiceIPR": 0, "presEstatica": [240, 250, 260, 255]},
    {"indiceIPR": 1, "IP": [20, 30, 15]}
  ],
  "psep": {"pressao": [25, 10, 30]},
  "GasLift": {"vazGas": [150000, 100000, 80000, 200000]},
  "choke": {"abertura": [0.2, 0.8, 0.1]}
}
```

**Activated flags:** `vipr = 1`, `vpsep = 1`, `vgasinj = 1`, `vchk = 1`

**Dimensions:**
- IPR[0].presEstatica: 4 values → `parseriePres = 4`
- IPR[1].IP: 3 values → `parserieIP = 3`
- psep.pressao: 3 values → `parseriePres = 3`
- GasLift.vazGas: 4 values → `parserieVazGas = 4`
- choke.abertura: 3 values → `parserieAbre = 3`

**dim** = 5 independent dimensions

**nVariaveis** = 4 × 3 × 3 × 4 × 3 = **432 cases**

Each case is a unique combination such as:
- Case 0: IPR₀.pres=240, IPR₁.IP=20, psep=25, GL=150000, choke=0.1
- Case 1: IPR₀.pres=240, IPR₁.IP=20, psep=25, GL=150000, choke=0.2
- ...
- Case 431: IPR₀.pres=260, IPR₁.IP=30, psep=30, GL=200000, choke=0.8

> **Note:** Values within each array are sorted by `ASens` after parsing, so the actual enumeration order uses sorted values regardless of the order in the JSON.
