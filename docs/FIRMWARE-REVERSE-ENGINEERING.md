# FoxESS EP-Series Manager Firmware Reverse Engineering

## Purpose and legal/evidence boundary

This document records the manager-firmware-derived evidence used during the independent community reverse engineering of the FoxESS EP-Series CAN interface. Its purpose is to explain the parser structures and numerical relationships that strengthened the public protocol map; it is not a firmware-distribution document or an official FoxESS specification.

The repository applies the following practical boundaries:

- No proprietary FoxESS firmware binary is distributed by this repository.
- Only derived protocol findings needed to explain interoperability are recorded here.
- Firmware analysis is used primarily to support byte boundaries, integer widths, endianness and cross-field relationships.
- Genuine EP12 CAN captures and real inverter/app behaviour remain higher-priority evidence.
- Reverse-engineered names such as “model parameter A”, “fine state” or “capacity-adjusted state” are neutral project labels, not official FoxESS terminology.
- A parsed field is not treated as semantically solved merely because its load instruction or storage width is visible.
- No claim is made that every branch, internal variable or native battery-model calculation has been fully understood.

The evidence labels used throughout this document are:

| Label | Meaning |
| --- | --- |
| **Hardware/app-confirmed** | Observed on the real inverter, battery installation and/or FoxESS app during a controlled test |
| **Capture-confirmed** | Directly observed in genuine EP12 CAN traffic |
| **Firmware-supported** | Supported by the analysed manager firmware's parsing, data flow or field use |
| **Strongly inferred** | Multiple evidence sources agree, but decisive direct proof is still missing |
| **Provisional** | A useful interpretation that remains open to correction |
| **Unresolved** | The available evidence does not justify a reliable semantic claim |

Evidence priority is:

1. real hardware/app behaviour;
2. genuine EP12 CAN captures;
3. manager-firmware analysis;
4. the frozen v67 implementation; and
5. historical comments or assumptions.

Firmware-derived interpretations are rejected or weakened whenever stronger capture or hardware evidence disagrees.

## Why firmware analysis was useful

CAN captures show what crossed the bus, but a changing eight-byte payload does not automatically reveal how the receiver interprets it. Short captures were especially capable of making numeric low bytes look like status flags.

Manager-firmware analysis helped answer structural questions that captures alone could not settle reliably:

- whether adjacent bytes form independent values or one multi-byte integer;
- the width and little-endian grouping of parsed quantities;
- whether a frame contains two complete fields rather than unrelated reserved bytes;
- whether coarse and fine values in different frames feed the same internal model quantity;
- whether a value is treated as model/configuration data or live telemetry;
- whether two representations are likely to be used by the same manager-side model; and
- whether an apparent state byte is actually the least-significant byte of a changing numeric field.

This was particularly important for `0x1878`, `0x1879` and the `0x1900`-series model frames. Firmware structure narrowed the valid interpretations; captures and hardware then established which remaining interpretation matched native and real-world behaviour.

## Method

The analysis workflow was intentionally evidence-led and kept separate from firmware redistribution:

1. Identify manager-side CAN receive and parsing paths associated with specific EP response IDs.
2. Record only the field accesses, widths, byte groupings and relationships relevant to interoperability.
3. Compare those boundaries with genuine single- and dual-EP12 CAN captures.
4. Check whether candidate interpretations remain consistent across startup, idle, charging, discharging and model-state changes.
5. Implement only the best-supported mapping in Battery-Emulator.
6. Test important hypotheses on the real inverter and FoxESS app.
7. Reject an interpretation when a longer capture or controlled hardware experiment contradicts it.

This document does not describe how to extract protected firmware, bypass device security or reproduce proprietary decompilation. It also avoids large decompiled functions, raw firmware dumps and unnecessary disassembly.

## Firmware-derived evidence versus proof

