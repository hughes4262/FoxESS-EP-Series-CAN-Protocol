# Known Limitations and Deferred Upstream Work

## Scope

v67 is a frozen, hardware-proven interoperability checkpoint for the standalone FoxESS EP-Series Battery-Emulator protocol.

On the validated real installation, the inverter recognises the virtual EP battery, contactors operate, charging and discharging work, the main live-data and limit paths operate, and the important Fox Battery Details energy and cycle paths have been demonstrated.

That does not mean that every native Fox internal field, model parameter or cloud-side calculation has been fully decoded. Some remaining limitations affect release quality and evidence completeness rather than immediate CAN interoperability.

Those limitations are documented here rather than hidden, overstated or prematurely implemented.

## Current release status

v67 is:

- hardware-proven on the validated test installation;
- suitable as the documented reverse-engineering baseline; and
- not yet considered a polished production or upstream release.

The principal release blocker is loss of cumulative history across Battery-Emulator reboot or reflash.

This limitation does not mean that normal charge or discharge operation is unstable. Charging, discharging, contactor operation, readiness handling and the principal live-data paths are already demonstrated on the validated installation.

## Major limitation: volatile cumulative counters

Frozen v67 currently generates its Fox-facing cumulative histories in RAM.

The affected values include:

- `0x187A` cumulative charged energy;
- `0x187A` cumulative discharged energy;
- `0x1878` absolute throughput derived from the local charged/discharged energy totals;
- `0x1879` cumulative charged capacity;
- `0x1879` cumulative discharged capacity; and
- the locally derived and transmitted equivalent-cycle value.

When Battery-Emulator restarts or is reflashed, those local histories can restart from zero or a lower value while Fox retains earlier cumulative context.

Real hardware/app observations showed that this loss of continuity can produce:

- negative Daily Charged Energy;
- negative Daily Discharged Energy;
- loss of continuity in cumulative values;
- Total Charged fallback behaviour returning; and
- the locally transmitted Battery Cycles value returning to a lower value or zero because its volatile source history has been lost.

The project does **not** claim that Fox Cloud's own internally stored Battery Cycles value has been proved to reset.

The proven reset is the locally generated and transmitted v67 cycle source, because the cumulative local counters feeding that calculation are volatile.

Persistent cumulative history is therefore a current release-quality requirement.

Persistence implementation and design are deliberately deferred and are outside this documentation phase.

## Why persistence is deferred upstream work

The FoxESS EP protocol requires monotonically continuous cumulative history if its Fox-facing lifetime and derived values are to remain coherent across Battery-Emulator restart and reflash.

The safest persistence mechanism should fit the wider Battery-Emulator architecture rather than being improvised inside protocol documentation.

Later upstream integration will need to determine:

- where persistent FoxESS installation history belongs;
- how it should interact with Battery-Emulator's existing persistence infrastructure;
- how upgrades and reflashes preserve continuity;
- how migration from a volatile build is handled; and
- how batteries that provide genuine lifetime counters should be treated relative to integrations that require locally generated histories.

This document does not answer those questions and does not prescribe an NVM record layout, flash-write strategy, migration algorithm or storage architecture.

## Commissioning mitigation is not persistence

The recommended low-SOC, charge-first commissioning sequence can help establish a plausible direct charged history on a fresh zero-counter installation.

That sequence can reduce early Total Charged fallback problems by allowing the direct charged path to build genuine history before substantial discharge accumulates.

It does **not** preserve cumulative history across reboot or reflash.

See [`COMMISSIONING.md`](COMMISSIONING.md) for the commissioning procedure and its limitations.

## Native-versus-v67 fidelity differences

Both `0x1879` halves are now **Capture-confirmed** at `0.1 Ah/count`; EP12 Plus supplied the discharged-side progression missing from the older short capture. The remaining limitations concern fidelity and hardware-review scope rather than that mapping:

