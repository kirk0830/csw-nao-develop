---
name: "general-theoretical-background"
description: "Reference for the theory behind CSW/NSW construction, generalized spillage, reference systems, basis hierarchy, and systematic convergence to the complete basis set. Read it to answer 'why' questions behind the ORBGEN input parameters."
---

# general-theoretical-background — the theory behind CSW-NAO

A no-code reference answering the "why" questions behind the parameters this skill series touches (`ecutjy`, `ecutwfc`, `rcut`, `lmax`, `bessel_nao_rcut`, `geoms`, spillage, reference states, basis hierarchy). It is grounded in the method paper:

> **Systematically Improvable Numerical Atomic Orbital Basis Using Contracted Truncated Spherical Waves**, arXiv:2603.13995 (CSW-NAO).

For the vocabulary of *basis size* (SZ/DZ/TZ, minimal basis, pVnZ vs nZmP) see `general-encyclopedia-zeta`. This skill is about *why the method works* and *how each knob affects completeness/transferability*.

## The central object: an NAO is `radial function × spherical harmonic`

An atomic orbital is

```
φ(r) = χ_lζ(r) · Y_lm(r̂)
```

All the flexibility lives in the **radial part** `χ_lζ(r)`. Constructing a basis set = (1) choosing a parametrization of `χ`, and (2) defining an optimization problem to fix the parameters. `l` is angular momentum, `ζ` indexes multiple radial functions for the same `l`, `m` is magnetic quantum number.

## Parametrization: truncated spherical waves (TSW) and NSW

A TSW radial function is a combination of spherical Bessel functions, strictly zero beyond the cutoff radius `r_c`:

```
χ_lζ(r) = Σ_q j_l(θ_lq · r / r_c) · c_lqζ          (r ≤ r_c)
        0                                            (r > r_c)
```

where `j_l` is a spherical Bessel function, `θ_lq` is its `q`-th positive zero, `r_c` is the cutoff radius, `c_lqζ` are the contraction coefficients.

**Why this is special — spherical waves are kinetic-energy eigenstates:**

```
−∇²( j_l(kr)Y_lm(r̂) ) = k² ( j_l(kr)Y_lm(r̂) )
```

So, just like a plane-wave basis, the number of spherical waves `N_l` per angular momentum is controlled by a **kinetic-energy cutoff `E_c`**:

```
N_l = max{ q | θ_lq < r_c·√E_c }
```

**Why these are "systematically improvable":** as `r_c`, `l_max`, and `E_c` all tend to infinity, the TSW set converges to the **complete basis set (CBS)**. For a fixed `r_c`, constructing the NAO is equivalent to fixing the contraction coefficients `c` of the TSWs. → This is exactly why `bessel_nao_rcut`, `lmax`, and `ecutjy` appear in the input: they are the three completeness knobs.

**NSW (Nodeless Spherical Waves) — smoothing without a smoothing function.** TSWs have nonzero derivative at `r = r_c`. Prior works added a smoothing function, which costs an extra parameter and breaks analytic integrals. This work instead searches the **subspace of combinations whose first derivatives vanish at `r_c`**:

```
Σ_q j_l(θ_lq r/r_c) K_qλ   ,   d^m/dr^m ξ_lλ(r)|_{r=r_c} = 0, m = 1..M
```

Because the spherical Bessel equation gives `D_2q = (−2/r_c)D_1q`, suppressing the first derivative automatically suppresses the second. Taking only the first two derivatives (the code default `M = 2`) yields the set called **NSW**, with **no extra smoothing parameter and pure TSW analytic integrals preserved**. The input `primitive_type` selects how many such NSW to keep per `l`.

## Optimization: generalized spillage = kinetic-operator trace in the residual space

The original **CGH spillage** (Sánchez-Portal → Chen, Guo, He) is the square of how much reference wavefunction leaks out of the NAO space:

```
S = Σ_nk ‖ (1 − P_k) |ψ_nk^PW⟩ ‖²
```

where `|ψ^PW⟩` are reference states from high-quality plane-wave (PW) calcns, and `P_k = Σ_μν |φ_μk⟩ S^{-1}_μν(k) ⟨φ_νk|` projects onto the NAO Bloch subspace (`S` = overlap matrix). **LRH** added a momentum (`p̂`) gradient term:

```
S′ = S + Σ_nk ‖ p̂ (1 − P_k) |ψ^PW⟩ ‖²
```

This work generalizes both. It minimizes the trace of a general operator over the **residual subspace** `(1 − P_k)|ψ^NSW⟩`:

```
S̃ = Σ_nk ⟨ψ^NSW| (1 − P_k) Ô (1 − P_k) |ψ^NSW⟩
```

- Choosing the reference as occupied (optionally + some virtual) NSW states and `Ô = p̂²` (kinetic operator) reproduces the generalized spillage **minimizing the trace of the kinetic operator in the residual space**.
- Replacing the reference with PW states and `Ô` with `Ŝ` (`Ŝ + T̂`) reduces Eq.~11 back to the CGH spillage (LRH gradient spillage).

**Practical reading for the user:** the quantity an ORBGEN run actually lowers is this kinetic-residual trace, printed to stdout as the **spillage value**. A well-optimized basis set reaches a **lower spillage** for the same reference set. This is the number we grep from stdout to judge whether an optimization "looks good" (see `orbgen-optimize`).

