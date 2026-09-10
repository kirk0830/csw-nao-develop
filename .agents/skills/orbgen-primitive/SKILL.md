---
name: "orbgen-primitive"
description: "Build the primitive-basis part of an ORBGEN input JSON (fit_basis, ecut, bessel_nao_rcut, primitive_type). Invoke when composing or templating an ORBGEN input that needs primitive basis settings. Reference geometries are handled by orbgen-reference-geometry."
---

# orbgen-primitive — primitive basis

Composes the primitive-basis settings of the ORBGEN input (the DFT reference structure geometries live in `orbgen-reference-geometry`).

## Primitive basis block

```json
{
  "fit_basis": "jy",
  "ecutwfc": 60,
  "ecutjy": 40,
  "bessel_nao_rcut": [10],
  "primitive_type": "reduced"
}
```

- `fit_basis`: `jy` (contracted NSW, default) or `pw` (plane-wave reference, ~ PTG-DPSI/LRH). Ask the user if unsure; `jy` is the safe default.
- `ecutwfc`: plane-wave / realspace-grid convergence needed by the *pseudopotential*; make it large enough that the pseudo (via ABACUS) is converged.
- `ecutjy`: kinetic-energy cutoff of the underlying NSW/jy spherical-Bessel expansion (defaults to `ecutwfc` if omitted).
- `bessel_nao_rcut`: list of int truncation radii, in Bohr. `orbgen` will loop the whole workflow for each rcut. Picked by `orbgen-converge-rcutlmax`; otherwise take the user's value.
- `primitive_type`: keep `"reduced"` for general use.

### Sizing `ecutwfc` / `ecutjy` — test or pick

`ecutwfc` and `ecutjy` are hyperparameters to fix **ahead of** the orbital run (see `tools/README.md`).

- `ecutwfc`/grid: the *pseudopotential*'s own grid/plane-wave convergence. Ask the user whether it has been tested; if not, run a PW/ecut convergence check.
- `ecutjy`: kinetic-energy cutoff of the NSW/jy expansion. Run the band-structure convergence test in **`orbgen-converge-ecutjy`** (eta criterion, `tools/JYEkinConvTest*`); the project reference answer is `ecutjy=60` for ~1 kcal/mol chemical accuracy.
- `lmax`/`rcut`: benchmark against a PW reference via **`orbgen-converge-rcutlmax`** (`tools/JYLmaxRcutJointConvTest*`).

**Ask the user** whether they want to run these tests or just pick values on the spot ("拍脑袋"). Quick-pick heuristics to offer:

- Recommended quick rule: **`ecutwfc = ecutjy + 50 Ry`** (empirically derived from grid-integration convergence tests).
- Historical (v2.0-era) default: `ecutjy == ecutwfc`, both taken blindly as **`100`**. Recorded as project lore — usable as a cheap starting point, not a recommendation to prefer over the tested values above.

## Reference geometries

The `geoms` block (which reference structures, bond lengths, `nspin`, `lmaxmax`, etc.) is **handled by `orbgen-reference-geometry`**, not here. See:
- [`orbgen-reference-geometry`](orbgen-reference-geometry/SKILL.md) — `proto`/`pertkind`/`pertmags`/`lmaxmax`/`nbands`/`nspin`, including the bond-length `auto` defaults and the singular-overlap-per-bond-length failure mode.

## Output

Return the primitive-basis JSON fragment to the orchestrator so it can be merged with the `geoms` (see `orbgen-reference-geometry`) and the `orbitals` block (see `orbgen-optimize`) before running `orbgen`.