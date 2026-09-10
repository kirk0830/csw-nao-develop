---
name: "orbgen-primitive"
description: "Build the reference-geometry and primitive-basis part of an ORBGEN input JSON (fit_basis, ecut, bessel_nao_rcut, primitive_type, geoms). Invoke when composing or templating an ORBGEN input that needs reference structures."
---

# orbgen-primitive — reference geometries + primitive basis

Composes the DFT reference data and the primitive basis settings of the ORBGEN input.

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
- `ecutjy`: kinetic-energy cutoff of the NSW/jy expansion. Run the band-structure convergence test in **`orbgen-converge-ecutjy`** (η criterion, `tools/JYEkinConvTest*`); the project reference answer is `ecutjy=60` for ~1 kcal/mol chemical accuracy.
- `lmax`/`rcut`: benchmark against a PW reference via **`orbgen-converge-rcutlmax`** (`tools/JYLmaxRcutJointConvTest*`).

**Ask the user** whether they want to run these tests or just pick values on the spot ("拍脑袋"). Quick-pick heuristics to offer:

- Recommended quick rule: **`ecutwfc = ecutjy + 50 Ry`** (empirically derived from grid-integration convergence tests).
- Historical (v2.0-era) default: `ecutjy == ecutwfc`, both taken blindly as **`100`**. Recorded as project lore — usable as a cheap starting point, not a recommendation to prefer over the tested values above.

## Reference geometries block

```json
{
  "geoms": [
    {
      "proto": "dimer",
      "pertkind": "stretch",
      "pertmags": [1.62, 1.82, 2.22, 2.72, 3.22],
      "nbands": 20,
      "nspin": 1,
      "lmaxmax": 2
    }
  ]
}
```

- `proto`: `dimer`, `trimer`, `square`, `tetrahedron`, `octahedron`, `cube`, or a structure file path.
- `pertkind`: `stretch` only is currently supported.
- `pertmags`: list of perturbation magnitudes (bond lengths) or the string `auto`.
- `nbands` (positive int): number of states included in the spillage.
- `nspin`: spin polarization.
- `lmaxmax`: max angular momentum of the basis — must be ≥ any orbital's l.

Include more structures (e.g. a trimer) for better transferability, but note that a trimer can hurt smoothness; dimer-only is the recommended default.

## Output

Return the merged primitive + `geoms` JSON fragment to the orchestrator so it can append the `orbitals` block before running `orbgen`.