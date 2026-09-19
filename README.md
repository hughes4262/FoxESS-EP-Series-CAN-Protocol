# FoxESS EP-Series CAN Protocol

Reverse-engineered documentation and a standalone battery-side CAN implementation for using the FoxESS EP-Series interface with [Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator).

The protocol represents one aggregate virtual EP battery using generic Battery-Emulator datalayer information. Its purpose is to let supported alternative high-voltage batteries provide the live state, limits, status, capacity, energy and model information expected by a compatible FoxESS inverter.

> **Built on Battery-Emulator:** This protocol was developed and hardware-tested using [Dala's Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator) as the underlying battery integration and CAN framework. Many thanks to Dala and the Battery-Emulator contributors for creating and maintaining the project that made this work possible.

> [!NOTE]
> This is independent community reverse-engineering work. It is not official FoxESS documentation and is not affiliated with or endorsed by FoxESS.

## Current status

**v67 is the frozen, hardware-proven standalone reference checkpoint.** It has operated on a real FoxESS KH-series inverter and populated the FoxESS app's Battery Details data. FoxESS EP-Series support was included in Battery-Emulator v12.5.0; that upstream release is separate from the frozen v67 source documented in this repository, which is not yet a final standalone release.

| Item | Current position |
| --- | --- |
| Protocol checkpoint | v67 |
| Validation | Real inverter, battery, FoxESS cloud/app and CAN testing |
| Source status | Frozen hardware-proven v67; public documentation complete |
| Upstream status | FoxESS EP-Series support released in Battery-Emulator v12.5.0; standalone v67 retained as the documented reference |
| Main release blocker | Persistent cumulative counters across restart and reflash |

The frozen v67 checkpoint is retained as a standalone reference for review and evidence-led documentation. It should not be described as a final standalone release until its remaining release requirements are met.

## Why this project exists

The available FoxESS support did not yet reproduce the complete EP-Series behaviour and data needed for the intended Battery-Emulator integration. In particular, further work was required around readiness, the EP capacity/model frames, Fox Cloud energy values and equivalent cycles.

The implementation was developed by combining:

- genuine Fox CAN captures covering old single/dual EP12, EP12 Plus and CQ7;
- analysis of Fox manager-firmware behaviour;
- comparison with earlier FoxESS CAN research and existing code; and
- repeated testing on a real inverter, alternative battery and FoxESS app.

This repository records both the resulting implementation and the evidence behind it. Detailed byte mappings and long evidence tables are deliberately kept out of this front page.

## Hardware-proven behaviour

On the current test system, v67 has demonstrated:

- startup communication and generic battery-readiness handling;
- contactor closure and contactor-state reporting;
- controlled charging and discharging using the battery's reported limits;
- live state of charge, pack voltage, signed current and temperature reporting;
- state of health, nominal energy and remaining-energy presentation;
- Daily Charged and Daily Discharged energy movement;
- Total Discharged energy movement;
- independent Total Charged behaviour after Fox accepted the genuine charged-capacity path; and
- Battery Cycles moving from `0` to `1` at approximately one equivalent full cycle.

Hardware-proven means observed on the real test installation. It does not imply that every FoxESS inverter model, firmware version or Battery-Emulator battery integration has been validated.

### Tested environment

The principal real-hardware validation used a FoxESS KH9 hybrid inverter and a 40 kWh Nissan Leaf battery connected through Battery-Emulator hardware. The Nissan Leaf was the test battery; it is not a protocol dependency.

## Key protocol discoveries

The work identified or materially strengthened several parts of the EP-Series data model:

| CAN frame | Concise role in v67 | Evidence position |
| --- | --- | --- |
| `0x187A` | Directional cumulative charged and discharged energy | Hardware/app-confirmed |
| `0x1878` | Individual-unit SOC and absolute charged-plus-discharged throughput | Supported by capture, firmware and implementation evidence |
| `0x1879` | Directional cumulative charged and discharged capacity | Both halves capture-confirmed; charged path hardware/app-confirmed |
| `0x1875` | Equivalent-cycle count | Hardware/app-confirmed from the observed `0` to `1` transition |
| `0x1900`–`0x1909` | EP capacity, limits, model and state information | Mixed evidence; field-level confidence is recorded in the frame map |
| `0x1910`–`0x1919` | Newer-family extension, absent on old EP12 and present on audited EP12 Plus/CQ7 | Capture-confirmed presence; family-dependent content and compatibility requirement partly unresolved; not transmitted by frozen v67 |

The [CAN frame map](docs/CAN-FRAME-MAP.md) records the byte order, scaling, signedness and field-by-field confidence. Fields that remain inferred or unresolved are not promoted to confirmed status without new evidence.

## Energy-counter architecture

v67 maintains separate installation counters and exposes three related views of battery use:

- `0x187A` carries cumulative charged and discharged energy in opposite halves of the frame, at `0.1 kWh` per count.
- `0x1878` carries unit SOC and absolute energy throughput in Wh, calculated as charged energy plus discharged energy.
- `0x1879` carries directional cumulative capacity at `0.1 Ah` per count: both halves are capture-confirmed, and the charged half is hardware/app-confirmed as Fox's accepted independent Total Charged path.

These frames are related, but they are not interchangeable. Real hardware testing showed that Fox can use the charged side of `0x1879` as an independent Total Charged path once its plausibility conditions are satisfied. During subsequent discharge, the accepted Total Charged value remained fixed instead of following discharged energy.

The detailed counter relationships, units and test sequence are documented in [Energy counters](docs/ENERGY-COUNTERS.md).

## Native capture versus frozen v67

Later cross-model captures corrected several native interpretations without changing the frozen hardware-proven implementation. Native battery-origin current is positive while charging and negative while discharging, while tested v67/app-facing current has the opposite sign. Native `0x187B` charging/discharging codes are also opposite to v67, native `0x1903` is SOC-scaled nominal energy rather than a universal fixed `/200` value, and newer captures show a richer `0x1908` container. These differences require controlled hardware review and do not invalidate the demonstrated KH9 result.

The `0x1910`–`0x1919` extension is not currently required for compatibility already demonstrated by v67 on the tested KH9. Whether newer inverter or manager firmware requires it remains unresolved; it has not been implemented from documentation evidence alone.

## Fox Total Charged fallback

Fox applies a plausibility or fallback model when a fresh genuine charged counter is too low relative to discharged energy. This fallback behaviour was observed on real hardware. Its relationship of approximately `0.9203` is notably consistent with the product of the observed Fox model coefficients:

```text
0.942 × 0.977 ≈ 0.9203
```

In the observed fallback region, Fox's charged estimate behaved approximately like discharged energy divided by `0.9203`. After the genuine `0x1879` charged-capacity counter overtook the fallback, Fox selected the independent charged path and Total Charged stopped increasing during discharge. If later discharge again makes that relationship implausible, Fox can return to the fallback.

The approximately `0.9203` relationship is therefore evidence of Fox's fallback/model behaviour. It does **not** show that `0x1879` failed. The detailed derivation and remaining uncertainty around Fox's selection threshold belong in the dedicated energy-counter document.

## Equivalent cycles

v67 calculates equivalent cycles from total bidirectional throughput:

```text
equivalent cycles = (charged Wh + discharged Wh) / (2 × battery capacity Wh)
```

The transmitted value uses whole-cycle integer truncation. On the real test system, the Fox Battery Cycles value moved physically from `0` to `1` at about `69.6 kWh` of combined throughput for a battery represented at approximately `35 kWh`. This is evidence for the first threshold transition only; longer-term behaviour still requires normal release validation.

## Battery-agnostic design

The standalone protocol is intended to consume generic Battery-Emulator datalayer values for battery state, limits, permissions, faults, capacity, cell measurements and temperatures.

It must not depend on Nissan Leaf-specific structures, a fixed Leaf capacity, a 96-cell assumption or Leaf-only temperature data. The test pack provides real validation, while the CAN interface remains suitable for other battery integrations that correctly populate the generic datalayer.

Readiness also remains generic: battery communication must be live, the system active, contactor closure permitted and faults absent before the battery is considered ready. Power-path reporting begins only when the contactors are engaged.

## Evidence and confidence

Repository claims will use explicit confidence labels:

| Label | Meaning |
| --- | --- |
| **Hardware/app-confirmed** | Observed on the physical inverter, battery and/or FoxESS app |
| **Capture-confirmed** | Directly observed in a genuine Fox CAN capture |
| **Firmware-supported** | Supported by the decoded Fox manager-firmware behaviour |
| **Strongly inferred** | Multiple evidence sources agree, but direct native proof is still missing |
| **Unresolved** | Available evidence is insufficient for a reliable mapping |

Evidence priority is real hardware, genuine CAN captures, manager-firmware analysis and then older comments or assumptions. Disproved mappings are retained as rejected findings rather than silently revived.

No proprietary Fox firmware binaries will be published. Firmware documentation will contain only the derived analysis needed to explain the protocol findings. Original CAN captures should remain unchanged, with processed notes kept separately from raw evidence.

## Commissioning guidance

For a fresh installation whose local counters begin at zero, enable the EP protocol at a relatively low battery SOC and establish enough genuine charged energy before allowing substantial discharge.

This gives the charged counter time to become plausible before Fox's discharged-energy fallback overtakes it. It is a commissioning mitigation for zero-start counters, not a substitute for persistent storage.

See [Commissioning](docs/COMMISSIONING.md) for the step-by-step procedure and safeguards.

## Known limitation: counter persistence

> [!WARNING]
> The v67 installation energy and capacity counters are currently held in RAM. They reset when the emulator restarts or is reflashed.

Fox retains its previous cloud/app baselines. If the local cumulative counters restart from zero, Fox can calculate negative Daily Charged or Daily Discharged values and reactivate its Total Charged fallback. Reboot or reflash also resets the locally derived and transmitted equivalent-cycle value because its source counters are volatile.

Persistence is therefore the principal remaining release-quality blocker. Its implementation is intentionally deferred for discussion with Dala and the Battery-Emulator project so that any solution fits upstream storage, migration and firmware-update expectations. This repository does not currently modify Battery-Emulator's shared NVM or core infrastructure.

## Documentation

The approved repository structure separates the front-page summary, frozen source, detailed interpretation and raw-evidence guidance:

| Resource | Purpose | Status |
| --- | --- | --- |
| [`README.md`](README.md) | Project overview and current status | Current |
| [`src/FOXESS-EP-CAN.cpp`](src/FOXESS-EP-CAN.cpp) | Frozen v67 implementation | Frozen source |
| [`src/FOXESS-EP-CAN.h`](src/FOXESS-EP-CAN.h) | Frozen v67 declarations and frame definitions | Frozen source |
| [`docs/CAN-FRAME-MAP.md`](docs/CAN-FRAME-MAP.md) | Detailed frame and field map | Current |
| [`docs/ENERGY-COUNTERS.md`](docs/ENERGY-COUNTERS.md) | Energy, capacity, fallback and cycle findings | Current |
| [`docs/COMMISSIONING.md`](docs/COMMISSIONING.md) | Commissioning and validation procedure | Current |
| [`docs/TESTING-AND-EVIDENCE.md`](docs/TESTING-AND-EVIDENCE.md) | Test environment, proof record and confidence rules | Current |
| [`docs/FIRMWARE-REVERSE-ENGINEERING.md`](docs/FIRMWARE-REVERSE-ENGINEERING.md) | Derived manager-firmware findings; no firmware binaries | Current |
| [`docs/KNOWN-LIMITATIONS.md`](docs/KNOWN-LIMITATIONS.md) | Known limitations, evidence boundaries and deferred upstream work | Current |
| [`captures/README.md`](captures/README.md) | Capture provenance, preservation and usage notes | Current |
| [`LICENSE`](LICENSE) | GNU General Public License version 3 | Current |

## Credits and attribution

- [Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator), created and maintained by Dala and its contributors, provides the upstream platform and generic battery datalayer.
- Earlier public FoxESS CAN research used during comparison includes [FozzieUK/FoxESS-Canbus-Protocol](https://github.com/FozzieUK/FoxESS-Canbus-Protocol) and [rand12345/solax_can_bus](https://github.com/rand12345/solax_can_bus).
- EP12 capture work, protocol investigation and real-hardware/app testing for this repository were carried out by Hughes4262.

FoxESS product names and trademarks remain the property of their respective owners.

## License

This repository is licensed under the [GNU General Public License version 3](LICENSE).
