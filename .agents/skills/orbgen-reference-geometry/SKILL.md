---
name: "orbgen-reference-geometry"
description: "Build the reference geometries (geoms) of an ORBGEN input JSON (proto, pertkind, pertmags, lmaxmax, nbands, nspin). Invoke when composing or templating the geoms section that defines which reference structures / bond lengths are used for fitting."
---

# orbgen-reference-geometry — the `geoms` block

Composes the `geoms` section: the set of reference structures and their perturbations on which the DFT reference data is computed and the orbital is fit.

## Block shape

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

Compulsory keys per geom (`SIAB/io/param.py#GEOM_COMPULSORY_`): `proto`, `pertkind`, `pertmags`, `lmaxmax`. `nbands`, `nspin`, `celldm` are optional per-geom DFT knobs.

## `proto` — pick `dimer`

`dimer`, `trimer`, `square`, `tetrahedron`, `octahedron`, `cube`, or a structure file path (any custom file whose path is passed). Validated in `SIAB/io/param.py#GeomAssert`.

> **Recommendation: use only `dimer`.** More/bigger protos are meant to improve transferability, but with too few reference samples the added structures produce outliers that pollute the fitted orbital quality. `dimer` is the safe, supported default. (`DEFAULT_BOND_LENGTH` in `SIAB/abacus/io.py` only covers `dimer`/`trimer` anyway.)

## `pertkind` — stretch only

`pertkind` is the **perturbation type**. Only `stretch` is implemented; `shear`/`twist` are reserved but raise `NotImplementedError` (`SIAB/io/param.py`, `SIAB/abacus/api.py#_build_pert`). Defaults to `stretch` if omitted — you can rely on it.

## `pertmags` — bond lengths, manual list or `auto`

`pertmags` is the **perturbation magnitude**. For a `dimer` (stretch) that literally means the **bond length(s)** in Bohr.
- A **list of int/float**: bond lengths in Bohr (e.g. `[1.75, 2.0, 2.25, 2.75, 3.75]`).
- **`"auto"`**: expand to a sensible bond-length series. Resolution order (`SIAB/abacus/run.py#_build_abacus`):
  1. look up the element in `DEFAULT_BOND_LENGTH[proto]` (`SIAB/abacus/io.py`);
  2. if absent, fall back to a **bond-length scan** (Morse fit plus a 1.5 meV energy filter, `SIAB/abacus/blscan.py`).

Bonus: the lookup table already ships curated per-element bond lengths for most elements — prefer `auto` when unsure of the bond length.

## `lmaxmax` & per-geom DFT knobs

- `lmaxmax` (non-negative int, or dev-string `=N`): max angular momentum of the basis — must be ≥ any orbital's l.
- `nbands` (positive int): number of states included in the spillage.
- `nspin`: spin polarization. Setting it enables open-shell wavefunctions, but there is **no observed need for it** — keep `nspin: 1` (closed shell).
- `celldm` (positive, default `1.0`): lattice constant scale.

## Avoiding singular overlap per bond length

The `rcut` × `lmax` combination can make the overlap matrix **singular for some bond lengths** (the DFT/SCF then stalls or fails to converge — see the `scalapack_gvx` fail-fast note in `orbgen-converge-rcutlmax`). Rule of thumb: **stop the calculation at that geometry and drop that bond length from `pertmags`.**

This only works if `pertmags` is an explicit list of numbers. To recover the exact values being tested, either:
- read the per-element defaults out of `DEFAULT_BOND_LENGTH` in `SIAB/abacus/io.py`, or
- list the generated job directory (one subfolder per bond length) and see which ones actually ran / failed.

Then resubmit with the offending bond length removed.

When in doubt about bond lengths, just use `"pertmags": "auto"`.

## Output

Return the `geoms` JSON fragment to the orchestrator so it can be merged with the primitive basis (see `orbgen-primitive`) and the `orbitals` block (see `orbgen-optimize`) before running `orbgen`.