- genuine battery-origin `0x1873` current is positive while charging and negative while discharging, but frozen v67 negates generic current and produced the opposite **Hardware/app-confirmed** app-facing sign on the tested KH9;
- native `0x187B` byte 1 uses `0x01` charging and `0x02` discharging, while frozen v67 uses those two codes in the opposite direction;
- native `0x1903` is SOC-scaled nominal energy at `0.1 kWh/count`, while frozen v67 retains the earlier `nominal_energy_Wh / 200` approximation; and
- frozen v67's simple `0x1908` states and capacity-adjusted second value do not reproduce all later native forms.

These discrepancies do not invalidate the tested v67 interoperability result. They require separate controlled hardware review before any functional code change.

## Total Charged internal algorithm remains unresolved

Hardware testing establishes that:

- Fox has a direct charged path;
- v67's `0x1879` charged-capacity source can be accepted by Fox; and
- Fox can also use a discharged-derived fallback.

The complete internal calculation remains unresolved.

The current evidence does not establish:

- the exact Ah-to-kWh conversion used by Fox for the direct path;
- the exact plausibility threshold;
- any hysteresis or state logic;
- the exact fallback-selection mechanism; or
- the exact mathematical role or order of model coefficients such as `0.942` and `0.977`.

The hardware-observed fallback relationship of approximately `0.9203` is notably consistent with:

```text
0.942 × 0.977 = 0.920334
```

That numerical agreement is important evidence, but it does **not** prove that this multiplication is Fox's literal internal Total Charged formula.

See [`ENERGY-COUNTERS.md`](ENERGY-COUNTERS.md) for the complete evidence and interpretation.

## Native equivalent-cycle algorithm remains unresolved

Hardware/app testing establishes that:

- `0x1875` bytes 6–7 are accepted and displayed by Fox as Battery Cycles; and
- the v67 transmitted value moved from `0` to `1` near the expected first equivalent-cycle threshold on the real installation.

Frozen v67 source establishes that its calculation is:

```text
(charged Wh + discharged Wh) /
(2 × datalayer.battery.info.total_capacity_Wh)
```

This proves the v67 implementation formula.

It does **not** independently prove that a genuine Fox EP12 BMS internally derives its own cycle count using the identical algorithm.

`datalayer.battery.info.total_capacity_Wh` is the generic Battery-Emulator capacity input used by v67. Its exact source and semantics can vary between battery integrations, so it should not be described universally as strictly "rated energy".

## Partially decoded `0x1900` model fields

Basic interoperability does not require pretending that every Fox internal model parameter is fully understood.

Important unresolved or provisional areas include:

- `0x1901` exact field types, scales and physical semantics;
- `0x1902` word 0 exact physical meaning;
- the exact source, filtering or interpretation of the temperature-like field in `0x1902`;
- `0x1904` exact max/min location order and temperature/location packing;
- `0x1905` bytes 0–1, which are model/family-specific (`320` on old EP12 and `1240` on CQ7) rather than a universal 3.20 V nominal-cell field;
- `0x1906` parameters A and B exact physical meanings;
- the exact distinction among the `0x1905`, `0x1907` and `0x1908` SOC/SOE/model-state quantities;
- the internal flags/subfields in the expanded `0x1908` bytes 0–3 container;
- family/revision-dependent `0x1915`/`0x1916` semantics; and
- the purpose of `0x1909`.

The older exact `0x1905` byte-5/byte-6 truncation identities are also not universal native packet identities: the cross-model audit found small EP12 Plus and older-startup exceptions. Frozen v67 may enforce them as implementation choices.

The newer-family `0x1910`–`0x1919` extension is absent from old EP12 and present in the audited EP12 Plus and CQ7 traffic. Frozen v67 does not transmit it and already works on the tested KH9 without it. Whether newer inverter or manager firmware requires the extension remains **Unresolved**; `0x190A`–`0x190F` were not observed and no payloads are inferred.

