# Dependency Refactor Design: Pixi Support + README Update

## Goal

Refactor the dependency setup of SIAB so that it supports both the existing `pip install -e .` workflow and a new `pixi` workflow. Update `README.md` to document both installation paths, and add an "Advanced usage" section explaining the `model` keyword and its sub-keywords in the `orbitals` section.

## Scope

- Modify [`pyproject.toml`](file:///workspace/pyproject.toml) to add `[tool.pixi.*]` configuration.
- Do **not** create a separate `pixi.toml`; keep the manifest unified in `pyproject.toml`.
- Update [`README.md`](file:///workspace/README.md):
  1. Installation section covers conda/pip and pixi as two parallel, mutually exclusive options.
  2. New "Advanced usage" section documenting `orbitals[i].model` and related sub-keywords.

## Pixi Configuration

### Channels and project metadata

```toml
[tool.pixi.project]
name = "SIAB"
channels = ["conda-forge"]
platforms = ["linux-64"]
```

### Dependencies

All runtime dependencies declared in `[project.dependencies]` are mirrored in pixi. Python packages that are available on PyPI are declared as pypi-dependencies so that versions stay aligned with the existing pip workflow:

```toml
[tool.pixi.pypi-dependencies]
SIAB = { path = ".", editable = true }
numpy = "*"
matplotlib = "*"
scipy = "<1.10"
torch = "*"
torch_optimizer = "*"
torch_complex = "*"
addict = "*"
```

### Feature: with-abacus

ABACUS is distributed on `conda-forge`. A dedicated feature adds it as a conda dependency instead of running `conda install` inside a task:

```toml
[tool.pixi.feature.with-abacus.dependencies]
abacus = "*"
mpich = "*"
libblas = { version = "*", build = "*mkl*" }
```

### Environments

```toml
[tool.pixi.environments]
default = { solve-group = "default" }
with-abacus = { features = ["with-abacus"], solve-group = "default" }
```

### Tasks

```toml
[tool.pixi.tasks]
basic = "python -m pip install -e . --no-deps"
with-abacus = "python -m pip install -e . --no-deps && abacus --version"
```

Usage:

```bash
# Option A: basic environment (no ABACUS)
pixi install
pixi run basic

# Option B: environment with ABACUS installed via conda-forge
pixi install -e with-abacus
pixi run with-abacus
```

## README.md Updates

### 1. Installation as two mutually exclusive options

Add a new top-level "Installation" section that presents conda/pip and pixi as two parallel, choose-one paths. Make it explicit that users should pick only one of them.

**Option 1: conda + pip**

```bash
conda create -n orbgen python=3.10
conda activate orbgen
git clone https://github.com/MCresearch/ABACUS-CSW-NAO.git
cd ABACUS-CSW-NAO
pip install .
```

**Option 2: pixi**

```bash
git clone https://github.com/MCresearch/ABACUS-CSW-NAO.git
cd ABACUS-CSW-NAO

# basic environment
pixi install
pixi run basic

# environment with ABACUS
pixi install -e with-abacus
pixi run with-abacus
```

### 2. Advanced usage: `orbitals[i].model`

Append a new section at the end of the README explaining that each dict in the `orbitals` list can carry an initialization model. Supported `model` values:

- `"atomic"` (default): use a prior single-atom ABACUS calculation as the initial guess. Requires `model_kwargs.jobdir`.
- `"random"`: random coefficients. Accepts `model_kwargs.random_seed`.
- `"ones"`: identity-like initialization, no extra kwargs.
- `"hydrogen"`: hydrogen-like orbitals. Accepts `model_kwargs.slater` (bool) and `model_kwargs.otherelem` (str).
- `"pretrained"`: restart from an existing `.orb` file. Requires `model_kwargs.pretrained` (path).

Also mention related per-orbital keywords:

- `fix_components`: list of lists of zeta indices to freeze during optimization.
- `filename`: custom output filename for the generated orbital.
- `greedygrow` + `nzeta_max`: enable greedy angular-momentum expansion.

Include a short JSON example demonstrating `model` and `model_kwargs`.

## Files to Modify

- [`pyproject.toml`](file:///workspace/pyproject.toml)
- [`README.md`](file:///workspace/README.md)

## Success Criteria

- `pixi install` resolves without error.
- `pixi install -e with-abacus` installs ABACUS from conda-forge.
- `pixi run basic` and `pixi run with-abacus` complete successfully.
- README clearly states that conda/pip and pixi are mutually exclusive installation options.
- README contains the new Advanced usage section with accurate descriptions derived from the code.
