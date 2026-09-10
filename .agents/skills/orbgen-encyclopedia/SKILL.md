---
name: "orbgen-encyclopedia"
description: "Reference for the concepts that drive ORBGEN inputs (single/double/triple zeta, polarization function, the nzeta array, valence-electron layers). Invoke to explain or decide basis-set level / nzeta."
---

# orbgen-encyclopedia — basis-set concepts

A no-code reference that teaches the vocabulary used everywhere else in this skill series. Read this before interpreting or writing `nzeta` so that "minimal", "polarized", "double zeta" etc. are grounded in something concrete.

## What is a "zeta"?

A **zeta** is one basis function for a given angular momentum. The total basis is described by how many zeta functions you give to each angular momentum channel:

| l | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| name | s | p | d | f |

In an ORBGEN orbital, `nzeta` is a list indexed by l, e.g. `[n_s, n_p, n_d, n_f, ...]`:

- `nzeta: [2, 2, 1]` = 2s + 2p + 1d = 5 zeta functions total.
- `nzeta: [1, 1, 0]` = 1s + 1p + 0d = 2 zeta functions.

## Single, double, triple zeta

- **Single zeta (SZ)** — one function per angular momentum. Also written 1s1p, 2s2p, etc.
- **Double zeta (DZ)** — two functions per l (a tight and a diffuse one for the same l). 2s2p…
- **Triple zeta (TZ)**, quadruple zeta (QZ) — 3 (4) functions per l.

### Single zeta comes from the pseudopotential

The SZ count is not arbitrary: for each angular momentum, SZ uses exactly one function per **occupied valence shell of that l**. You read it from the **valence electron layers** recorded in the pseudopotential file.

Example (Si): the pseudopotential keeps only the valence electrons `3s² 3p²`. There is one occupied s shell (3s) and one occupied p shell (3p), so

```
nzeta = [1, 1, 0]     # 1s + 1p, and 0 d orbitals
```

This is the minimal basis for Si. General rule per l: **number of occupied valence shells of that l ⇨ SZ nzeta for that l**.

Careful: the *valence* configuration in the pseudo is usually a subset of the full atomic configuration (e.g. Si is `[Ne] 3s² 3p²`). Only the pseudo's valence shells count. The repository stores full ground-state configurations in `SIAB/data/build.py` (`AtomSpecies.ground_state_atomic_electronic_configuration`) and derives the valence shells from the pseudo's `zval` — useful cross-checks when a pseudo is ambiguous.

## Polarization functions

A **polarization function** adds a higher-`l` channel that is empty in the ground state (e.g. a d on a 1s1p basis). It lets the atomic orbitals respond to the asymmetric environment of a molecule/solid. Convention:

- SZP / SZ(d…) — single zeta + a polarization d: `[1, 1, 1]`.
- DZP — double zeta + polarization, e.g. `[2, 2, 1]`.
- SV / SV(d…) — "split valence" (usually = SZ/SZP in this context).

## Practical tiers (used by the `orbgen` entry skill)

| Tier | nzeta | meaning |
|------|-------|---------|
| `minimal` | e.g. `[1, 1, 0]` | SZ from the pseudo's valence layers |
| `polarized` (default) | e.g. `[2, 2, 1]` | DZ + 1 d polarization |

Higher zeta (more diffuse functions per l) raises accuracy and cost; `lmaxmax` in the geoms block must be ≥ the highest l used in `nzeta`.

## How to use this skill

- To **decide** `nzeta`: read the pseudo's valence layers (per l) for the SZ baseline, then add polarization/diffuse functions per the desired tier.
- To **explain** an input or ORBGEN output: translate `nzeta` back into 1s1p, 2s2p1d, etc. in plain terms.