| Finding type | Firmware can support | Additional proof required |
| --- | --- | --- |
| Field width and endianness | Loads, masks, shifts and storage operations can strongly establish byte grouping and integer width | A second firmware build or capture comparison may still be useful where compiler output is ambiguous |
| Two fields feeding related model state | Data flow can show that values enter the same manager-side model or that one is a coarse copy of another | Captures should demonstrate the numerical relationship across changing values |
| Official physical meaning | Firmware can narrow the role from structure and downstream use | Genuine captures, model comparisons, hardware behaviour or independent documentation are normally required |
| Internal formula | A complete and unambiguous control/data-flow path may establish a calculation | Nearby constants, similar ratios or partial decompilation are insufficient by themselves |
| Cloud or app behaviour | Firmware may show a candidate source path | A real inverter/app test is required to establish what is accepted, displayed or selected |
| Native battery algorithm | Manager parsing can show what the inverter consumes | It does not prove how a native EP12 generated the value |
| v67 calculation | Source inspection establishes exactly what v67 sends | It does not prove that Fox's native battery or manager uses the same formula internally |

This distinction is central to the document. “Firmware-supported” means that the decoded structure or relationship is real evidence; it does not silently promote a project label into an official FoxESS field name.

## Main EP frame parsing findings

### `0x1878`

The analysed manager parsing supports the following field structure:

| Bytes | Firmware-supported structure |
| --- | --- |
| 0 | Separate byte |
| 1 | Separate byte |
| 2–3 | One separate 16-bit field |
| 4–7 | One 32-bit little-endian field |

This structure was important because an early short charging capture contained `20 00 00 00` in bytes 4–7. When read visually as separate bytes, byte 4 could be mistaken for a fixed charging flag. The manager instead consumes the four bytes as one number. The same bytes decode to decimal 32, consistent with approximately 32 Wh transferred at that point.

The evidence layers are deliberately separated:

| Finding | Evidence status |
| --- | --- |
| Bytes 4–7 are one 32-bit little-endian-style field | **Firmware-supported** |
| Byte 1 tracks individual-unit SOC in whole percent | **Capture-confirmed** |
| Bytes 4–7 behave as cumulative absolute bidirectional throughput at 1 Wh/count | **Capture-confirmed**, with firmware-supported field structure |
| Sending charged-only Wh in bytes 4–7 did not establish an independent Total Charged source | **Hardware/app-confirmed** by the v66 experiment |

The final semantic interpretation therefore did not come from firmware alone. The longer genuine EP12 capture showed the field rising toward approximately 240 while measured transfer was approximately 223 Wh charged plus 23 Wh discharged. That established the best-supported meaning as cumulative absolute throughput. The v66 hardware experiment then independently rejected the idea that this field was Fox's direct Total Charged source.

### `0x1879`

The analysed manager parsing treats the eight-byte payload as two 32-bit fields. That finding established that `0x1879` was not merely eight unrelated or permanently reserved bytes.

The stronger evidence was then supplied by genuine traffic and hardware:

| Finding | Evidence status |
| --- | --- |
| Payload is parsed as two 32-bit fields | **Firmware-supported** |
| Bytes 0–3 progress only during charging in the genuine dual-EP12 capture | **Capture-confirmed** |
| Bytes 0–3 represent cumulative charged capacity at 0.1 Ah/count | **Capture-confirmed**; the threshold pattern supports the scale exceptionally strongly |
| Fox accepted the v67 charged-capacity path as an independent charged source | **Hardware/app-confirmed** |
| Bytes 4–7 are the matching cumulative discharged-capacity field at 0.1 Ah/count | **Strongly inferred** pending a native discharge transition |

In the genuine dual-unit charge, bytes 0–3 progressed `0 → 1 → 2 → 3 → 4`. Paired transitions occurred near the energy required for two parallel EP12 units to cross successive 0.1 Ah boundaries. The available native discharge span was only about 23 Wh, so it did not cross the expected first discharged-capacity threshold.

The manager structure supports the directional pair, but it does not by itself establish Fox's exact Ah-to-kWh conversion. Hardware proved that the charged path is accepted: once sufficient v67 charged history existed, Total Charged remained at 19.40 kWh while Total Discharged increased from 11.20 to 11.50 kWh. The manager's exact voltage, calibration and model calculation remains **Unresolved**.

