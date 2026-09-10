# Systematically Improvable Numerical Atomic Orbital Basis Using Contracted Truncated Spherical Waves

This repository contains the code for generating systematically improvable numerical atomic orbital basis sets using contracted truncated spherical waves. The code is implemented in Python and is designed to be user-friendly and efficient.

## Installation

The installation of the code is straightforward. You can create a conda environment and install the required dependencies using the following commands:

```bash
conda create -n orbgen python=3.10
conda activate orbgen
git clone https://github.com/MCresearch/ABACUS-CSW-NAO.git
cd ABACUS-CSW-NAO
pip install .
```

the lines above will create a new conda environment named `orbgen`, activate it, clone the repository, navigate into the cloned directory, and install the package using pip.

## Usage

### Pseudopotential

ABACUS requires the numerical atomic orbital and the pseudopotential must match to get the best performance. The pseudopotential must be selected before generating the basis set. A possible resource may be [APNS-PPORB](https://www.aissquare.com/datasets/detail?pageType=datasets&name=ABACUS-APNS-PPORBs-v1%253Apre-release&id=326), in which the pseudopotentials are tested in both the efficency and precision aspects.

Other pseudopotential resources can be found in the online documentation of ABACUS: https://abacus.deepmodeling.com/en/latest/advanced/pp_orb.html.

After ensuring the pseudopotential is ready, you can have the following lines in your input script to generate the basis set:

```json
{
    "element": "Si",
    "pseudo_dir": "/path/to/pseudopotential",
    "ecutwfc": 100
}
```

, in which `element` is the chemical element for which you want to generate the basis set, `pseudo_dir` is the directory where the pseudopotential files are located, and `ecutwfc` controls the precision of grid integration, which is related to the precision of the generated basis set. A higher `ecutwfc` value will result in a more accurate basis set but will also increase the computational cost.

### Primitive basis

Similar with the Gaussian Type Orbital (GTO) basis sets, in which many primitive GTOs are contracted to form a CGTO (contracted GTO), the primitive basis functions here are the "smoothed" truncated spherical waves (NSW). By setting a truncation radius, a maximal angular momentum, a kinetic energy cutoff, a set of NSW can be uniquely defined. Then you can have the following lines in your input script:

```json
{
    "fit_basis": "jy",
    "ecutjy": 100,
    "primitive_type": "reduced",
    "bessel_nao_rcut": [10]
}
```

If you have read our paper, you may be curious about whether the "fit_basis" can also be something other than "jy", our answer is yes. Try "pw" and you will get a reference wavefunction expanded in plane waves, which is SIMILAR (but not exactly the same) to the PTG-DPSI (LRH) basis set. 

The `primitive_type` should always be kept as `"reduced"` for the general users.

You may also notice that the `bessel_nao_rcut` is assigned as a list of integers instead of a single integer. So our answer is also yes, you can assign a list of integers to `bessel_nao_rcut`, and the code will generate multiple primitive basis sets with different truncation radii. However, this would not be quite useful since we will introduce a systematic way to determine the truncation radius and the maximal angular momentum in the following.

### Spillage optimization

The spillage is defined as the penalty function to measure the difference between the reference wavefunction and the wavefunction expanded in the generated basis set. By minimizing the spillage, we can optimize the contraction coefficients of the basis functions. The behavior of the spillage optimization is controlled by the following parameters:

```json
{
    "spill_guess": "atomic",
    "optimizer": "scipy.bfgs",
    "max_steps": 9000
}
```

The optimizers implemented in torch are also supported, but we still recommend the optimizers `scipy.bfgs` because we have implemented the analytical gradient of the spillage. To use the torch's optimizers, you can set as:

```json
{
    "optimizer": "torch.swats",
    "torch.lr": 1e-3
}
```

, this will use the SWATS optimizer with a learning rate of 1e-3. You can also try other optimizers implemented in torch, such as Adam, yogi, etc.

### Reference geometries

To enhance the transferability of the generated basis set, we can averaging over spillage functions calculated from different structures:

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

With these lines, you create one series of geometries by perturbing the bond length of a homonuclear dimer prototype. The `pertmags` is the list of perturbation magnitudes, which are the bond lengths in this case. You can also create other series of geometries by perturbing the bond length of a trimer, etc. 

In principle, ideally, the more geometries you include, the more transferable the generated basis set will be. However, in some cases we find that including trimer might be detrimental to the smoothness of the basis set. So we recommend to include only the dimer geometries for the general users.

The `nbands` controls the number of states to calculate and include in the spillage function, the `nspin` controls the spin polarization, and the `lmaxmax` controls the maximal angular momentum of the basis functions.

### Basis set contraction

After the reference states being calculated (either being expanded with the plane wave or the NSW primitive basis) and the spillage optimization strategy being configured, you can have the following lines to contract the NSW to form the final NAO basis set:

```json
{
    "orbitals": [
        {
            "nzeta": [1, 1, 0],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": null
        }
    ]
}
```

These lines will tell the code to contract the primitive basis functions to form a basis set with 1 s-type orbital and 1 p-type orbital, by minimizing the spillage function calculated from the first series of geometries (by setting the `geoms` to `[0]`) and only including the occupied states (`nbands` is set to `occ`, although it can be set to other values like an integer that is not greater than `nbands`, or `all`). For the first contraction, the `checkpoint` is always set to `null`.

Larger basis set can be generated based on defined-above smaller basis set. For example, you can have the following lines to generate a basis set with 2 s-type, 2 p-type and 1 d-type (a polarized) orbitals:

```json
{
    "orbitals": [
        {
            "nzeta": [1, 1, 0],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": null
        },
        {
            "nzeta": [2, 2, 1],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": 0
        }
    ]
}
```

, in which the second contraction will be based on the first contraction (by setting the `checkpoint` to `0`), and the spillage function is still calculated from the first series of geometries and only including the occupied states.

Please note, you should always make sure that the maximal angular momentum requested in this section is not larger than the `lmaxmax` defined in the reference geometries section.

## Advanced orbital options

The `orbitals` entries shown above only carry the compulsory keys `nzeta`, `geoms`, `nbands` and `checkpoint`. Besides these, every orbital entry additionally supports the following optional keys to fine-tune how it is initialized and grown. All of them are pure keywords of the current ORBGEN v3.0 input script.

### Initialization models: `model` and `model_kwargs`

During the spillage optimization, each orbital starts from an initial guess of the primitive contraction coefficients. This is controlled by the initialization **model**. The top-level `spill_guess` key only sets the *global default*; each orbital entry can override it with its own `model`, together with a per-model `model_kwargs` dict:

```json
{
    "spill_guess": "atomic",
    "orbitals": [
        {
            "nzeta": [1, 1, 0],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": null
        },
        {
            "nzeta": [2, 2, 1],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": 0,
            "model": "hydrogen",
            "model_kwargs": {"slater": true}
        }
    ]
}
```

In the example above, the second orbital would be initialized with screened hydrogen-like orbitals instead of the global `atomic` model. `model_kwargs` are automatically filtered to only the keys that are meaningful for the chosen `model`, so it is safe to reuse one dict across orbitals with different models.

The supported models are:

| model | purpose | valid `model_kwargs` keys |
|------|---------|--------------------------|
| `ones` | identity/unit initial guess, fastest but often poor | none |
| `random` | random coefficients (PRB 103, 235131 (2021)) | `seed` |
| `atomic` | from a single-atom (monomer) DFT calculation | `jobdir` (required), `vloc_aux`, `lloc_min` |
| `hydrogen` | hydrogen-like orbitals, with or without Slater screening | `slater`, `otherelem` |
| `pretrained` | restart/hot-start from an existing `.orb` file | `pretrained` |

Notes on the individual models:

- `atomic` requires a `jobdir` pointing to the monomer calculation. Empirically, the pure atomic calculation cannot give satisfying starting points for high angular momentum orbitals (e.g. g orbitals). To improve this, provide `vloc_aux` (a file describing an auxiliary local potential) and `lloc_min`, beyond which angular momentum the coefficients are initialized by solving the radial Schrödinger equation under that auxiliary potential. `lloc_min` defaults to `4`. Unless you override `model` per orbital, `atomic` is the value coming from the top-level `spill_guess`.
- `hydrogen` with `otherelem` set to another element of higher Z is useful to avoid the truncation of the generated hydrogen-like radial functions at the cutoff radius.
- `pretrained` with `pretrained` pointing to an `.orb` file lets you continue from a previously generated orbital.

### Basis growth over zeta counts: `greedygrow` and `nzeta_max` (experimental)

> Note the division of labour: the convergence test workflow above (`tools/JYLmaxRcutJointConvTest*`) determines the *completeness* parameters **`rcut` and `lmax`**. This section is about a **different** knob — growing the number of **zeta** functions (`nzeta`) for a given `rcut`/`lmax`. In the common workflow you set `rcut`/`lmax` via the convergence test and specify an explicit `nzeta` per angular momentum; you do not need `greedygrow` at all.

`greedygrow` lets an orbital **grow its own zeta counts** until the spillage stops decreasing. Set `"greedygrow": true` and give an upper bound `nzeta_max`:

```json
{
    "orbitals": [
        {
            "nzeta": [1, 1, 0],
            "geoms": [0],
            "nbands": "occ",
            "checkpoint": null,
            "greedygrow": true,
            "nzeta_max": [3, 3, 3]
        }
    ]
}
```

Starting from `nzeta`, the greedy algorithm tries adding one more zeta function to each angular momentum, keeps the one that most reduces the spillage per `(2l+1)` basis functions (accounting for the computational cost), and repeats until no angular momentum can lower the spillage (or `nzeta_max` is reached). `nzeta_max` must be element-wise greater than or equal to `nzeta`.

**Caveat — not recommended for production.** This is a hidden/experimental option: it is not part of the validated input schema (it is only honored because unknown keys are passed through), and, being a greedy heuristic over repeated non-convex spillage optimizations, it is difficult to make reliable — the spillage surface can be uneven enough that the per-step greedy choice is noisy. Empirical experience favours **manually specifying an explicit `nzeta` per angular momentum** (the basis-contraction scheme in the preceding section) over automatic growth. Consider `greedygrow` experimental and for prototyping only.

### Other per-orbital options

- `filename`: customize the name of the output `.orb` file for this orbital.
- `fix_components`: a nested list that freezes (keeps constant during optimization) the given contraction coefficients; useful for constrained basis sets.

### Automatic zeta counts: `nzeta` as a string

Besides a list of integers, `nzeta` of an orbital can be a string of the form

```
auto:(twsvd|amwsvd):<threshold>[:(max|mean)]
```

e.g. `"nzeta": "auto:twsvd:0.8:max"`. The code then infers the number of zeta functions for each angular momentum from the reference data using the `twsvd` or `amwsvd` method, keeping only those below the given singular-value threshold. The trailing `:max|:mean` selects how the statistics are combined, defaulting to `max`.

## Compute and environment options

The following top-level keys control how the underlying ABACUS DFT calculations are launched:

```json
{
    "environment": "",
    "mpi_command": "mpirun -np 8",
    "abacus_command": "abacus",
    "nthreads_rcut": 4,
    "max_steps": 9000
}
```

- `environment` / `mpi_command` / `abacus_command` describe how to invoke ABACUS (module loading, the MPI launcher and its process count, and the executable). If `abacus_command` is `null`/missing, ABACUS is not run and only the jy/orbital preparation is performed.
- `nthreads_rcut` is the number of threads used for the spillage optimization.
- `max_steps` caps the number of optimization steps per contraction.
- Optimizer-specific keys prefixed with `torch.` or `scipy.` (e.g. `optimizer`, `torch.lr`) are forwarded to the chosen optimizer.

## Systematic way to determine the truncation radius and the maximal angular momentum

The truncation radius and the maximal angular momentum are two important parameters that control the completeness of the basis set. We suggest a systematic way to determine these two parameters by varying them and checking the convergence behavior of the relative total energy error of a test system calculated with the primitive basis set with respect to the reference plane wave calculation. 

Please see the workflow `tools/JYLmaxRcutJointConvTestGenerator.py` for the details of how to perform this test. The workflow will generate a series of input scripts for the primitive basis set generation with different truncation radii and maximal angular momenta, and then you can run these input scripts to get the convergence behavior of the relative total energy error. A postprocessing script is also provided to plot the convergence behavior, which is `tools/JYLmaxRcutJointConvTestReader.py`. By analyzing the convergence behavior, you can determine the truncation radius and the maximal angular momentum that can achieve a good balance between accuracy and computational cost for your system of interest.
