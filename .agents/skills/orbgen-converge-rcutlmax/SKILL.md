---
name: "orbgen-converge-rcutlmax"
description: "Run and post-process the rcut/lmax joint convergence test for ORBGEN primitive basis sets, against a plane-wave reference, to pick truncation radius and maximal angular momentum. Invoke when truncation parameters are unknown. For ecutjy convergence see orbgen-converge-ecutjy."
---

# orbgen-converge-rcutlmax — rcut × lmax joint convergence

`rcut` (truncation radius) and `lmax` (maximal angular momentum) control the *completeness* of the primitive basis. This sub-skill determines them by varying both and watching the **relative total-energy error of the primitive (NSW/CSW) basis vs. a plane-wave reference** over several dimer bond lengths.

Reference implementation: `tools/JYLmaxRcutJointConvTest{Generator,Driver,Reader}.py`.

## Acceptance criterion

**The primitive basis is deemed converged when its total energy lies within a target of the plane-wave reference.** The project convention (see `tools/README.md`) targets **~1 kcal/mol "chemical accuracy"** — the published CSW-NAO defaults satisfying this are `ecutjy=60`, `lmax=3`, `rcut=8`. Use the user's target if they have one; otherwise 1 kcal/mol is the default.

Choose the **smallest** `(lmax, rcut)` meeting the target (balancing accuracy vs. cost). Judge against the **averaged-over-bond-lengths** panel of the plot, not a single geometry.

## Why a plane-wave reference, and why several bond lengths

- rcut/lmax completeness can only be judged against a genuinely complete basis → compare to a **plane-wave** calculation (`basis_type: pw`, `ks_solver: dav`) at the same geometry.
- More bond lengths give a more transferable answer, not a single-geometry fluke. The generator ships per-element dimer bond-length tables (`DIMER_BOND_LENGTH_7/5POINTS`) so many distorted dimer geometries are swept.
- The NAO total energy minus the PW total energy at each bond length is the plotted error (∫∫; default `kcal/mol`, may use logscale).

## Method background

The contraction **minimizes the trace of the kinetic operator in the residual space**, generalizing the spillage-minimizing scheme [M. Chen et al., J. Phys. Condens. Matter 22, 445501 (2010); P. Lin et al., Phys. Rev. B 103, 235131 (2021)]. The CSW implementation is described in the repo's paper, **arXiv:2603.13995** ("Systematically Improvable NAO Basis Using Contracted Truncated Spherical Waves"): using contracted truncated spherical waves (instead of plane waves) as the expansion basis bridges reference states and NAOs more effectively and removes spurious periodic-image interactions, improving transferability.

## Setup constraints

- `ecutjy` is fixed (project default `60` Ry) for this test; only `lmax`/`rcut` vary.
- `ecutgrid` (the `ecutwfc` passed to the driver) must be **≥ `ecutjy + 10`**, and larger than the convergence value of *both* Vloc and the jY grid integration.
- `celldm` must be large enough that cell images don't overlap the dimer at the largest rcut — the generator auto-resizes it upward if needed.
- The test runs **NSCF** (optionally SCF) energy evaluation per `(bond length, lmax, rcut)`, so it is heavier than the ecutjy test.
- jY orbitals are pseudopotential-independent: the same primitive set is shared across bond lengths (only the element label is renamed), saving cost.

## Risks of over-large `rcut` / `lmax`

Bigger is *generally* more complete, but there are real failure modes an agent must watch for:

- **Near-singular overlap matrix.** Very diffuse functions from large `rcut`, or too many/high-`l` functions, can produce an overlap matrix that is (nearly) singular.
- **`ks_solver: genelpa` (default) silently hangs.** With a singular overlap, the SCF appears to *freeze — no output, no error*. If a run shows no progress/output, suspect this before anything else.
  - **Detection:** check the solver output for stalls; if `genelpa` is set and output stops, treat as a possible singular-overlap hang.
  - **Warning the user:** when `rcut`/`lmax` are pushed large, proactively warn that a singular overlap could hang SCF.
- **Fail-fast mitigation:** switch to **`ks_solver: scalapack_gvx`**. It does not *fix* the singularity, but it **fails fast** — ABACUS errors out immediately instead of hanging, making the failure obvious and debuggable. In agent-mode runs this converts a silent freeze into a clear diagnostic.

Practical stance: prefer fail-fast (`scalapack_gvx`) during exploration/agent runs; keep `genelpa` for clean production systems after `rcut`/`lmax` are settled.

Tool note: `tools/JYLmaxRcutJointConvTestDriver.py` already runs its NAO/eval stages with **`scalapack_gvx`** → this sweep is fail-fast by construction. (The separate `ecutjy` test still uses `genelpa` — see `orbgen-converge-ecutjy`.)

## Steps

1. **Sanity-check `celldm`:** the generator raises if `celldm` is too small for the largest bond length + rcut, and warns/resets when it can be pruned to save resources.
2. **Generate the sweep:**
   ```
   python3 tools/JYLmaxRcutJointConvTestGenerator.py
   ```
   Configure in its `__main__`/call: `elem`, `ecutjy` (fixed), `r` (e.g. `[6..12]`), `l` (e.g. `[2,3,4]`), `ecutgrid` (≥ ecutjy+10), `celldm`, `nscf` (True), `fpsp`, `orbgen`. It writes a `driver.json` per bond length under `JYLmaxRcutJointConvTest-{elem}/{elem}-{bl}/`.
3. **Run the jobs:** each bond-length folder runs
   `python3 JYLmaxRcutJointConvTestDriver.py -i driver.json`
   executing the PW reference and each lcao evaluation. Locally set `abacustest='__local__'` to generate only and run manually; otherwise it submits to Bohrium via `abacustest -i registry.dp.tech/dptech/abacus:3.8.1 -f <bldir> -c "python3 JYLmaxRcutJointConvTestDriver.py -i driver.json"`.
4. **Post-process / plot:**
   ```
   python3 tools/JYLmaxRcutJointConvTestReader.py
   ```
   (or call `main(target, walk, unit='kcal/mol', logscale=True)`). It parses `running_scf.log` total energies for PW and each orb, plots per-bond-length panels plus an averaged panel, and saves `JYLmaxRcutJointConvTest.png`.
5. **Recommend** the smallest `rcut`/`lmax` combo whose (averaged) relative total-energy error meets the acceptance criterion, and return it to the orchestrator.

## Returning to the caller

Feed the chosen `rcut`(s) into `bessel_nao_rcut`, and the chosen max angular momentum into `geoms[].lmaxmax` (make sure the orbitals never request an l larger than `lmaxmax`).