Fit vs projection: because NSW are truncated in real space they are not exactly a subspace of PW, so this is a **fitting** problem, not a pure projection.

## Reference systems (the `geoms` section)

- Default reference = a **homonuclear dimer series at several bond lengths** (≥ 4 bond lengths per species; LRH also used trimers at TZ level).
- The more (distorted) reference geometries, the more **transferable** the basis — you are averaging the spillage over multiple chemical environments.
- **Empirical caveat:** with too few samples, extra structures (e.g. trimer) occasionally introduce outliers that hurt overall smoothness, so **dimer-only is recommended** (see `orbgen-reference-geometry`).

**Why use NSW instead of PW as the expansion basis for reference states** (a core point of the paper):

1. **Better bridging** — NSW and NAO are made of the same spherical waves, so they connect reference states to NAOs more effectively.
2. **Eliminates periodic-image artifacts** — expanding reference states in PW can generate **unphysical virtual states** from the overlap of highly-delocalized orbitals with their periodic images, causing **artificial bonding across the vacuum**. Because NSW are truncated in real space (≈ applying a spherical confining potential), their tails are cut off and **every solved NSW state is genuinely useable** for building NAOs. This is what makes it possible to safely include virtual states (see below).
3. **Better transferability** for the same number of basis functions.

## Basis hierarchy & the convergence workflow (where the input knobs meet)

- Uses a Dunning-style hierarchy: **minimal (SZ) → pVDZ → pVTZ**, plus restricted variants (pVTZ⁻, pVQZ=). Growth is **shell-wise**: you keep previously generated orbitals fixed and add functions step by step. → This is exactly the `checkpoint` cascade in the `orbitals` block (`SZ → DZ → DZP → TZP → TZDP`).
- **Systematic convergence to CBS** is controlled by `r_c` and `l_max` (given a fixed `E_c`). Define `ε^{PW}_{NSW}` = the **total-energy difference between the NSW primitive basis and a PW reference** for a dimer. Competing effects:
  - alkali metals (Na): error dominated by **`r_c`**,
  - groups IV–VII (S, F): dominated by **`l_max`**,
  - reaching **1 kcal/mol chemical accuracy** generally needs the `f` component (`l_max = 3`) or higher.
  - **Selection rule:** pick the smallest `l_max` and `r_c` such that `ε^{PW}_{NSW}` converges to **0.1 kcal/mol (≈ 4.2 meV)**.
- This is the exact job of the `tools/JYLmaxRcutJointConvTest*` workflow and the `orbgen-converge-rcutlmax` skill. `l_max` ↔ `lmax`, `r_c` ↔ `bessel_nao_rcut`, `E_c` ↔ `ecutjy`.
- **Total energy (by the variational theorem) is the primary quantitative test of basis completeness** — a plane wave at high `ecutwfc` approximates the CBS, and (NAO energy − PW energy) is the incompleteness error.

## Including virtual states improves conduction-band transferability

- Adding some **unoccupied (virtual) states** into the spillage reference **improves conduction-band description with no change in the number of basis functions**.
- **This only works when the reference is expanded in NSW, not PW.** With a PW reference, including virtual states fails, because PW generates unphysical virtual states from periodic-image overlap (seen as spurious virtual states in Na at box sizes 30/40/50 a.u.). With NSW, truncation removes the delocalized tails so the added virtual states are physical and useful.
- **Empirical caveat — GaN counterexample:** including virtual states can *hurt* accuracy in some systems (GaN η₁₀ worsens with 2V/3V) when the missing low angular momentum (s/p/d) radial functions are not yet well converged. So it is not universally a free lunch.
- **Practical mapping:** `nbands` / the virtual-state choice in the input, and why `nbands: occ` vs an integer (`occ+N`) matter for conduction-band transferability (see `orbgen-reference-geometry`, `orbgen-optimize`).

## Terminology for judging "how good a basis is" (metrics used in the paper)

- **Total energy** vs a high-cutoff PW/CBS — the primary completeness test.
- **Chemical accuracy threshold** reused throughout: **1 kcal/mol ≈ 0.042 eV/atom ≈ 4.2 meV**.
- **η (eta)** — occupied-number-weighted band-energy error between two band structures, with an energy shift `ω` chosen to best align levels:
  ```
  η(A,B) = min_ω √( Σ_fk (ε_A − ε_B + ω)² / Σ_fk ) ,        f̃ = √(occ_A·occ_B)
  ```
  This is the criterion used by `orbgen-converge-ecutjy` (band-structure convergence).
- **η₁₀** — same eta but with the Fermi level shifted up **10 eV** to probe higher energy (more conduction bands).

## How this maps onto the skill series

- `general-encyclopedia-zeta` — size vocabulary (SZ/DZ/TZ…).
- `orbgen-converge-ecutjy` — fixes `E_c` via the `η` band-structure test.
- `orbgen-converge-rcutlmax` — fixes `r_c`/`l_max` via `ε^{PW}_{NSW}` total-energy vs PW.
- `orbgen-reference-geometry` — reference states (dimer bond-length series, `pertmags`).
- `orbgen-optimize` — the generalized-spillage minimization, `checkpoint` hierarchy, virtual-state/`nbands` choice, `vloc_aux`, and reading spillage from stdout.
- `orbgen-validate` — ground-truth checks (node count, .orb plot).