### Other main-frame relationships

Only relationships that materially support the approved frame map are summarised here:

| Relationship | Contribution of firmware analysis | Stronger corroboration |
| --- | --- | --- |
| `0x187B` contains separate rated/design and full/effective capacity fields | Supports capacity-oriented field grouping and manager use | Single/dual EP12 captures and app behaviour establish the practical capacity roles |
| `0x1873` bytes 6–7 and `0x1900` bytes 2–3 carry the same fine nominal-energy representation | Parser boundaries are consistent with separate 16-bit quantities | The matching native values and single/dual scaling are **Capture-confirmed** |
| `0x1903` bytes 6–7 carry a coarser form of the same installed nominal energy | Manager-side structure supports a separate coarse model input | Raw 57/115 single/dual behaviour is **Capture-confirmed** |

The complete main-frame byte map remains in [CAN-FRAME-MAP.md](CAN-FRAME-MAP.md). This document records only the firmware-derived contribution to those conclusions.

## `0x1900`-series overview

The extended group mixes installed-capacity descriptions, static model data, live capability values and several related battery-state quantities. It should not be treated as ten copies of one data type.

| Frame | Firmware/capture role | Strongest supported interpretation | Evidence status |
| --- | --- | --- | --- |
| `0x1900` | Capacity/energy summary | Capacity-related value, fine installed nominal energy and fixed trailing model bytes | Structure and scaling **Capture-confirmed**; parser use **Firmware-supported**; trailing meaning **Unresolved** |
| `0x1901` | Static model/calibration payload | Fixed model-specific data | Payload **Capture-confirmed**; candidate types **Provisional**; semantics **Unresolved** |
| `0x1902` | Four-word capability/model frame | Model coefficient-like value, maximum discharge power, resistance-like value and temperature-like value | Word structure **Firmware-supported**; meanings range from confirmed to provisional |
| `0x1903` | Coarse installed-energy input | Installed nominal energy at 200 Wh/count in bytes 6–7 | **Capture-confirmed**; parser relationship **Firmware-supported** |
| `0x1904` | Extreme-measurement locations | Two cell-extreme positions and two temperature/location-style codes | Structure **Strongly inferred**; raw behaviour **Capture-confirmed**; order/packing **Unresolved** |
| `0x1905` | Compact model/state summary | Model reference, several coarse SOC/SOE-like values and a model-state byte | Cross-frame relationships **Capture-confirmed**; official names **Unresolved** |
| `0x1906` | Model parameters A and B | Two slowly changing per-unit/model quantities separated by zero words | Structure and values **Capture-confirmed** and **Firmware-supported**; physical meanings **Unresolved** |
| `0x1907` | High-resolution model state | Two 32-bit battery-state slots, with the second feeding `0x1905` byte 5 | Structure **Firmware-supported**; relationship **Capture-confirmed**; first meaning **Unresolved** |
| `0x1908` | Model phase and adjusted state | Model/status state plus fine capacity-adjusted SOC/SOE-like quantity | Structure/relationship **Capture-confirmed** and **Firmware-supported**; official names **Unresolved** |
| `0x1909` | Present all-zero slot | Zero-filled final frame in the extended group | Wire behaviour **Capture-confirmed**; purpose **Unresolved** |

## `0x1900` — capacity / nominal-energy summary

The supported payload structure is:

| Bytes | Best-supported interpretation | Scale/status |
| --- | --- | --- |
| 0–1 | Capacity-related value associated with the represented unit/model | 0.1 Ah/count; **Capture-confirmed** scale, scope partly **Provisional** |
| 2–3 | Nominal installed energy | 10 Wh/count; **Capture-confirmed** |
| 4–7 | Fixed/model trailing field | Raw behaviour **Capture-confirmed**; type and meaning **Unresolved** |

Native examples include:

- one EP12: `2C 01 80 04 00 00 3C 46`;
- two EP12 units: `2C 01 00 09 00 00 3C 46`.

