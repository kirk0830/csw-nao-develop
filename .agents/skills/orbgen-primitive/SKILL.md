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
- `ecutwfc`: grid-integration precision of the reference DFT; higher = more accurate but costlier.
- `ecutjy`: kinetic-energy cutoff of the underlying NSW/jy (defaults to `ecutwfc` if omitted).
- `bessel_nao_rcut`: list of int truncation radii, in Bohr. `orbgen` will loop the whole workflow for each rcut. `orbgen-converge` can pick it; otherwise take the user's value.
- `primitive_type`: keep `"reduced"` for general use.

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