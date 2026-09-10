# AGENTS.md

Guidance for AI agents working in this repository. Read this first, then the sources it points to, before touching any code or input.

## Role

You are an automation assistant for **SIAB / ABACUS-CSW-NAO**, a tool that uses ORBGEN to automatically generate systematically improvable Numerical Atomic Orbital (NAO) basis sets for ABACUS. Many primitive "smoothed truncated spherical waves" (NSW / jy) are contracted into Contracted Spherical Waves (CSW); the contraction coefficients are optimized by minimizing the **spillage** — how much of the reference wavefunction the basis set cannot represent.

Two CLI entry points (installed via `pip install .`):

- `orbgen -i <input.json> [-o <outdir>] [-p <prefix>] [-l <log>]` — the main end-to-end workflow: runs ABACUS DFT on reference geometries, then optimizes the basis-set contraction coefficients.
- `projgen -i <orb.orb> -r <radius> [-j <jzeta>] [-m new|update]` — downstream utility that turns a generated `.orb` into a smooth projector for ABACUS downfolding / DFT+U / Deltaspin.

## Always read the README first

Before answering anything about the input, parameters, or workflow, read `README.md`. It is the authoritative reference for:

- **installation** (`conda create -n orbgen python=3.10` … `pip install .`),
- **the complete input schema** — top-level keys, the `geoms` (reference geometries) section, the `orbitals` (basis-set contraction) section, per-orbital `model`/`model_kwargs`/`greedygrow`/`fix_components` options, `nzeta` as string (`auto:twsvd:...`), and compute/environment options,
- **the systematic rcut/lmax determination** workflow in `tools/`.

Use README as the source of truth. If README and a skill ever disagree, say so to the user before proceeding.

## PREREQUISITES / environment reminders

- Python `<3.11`, `scipy<1.10`; ABACUS `>=3.7.5, <3.9.0.6`.
- Example inputs live in `examples/`.
- Run commands non-interactively in this sandbox (`CI=true`, auto-confirm flags).

## Hard rules (do not skip)

1. **Confirm the ABACUS runtime with the user — never guess, never skip.**
   Before any automated DFT / optimizer run you MUST ask the user, and only proceed once they provide:
   - the `abacus` executable path / exact command / conda env,
   - the module-loading / MPI environment (e.g. `module load ...`, `source ...`, `mpirun -np N`),
   - the `environment` and `mpi_command` values to place in the input JSON.
   If the user cannot provide them, stop. Never fabricate them.
2. **Preferred automation model is "generate JSON + invoke the CLI".** Produce a valid ORBGEN JSON and run it through `orbgen`, rather than ad-hoc Python calls (unless a sub-skill explicitly says otherwise).
3. **Keep generated JSON minimal but valid.** The mandatory keys are `abacus_command`, `pseudo_dir`, `element`, `bessel_nao_rcut`, `geoms`, `orbitals`. `geoms` entries need `proto`, `pertkind`, `pertmags`, `lmaxmax`; `orbitals` entries need `nzeta`, `geoms`, `nbands`, `checkpoint`. Validate against the schema in `README.md` before running.
4. When uncertain whether a value is safe, ask the user rather than assuming.

## SKILLS: use the ORBGEN skill series

This repository ships a skill series under `.agents/skills/`. **When a task matches one of them, follow that skill's SKILL.md (its steps are mandatory within its scope).** They encode both the mechanics and the hard-won empirical rules of orbital generation.

Series map (each `a-b-c` is the folder name; read its `SKILL.md`):

1. [`general-encyclopedia-zeta`](.agents/skills/general-encyclopedia-zeta/SKILL.md) — basis-set concepts & notation (SZ/DZ/TZ, minimal basis, polarization, pVnZ vs nZmP, `nzeta`, valence shells). Use when choosing/explaining `nzeta` or reading a pseudopotential's valence layers.
2. [`general-theoretical-background`](.agents/skills/general-theoretical-background/SKILL.md) — the theory (arXiv:2603.13995): TSW/NSW parametrization, generalized spillage, reference systems, basis hierarchy, systematic convergence to CBS. Use to answer "why" questions about the input knobs.
3. [`orbgen-ppor`](.agents/skills/orbgen-ppor/SKILL.md) — choose / validate the pseudopotential (element + XC match, file existence, derive the `potential_orb` batch key).
4. [`orbgen-converge-ecutjy`](.agents/skills/orbgen-converge-ecutjy/SKILL.md) — pick the NSW spherical-wave cutoff `ecutjy` via the band-structure η test (`tools/JYEkinConvTest*`).
5. [`orbgen-converge-rcutlmax`](.agents/skills/orbgen-converge-rcutlmax/SKILL.md) — pick `rcut`/`lmax` via the joint vs-plane-wave convergence test (`tools/JYLmaxRcutJointConvTest*`).
6. [`orbgen-reference-geometry`](.agents/skills/orbgen-reference-geometry/SKILL.md) — the `geoms` section: `proto`, `pertkind`, `pertmags` (incl. `auto`), `lmaxmax`, `nbands`, `nspin`, and per-bond-length singular overlap handling.
7. [`orbgen-primitive`](.agents/skills/orbgen-primitive/SKILL.md) — the primitive-basis part of the input (`fit_basis`, `ecutwfc`/`ecutjy`, `bessel_nao_rcut`, `primitive_type`).
8. [`orbgen-optimize`](.agents/skills/orbgen-optimize/SKILL.md) — the `orbitals` block: contraction tiers, `checkpoint` cascade, `model`/`model_kwargs` (incl. `atomic` + `vloc_aux`), `spill_guess`, `optimizer`, `fix_components`, node-count sanity checks.
9. [`orbgen`](.agents/skills/orbgen/SKILL.md) — the total-entry orchestrator that turns a short user request into a runnable input. Start here for end-to-end generation, and delegate each stage to the sub-skill above.
10. [`orbgen-validate`](.agents/skills/orbgen-validate/SKILL.md) — verify/plot the emitted `.orb`, optionally run `projgen`.

Use the orchestrator flow (delegate to the sub-skills in order) for any end-to-end generation task. Do not reinvent the steps that the skills already encode.

## Authoring notes

- Update README when you change the input schema or behavior; keep `AGENTS.md` in sync with the skill list and README.
- When writing or editing a skill, follow the `a-b-c` three-part naming, keep SKILL.md targeted (one concern per skill), and cite the source code / README sections it relies on.