---
name: "orbgen-validate"
description: "Verify and visualize a generated NAO basis set (.orb) and optionally produce a smooth projector via projgen. Invoke after orbgen finishes, to sanity-check outputs before they are used in ABACUS."
---

# orbgen-validate — verify & visualize outputs

Sanity-checks an `.orb` produced by `orbgen` and optionally turns it into a projector.

## Verify the output

1. Locate the emitted `.orb` for the element (the orchestrator passes the output dir).
2. Confirm it opens/parses (e.g. `read_nao`) and that `rcut`, `ecut`, `nzeta` match expectations.
3. Check the spillage trend logged during optimization (the final spillage should be small — lower is better).
4. Plot the radial functions; the orbitals should be smooth and decay to ~0 at the cutoff.

## Optional downstream: `projgen`

If the user needs a projector (for ABACUS downfolding / DFT+U / Deltaspin), run:

```bash
projgen -i <orb>.orb -r <radius>
```

- `-r` (required) — cutoff radius, must be ≤ the orbital's rcut.
- `-j` — global index of the zeta function to use (default: first zeta of each l); negative counts from the end.
- `-m new|update` — write only the projector or append it to the existing zeta functions.
- Output defaults to `<orb>.<radius>au.proj`; a `projgen.png` plot is also written.

## Rules

- Do not claim the basis is "good" without checking the final spillage value and the smoothness plot.
- Surface any parse errors or unexpected shapes instead of auto-fixing silently.