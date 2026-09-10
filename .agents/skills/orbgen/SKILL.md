---
name: "orbgen"
description: "End-to-end numerical atomic orbital (NAO) basis generation for ABACUS via ORBGEN. Invoke when the user asks to generate a basis set / orbital(s) for an element or to templating an ORBGEN run."
---

# orbgen — End-to-end NAO basis generation

Total-entry orchestrator of the ORBGEN skill series. It turns a short user request into a complete, runnable ORBGEN input and executes it, delegating each stage to a sub-skill. It favors sensible defaults for ordinary users while exposing every advanced option for power users.

## Skill series map

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
3. **computational budget / target quality** — one of:
   - `minimal` (e.g. 1s1p),
   - `polarized` (e.g. 2s2p1d — recommended default),
   - `automatic` (use `greedygrow` + `nzeta_max` so the basis grows to convergence).
4. **rcut / lmax** — if the user has no preference, run `orbgen-converge`; otherwise accept their values or defaults.

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
| growth | manual contractions | `greedygrow` + `nzeta_max` |
| primitive type | `reduced` | `primitive_type` |

For "automatic" quality, build a single orbital with `greedygrow: true` and `nzeta_max` set to the user's upper bound; `orbgen-optimize` will run the greedy algorithm to convergence.

Always confirm the generated JSON against the scheme in `README.md` (compulsory keys listed there) before running `orbgen`.