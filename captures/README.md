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

## 27 September 2026 KH9 / Battery-Emulator charge/discharge capture set

These 15 unique raw exports came from the principal FoxESS KH9 / Stark CMR / Nissan Leaf ZE1 installation during a forced-grid-charge session and the subsequent discharge period. They contain both battery-side traffic and the Fox inverter-side traffic exposed by Battery-Emulator's CAN Tools logger.

The later charging slices were taken at high SOC while the Leaf BMS charge-power allowance was already reducing. They are suitable for frame decoding but **must not** be used alone to explain the separate earlier low-SOC charge-throttling event. The discharge set supplied the missing opposite-direction `0x1871/0x07` evidence.

Six additional phone downloads were byte-identical duplicates and are intentionally not committed twice. Original retained exports are stored unchanged under `captures/2026-09-27-kh9-charge-discharge/`.

| Capture | Size | SHA-256 |
| --- | ---: | --- |
| `canlog_8d05h14m45s (1).txt` | `10,927` bytes | `5713a6880c3c295e27e64eb09adbefdc6d0dd5513eacd739b08c430870148c58` |
| `canlog_8d05h14m48s (1).txt` | `1,992` bytes | `62ef13c144ced6dbea6480b0bf9ae04788f67fb3424464185ccc4a4951224047` |
| `canlog_8d05h14m51s (1).txt` | `3,522` bytes | `dca7c2cb439a562b0331b15ed32dc9eceb7eff9010bf9f2db6a55008f3a44c42` |
| `canlog_8d05h14m54s (1).txt` | `13,552` bytes | `6ec953099538b19777e66061ee4ceaebccf22ec29ddece59c53ad3aad55d183c` |
| `canlog_8d05h15m00s (1).txt` | `8,097` bytes | `11cf9050d82879021a80bf3f51249fdb36cca77f332613fc678b5cfaf5928c23` |
| `canlog_8d05h15m03s (1).txt` | `5,783` bytes | `34e3431cf6294ca30317492464f1b3f994887a5879e16c64e39d8c9e516cb631` |
| `canlog_8d05h59m48s (1).txt` | `993` bytes | `ab4b18363fd7a9b989ce3e5d734b610580fecd6cc47b74af9a4d282e7a5102a9` |
| `canlog_8d05h59m51s (1).txt` | `3,371` bytes | `6d2938f9ef707f2295b44e8177b9546aa8464b4170d3c017831cae4108d7d854` |
| `canlog_8d05h59m54s (1).txt` | `4,320` bytes | `14ffecaef01049ac89032053a8892eff6956a3be3a6d6312dcb400b77e3f92b6` |
| `canlog_8d06h03m41s (1).txt` | `11,742` bytes | `66de92df30d254df38161d1726cdbb37a62274272e2302bb6a5930a87b82807f` |
| `canlog_8d06h03m43s (1).txt` | `10,941` bytes | `0fe0122345a27d7181f08a64778bf621ea9bb1609182d0134619ab814828b890` |
| `canlog_8d06h03m45s (1).txt` | `14,381` bytes | `c4f72082c5ccf9170812f43e99c22eb35f3f106b5fac196488698992b796966f` |
| `canlog_8d06h03m47s (1).txt` | `6,830` bytes | `7668cd6d9dd0a34f55f5f4ee2e5ee8ffe94c05705203060bd09c5e0e212054b4` |
| `canlog_8d06h03m52s (1).txt` | `3,720` bytes | `dea47235bd3b075be5b8b8f10a02fd06ee3badce52b275c4209c83d5a11da560` |
| `canlog_8d06h03m54s (1).txt` | `7,185` bytes | `9ce0e3633ed777e15427b2b5e34947c8aaaadd4a5d4515b0c93dcb2cc45d4c1b` |

Key inverter-origin examples from this set:

- charging: `07 33 67 FF AA 0F 02 3C` → `-15.3 A`, `401.0 V`;
- discharging: `07 33 E2 00 27 0F 02 3C` → `+22.6 A`, `387.9 V`.

This makes bytes 2–3 and bytes 4–5 of the observed `0x1871/0x07` form **Capture-confirmed** as signed current at `0.1 A/count` and voltage at `0.1 V/count` respectively on the tested KH9. Byte 1 and bytes 6–7 remain **Unresolved**.
## Evidence handling

- Preserve original raw capture files unchanged.
- Verify byte size and SHA-256 before adding or analysing a raw capture; never reconstruct a missing original.
- Retain each capture's provenance and test context.
- Keep original/raw evidence separate from processed notes, decoded tables and derived calculations.
- Useful provenance may include the date and time, operating state, inverter and battery context, capture tool or interface, and firmware version only where it is actually known.
- Label unknown provenance as **unknown** rather than guessing.
- Never place proprietary FoxESS firmware in this directory.

The existence of this directory and the inventory above do not mean that raw captures are included or must be published. Publication of a raw capture is not required merely to populate the directory.
