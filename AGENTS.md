# AGENTS.md

Guidance for AI agents working in this repository.

## What this project is

`SIAB` (ABACUS-CSW-NAO) automatically generates systematically improvable Numerical Atomic Orbital (NAO) basis sets for ABACUS. The basis functions are Contracted Spherical Waves (CSW): many primitive "smoothed truncated spherical waves" (NSW / jy) are contracted so that the spillage — how much of the reference wavefunction the basis set cannot represent — is minimized.

The two command-line entry points (installed via `pip install .`):

- `orbgen -i <input.json> [-o <outdir>] [-p <prefix>] [-l <log>]` — the main end-to-end workflow. It first runs ABACUS DFT on reference geometries, then optimizes the basis-set contraction coefficients.
- `projgen -i <orb.orb> -r <radius> [-j <jzeta>] [-m new|update]` — downstream utility that turns a generated `.orb` into a smooth projector for ABACUS downfolding / DFT+U / Deltaspin.

Requirements worth remembering: Python `<3.11`, `scipy<1.10`; ABACUS version `>=3.7.5, <3.9.0.6`. Example inputs live in `examples/`. A full parameter reference is in `README.md`.

## Hard rules (do not skip)

1. **Confirm the ABACUS runtime with the user — never guess, never skip.**
   Before any automated DFT/optimization run you MUST ask the user, and only proceed once they provide:
   - the path to the `abacus` executable (or the exact command / conda env needed),
   - the module-loading / environment setup required (e.g. `module load ...`, `source ...`, `mpirun -np N`),
   - the `environment` and `mpi_command` values to place in the input JSON.
   If the user cannot provide them, stop and do not fabricate values.
2. **Preferred automation model is "generate JSON + invoke CLI".** Skills produce a valid ORBGEN JSON input and run it through `orbgen`. Prefer readable, reproducible, commit-able JSON over ad-hoc Python calls, unless a sub-skill explicitly says otherwise.
3. **Skills live in `.agents/skills/`** as `SKILL.md` files. Follow their mandatory steps when invoked.
4. **Keep generated JSON minimal but valid.** Compulsory keys: `abacus_command`, `pseudo_dir`, `element`, `bessel_nao_rcut`, `geoms`, `orbitals`. `geoms` entries need `proto`, `pertkind`, `pertmags`, `lmaxmax`; `orbitals` entries need `nzeta`, `geoms`, `nbands`, `checkpoint`. Verify the JSON with the scheme documented in `README.md` before running.

## Authoring notes

- Run commands non-interactively (`CI=true`, auto-confirm flags) in this sandbox.
- When uncertain whether a value is safe, ask the user rather than assuming.