v67 uses conservative, internally consistent generic mappings or stable capture-backed values where Battery-Emulator has no equivalent generic source.

These uncertainties remain visible because they are genuine evidence boundaries. They are not evidence that the already demonstrated charging, discharging and core live-data interoperability is incomplete.

See [`CAN-FRAME-MAP.md`](CAN-FRAME-MAP.md) and [`FIRMWARE-REVERSE-ENGINEERING.md`](FIRMWARE-REVERSE-ENGINEERING.md).

## Generic emulation versus native Fox model behaviour

v67 deliberately does not attempt to reproduce every internal EP12 battery-model transition.

Examples include:

- using stable generic model parameters instead of reproducing every native startup or model update;
- mirroring generic reported SOC into multiple Fox-facing higher-resolution model-state fields where Battery-Emulator has no separate generic equivalent; and
- representing the supported battery as an aggregate virtual EP battery rather than attempting to reproduce every native per-unit internal behaviour.

This is deliberate.

The standalone protocol is intended to remain battery-agnostic and consume generic Battery-Emulator values. It must not become Nissan Leaf-specific merely to imitate every value observed in one physical test battery or one genuine Fox capture.

## Hardware coverage limitation

The protocol has strong real-hardware evidence, but that evidence comes from a finite validation environment.

The principal validated installation uses:

- a FoxESS KH9 hybrid inverter;
- Battery-Emulator on Stark CMR hardware; and
- a Nissan Leaf battery as the physical test battery.

The Nissan Leaf battery is part of the validation environment, not a protocol dependency.

Current testing does not establish universal compatibility across:

- every FoxESS KH inverter model;
- every manager, master or slave firmware version;
- every regional firmware variant;
- every Battery-Emulator hardware platform;
- every supported donor-battery integration; or
- every possible single, dual or multi-EP topology.

The protocol is designed to be generic, but broader compatibility should be confirmed through additional contributor hardware and evidence before wide compatibility is claimed.

## Firmware-version coverage

Some hardware tests were recorded without a complete contemporaneous record of the exact inverter firmware versions associated with every checkpoint.

Missing version associations should not be reconstructed from memory or inferred after the fact.

Where exact firmware provenance is available, it can be added to [`TESTING-AND-EVIDENCE.md`](TESTING-AND-EVIDENCE.md).

Firmware-version coverage is therefore incomplete.

That does not invalidate the recorded hardware observations themselves; it limits how precisely those observations can be tied to particular firmware releases.

## App/cloud behaviour

FoxESS app and cloud presentation introduce their own limits to protocol testing.

In particular:

- app/cloud refresh can lag physical CAN events;
- displayed values can be quantised;
- cloud-side baselines and history can persist across Battery-Emulator resets; and
- an app value can prove what Fox accepted or displayed without revealing every internal manager or cloud calculation behind it.

An app delay is not, by itself, evidence of a protocol defect.

This project does not claim that FoxESS cloud algorithms have been fully reverse-engineered.

## Raw capture coverage

The genuine Fox capture set is strong but finite and now includes old single/dual EP12, EP12 Plus and CQ7 evidence.

Known capture coverage includes startup, charging, discharge and dual-unit behaviour.

Some fields remain unresolved because the available native captures did not include:

- every possible SOC, temperature or power region;
- fault conditions;
- service or replacement states;
- controlled native EP12 power-cycle persistence testing; or
- every possible multi-unit configuration.

These gaps are evidence boundaries. They do not justify manufacturing unobserved behaviour or treating unresolved fields as necessary for basic interoperability.

## Known behaviour that is NOT a mapping failure

