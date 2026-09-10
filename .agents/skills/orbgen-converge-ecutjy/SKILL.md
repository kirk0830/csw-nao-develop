---
name: "orbgen-converge-ecutjy"
description: "Run and post-process the ecutjy (spherical-wave kinetic-energy cutoff) convergence test for ORBGEN primitive basis sets, using the band-structure eta criterion. Invoke when the NSW/jy cutoff is unknown. For rcut/lmax convergence see orbgen-converge-rcutlmax."
---

# orbgen-converge-ecutjy — ecutjy convergence

`ecutjy` is the **kinetic-energy cutoff of the spherical-wave (NSW/jy) basis** used to build the primitive orbitals. This sub-skill determines how large it must be for the NSW expansion to represent the atomic wavefunctions.

Reference implementation: `tools/JYEkinConvTest{Generator,Driver,Reader}.py`.

## Why a single atom suffices

The cutoff just has to include every spherical wave whose kinetic energy is below the cutoff. If the NSW basis can represent a single atom's wavefunction, it will also represent a molecule's — so the test uses a **single atom / monomer in a cell**, making it cheap.

## Acceptance metric: η (band-structure convergence), not total energy

Total energy is a poor indicator of how many NSW functions are enough (occupied-state bias; tight dimers need far more). Instead the test compares the whole **band structure** of each `ecutjy` against the assumed-converged reference (the largest `ecutjy`), allowing a global energy shift to be absorbed:

```
η = min_ω  sqrt( Σ_{n,k} f̃ (e₁,ₙₖ − e₂,ₙₖ + ω)² / Σ_{n,k} f̃ ),   f̃ = √(occ₁·occ₂)  (only if occ-weighted)
```

- Read from each ABACUS `OUT.*/istate.info` (band energies + occupations) and `running_scf.log`.
- **Band selection:** pass an explicit `nzeta` (e.g. `[3,3,2,1]` ≈ Dunning cc-pVTZ) to auto-pick each angular momentum's bands by degeneracy (`2l+1`) from the assumed-converged reference; or pass `ibands`; otherwise all bands are used.
- **Reference thresholds** (default unit eV, plotted on log scale):
  - `η < 20 meV` → **err: not-bad**
  - `η < 10 meV` → **err: safe**
- The `ecutjy` value just past the knee below the desired threshold is the answer; project default is `ecutjy=60` for ~1 kcal/mol chemical accuracy.

## Setup (defaults)

- Fixed for the test: `nzeta=[1,1,0]` (1s1p), `lmax=3`, `rcut=10`, dimer geometry, `ecutgrid` large (default `250`).
- Sweep `ecutjy`, e.g. `[10, 20, 40, 60, 80, 100, 150, 200]`.
- `client_solver` note: `tools/JYEkinConvTestDriver.py` runs with **`ks_solver: genelpa`** and hardcodes **`ecutwfc: 100`**. At large `rcut`/`lmax` a near-singular overlap can make `genelpa` **silently hang** (no output); if this test stalls, suspect singular overlap and try `scalapack_gvx` (fail-fast). See `orbgen-converge-rcutlmax` for the full discussion.

## Steps

1. **Generate** the per-`ecutjy` jY orbitals and the driver inputs:
   ```
   python3 tools/JYEkinConvTestGenerator.py
   ```
   Configure in its `__main__`/call: `elem` (e.g. `Si`), `ecutjy` (list), `ecutgrid`, `celldm`, `fpsp`, `orbgen`, `lmax`, `rcut`. It runs `orbgen` for each `ecutjy` into `{elem}-jy-lmax{lmax}-{rcut}au`, copies the pseudopotential, and writes `driver.json` under `JYEkinConvTest-{elem}/ecutjy-{...}/`.
2. **Run the jobs:** each job folder runs
   `python3 JYEkinConvTestDriver.py -i driver.json`.
   Locally set `abacustest='__local__'` to generate only and run manually; otherwise it submits to Bohrium via `abacustest -i registry.dp.tech/dptech/abacus:3.8.1 -f <elemdir> -c "python3 JYEkinConvTestDriver.py -i driver.json"`.
3. **Post-process / plot:**
   ```
   python3 tools/JYEkinConvTestReader.py
   ```
   (or call `main(src, iterparse=True, nzeta=[3,3,2,1], occ_wt=False)`). It computes η vs the largest `ecutjy`, dumps `{ekin, eta}` JSON, and plots `EkinConvTest.png` with the 20/10 meV threshold lines.
4. **Recommend** the smallest `ecutjy` whose η is below the chosen threshold, and return it to the orchestrator.

## Returning to the caller

Feed the chosen `ecutjy` into the ORBGEN input's `ecutjy` (or rely on the default `ecutwfc`). Remember the quick-pick fallback lives in `orbgen-primitive` (`ecutwfc = ecutjy + 50 Ry`; historic v2.0 default `ecutjy == ecutwfc == 100`).