---
name: "general-encyclopedia-zeta"
description: "Reference for basis-set zeta concepts and notations (single/double/triple zeta, minimal basis, polarization, pVnZ vs nZmP, the nzeta array, valence-electron layers). Invoke to explain or decide basis level / nzeta."
---

# general-encyclopedia-zeta — basis-set concepts & notations

A no-code reference that teaches the vocabulary used across this skill series. Read it before interpreting or writing `nzeta` so that "minimal", "single zeta", "polarized", "pVDZ", "TZDP"… are grounded in something concrete.

## What is a "zeta"?

A **zeta** is one basis function for a given angular momentum. The basis is described by how many zeta functions you give to each angular momentum channel:

| l | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| name | s | p | d | f |

In an ORBGEN orbital, `nzeta` is a list indexed by l, e.g. `[n_s, n_p, n_d, n_f, …]`:

- `nzeta: [2, 2, 1]` = 2s + 2p + 1d = 5 zeta functions total.
- `nzeta: [1, 1, 0]` = 1s + 1p + 0d = 2 zeta functions.

## Single, double, triple zeta

- **Single zeta (SZ)** — one function per angular momentum. Also written 1s1p, 2s2p, etc.
- **Double zeta (DZ)** — two functions per l (a tight and a more diffuse function for the same l). 2s2p…
- **Triple zeta (TZ)**, quadruple zeta (QZ) — 3 (4) functions per l.

### Single zeta comes from the pseudopotential — SZ is also the minimal basis

SZ, aka the **minimal basis**, is not arbitrary: for each angular momentum it uses exactly one function per **occupied valence shell of that l**, read from the **valence electron layers** recorded in the pseudopotential file.

Example (Si): the pseudopotential keeps only the valence electrons `3s² 3p²`. There is one occupied s shell (3s) and one occupied p shell (3p), so

```
nzeta = [1, 1, 0]     # 1s + 1p, and 0 d orbitals
```

This is the minimal basis for Si. General rule per l: **number of occupied valence shells of that l ⇨ SZ nzeta for that l.**

Careful: the *valence* configuration in the pseudo is usually a subset of the full atomic configuration (e.g. Si is `[Ne] 3s² 3p²`); only the pseudo's valence shells count. The repository stores full ground-state configurations in `SIAB/data/build.py` (`AtomSpecies.ground_state_atomic_electronic_configuration`) and derives valence shells from the pseudo's `zval` — useful cross-checks when a pseudo is ambiguous.

## Polarization functions

A **polarization function** adds a higher-`l` channel that is empty in the ground state (e.g. a d on a 1s1p basis). It lets the atomic orbitals respond to an asymmetric environment (bonds, crystal field). Smoke-test names: SZP / SZ(d…) = single zeta + polarization d (`[1,1,1]`); DZP = double zeta + polarization (`[2,2,1]`).

## Two notation families: pVnZ (Dunning) vs nZmP (def2-like)

The same "quality" is spelt out differently, and the difference matters for `nzeta`.

### pVnZ (Dunning cc-pVnZ) — polarization count is implicit

The polarization angular momentum climbs automatically with each "n"; no separate polarization count is spelled out:

| basis | nzeta (Si) | l finishing |
|---|---|---|
| SZ / minimal | 1s1p  `[1,1,0]` | p |
| DZP ≈ cc-pVDZ | 2s2p1d  `[2,2,1]` | d |
| ≈ cc-pVTZ | 3s3p2d1f  `[3,3,2,1]` | f |
| cc-pVQZ | 4s4p3d2f1g  `[4,4,3,2,1]` | g |

→ Each step up the correlation-consistent ladder also raises the **highest** polarization l (d → f → g) and adds one more zeta to every lower l.

### nZmP (def2-like) — polarization count is explicit

Here you write out the number of polarization functions (`mP`). By default the polarization functions accumulate at the **first** higher l, so reaching the next angular momentum takes an extra "P":

- TZDP = 3s3p2d `[3,3,2,0]` (triple zeta, **double** polarization — both polarizations are d).
- To reproduce pVTZ's `[3,3,2,1]` you need **TZDPP** = 3s3p2d1f — "add one more polarization", which lands on f.

So: **the nzeta-equivalent of pVTZ is TZDPP, not TZDP.** In the familiar def2 family this level corresponds roughly to **def2-TZVDPP** (TZDP ≈ def2-TZVDP, DPP ≈ def2-TZVDPP). The exact def2 zeta counts per l differ from CSW's because def2 compositions are not strictly valence-shell-based — treat the def2 mapping as a good qualitative anchor, not a literal `nzeta`.

### Practical tiers (used by the `orbgen` entry skill)

| Tier | nzeta | meaning |
|------|-------|---------|
| `minimal` | e.g. `[1, 1, 0]` | SZ from the pseudo's valence layers |
| `polarized` (default) | e.g. `[2, 2, 1]` | DZ (pVDZ) + a d polarization |

For a higher level, pick an explicit `nzeta` from the tables above. Always keep `lmaxmax` in the geoms block ≥ the highest l in `nzeta`.

## How to use this skill

- To **decide** `nzeta`: take the SZ baseline from the pseudo's valence layers (per l), then add polarization/diffuse functions per the desired tier/notation.
- To **explain** an input or ORBGEN output: translate `nzeta` back into 1s1p, 2s2p1d, 3s3p2d1f, … in plain terms, and name which notation family it matches (pVnZ vs nZmP).