The energy field doubles from raw 1152 to 2304, representing 11.52 kWh and 23.04 kWh. By contrast, the native capacity-related field in bytes 0–1 remains consistent with per-unit behaviour in the available evidence. Frozen v67 represents the attached Battery-Emulator system as one aggregate virtual battery where appropriate and therefore sends an aggregate rated-capacity value. That is a deliberate emulation choice, not a claim that every native multi-EP system uses an aggregate value in this field.

If bytes 4–7, `00 00 3C 46`, are interpreted as a little-endian IEEE-754 float, they produce 12032.0. This is only a **Provisional** interpretation aid. The float type, physical unit and official meaning are not established.

## `0x1901` — static model/calibration-like data

The observed payload is:

`F0 0F 00 00 40 00 33 42`

It remains unchanged across the supplied startup, charging, discharging and single/dual-unit evidence. Its wire behaviour as static model/calibration-like data is therefore well established.

Candidate numerical decodings include:

- bytes 0–3 as a little-endian integer: 4080;
- bytes 4–7 as a little-endian IEEE-754 float: approximately 44.750244.

These are **Provisional** decoding aids only. The exact field boundaries, data types, scales and physical semantics remain **Unresolved**. The stable payload should not be labelled as live voltage, current, energy, SOC or a cumulative counter merely because one candidate numerical interpretation resembles an engineering value.

## `0x1902` — model coefficient / discharge capability / resistance / temperature

The strongest structural interpretation is four 16-bit little-endian words. A settled example is:

`AE 03 39 2C 7C 03 1C 00`

which decodes to raw words 942, 11321, 892 and 28.

| Word / bytes | Strongest current interpretation | Evidence boundary |
| --- | --- | --- |
| Word 0, bytes 0–1 | Model/capacity/efficiency-like coefficient; settled value 942 | Raw value **Capture-confirmed**; model role **Firmware-supported**; physical name **Provisional** |
| Word 1, bytes 2–3 | Maximum permitted discharge power in watts | Mapping **Capture-confirmed** and **Firmware-supported** |
| Word 2, bytes 4–5 | Equivalent-resistance-like model value; settled value 892 | Role **Firmware-supported** and **Strongly inferred**; best-supported scale 0.1 mΩ/count |
| Word 3, bytes 6–7 | Filtered or model-temperature-like whole-degree value | Temperature relationship **Strongly inferred**; exact native sensor/filter source **Unresolved** |

Word 0 must not be given an invented official name. Interpreting 942 as 0.942 is useful in the later model-factor comparison, but does not prove whether Fox calls it an efficiency, capacity, health or another correction coefficient.

For word 2, the single/dual behaviour and equivalent-circuit relationships support 0.1 mΩ/count as the best current scale, making raw 892 equivalent to 89.2 mΩ. This remains a best-supported reverse-engineered interpretation rather than an official specification.

Frozen v67 sends the discharge-power word from the generic Battery-Emulator maximum-discharge-power value only when the battery is ready and maximum discharge current is non-zero; otherwise it sends zero. It also clamps the transmitted value to the 16-bit field. This readiness/limit gating is a v67 source fact layered on top of the reverse-engineered field mapping.

## `0x1903` — coarse installed nominal energy

Available native traffic and frozen v67 use:

- bytes 0–5: zero;
- bytes 6–7: installed nominal energy at 200 Wh/count.

Native examples are raw 57 for one EP12 and raw 115 for two:

- 57 × 200 Wh = approximately 11.4 kWh;
- 115 × 200 Wh = approximately 23.0 kWh.

The fine nominal-energy representation is 11.52 kWh for one unit and 23.04 kWh for two. Integer quantisation therefore explains the apparent 57-to-115 change:

- 1152 fine counts ÷ 20 = 57 remainder 12;
- 2304 fine counts ÷ 20 = 115 remainder 4.

This is consistent with calculating the coarse value after system aggregation. It does not require the dual raw value to be exactly `57 × 2 = 114`. The relationship is **Capture-confirmed**, with the manager-side parsing relationship **Firmware-supported**.

