---
name: "orbgen-converge"
description: "Run and post-process the rcut/lmax convergence test for ORBGEN primitive basis sets, to pick truncation radius and maximal angular momentum. Invoke when truncation parameters are unknown."
---

# orbgen-converge — rcut / lmax convergence study

`rcut` (truncation radius) and `lmax` (maximal angular momentum) control the completeness of the basis. This sub-skill determines them by varying both and watching the relative total-energy error of a test system vs. a plane-wave reference.

## Steps

1. **Generate the sweep** with the generator workflow:
   `tools/JYLmaxRcutJointConvTestGenerator.py`
   It writes a series of primitive-basis ORBGEN input scripts across a grid of `rcut` and `lmax`.
2. **Run** each generated input with `orbgen -i <input>.json` (after confirming the ABACUS runtime per the top-level rule).
3. **Post-process / plot** the results with:
   `tools/JYLmaxRcutJointConvTestReader.py`
4. **Recommend** the smallest `rcut`/`lmax` combo that yields an acceptably converged relative total-energy error, balancing accuracy vs. cost. Return these values to the orchestrator.

## Returning to the caller

Feed the chosen `rcut`(s) into `bessel_nao_rcut`, and the chosen max angular momentum into `geoms[].lmaxmax` (make sure the orbitals never request an l larger than `lmaxmax`).