---
name: "orbgen-optimize"
description: "Build and run the spillage-optimization and contraction (orbitals) block of an ORBGEN input: model/model_kwargs, greedygrow/nzeta_max, checkpoint cascade, optimizer. Invoke when determining the final atomic orbitals block or executing the contraction."
---

# orbgen-optimize — spillage optimization & contraction

Composes and runs the `orbitals` block, then executes `orbgen`. This is where the actual NAO contraction happens.

## Basic contraction (checkpoint cascade)

```json
{
  "orbitals": [
    { "nzeta": [1, 1, 0], "geoms": [0], "nbands": "occ", "checkpoint": null },
    { "nzeta": [2, 2, 1], "geoms": [0], "nbands": "occ", "checkpoint": 0 }
  ]
}
```

- First contraction has `checkpoint: null`. Later ones set `checkpoint` to the index of the smaller basis they build on.
- `nzeta`: number of zeta per angular momentum.
- `geoms`: **a list of int**, indices into the `geoms` block — each `geoms` entry (a single dict) is one reference structure/perturbation, and this int says which ones this orbital draws its reference DFT wavefunctions from. Pass `[0]` for one geometry or `[0, 1]` for several (not a bare int).
- `nbands`: `occ`, `all`, or an int ≤ the geom's `nbands`.
- The max l in any `nzeta` must be ≤ the geom's `lmaxmax`.

## Initialization: `model` / `model_kwargs`

Each orbital's initial guess comes from a model. The top-level `spill_guess` sets the global default; an orbital can override with `model` + `model_kwargs`:

```json
{
  "spill_guess": "atomic",
  "orbitals": [
    { "nzeta": [2, 2, 1], "geoms": [0], "nbands": "occ", "checkpoint": 0,
      "model": "hydrogen", "model_kwargs": { "slater": true } }
  ]
}
```

Supported models (see README for full notes): `ones`, `random` (needs `seed`), `atomic` (`jobdir` required; optional `vloc_aux`, `lloc_min`), `hydrogen` (`slater`, `otherelem`), `pretrained` (`pretrained` → an existing `.orb`). `model_kwargs` are filtered automatically to the keys valid for the chosen model.

## Automatic growth over zeta: `greedygrow` + `nzeta_max` (experimental)

**Division of labour:** the convergence test (`orbgen-converge-rcutlmax`) fixes `rcut`/`lmax`; `orbgen-converge-ecutjy` fixes `ecutjy`. `greedygrow` acts only on `nzeta` (per-l zeta counts). Prefer specifying `nzeta` explicitly (the checkpoint cascade above); treat `greedygrow` as experimental.

If explicitly requested, `greedygrow: true` adds zeta functions greedily until spillage stops improving:

```json
{
  "orbitals": [
    { "nzeta": [1, 1, 0], "geoms": [0], "nbands": "occ", "checkpoint": null,
      "greedygrow": true, "nzeta_max": [3, 3, 3] }
  ]
}
```

`nzeta_max` must be element-wise ≥ `nzeta`.

> **Caveat:** `greedygrow` is hidden/experimental — not part of the validated input schema, and reliability is limited because it repeatedly re-runs non-convex spillage optimizations and picks a single noisy greedily-best l. Do not offer it as the default automation path; use explicit `nzeta` (checkpoint cascade) as the default, unless the user explicitly asks for greedy growth.

## Other per-orbital options

- `filename`: custom output `.orb` name.
- `fix_components`: nested list of contraction coefficients to keep frozen.
- `nzeta` as a string for automatic zeta counts, e.g. `"auto:twsvd:0.8:max"` (methods `twsvd` / `amwsvd`, optional `:max|:mean`).

## Top-level optimizer options

```json
{
  "optimizer": "scipy.bfgs",
  "max_steps": 9000,
  "nthreads_rcut": 4,
  "torch.lr": 0.001
}
```

Prefer `scipy.bfgs` (has an analytical spillage gradient). Keys prefixed `scipy.` / `torch.` are forwarded to the optimizer.

## Run

After assembling the full JSON, verify the compulsory keys ({`abacus_command`, `pseudo_dir`, `element`, `bessel_nao_rcut`, `geoms`, `orbitals`}) and run:

```bash
orbgen -i <input.json> -o <outdir>
```

(Make sure the ABACUS runtime was confirmed with the user first.)