Bytes 0–5 were zero in the available evidence. They should not be described as universally unused by every Fox battery or firmware build.

## `0x1904` — extreme-value locations

The best-supported structure is:

| Bytes | Best-supported role | Status |
| --- | --- | --- |
| 0–1 | First cell-voltage extreme position/location | **Strongly inferred**; native max/min order **Unresolved** |
| 2–3 | Second cell-voltage extreme position/location | **Strongly inferred**; native max/min order **Unresolved** |
| 4 | First temperature-extreme or sensor/location-style code | **Provisional** |
| 5 | Second temperature-extreme or sensor/location-style code | **Provisional** |
| 6–7 | Zero in current evidence | **Capture-confirmed** zero; purpose **Unresolved** |

The first two fields behave like 1-based cell positions. They complement the extreme values sent elsewhere, but the available evidence does not decisively prove which native field is maximum and which is minimum.

Bytes 4–5 contain location/address-style values rather than ordinary temperatures. Examples include `0x3A` in byte 4 and `0x0A` or `0x12` in byte 5 for observed single/dual configurations, with other transitional values during startup. Their nibble packing, battery-unit selection, thermistor numbering and max/min order remain **Unresolved**.

Frozen v67 places its generic virtual maximum- and minimum-voltage-cell positions into bytes 0–3 and uses capture-backed generic location codes in bytes 4–5. That does not claim that native Fox sensor numbering has been fully decoded.

## `0x1905` — coarse model/SOC state

A representative native operational payload is:

`40 01 32 08 61 32 2F 00`

The best-supported relationships are:

| Bytes | Relationship | Evidence boundary |
| --- | --- | --- |
| 0–1 | Static/model reference raw 320 | Raw value **Capture-confirmed**; a 3.20 V-like interpretation is **Strongly inferred**, not official naming |
| 2 | Primary internal SOC-like whole-percent value | **Capture-confirmed** relationship; exact source distinction **Unresolved** |
| 3 | Operating/model-state byte, including startup and valid-state changes such as `0x04` and `0x08` | Wire states **Capture-confirmed**; official state names **Unresolved** |
| 4 | Coarse form of `0x1906` parameter B divided by 10 | **Capture-confirmed** cross-frame relationship |
| 5 | Coarse form related to the second `0x1907` high-resolution state divided by 10 | **Capture-confirmed** |
| 6 | Coarse form related to the `0x1908` fine capacity-adjusted state divided by 10 | **Capture-confirmed** |
| 7 | Zero | **Capture-confirmed** |

The exact semantic difference between byte 2 and byte 5 is not fully resolved. Both are battery charge/energy-state-like values, but they can update differently. This document therefore avoids declaring one to be an official “SOC” and the other an official “SOE”.

Likewise, interpreting raw 320 as a 3.20 V model reference is plausible and strongly supported by the scale, but “nominal cell voltage” remains a project interpretation rather than a Fox-published field name.

## `0x1906` — model parameters A and B

The payload is structured as:

| Bytes | Field |
| --- | --- |
| 0–1 | Parameter A |
| 2–3 | Zero |
| 4–5 | Parameter B |
| 6–7 | Zero |

Native observations include:

- parameter A around 80 or 81 in settled operation;
- parameter A around 46 or 47 during startup;
- parameter B around 970 or 977 depending on the captured model state.

Two conclusions are stronger than any proposed physical name:

1. Parameter B is not a simple charge/discharge flag. The value changes too slowly, persists across direction changes and can occur in more than one operating direction.
2. `0x1905` byte 4 is the integer coarse form of parameter B divided by 10. Both 970 and 977 therefore appear as 97 in the coarse field.

Parameter A appears equivalent-circuit or resistance related in the available model behaviour, and parameter B appears associated with a capacity/efficiency/resistance model. Those physical descriptions remain **Provisional**. The official names, scales and precise state-transition triggers are **Unresolved**.

Frozen v67 deliberately sends stable generic values, parameter A = 81 and parameter B = 970, because Battery-Emulator does not expose generic equivalents of the native dynamic Fox model parameters. It keeps `0x1905` byte 4 consistent with parameter B. This does not reproduce every native startup or slow-update transition and is not claimed to do so.