| Behaviour | Current interpretation |
| --- | --- |
| Total Charged rises during physical discharge while Daily Charged is static | Likely Fox fallback behaviour, not proof that current sign is wrong |
| Total Charged later returns to fallback after the direct path previously worked | Source-selection/plausibility behaviour, not proof that `0x1879` failed |
| Battery Cycles stays at `0` for a long time | Expected until the whole equivalent-cycle threshold is crossed |
| `0x1879` discharged half remains zero in a short genuine native discharge capture | Insufficient transferred charge to cross the expected `0.1 Ah` threshold |
| App values lag physical CAN activity | Cloud/app update timing can delay presentation |
| Negative daily values appear after reboot/reflash | Volatile local cumulative history versus retained Fox context, not a newly broken frame map |
| Native `0x190x` values differ slightly from v67 stable generic values | Expected generic emulation where no equivalent Battery-Emulator generic source exists |
| Native current sign or `0x187B` direction codes differ from frozen v67 | Documented fidelity discrepancy requiring controlled hardware review, not a reason to rewrite the successful KH9 history |

## Deferred upstream integration items

| Item | Why deferred | Release significance |
| --- | --- | --- |
| Persistent cumulative history | Requires integration with wider Battery-Emulator persistence architecture rather than an isolated documentation-phase design | **Release blocker** for a polished upstream release |
| Migration and continuity across firmware updates | Fox-facing cumulative history must remain coherent across upgrades, but the mechanism belongs to later integration design | Part of resolving the persistence release blocker |
| Broader hardware validation | Current hardware evidence is strong but finite | Desirable before claiming wide compatibility |
| Controlled review of native-v67 current and `0x187B` direction differences | Functional changes could affect the tested KH9 behaviour and must not be inferred from captures alone | Hardware-review requirement |
| Newer-family extension compatibility | It is unresolved whether newer inverter/manager firmware requires `0x1910`–`0x1919` | Compatibility evidence gap; not required on the tested KH9 |
| Firmware-version provenance for recorded tests | Some checkpoint-to-firmware associations were not recorded contemporaneously | Documentation-quality improvement |
| Remaining model-field semantics | Several `0x190x` fields remain Provisional or Unresolved | Reverse-engineering improvement; not necessarily an interoperability blocker |

## What is NOT currently deferred

The following behaviour has already been demonstrated in v67 on the validated installation:

- the inverter recognises the virtual EP battery;
- contactors operate;
- charging works;
- discharging works;
- generic readiness logic works;
- core live data and battery limits operate;
- `0x187A` directional charged and discharged energy works;
- `0x1878` absolute-throughput mapping is established;
- the `0x1879` charged-capacity/direct Total Charged path works;
- the Battery Cycles field/display works through the first observed `0 → 1` transition; and
- the protocol remains battery-agnostic in design.

These statements describe the validated installation only and should not be expanded into unsupported claims of universal hardware or firmware compatibility.

## Release-readiness summary

| Area | Status |
| --- | --- |
| Core EP communication | **Hardware-proven** |
| Charging/discharging | **Hardware-proven** |
| Directional energy | **Hardware/app-confirmed** |
| Absolute throughput | **Capture-confirmed and implemented** |
| Independent charged-capacity path | **Hardware/app-confirmed** |
| Discharged-capacity native proof | **Capture-confirmed** |
| Equivalent-cycle field/display | **Hardware/app-confirmed** |
| Native Fox cycle formula | **Unresolved** |
| Total Charged fallback behaviour | **Hardware/app-confirmed** |
| Exact fallback/direct selection algorithm | **Unresolved** |
| Counter persistence | **Not implemented / release blocker** |
| Broad inverter/firmware coverage | **Additional validation desirable** |
| Native-v67 current/`0x187B` fidelity | **Controlled hardware review required** |
| Newer-family `0x1910`–`0x1919` requirement | **Unresolved** |
| Battery-agnostic design | **Preserved** |

v67 is a strong hardware-proven reverse-engineering checkpoint.

The remaining major release-quality problem is cumulative-history persistence, while several deeper Fox internal-model semantics remain intentionally unresolved rather than being overstated.
