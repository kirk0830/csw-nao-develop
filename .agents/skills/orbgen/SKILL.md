---
name: "orbgen"
description: "End-to-end numerical atomic orbital (NAO) basis generation for ABACUS via ORBGEN. Invoke when the user asks to generate a basis set / orbital(s) for an element or to templating an ORBGEN run."
---

# orbgen — End-to-end NAO basis generation

Total-entry orchestrator of the ORBGEN skill series. It turns a short user request into a complete, runnable ORBGEN input and executes it, delegating each stage to a sub-skill. It favors sensible defaults for ordinary users while exposing every advanced option for power users.

## Skill series map

- [`general-encyclopedia-zeta`](general-encyclopedia-zeta/SKILL.md) — teaches basis-set concepts & notations (SZ/DZ/TZ, minimal basis, polarization, pVnZ vs nZmP, `nzeta`, valence layers). Read it when explaining or choosing `nzeta`.
- [`orbgen-ppor`](orbgen-ppor/SKILL.md) — choose / validate the pseudopotential.
- [`orbgen-converge`](orbgen-converge/SKILL.md) — pick `rcut`/`lmax` via a convergence test.
- [`orbgen-primitive`](orbgen-primitive/SKILL.md) — reference geometries + primitive basis.
- [`orbgen-optimize`](orbgen-optimize/SKILL.md) — spillage optimization & contraction.
- [`orbgen-validate`](orbgen-validate/SKILL.md) — check & plot outputs, optional `projgen`.

## MANDATORY first step

Before doing anything, follow the hard rule in `AGENTS.md`:

**Ask the user for the ABACUS runtime and do not proceed without it.** You need:
- the `abacus` executable path / command,
- the module-loading / environment (module load, conda env, etc.),
- the `environment` and `mpi_command` values for the JSON.
If the user can't provide these, stop. Never fabricate them.

## Gather requirements

Collect (ask if not given):
1. **element** (e.g. `Si`).
2. **pseudo_dir** — path to the matching pseudopotential (delegate to `orbgen-ppor` to confirm the match).
3. **computational budget / target quality** — one of the explicit contraction tiers:
   - `minimal` (e.g. `nzeta: [1, 1, 0]` = 1s1p),
   - `polarized` (e.g. `nzeta: [2, 2, 1]` = 2s2p1d — recommended default).
   These tiers set an explicit `nzeta` per angular momentum; see `orbgen-optimize`.

> The `minimal` tier is NOT guessed: derive it by reading the pseudopotential's valence shells (the agent parses the pseudo; see `general-encyclopedia-zeta`) and confirm it with the user.
4. **rcut / lmax** — if the user has no preference, run `orbgen-converge` to determine them; otherwise accept their values or defaults.

> Do **not** default to automatic nzeta growth (`greedygrow`). It is an experimental/hidden option and empirical experience finds it unreliable (repeated non-convex spillage optimizations, uneven spillage surface). Set `rcut`/`lmax` from the convergence test and specify `nzeta` explicitly.

## Run the orchestrator flow

1. Confirm ABACUS runtime (mandatory, above).
2. `orbgen-ppor` — resolve and validate `pseudo_dir`.
3. `orbgen-converge` — if requested or if rcut/lmax are unknown, determine them.
4. `orbgen-primitive` — build the `geoms` + primitive basis part of the input.
5. `orbgen-optimize` — build the `orbitals` block (default contraction scheme + optional advanced options) and run `orbgen`.
6. `orbgen-validate` — verify the emitted `.orb` file, plot it, and (optionally) generate a projector with `projgen`.

## Defaults vs. advanced override

Use these defaults unless the user asks for something else:

| Option | Default | Advanced key |
|--------|---------|--------------|
| optimizer | `scipy.bfgs` | `optimizer`, `scipy.*` / `torch.*` keys |
| max steps | `9000` | `max_steps` |
| initialization | `atomic` | `spill_guess` / per-orbital `model` + `model_kwargs` |
| growth | explicit `nzeta` per l (checkpoint cascade) | `greedygrow` + `nzeta_max` (experimental, not recommended) |
| primitive type | `reduced` | `primitive_type` |

Use explicit `nzeta` values via a checkpoint cascade (`orbgen-optimize`); `greedygrow` is a hidden/experimental option and should not be a default (see caveat above).

Always confirm the generated JSON against the scheme in `README.md` (compulsory keys listed there) before running `orbgen`.