## `0x1907` — high-resolution model state

The manager-supported layout is two 32-bit slots:

| Bytes | Best-supported role | Status |
| --- | --- | --- |
| 0–3 | First high-resolution voltage/model-sensitive battery-state quantity | Structure **Firmware-supported**; scale **Strongly inferred**; exact semantic **Unresolved** |
| 4–7 | Second high-resolution SOC/SOE-style battery-state quantity | Structure **Firmware-supported**; 0.1%-style scale and coarse relationship **Capture-confirmed** |

The observed native values occupy the lower active portions of each slot, but the parser boundaries support complete 32-bit fields. The second value has a particularly strong relationship to `0x1905` byte 5:

`0x1905 byte 5 = floor(0x1907 second state / 10)`

The first value is more sensitive to voltage/model changes and can differ from the second. Candidate descriptions include voltage-derived SOC, OCV-related state or SOE, but the exact Fox semantic remains **Unresolved**.

Frozen v67 mirrors generic reported SOC into both model-state fields because Battery-Emulator does not expose separate generic equivalents for every native Fox internal state variable. That interoperability choice does not make the two native values semantically identical to SOC.

## `0x1908` — model state / fine capacity-adjusted quantity

The best-supported layout is:

| Bytes | Best-supported role | Status |
| --- | --- | --- |
| 0–3 | Model/status state | Field relationship **Firmware-supported** and **Capture-confirmed**; official states **Unresolved** |
| 4–7 | Fine capacity-adjusted SOC/SOE-like quantity | 0.1%-style relationship **Capture-confirmed**; physical name **Strongly inferred** |

Native/current evidence includes model-state values 0, 14 and 16 under different startup and operating conditions. Descriptions such as “initialising” and “operational” are useful project labels, not official Fox enumerations.

The exact cross-frame relationship is:

`0x1905 byte 6 = floor(0x1908 fine state / 10)`

That relationship is much stronger than the precise physical name of the fine quantity. Frozen v67 derives a capacity-adjusted state from its generic fine SOC and full/rated-capacity relationship, then uses model states 14 and 16 according to its own readiness model. It does not reproduce the observed native state 0 in every circumstance.

## `0x1909` — zero / unresolved

`0x1909` has DLC 8 and an all-zero payload in all available genuine EP12 captures:

`00 00 00 00 00 00 00 00`

Frozen v67 also transmits eight zero bytes. The wire behaviour is **Capture-confirmed**. Its semantic purpose is **Unresolved**. It may be an unused extension slot, a field populated only in conditions not captured, or a slot used by another model or firmware version; none of those possibilities is promoted to a confirmed role.

## Cross-frame model relationships

| Relationship | What is established | Evidence status |
| --- | --- | --- |
| `0x1905` byte 4 = `floor(0x1906 parameter B / 10)` | Raw 970 and 977 both produce coarse 97 | **Capture-confirmed**; parser relationship **Firmware-supported** |
| `0x1905` byte 5 = `floor(0x1907 second state / 10)` | Fine second state feeds the coarse byte | **Capture-confirmed** and **Firmware-supported** |
| `0x1905` byte 6 = `floor(0x1908 fine state / 10)` | Fine capacity-adjusted state feeds the coarse byte | **Capture-confirmed** and **Firmware-supported** |
| `0x1873` bytes 6–7 = `0x1900` bytes 2–3 | Same installed nominal energy at 10 Wh/count | **Capture-confirmed**; field boundaries **Firmware-supported** |
| `0x1903` bytes 6–7 = coarse form of the same nominal energy | Installed nominal energy at 200 Wh/count | **Capture-confirmed**; parser role **Firmware-supported** |
| Single/dual `0x1900` energy | Raw 1152 becomes 2304 | **Capture-confirmed** system aggregation |
| Single/dual `0x1903` energy | Raw 57 becomes 115 after coarse quantisation | **Capture-confirmed** system aggregation |
| Native `0x1900` bytes 0–1 versus system energy fields | Capacity-related value appears per-unit while energy fields aggregate | **Capture-confirmed** in available EP12 evidence; universal scope **Provisional** |

