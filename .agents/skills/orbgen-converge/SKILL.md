---
name: "orbgen-converge"
description: "Run and post-process the rcut/lmax convergence test for ORBGEN primitive basis sets, to pick truncation radius and maximal angular momentum. Invoke when truncation parameters are unknown."
---

# orbgen-converge — rcut / lmax convergence study

`rcut` (truncation radius) and `lmax` (maximal angular momentum) control the *completeness* of the primitive basis. This sub-skill determines them by varying both and watching the **relative total-energy error of the primitive (CSW/NAO) basis vs. a plane-wave reference**. It also fixes `lmax` for the reference geometries.

## Acceptance criterion

**The primitive basis is deemed converged when its total energy lies within a target of the plane-wave reference.** The project convention (see `tools/README.md`) targets **~1 kcal/mol "chemical accuracy"** — the published CSW-NAO defaults satisfying this are `ecutjy=60`, `lmax=3`, `rcut=8`. Use the user's target if they have one; otherwise 1 kcal/mol is the default.

Method background: the contraction **minimizes the trace of the kinetic operator in the residual space**, generalizing the spillage-minimizing scheme [M. Chen et al., J. Phys. Condens. Matter 22, 445501 (2010); P. Lin et al., Phys. Rev. B 103, 235131 (2021)]. The CSW implementation is described in the repo's paper, **arXiv:2603.13995** ("Systematically Improvable NAO Basis Using Contracted Truncated Spherical Waves"); using contracted truncated spherical waves (instead of plane waves) as the expansion basis bridges reference states and NAOs more effectively and removes spurious periodic-image interactions, improving transferability.

## Risks of over-large `rcut` / `lmax`

Bigger is *generally* more complete, but there are real failure modes an agent must watch for:

- **Near-singular overlap matrix.** Very diffuse functions from large `rcut`, or too many/high-`l` functions, can produce an overlap matrix that is (nearly) singular.
- **`ks_solver: genelpa` (default) silently hangs.** With a singular overlap, the SCF appears to *freeze — no output, no error*. If a run shows no progress/output, suspect this before anything else.
  - **Detection:** check the solver output for stalls; if `genelpa` is set and output stops, treat as a possible singular-overlap hang.
  - **Warning the user:** when `rcut`/`lmax` are pushed large, proactively warn that a singular overlap could hang SCF.
- **Fail-fast mitigation:** switch to **`ks_solver: scalapack_gvx`**. It does not *fix* the singularity, but it **fails fast** — ABACUS errors out immediately instead of hanging, making the failure obvious and debuggable. In agent-mode runs this converts a silent freeze into a clear diagnostic.

Practical stance: prefer fail-fast (`scalapack_gvx`) during exploration/agent runs; keep `genelpa` for clean production systems after `rcut`/`lmax` are settled.

## Steps

1. **Generate the sweep** with the generator workflow:
   `tools/JYLmaxRcutJointConvTestGenerator.py`
   It writes a series of primitive-basis ORBGEN input scripts across a grid of `rcut` and `lmax`.
2. **Run** each generated input with `orbgen -i <input>.json` (after confirming the ABACUS runtime per the top-level rule).
3. **Post-process / plot** the results with:
   `tools/JYLmaxRcutJointConvTestReader.py`
4. **Recommend** the smallest `rcut`/`lmax` combo whose relative total-energy error meets the acceptance criterion (balancing accuracy vs. cost), and return it to the orchestrator.

## Returning to the caller

Feed the chosen `rcut`(s) into `bessel_nao_rcut`, and the chosen max angular momentum into `geoms[].lmaxmax` (make sure the orbitals never request an l larger than `lmaxmax`).