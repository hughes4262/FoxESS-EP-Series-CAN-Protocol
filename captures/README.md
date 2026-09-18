# CAN capture evidence

This directory is reserved for FoxESS EP-Series CAN capture evidence and its provenance. Genuine old EP12, EP12 Plus and CQ7 captures formed part of the evidence used to develop and validate this protocol.

## Audited capture inventory

The completed cross-model audit recorded the following original-file checks. A listed hash documents the audited evidence; it does not imply that the raw file is committed to this public repository.

| Capture | Size | SHA-256 | Repository state |
| --- | ---: | --- | --- |
| `EP12 Startup-discharge-charge-2026-09-18_08-20-38.txt` | `15,729,384` bytes | `439c9c078f04880297d9e30ff377a2ce20cc0f2bf011c309843736b87afa62af` | Not included; the genuine original file was not available in the documentation-update environment |
| `CQ7 Startup-2026-09-09_09-57-49.txt` | `6,620,077` bytes | `f63ee9abc71524fa7c1dd5d1929a5fb06df15dc3beda48fa71396ab901fb1e93` | Not included; the genuine original file was not available in the documentation-update environment |

Historical audited captures were also identified by these SHA-256 values; filenames and sizes are not reconstructed where they were not supplied to this repository update:

| Historical evidence label | SHA-256 |
| --- | --- |
| Old single EP12 startup | `41f4e06f92801180a9d4725905f2580fb3808e11d87a807f0a7c3960af11cc19` |
| Old single EP12 charging-labelled capture | `cea860973938b0a7af602eabe08ab0e79416feb2c98742200961ec3a4c2e93a4` |
| Old single EP12 discharging-labelled capture | `099dceac8e890d8636c2448e5ed3212f81fc93f455b6caccccba2b13bf4b63e3` |
| Old dual EP12 long capture | `62469ef0e243211a9cb738b47689fe6e56065ce3ce0d46493302d3b5dc0f9a95` |

## Evidence handling

- Preserve original raw capture files unchanged.
- Verify byte size and SHA-256 before adding or analysing a raw capture; never reconstruct a missing original.
- Retain each capture's provenance and test context.
- Keep original/raw evidence separate from processed notes, decoded tables and derived calculations.
- Useful provenance may include the date and time, operating state, inverter and battery context, capture tool or interface, and firmware version only where it is actually known.
- Label unknown provenance as **unknown** rather than guessing.
- Never place proprietary FoxESS firmware in this directory.

The existence of this directory and the inventory above do not mean that raw captures are included or must be published. Publication of a raw capture is not required merely to populate the directory.