These relationships explain why apparently redundant values can coexist: the manager receives fine and coarse forms, per-unit and system-level quantities, and multiple internal model-state estimates. They do not prove the official Fox name of every consumer variable.

## The 942 / 970 / 977 model-factor finding

This numerical relationship materially influenced the Total Charged investigation, but its limits are important.

Relevant genuine Fox observations include:

- `0x1902` word 0 at 942 in settled model data;
- `0x1906` parameter B at 977 in relevant genuine charging data; and
- `0x1906` parameter B at 970 in other startup, discharge and model states.

Interpreting the two relevant values as permille-style coefficients gives:

`0.942 × 0.977 = 0.920334`

The independently observed hardware/app fallback ratio was approximately 0.9203. For example:

| Observation | Ratio |
| --- | ---: |
| 8.90 / 9.67 | 0.920372 |
| 9.10 / 9.89 | 0.920121 |
| 0.942 × 0.977 | 0.920334 |

The approved conclusion is:

> The observed fallback relationship is notably consistent with the product of the two observed Fox model coefficients.

It does not establish that the manager literally multiplies those two coefficients when calculating or selecting the fallback. The exact sequence of operations, parameter meanings, source-selection rule, switching threshold, hysteresis and fallback calculation remain **Unresolved**.

The fallback behaviour itself is **Hardware/app-confirmed**. Its association with the model coefficients is **Strongly inferred**. See [ENERGY-COUNTERS.md](ENERGY-COUNTERS.md) for the controlled hardware observations and the direct-versus-fallback evidence.

Frozen v67 uses a fixed parameter B value of 970. It therefore does not necessarily reproduce the native 977 value present in the capture used for the numerical comparison. The comparison documents native evidence; it is not a claim about the exact product of v67's fixed values.

## What firmware analysis disproved

| Rejected interpretation | Why it was rejected | Evidence |
| --- | --- | --- |
| `0x1878` byte 4 is an independent charging flag | The manager parses bytes 4–7 as one 32-bit field; longer capture behaviour establishes the final throughput meaning | Structure **Firmware-supported**; meaning **Capture-confirmed** |
| `0x1879` is eight unused or reserved bytes | The manager parses two 32-bit fields, and the first field progresses during genuine charging | **Firmware-supported** plus **Capture-confirmed** |
| 970 and 977 are simple discharge/charge states | They persist across direction changes, change slowly with model state and have a coarse cross-frame relationship to `0x1905` byte 4 | **Capture-confirmed** behaviour; **Firmware-supported** relationship |

Hardware later supplied an additional rejection: the v66 charged-only `0x1878` experiment did not create Fox's independent Total Charged path. That is a **Hardware/app-confirmed** conclusion rather than a firmware-only finding.

## Firmware findings intentionally not promoted to confirmed semantics

Successful emulation does not require pretending that every native Fox internal variable has been decoded. The following remain deliberately limited:

- exact official names and field types for the `0x1901` parameters;
- exact physical meaning of `0x1902` word 0;
- exact native sensor/filter source for `0x1902` word 3;
- exact official meaning and scale of `0x1906` parameters A and B;
- exact distinction among the several `0x1905`, `0x1907` and `0x1908` SOC/SOE/model quantities;
- exact native max/min order and temperature/location packing in `0x1904`;
- the purpose of `0x1909`;
- exact manager conversion from `0x1879` cumulative Ah to displayed charged kWh;
- exact Total Charged direct/fallback selection algorithm;
- exact native Fox equivalent-cycle calculation.

The v67 source proves its own transmitted-cycle formula, and hardware proves that the Fox Battery Cycles field/display accepted the first v67 0-to-1 transition. Neither establishes Fox's native internal cycle algorithm.

## Firmware provenance and reproducibility

Public reverse-engineering notes should preserve useful provenance without redistributing proprietary material:

- Record the firmware family and version used for analysis when they are genuinely known.
- Associate derived notes with the analysed build where possible.
- Retain cryptographic hashes only where doing so is appropriate and legally safe.
- Do not redistribute FoxESS firmware binaries through this repository.
- Do not publish large decompiled functions or raw disassembly when a derived field description is sufficient.
- Do not imply that a finding applies to every Fox inverter, battery model or manager-firmware version without evidence.
- Cross-check manager-derived findings against genuine CAN captures and real hardware wherever possible.
- Where the source platform, build or version is uncertain, use the generic description “the analysed Fox manager firmware” rather than guessing.

The physical hardware-validation platform is documented separately in [TESTING-AND-EVIDENCE.md](TESTING-AND-EVIDENCE.md). Hardware platform identity must not be used to infer the provenance of an analysed firmware image unless that link is independently established.

## Relationship to the hardware-proven v67 implementation

Frozen v67 uses firmware findings selectively.

Some fields reproduce native structure closely:

- `0x1878` preserves the manager-supported field boundaries and sends capture-confirmed absolute throughput;
- `0x1879` sends the manager-supported two-field directional-capacity structure;
- `0x1902` preserves four 16-bit words and uses the confirmed discharge-power slot;
- `0x1903` uses the capture-confirmed 200 Wh installed-energy representation;
- `0x1905`–`0x1908` preserve the established coarse/fine cross-frame relationships; and
- `0x1909` remains zero.

Other values are generic emulation choices:

- v67 represents the attached Battery-Emulator system as one aggregate virtual battery where appropriate;
- it maps generic reported SOC into multiple Fox-facing model-state fields because separate generic OCV/SOE/model values are not available;
- it derives a generic capacity-adjusted state for `0x1908`;
- it uses stable captured values for native model parameters that Battery-Emulator cannot derive generically; and
- it applies generic readiness and battery-limit gating to the `0x1902` discharge-power value.

This is deliberate. A battery-agnostic implementation should not hard-code Nissan Leaf behaviour, Leaf-only capacity assumptions, Leaf temperature arrays or other source-battery-specific state merely to imitate every native EP12 internal variable.

Hardware success validates the resulting interoperability: contactors, charging, discharging, Battery Details reporting, the independent charged path and the first v67 cycle transition all operated on the real system. The field-by-field evidence labels still preserve the distinction between native Fox meaning, capture-backed wire behaviour and a generic Battery-Emulator mapping choice.

## Remaining firmware questions

The following reverse-engineering questions remain genuinely unresolved:

- What exact voltage, averaging, calibration or model calculation converts `0x1879` charged capacity into displayed Total Charged kWh?
- What exact comparison, threshold, hysteresis and state logic select the direct Total Charged source or the fallback?
- What is the exact physical meaning of `0x1902` word 0?
- What are the exact physical meanings and state-transition trigger for `0x1906` parameters A and B?
- How are the `0x1904` temperature/location codes packed, and what is the decisive native max/min order?
- What is the semantic distinction among the `0x1905`, `0x1907` and `0x1908` battery-state quantities?
- What are the exact `0x1901` field types, scales and semantics?
- What is the purpose of `0x1909`, including whether it becomes non-zero in unobserved fault or service conditions?

These are evidence gaps, not a protocol-development roadmap.

## Summary

Manager-firmware analysis materially strengthened the FoxESS EP-Series reverse engineering by establishing parser structure and cross-frame relationships, especially for `0x1878`, `0x1879` and the `0x1900` model frames.

It showed, among other things, that `0x1878` bytes 4–7 form one 32-bit field, that `0x1879` is parsed as two 32-bit fields, and that several coarse values in `0x1905` correspond to finer model values in `0x1906`, `0x1907` and `0x1908`.

Firmware analysis is not treated as an official specification. Final mappings are accepted only where they agree with the stronger evidence from genuine EP12 CAN captures and/or real inverter/app testing. Values whose official physical meanings remain unknown are kept **Provisional** or **Unresolved**, even when frozen v67 can emulate them successfully.
