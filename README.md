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

## Systematic way to determine the truncation radius and the maximal angular momentum

The truncation radius and the maximal angular momentum are two important parameters that control the completeness of the basis set. We suggest a systematic way to determine these two parameters by varying them and checking the convergence behavior of the relative total energy error of a test system calculated with the primitive basis set with respect to the reference plane wave calculation. 

Please see the workflow `tools/JYLmaxRcutJointConvTestGenerator.py` for the details of how to perform this test. The workflow will generate a series of input scripts for the primitive basis set generation with different truncation radii and maximal angular momenta, and then you can run these input scripts to get the convergence behavior of the relative total energy error. A postprocessing script is also provided to plot the convergence behavior, which is `tools/JYLmaxRcutJointConvTestReader.py`. By analyzing the convergence behavior, you can determine the truncation radius and the maximal angular momentum that can achieve a good balance between accuracy and computational cost for your system of interest.
