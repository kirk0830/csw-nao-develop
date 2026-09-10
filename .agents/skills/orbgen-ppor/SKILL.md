---
name: "orbgen-ppor"
description: "Select and validate the pseudopotential for ORBGEN/ABACUS: match the element and XC functional, confirm the file exists, and derive the batch potential_orb key. Invoke when a basis-set run needs a pseudopotential."
---

# orbgen-ppor — Pseudopotential selection & validation

During ORBGEN the numerical atomic orbital and the pseudopotential must match (same element, same XC functional) to get best accuracy. This sub-skill resolves `pseudo_dir`.

## Steps

1. **Element.** Confirm the element symbol (must match the system you are generating orbitals for). Ask if unknown.
2. **Source.** Suggested resources, in order:
   - [APNS-PPORB](https://www.aissquare.com/datasets/detail?pageType=datasets&name=ABACUS-APNS-PPORBs-v1%253Apre-release&id=326) — pseudopotentials pre-tested for both efficiency and precision.
   - ABACUS online docs: <https://abacus.deepmodeling.com/en/latest/advanced/pp_orb.html>.
   - A user-provided path.
3. **Validate.** Check that `pseudo_dir` exists and is the right file for the element. Infer the XC functional from the pseudopotential filename when possible (e.g. `Si_ONCV_PBE-1.0.upf`) so it stays consistent with the reference DFT.
4. **Output.** Return the validated `pseudo_dir` to the caller for placement in the top-level `pseudo_dir` key.

## Rules

- Do not guess a path that does not exist.
- Note whether it is a UPF or other type ABACUS supports, and surface any mismatch with the requested element.