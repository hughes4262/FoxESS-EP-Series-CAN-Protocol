# FoxESS EP-Series CAN Frame Map

## Scope and evidence rules

This document maps the frozen, hardware-proven v67 FoxESS EP-Series CAN implementation. The source checkpoint used for this draft is:

- `FOXESS-EP-CAN.cpp`: SHA-256 `15afba8e189bd572a9e45b4ef193473c9706d22f62feded5cc362427f6ad9516`
- `FOXESS-EP-CAN.h`: SHA-256 `0ec370e52427b57d51d23916407f27a444ce8d25c560f90ac790fb8aecbf4b0d`

The inventory below comes from those source files, not from an older handoff frame list. Frozen v67 transmits 63 distinct FoxESS EP CAN identifiers and receives/handles one identifier, `0x1871`.

This is reverse-engineered community documentation. It is not an official FoxESS protocol specification. The evidence base combines genuine EP12 CAN captures, manager-firmware-derived findings, frame audits, and real inverter/app tests. No proprietary firmware is reproduced here.

A frame can work correctly on real hardware even when the official Fox name or purpose of one of its fields remains unknown. Confidence is therefore assigned field-by-field. The `Overall confidence` column in the summary is only a navigation aid and never overrides the detailed tables.

When evidence conflicts, this document applies the following priority:

1. Real hardware behaviour
2. Genuine Fox CAN captures
3. Manager-firmware analysis
4. The frozen v67 implementation
5. Historical comments and older theories

The evidence labels mean:

| Label | Meaning in this document |
|---|---|
| **Hardware/app-confirmed** | A controlled real inverter, battery, or app test demonstrated the stated behaviour or displayed result. |
| **Capture-confirmed** | Genuine Fox traffic directly establishes the byte position, value, transition, relationship, or scale stated. |
| **Firmware-supported** | Manager-firmware analysis supports the field boundary, decoding, or use; the official Fox field name may still be unknown. |
| **Strongly inferred** | Multiple independent observations support the interpretation, but a decisive native transition or isolated hardware test is still missing. |
| **Provisional** | A useful implementation choice or working interpretation with limited supporting evidence. It must not be treated as an official Fox definition. |
| **Unresolved** | The wire bytes are known, but their semantic purpose or exact decoding is not. |

“v67 implementation” describes what the frozen code sends. “Native EP12 evidence” describes what genuine Fox hardware was observed to send. They are intentionally separated where v67 maps a generic Battery-Emulator value onto a field that was fixed, model-specific, or differently scoped on the captured EP12.

## Conventions

- CAN identifiers are written in hexadecimal, for example `0x1873`.
- All inventory entries are 29-bit extended-ID, Classical CAN frames (`ext_ID = true`, `FD = false`).
- Every transmitted frame constructed by v67 has an 8-byte payload (`DLC = 8`); the genuine `0x1871` forms mapped here also use DLC 8. The receive handler assumes the selector bytes it reads are present.
- Payload bytes are numbered `0` through `7`.
- Multi-byte integers are little-endian unless a field explicitly says otherwise: the least-significant byte is transmitted first.
- `uint16`, `int16`, and `uint32` denote unsigned 16-bit, signed two's-complement 16-bit, and unsigned 32-bit integers respectively.
- A scale such as `0.1 V/count` means `engineering value = raw integer × 0.1 V`.
- v67 integer divisions truncate unless a formula explicitly includes rounding. Values described as saturated or clamped stop at the stated wire-type limit.
- `dV`, `dA`, `dC`, `dAh`, and `pptt` in v67 source names mean `0.1 V`, `0.1 A`, `0.1 °C`, `0.1 Ah`, and percent-times-100 respectively.

### Current direction: native Fox versus frozen v67

Battery-Emulator's generic datalayer uses:

- positive current = physical charging
- negative current = physical discharging

Genuine battery-origin Fox traffic uses the same physical convention in `0x1873`:

- positive = charging
- negative = discharging

This is **Capture-confirmed**. Non-zero EP12 and EP12 Plus `0x0C05` unit currents use the same sign. CQ7 `0x0C05`–`0x0C08` remained zero in the audited capture, so that unit-frame result is not generalised to CQ7. Inverter-origin `0x1871` opcode `0x07` normally uses the opposite perspective during settled flow: negative while the battery charges and positive while it discharges; magnitudes need not be exact negatives. A 27 September 2026 KH9 charge/discharge capture independently matched bytes 2–3 to signed current at `0.1 A/count` and bytes 4–5 to voltage at `0.1 V/count` in both directions: `-15.3 A` at `401.0 V` while charging and `+22.6 A` at `387.9 V` while discharging.

Frozen v67 nevertheless transmits `-reported_current_dA` in `0x1873` and `0x0C05`, subject to readiness gating and `int16` saturation. Its app-facing result on the tested KH9 is therefore negative while charging and positive while discharging, and that behaviour is **Hardware/app-confirmed**. The native/v67 discrepancy requires controlled hardware review; this documentation correction does not change functional code. Energy and capacity accumulation continues to use the generic datalayer direction and must not be reversed.

### v67 readiness terms used below

Several frame values share these implementation gates:

| Term | v67 condition |
|---|---|
| Battery ready | Battery CAN alive, system `ACTIVE`, battery allows contactor closing, and neither system nor real BMS reports a fault. |
| Power path active | Battery ready and `contactors_engaged == 1`. |
| Charging active | Power path active and generic current is at least `+10 dA` (`+1.0 A`). |
| Discharging active | Power path active and generic current is at most `-10 dA` (`-1.0 A`). |
| Capacity model ready | Battery ready and the nominal-voltage/rated-energy/full-energy inputs needed for the capacity calculation are non-zero and valid. |

The 1 A activity deadband affects the `0x187B` operating-state byte. It does not change the sign convention of the live current fields.

## Request/response timing and groups

v67 is request-driven. It does not transmit these FoxESS frames on independent periodic timers. A received `0x1871` request sets a response flag; `transmit_can()` then emits the corresponding response after the relevant timer has been eligible for at least `10 ms`.

The `10 ms` value is a minimum gate between batches on the same response path, not a guaranteed exact inter-frame or inter-batch interval. Actual timing also depends on the main-loop call rate and CAN transmit buffering.

| `0x1871` selector | v67 action | v67 grouping/cadence |
|---|---|---|
| Byte 0 `0x01`, byte 4 `0x00` | Main BMS information | Five batches, each eligible after at least 10 ms. Source comment says the inverter asks every 1 s. |
| Byte 0 `0x01`, byte 4 `0x01` | Individual virtual-unit status | One `0x0C05` frame after the 10 ms gate. |
| Byte 0 `0x01`, byte 4 `0x02` | Detailed cell voltages | Seven batches covering 36 frames and 144 virtual positions; at least 10 ms between batches. |
| Byte 0 `0x01`, byte 4 `0x04` | Detailed temperatures | `0x0D21` and `0x0D22` together after the 10 ms gate. |
| Byte 0 `0x05` | Serial fragments | `0x1881`, `0x1882`, and `0x1883` together after the 10 ms gate. |
| Byte 0 `0x03` | Timestamp/keepalive form | Recognised; no reply. The source comment describes a 6 s cadence and bytes 2–7 as `YY MM DD hh mm ss`. |
| Byte 0 `0x02` | Acknowledgement form | Recognised; no reply. |
| Byte 0 `0x07` | Live inverter DC/battery feedback | Frozen v67 has no dedicated `0x07` handler; receiving it still refreshes the inverter-alive timer. No battery reply is generated. |

The main BMS response order is fixed by v67:

| Batch | Frames |
|---|---|
| 0 | `0x1872`, `0x1873`, `0x1874`, `0x1875` |
| 1 | `0x1876`, `0x1877`, `0x1878`, `0x1879` |
| 2 | `0x187A`, `0x187B`, `0x187F`, `0x1900` |
| 3 | `0x1901`, `0x1902`, `0x1903`, `0x1904` |
| 4 | `0x1905`, `0x1906`, `0x1907`, `0x1908`, `0x1909` |

The dynamic/mixed tables below describe values after `update_values()` has populated the frame objects. Several objects have header initialisers, but the two frozen protocol files alone do not prove that `update_values()` must run before the first request after startup. Static payloads are identified explicitly. Native startup observations are called out separately where the evidence supports them.

## Frame summary

| CAN ID | Direction | DLC | Main role | Dynamic/static | Overall confidence |
|---|---|---:|---|---|---|
| `0x1871` | Inverter → battery | 8 | Multiplexed request, timestamp, acknowledgement, inverter-alive and live inverter DC feedback traffic | Dynamic | Capture-confirmed structure and bidirectional `0x07` current/voltage; Unresolved remaining selector metadata |
| `0x1872` | Battery → inverter | 8 | Voltage and current limits | Dynamic | Capture-confirmed |
| `0x1873` | Battery → inverter | 8 | Live pack voltage/current/SOC and nominal energy | Mixed | Capture-confirmed; Hardware/app-confirmed operation |
| `0x1874` | Battery → inverter | 8 | Temperature extrema plus two position-like fields | Dynamic | Capture-confirmed temperatures; Provisional positions |
| `0x1875` | Battery → inverter | 8 | Average temperature, unit/status data, contactor state, equivalent cycles | Mixed | Capture-confirmed fields; Hardware/app-confirmed cycle transition |
| `0x1876` | Battery → inverter | 8 | Charge permission/status and cell-voltage extrema | Mixed | Capture-confirmed cell voltages; Strongly inferred permission semantic |
| `0x1877` | Battery → inverter | 8 | Fault/status and rotating controller/unit identity | Mixed | Capture-confirmed identity structure; Provisional fault code |
| `0x1878` | Battery → inverter | 8 | Individual-unit SOC and absolute energy throughput | Mixed | Capture-confirmed; Firmware-supported |
| `0x1879` | Battery → inverter | 8 | Directional cumulative capacity | Dynamic | Hardware/app-confirmed charged side; Capture-confirmed both native fields |
| `0x187A` | Battery → inverter | 8 | Directional cumulative energy | Dynamic | Hardware/app-confirmed |
| `0x187B` | Battery → inverter | 8 | SOH, operating state, rated and effective capacity | Mixed | Capture-confirmed; Hardware/app-confirmed display behaviour |
| `0x187F` | Battery → inverter | 8 | Fixed EP field; purpose unknown | Static | Capture-confirmed payload; Unresolved semantics |
| `0x1881`–`0x1883` | Battery → inverter | 8 each | Three serial/identity fragments | Static | Capture-confirmed framing; Provisional fixed identity |
| `0x1900` | Battery → inverter | 8 | Capacity/energy summary and fixed model bytes | Mixed | Capture-confirmed structure; Provisional scope; Unresolved trailing field |
| `0x1901` | Battery → inverter | 8 | Fixed model/calibration data | Static | Capture-confirmed payload; Unresolved semantics |
| `0x1902` | Battery → inverter | 8 | Model value, discharge-power limit, resistance-like value, temperature-like value | Mixed | Capture-confirmed structure; Strongly inferred and Provisional meanings |
| `0x1903` | Battery → inverter | 8 | SOC-scaled nominal energy | Mixed | Capture-confirmed across six capture sets |
| `0x1904` | Battery → inverter | 8 | Extreme-measurement location values | Mixed | Strongly inferred structure; Unresolved ordering/packing |
| `0x1905` | Battery → inverter | 8 | Compact family-specific battery-state/model summary | Mixed | Strong capture associations; Unresolved official semantics |
| `0x1906` | Battery → inverter | 8 | Per-unit model parameters | Static in v67 | Capture-confirmed values; Unresolved official semantics |
| `0x1907` | Battery → inverter | 8 | Two high-resolution battery-state estimates | Dynamic | Capture-confirmed structure; Strongly inferred scale; Unresolved first semantic |
| `0x1908` | Battery → inverter | 8 | Expanded status/model container and second model value | Dynamic | Capture-confirmed structure; Unresolved flags/subfields and exact semantics |
| `0x1909` | Battery → inverter | 8 | Zero-filled extended slot | Static | Capture-confirmed all-zero behaviour; Unresolved purpose |
| `0x0C05` | Battery → inverter | 8 | Individual virtual-unit status | Dynamic | Capture-confirmed |
| `0x0C1D`–`0x0CA9`, step `0x04` | Battery → inverter | 8 each | 144 virtual cell voltages, four per frame | Dynamic | Capture-confirmed format; Provisional generic remapping |
| `0x0D21`, `0x0D22` | Battery → inverter | 8 each | 16 virtual temperature positions, eight per frame | Dynamic | Capture-confirmed format; Provisional repeated min/max population |

## Detailed frame sections

### Inverter-to-battery frame

#### `0x1871` — multiplexed inverter request/status

| Bytes | Type / endian | Scale | v67 handling | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` opcode | — | Dispatches recognised forms `0x01`, `0x02`, `0x03`, and `0x05` | Request/message-class selector | Capture-confirmed | Any other value only refreshes the inverter-alive timer. |
| 4 when byte 0 = `0x01` | `uint8` selector | — | `0x00` main info; `0x01` unit status; `0x02` cell voltages; `0x04` temperatures | Requested data group | Capture-confirmed | v67 ignores unrecognised selector values. |
| 2–7 when byte 0 = `0x03` | Six bytes | Calendar components | No response | `YY MM DD hh mm ss` timestamp | Capture-confirmed for observed layout | Byte 1 in this form is not decoded by v67. |
| 2–3 when byte 0 = `0x07` | `int16` LE | `0.1 A/count` | Not decoded | Inverter-origin live battery/DC current | Capture-confirmed | Uses the inverter perspective during settled flow. KH9 examples: `67 FF` = `-153` = `-15.3 A` while charging; `E2 00` = `+226` = `+22.6 A` while discharging. |
| 4–5 when byte 0 = `0x07` | `uint16` LE | `0.1 V/count` | Not decoded | Inverter-origin live battery/DC voltage | Capture-confirmed | KH9 examples: `AA 0F` = `4010` = `401.0 V` while charging; `27 0F` = `3879` = `387.9 V` while discharging. |
| 1, 6–7 when byte 0 = `0x07` | Raw bytes | — | Not decoded | Status/metadata | Unresolved | Both tested KH9 directions retained byte 1 `0x33` and bytes 6–7 `02 3C`; do not assign semantics without further evidence. |
| 1–3, 5–7 in other request forms | Raw bytes | — | Not decoded | Request metadata/addressing | Unresolved | Do not assign names from the example payloads alone. |

Receiving any `0x1871` refreshes `CAN_inverter_still_alive`. Frozen v67 reads selector bytes without an explicit local DLC check; its CAN integration supplies the 8-byte frame object documented here.

### Main BMS response frames

#### `0x1872` — voltage and current limits

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | `0.1 V/count` | `info.max_design_voltage_dV` | Maximum design/charge voltage | Capture-confirmed | Direct generic datalayer mapping. |
| 2–3 | `uint16` LE | `0.1 V/count` | `info.min_design_voltage_dV` | Minimum design/discharge voltage | Capture-confirmed | Direct generic datalayer mapping. |
| 4–5 | `uint16` LE | `0.1 A/count` | `status.max_charge_current_dA` when battery ready; otherwise `0` | Maximum permitted charge current | Capture-confirmed; Hardware/app-confirmed operation | Unsigned magnitude; no Fox current-sign inversion. |
| 6–7 | `uint16` LE | `0.1 A/count` | `status.max_discharge_current_dA` when battery ready; otherwise `0` | Maximum permitted discharge current | Capture-confirmed; Hardware/app-confirmed operation | Unsigned magnitude; no Fox current-sign inversion. |

This is main-response batch 0.

#### `0x1873` — live pack data and nominal energy

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | `0.1 V/count` | `status.voltage_dV` | Live battery voltage | Capture-confirmed | Dynamic. |
| 2–3 | `int16` LE | `0.1 A/count` | `-status.reported_current_dA` when the power path is active; otherwise `0`; saturated to `[-32768, 32767]` | Live battery current | Native sign **Capture-confirmed**; v67/app sign **Hardware/app-confirmed** | Native Fox: positive charging, negative discharging. Frozen v67/tested app: negative charging, positive discharging. |
| 4 | `uint8` | `1 %/count` | `reported_soc / 100`, capped at `100` | Pack/system SOC | Capture-confirmed | Integer truncation. |
| 5 | `uint8` | — | `0x00` | Zero-filled companion byte | Capture-confirmed | Official purpose unresolved. |
| 6–7 | `uint16` LE | `10 Wh/count` | Reconstructed nominal energy divided by 10, then capped at `65535` | Advertised nominal battery energy | Capture-confirmed; Hardware/app-confirmed display behaviour | Genuine EP12 examples include `1152` = 11.52 kWh for one unit and `2304` = 23.04 kWh for two. |

For bytes 6–7, v67 starts with `info.total_capacity_Wh`. If `reported_total_capacity_Wh > 0`, it reconstructs a pre-SOH nominal value as:

`nominal_Wh = round(reported_total_capacity_Wh × 10000 / min(soh_pptt, 10000))`

When SOH is zero, v67 uses the reported effective energy directly. This is a generic Battery-Emulator implementation choice; the native captures establish the wire scale and single/dual-unit values, not this exact reconstruction formula.

This is main-response batch 0.

#### `0x1874` — temperature extrema and position-like fields

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `int16` LE | `0.1 °C/count` | `status.temperature_max_dC` | Maximum battery temperature | Capture-confirmed | Signed two's-complement value. |
| 2–3 | `int16` LE | `0.1 °C/count` | `status.temperature_min_dC` | Minimum battery temperature | Capture-confirmed | Signed two's-complement value. |
| 4–5 | `uint16` LE in v67 | `1 position/count` | v67 scans non-zero physical cell voltages, takes the maximum-voltage cell, maps its 1-based position into 1–144, or sends `0` if unavailable | First cell-position-like value; v67 treats it as maximum-voltage-cell position | Provisional | Native evidence has not isolated the official meaning strongly enough to elevate the v67 comment to a confirmed Fox mapping. |
| 6–7 | `uint16` LE in v67 | `1 position/count` | Same process for the minimum-voltage cell | Second cell-position-like value; v67 treats it as minimum-voltage-cell position | Provisional | `0x1904` is more directly supported as the native extreme-location frame; duplication or a different role here remains possible. |

For physical 1-based cell position `p` and usable physical cell count `N`, v67's virtual position is:

`ceil((p - 1) × 144 / N) + 1`, clamped to `144`.

This is a v67 implementation formula, not a claimed native Fox cell-addressing specification. For source batteries with more than 144 cells, it is not the exact inverse of the detailed-frame sampling formula, so an extrema position can identify a virtual region whose transmitted sample is not the physical extrema cell. This frame is in main-response batch 0.

#### `0x1875` — status, unit count, contactor state, and cycles

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `int16` LE | `0.1 °C/count` | `(temperature_max_dC + temperature_min_dC) / 2` | Average battery temperature | Capture-confirmed | Integer truncation. |
| 2 | `uint8` bitmap/status | — | Fixed `0x01` | Operational virtual-unit bitmap/status | Capture-confirmed wire relationship; Provisional official name | Genuine one-unit traffic uses `0x01`; two-unit traffic uses `0x03`. Frozen v67 exposes one virtual unit. |
| 3 | `uint8` | `1 unit/count` | `configured_number_of_modules`, set to `1` by `setup()` | Reported unit/module count | Capture-confirmed | Native one-unit and two-unit captures use `1` and `2`. |
| 4 | `uint8` state | — | `0x01` when contactors are engaged, else `0x00` | Contactor state | Hardware/app-confirmed | v67 does not require the broader battery-ready gate for this byte. |
| 5 | `uint8` | — | `0x00` | Zero/unused in observed battery-details use | Capture-confirmed zero; official purpose unresolved | — |
| 6–7 | `uint16` LE | `1 equivalent cycle/count` | `(charged_Wh + discharged_Wh) / (2 × datalayer.battery.info.total_capacity_Wh)`, only when battery ready; capped at `65535` | Equivalent full cycles from bidirectional throughput | Hardware/app-confirmed field/display and first threshold transition; v67 source-confirmed calculation | Integer floor. Hardware/app testing proves that Fox accepts and displays this field as Battery Cycles and that the real value physically moved from 0 to 1 near the expected first equivalent-cycle threshold. The v67 source proves this implementation formula; it does not independently prove that this is Fox's native internal cycle-calculation formula. |

The cycle calculation is based on both charge and discharge throughput. `datalayer.battery.info.total_capacity_Wh` is the generic Battery-Emulator capacity input used by v67; its exact source and semantics can vary between battery integrations, so it should not be described universally as strictly "rated energy". The detailed energy-counter behaviour belongs in [`ENERGY-COUNTERS.md`](ENERGY-COUNTERS.md). This frame is in main-response batch 0.

#### `0x1876` — charge permission/status and cell-voltage extrema

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` status | — | `0x01` when not battery ready, maximum charge current is zero, or SOC is at least 100%; otherwise `0x00` | Charge-block / charge-not-allowed indication | Strongly inferred | Bit 0 is associated with charge permission, but the complete native bitfield semantics remain unresolved. |
| 1 | `uint8` | — | `0x00` | Zero-filled | Capture-confirmed zero; official purpose unresolved | — |
| 2–3 | `uint16` LE | `1 mV/count` | `status.cell_max_voltage_mV` | Maximum cell voltage | Capture-confirmed | — |
| 4–5 | Two bytes | — | `00 00` | Zero-filled | Capture-confirmed zero; official purpose unresolved | — |
| 6–7 | `uint16` LE | `1 mV/count` | `status.cell_min_voltage_mV` | Minimum cell voltage | Capture-confirmed | — |

The historical header comment `BMS_PackTemps` does not describe the payload constructed by v67. No discharge-permission bit is claimed here; v67 communicates discharge permission through numeric limits such as `0x1872` and `0x1902`. This is main-response batch 1.

#### `0x1877` — fault/status and rotating identity

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` status | — | `0x02` on a system or real-BMS fault, otherwise `0x00` | Fault/status code | Provisional for `0x02`; Capture-confirmed normal zero | A genuine native fault transition is still required to confirm the exact Fox meaning of `0x02`. |
| 1–3 | Three bytes | — | `00 00 00` | Zero-filled | Capture-confirmed zero | Official purpose unresolved. |
| 4 | `uint8` identity | — | Main record: `0x6E`; virtual-unit record: `0xFF` | Battery type/subtype identity byte | Capture-confirmed structure | Exact official enumeration names are not known. |
| 5 | `uint8` | — | `0x00` | Zero-filled | Capture-confirmed zero | — |
| 6 | `uint8` version/identity | — | Main record: `0x0C`; virtual-unit record: `0x1C` | Firmware/version-like identity byte | Capture-confirmed structure | Do not interpret as a general Battery-Emulator firmware version. |
| 7 | `uint8` address/identity | — | Main record: `0x01`; unit record: `current_pack_info << 4` (`0x10` for v67's unit 1) | Controller/unit address-like byte | Capture-confirmed structure | Native dual-unit captures also contain a `0x20` unit record. |

The complete v67 payloads are normally:

- main controller record: `00 00 00 00 6E 00 0C 01`
- virtual unit record: `00 00 00 00 FF 00 1C 10`

Byte 0 becomes `0x02` under v67's fault condition. `current_pack_info` advances modulo the configured unit count plus the main record on each `update_values()` call, not on each `0x1877` transmission, so the exact record sequence depends on update cadence. This frame is in main-response batch 1.

#### `0x1878` — individual-unit SOC and absolute throughput

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` | — | `0x00` | Not consumed by the analysed manager path | Firmware-supported; Capture-confirmed zero | Official purpose unresolved. |
| 1 | `uint8` | `1 %/count` | `reported_soc / 100`, capped at `100` | Individual virtual-unit SOC | Capture-confirmed | Genuine values `0x31` and `0x32` are decimal 49% and 50%. |
| 2–3 | `uint16` LE field boundary | Unknown | `0x0000` | Separate zero-valued field | Firmware-supported boundary; Capture-confirmed zero | Exact meaning and scale unresolved. |
| 4–7 | `uint32` LE | `1 Wh/count` | `charged_energy_Wh + discharged_energy_Wh`, saturated at `UINT32_MAX` | Cumulative absolute energy throughput | Capture-confirmed; Firmware-supported | This is bidirectional throughput, not an independent “Total Charged” counter. |

The v67 counter in bytes 4–7 is the sum of its directional energy counters. See [`ENERGY-COUNTERS.md`](ENERGY-COUNTERS.md) for the separate discussion of Fox app energy behaviour. This frame is in main-response batch 1.

#### `0x1879` — directional cumulative capacity

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–3 | `uint32` LE | `0.1 Ah/count` | Integrated positive Battery-Emulator current; saturated at `UINT32_MAX` | Cumulative charged capacity | Hardware/app-confirmed | Genuine charging captures showed the count progress `0 → 1 → 2 → 3 → 4`; the app test isolated the charged side. |
| 4–7 | `uint32` LE | `0.1 Ah/count` | Integrated magnitude of negative Battery-Emulator current; saturated at `UINT32_MAX` | Cumulative discharged capacity | Capture-confirmed | EP12 Plus provides meaningful native progression; current integration independently supports the scale. Older captures alone were too short to prove a transition. |

v67 accumulates `dA × ms` separately by direction and completes one `0.1 Ah` count per `3,600,000 dA·ms`. Integration occurs only while the v67 power path is active. The old header label `Reserved EP field` is contradicted by stronger evidence. This frame is in main-response batch 1.

#### `0x187A` — directional cumulative energy

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–3 | `uint32` LE | `0.1 kWh/count` | `charged_energy_Wh / 100`, saturated at `UINT32_MAX` | Cumulative charged energy | Hardware/app-confirmed | Integer floor. |
| 4–7 | `uint32` LE | `0.1 kWh/count` | `discharged_energy_Wh / 100`, saturated at `UINT32_MAX` | Cumulative discharged energy | Hardware/app-confirmed | Integer floor. |

v67 integrates `voltage_dV × abs(current_dA) × elapsed_ms`, separates the result by Battery-Emulator current direction, and carries sub-Wh remainders. The wire fields are directional totals; they are not the same field as `0x1878` absolute throughput. Further app-counter context belongs in [`ENERGY-COUNTERS.md`](ENERGY-COUNTERS.md). This frame is in main-response batch 2.

#### `0x187B` — SOH, operating state, and capacity

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` | `1 %/count` | `soh_pptt / 100`, capped at `100` | State of health | Hardware/app-confirmed | Integer truncation. |
| 1 | `uint8` state | — | `0x04` not ready; `0x02` charging; `0x01` discharging; `0x00` ready/idle | Operating-state code | Native map Capture-confirmed; v67 operation Hardware/app-confirmed | Native Fox uses `0x01` charging, `0x02` discharging, `0x00` idle/no active direction, and `0x04` startup/not ready. `0x00` does not necessarily mean exactly zero current. Frozen v67 swaps the charging/discharging codes and applies a 1.0 A activity threshold. |
| 2–3 | Two bytes | — | `00 00` | Zero/reserved in observed traffic | Capture-confirmed zero | Official purpose unresolved. |
| 4–5 | `uint16` LE | `0.1 Ah/count` | Rated energy converted to capacity using v67's estimated nominal voltage; capped at `65535` | Rated/design capacity | Capture-confirmed field/scale; Firmware-supported | Native one-unit examples use `300` = 30.0 Ah; multi-unit system totals scale upward. |
| 6–7 | `uint16` LE | `0.1 Ah/count` | Reported effective energy converted in the same way; capped at `65535` | Full/effective capacity | Capture-confirmed field/scale; Hardware/app-confirmed display behaviour | Native values vary with reported usable capacity/SOH. |

Battery-Emulator exposes no dedicated generic nominal-voltage field here. v67 estimates `nominal_voltage_dV` as the rounded midpoint of the valid minimum and maximum design voltages, then computes:

`capacity_dAh = round(energy_Wh × 100 / nominal_voltage_dV)`

Rated energy prefers `total_capacity_Wh`; effective energy prefers `reported_total_capacity_Wh`. That calculation is v67-specific. The native evidence establishes the two capacity fields and their scale, not the midpoint-voltage formula. This frame is in main-response batch 2.

#### `0x187F` — fixed EP field

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Raw bytes | — | `01 00 00 01 00 00 00 00` | Fixed EP protocol field | Capture-confirmed payload; Unresolved semantics | The same payload appears across supplied startup, charging, discharging, and dual-unit captures. No descriptive Fox name is assigned. |

This frame is in main-response batch 2.

### Serial/identity response frames

#### `0x1881` — serial fragment 1

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` | `1 unit/count` | `0x00` | Virtual-unit index | Capture-confirmed | v67 forces unit 0 before transmission. |
| 1–7 | Seven ASCII bytes | — | `36 30 45 50 30 30 35` = `60EP005` | First serial/identity fragment | Capture-confirmed framing | Fixed in v67; not derived from the attached source battery. |

#### `0x1882` — serial fragment 2

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` | `1 unit/count` | `0x00` | Virtual-unit index | Capture-confirmed | v67 forces unit 0 before transmission. |
| 1–7 | Seven ASCII bytes | — | `30 34 37 4D 41 30 35` = `047MA05` | Second serial/identity fragment | Capture-confirmed framing | Fixed in v67. |

#### `0x1883` — serial fragment 3

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0 | `uint8` | `1 unit/count` | `0x00` | Virtual-unit index | Capture-confirmed | v67 forces unit 0 before transmission. |
| 1–7 | ASCII plus NUL padding | — | `32 00 00 00 00 00 00` = `2` plus six NUL bytes | Final serial/identity fragment and terminator padding | Capture-confirmed framing | Fixed in v67. |

Together, v67 exposes the fixed string `60EP005047MA052`. These three frames are sent together in response to `0x1871` opcode `0x05`.

### Extended `0x1900`-series response frames

The `0x1900`–`0x1909` family is documented conservatively. Observable boundaries and relationships are stated; neutral labels are used when the official Fox semantic name is not established.

#### `0x1900` — capacity/energy summary and fixed model bytes

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | `0.1 Ah/count` | Same v67 system-rated capacity used in `0x187B` bytes 4–5 | Capacity value associated with the represented unit/model | Capture-confirmed scale; Provisional scope in v67 | Native EP12 evidence kept this at 30.0 Ah per unit while a two-unit system's energy fields doubled. Frozen v67 represents the aggregate battery as one virtual unit and sends its aggregate rated capacity. |
| 2–3 | `uint16` LE | `10 Wh/count` | Same v67 nominal-energy count used in `0x1873` bytes 6–7 | Nominal installed energy | Capture-confirmed | Native examples: `1152` for one unit and `2304` for two units. |
| 4–7 | Four raw bytes | Unknown | `00 00 3C 46` | Fixed trailing model/calibration field | Capture-confirmed payload; Unresolved semantics | Interpreted as IEEE-754 little-endian, the bytes would equal `12032.0`, but neither that type nor its unit/purpose is established. |

Native one-unit payload: `2C 01 80 04 00 00 3C 46`. Native two-unit payload: `2C 01 00 09 00 00 3C 46`. This frame is in main-response batch 2.

#### `0x1901` — fixed model/calibration data

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Raw bytes | — | `F0 0F 00 00 40 00 33 42` | Fixed model/calibration payload | Capture-confirmed payload; Unresolved semantics | The supplied captures show this payload unchanged across conditions. |
| 0–3, if integer | `uint32` LE | Unknown | Raw value `4080` | First possible model parameter | Provisional | Field boundary and unit are not established. |
| 4–7, if float | IEEE-754 LE candidate | Unknown | Raw bytes decode to approximately `44.750244` | Second possible model parameter | Provisional | This is an interpretation aid, not a confirmed Fox type. |

This frame is in main-response batch 3.

#### `0x1902` — capability/model values

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | Possibly `0.001/count` | Fixed `942` | Model/capacity coefficient-like value | Capture-confirmed raw value; Provisional semantic name | Settled EP12 traffic commonly reports `942`; startup captures also contain lower values. Do not treat “efficiency coefficient” as an official name. |
| 2–3 | `uint16` LE | `1 W/count` | `max_discharge_power_W` when battery ready and maximum discharge current is non-zero; otherwise `0`; capped at `65535` | Maximum permitted discharge power | Capture-confirmed; Firmware-supported | No corresponding charge-power field is claimed in this frame. |
| 4–5 | `uint16` LE | Best-supported `0.1 mΩ/count` | Fixed `892` | Equivalent resistance-like model value | Strongly inferred; Firmware-supported | v67 uses the settled EP12 value because no generic live resistance source is available. The official Fox name remains unknown. |
| 6–7 | `uint16` LE | Best-supported `1 °C/count` | `(temperature_max_dC + temperature_min_dC) / 20`, clamped to `[0, 65535]` | Filtered/model temperature-like value | Strongly inferred | The temperature relationship is supported; the exact native filtering/source and official name are unresolved. |

This frame is in main-response batch 3.

#### `0x1903` — SOC-scaled nominal energy

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–5 | Six bytes | — | All zero | Zero-filled portion | Capture-confirmed zero | Official purpose unresolved. |
| 6–7 | `uint16` LE | Native: `0.1 kWh/count` | Frozen v67: `nominal_energy_Wh / 200`, capped at `65535` | Native SOC-scaled nominal energy | Capture-confirmed | Native relationship: `floor(U16(0x1873,b6-7) × 0x1873 byte4 / 1000)`. It matched 14,296/14,296 comparable responses across old single/dual EP12, EP12 Plus, and CQ7 captures. |

The former native interpretation as fixed nominal energy at `200 Wh/count` is disproved. The old captures held SOC at 50%, so `floor(E × 50 / 1000) = floor(E / 20)`, making raw values `57` and `115` appear compatible with the old interpretation. Varying SOC resolved the ambiguity. Frozen v67 retains the earlier `/200` approximation and remains hardware-proven on the tested KH9; this frame is in v67 main-response batch 3.

#### `0x1904` — extreme-measurement locations

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE in v67 | `1 position/count` | v67 virtual maximum-voltage-cell position, or `0` | First voltage-extreme position | Strongly inferred | Captures support two cell-extreme positions, but the native max/min ordering has not been decisively isolated. |
| 2–3 | `uint16` LE in v67 | `1 position/count` | v67 virtual minimum-voltage-cell position, or `0` | Second voltage-extreme position | Strongly inferred | Same ordering caution as bytes 0–1. |
| 4 | Raw location code | Unknown | Fixed `0x3A` | First temperature-extreme/address-like code | Provisional | Exact sensor-address packing and max/min order unresolved. |
| 5 | Raw location code | Unknown | `0x0A` for v67's configured one unit; code contains an unreachable-in-normal-v67 `0x12` branch for two configured units | Second temperature-extreme/address-like code | Provisional | Native captures vary with configuration/startup, including `0x0A`, `0x12`, and other transitional values. |
| 6–7 | Two bytes | — | `00 00` | Zero-filled | Capture-confirmed zero | Official purpose unresolved. |

The v67 max/min assignment in bytes 0–3 is an implementation choice based on its cell scan. Public documentation should retain the neutral “first/second extreme” wording until a controlled native capture proves the order. This frame is in main-response batch 3.

#### `0x1905` — compact battery-state/model summary

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | Model/family-specific | Fixed `320` | Model/family reference | Capture-confirmed raw values; semantic Unresolved | EP12 uses `320`, while CQ7 uses `1240`; the old universal 3.20 V nominal-cell interpretation is disproved. |
| 2 | `uint8` | `1 %/count` | Whole reported SOC, capped at `100`; forced to `0` until capacity model ready | Primary whole-percent battery-state value | Capture-confirmed relationship | SOC is the best-supported interpretation, but this is not an official field name. |
| 3 | `uint8` state | — | `0x04` before capacity model ready; `0x08` when ready | Compact model-state code | Capture-confirmed wire states | v67's “initialising/ready” wording is descriptive, not an official Fox enumeration. |
| 4 | `uint8` | Unknown | `floor(970 / 10) = 97`, derived from `0x1906` bytes 4–5 | Coarse copy of model parameter B | Capture-confirmed relationship | Do not call it an efficiency value without stronger evidence. |
| 5 | `uint8` | Best-supported `1 %/count` | `floor(0x1907 bytes 4–7 / 10)` | Coarse value strongly associated with the second `0x1907` value | Strong association, not universal identity | EP12 Plus has one matched-response exception; CQ7 matches strongly. Frozen v67 enforces exact truncation. |
| 6 | `uint8` | Best-supported `1 %/count` | `floor(0x1908 bytes 4–7 / 10)` | Coarse value strongly associated with the second `0x1908` value | Strong association, not universal identity | EP12 Plus has 42 matched-response exceptions and older startup evidence has an exception; CQ7 matches strongly. Frozen v67 enforces exact truncation. |
| 7 | `uint8` | — | `0x00` | Zero-filled | Capture-confirmed zero | Official purpose unresolved. |

A representative genuine operational payload is `40 01 32 08 61 32 2F 00`; an observed startup form is `40 01 00 04 61 00 00 00`. These examples demonstrate relationships, not official names. This frame is in main-response batch 4.

#### `0x1906` — per-unit model parameters

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | Unknown | Fixed `81` | Model parameter A | Capture-confirmed raw range; Unresolved semantics | Native operational traffic contains `80`/`81`; startup traffic contains lower values such as `46`/`47`. |
| 2–3 | Two bytes | — | `00 00` | Zero-filled | Capture-confirmed zero | Official purpose unresolved. |
| 4–5 | `uint16` LE | Unknown | Fixed `970` | Model parameter B | Capture-confirmed raw values; Unresolved semantics | Native traffic contains `970` and slowly changing values such as `977`. The field is not a charge/discharge state flag. |
| 6–7 | Two bytes | — | `00 00` | Zero-filled | Capture-confirmed zero | Official purpose unresolved. |

v67 intentionally sends a stable captured value rather than reproducing the native model's startup and slow-update behaviour. This distinction matters for future upstream review. This frame is in main-response batch 4.

#### `0x1907` — high-resolution battery-state estimates

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–3 | `uint32` LE | Best-supported `0.1 %/count` | Mirrors v67's fine SOC value because Battery-Emulator exposes no separate generic voltage/OCV/SOE estimate; zero until capacity model ready; capped at `1000` | First high-resolution, voltage/model-sensitive state estimate | Capture-confirmed structure; Strongly inferred scale; Unresolved exact semantic | Genuine EP12 behaviour indicates this can differ from the second estimate. v67 intentionally mirrors them. |
| 4–7 | `uint32` LE | `0.1 %/count` | `reported_soc / 10`, zero until capacity model ready, capped at `1000` | Second high-resolution SOC/SOE-like estimate | Field/association Capture-confirmed; boundary Firmware-supported | Usually associated with `0x1905` byte 5 by division by 10, but EP12 Plus has one matched-response exception. Whether Fox internally calls it SOC, SOE, or another state estimate remains unresolved. |

Observed native values occupy the low part of each 32-bit slot; the manager-derived field boundaries are 32-bit. This frame is in main-response batch 4.

#### `0x1908` — expanded status/model container

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–3 | Four-byte container | Unresolved | Frozen v67 sends simple LE32 values `14` before capacity-model readiness and `16` when ready | Expanded status/model container with likely flags/subfields | Capture-confirmed forms; exact definitions Unresolved | Native forms include `10 04 80 00`, `10 08 00 00`, `10 0A 40 00`, and `10 0C 80 00`; it must not be documented as a universal simple enum. |
| 4–7 | `uint32` LE | Best-supported `0.1 %/count` | Frozen v67 sends `fine_state × full_capacity_dAh / rated_capacity_dAh`, capped at `1000`; otherwise the fine value | Second model/state quantity | Capture-confirmed field boundary; exact semantic Unresolved | New genuine captures contradict the v67 capacity-adjusted formula as a universal native relationship. Its coarse association with `0x1905` byte 6 has exceptions. |

Frozen v67 may retain its capacity-adjusted approximation as an implementation choice. It is not a universal native Fox formula. This frame is in v67 main-response batch 4.

#### `0x1909` — all-zero extended slot

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Raw bytes | — | `00 00 00 00 00 00 00 00` | All-zero extended slot | Capture-confirmed wire behaviour; Unresolved purpose | Genuine EP12 payloads remained all zero in the supplied evidence. “Reserved” is not claimed as an official semantic name. |

This is the final frame in main-response batch 4.

### Native newer-family extension: `0x1910`–`0x1919`

These identifiers were absent from the audited old EP12 captures and present after `0x1909` in every observed complete normal main response from EP12 Plus and CQ7. No new request selector was observed. Passive ordering does not prove an internal scheduler design, and these frames are not claimed to be required by every inverter.

Frozen v67 does **not** transmit `0x1910`–`0x1919`; they are not part of its 63-frame transmit inventory. v67 already demonstrated compatibility on the tested KH9 without them. Whether newer inverter or manager firmware requires the extension is **Unresolved**, so documentation evidence alone is insufficient reason to implement it.

| Frames | EP12 Plus | CQ7 | Evidence / limits |
|---|---|---|---|
| `0x1910` | ASCII `EP12` plus zero padding | Eight spaces | Capture-confirmed bytes |
| `0x1911` | No separate meaning assigned by this audit | One space then zero padding | CQ7 bytes Capture-confirmed; do not infer text semantics from padding |
| `0x1912`–`0x1914` | `0x1912 + 0x1913` concatenate to `EP12 Plus (w)`; `(w)` meaning Unresolved | Zero | Capture-confirmed bytes; no CQ7 text meaning assigned |
| `0x1915` | First LE32 fixed at `4`; second LE32 monotonic/seconds-like | Exact cross-frame structure described below | Family/revision dependent |
| `0x1916` | First LE32 monotonic/seconds-like; second LE32 zero | All zero | EP12 Plus uptime/runtime/seconds-like interpretation Strongly inferred, not official semantics |
| `0x1917`–`0x1919` | All zero | All zero | Capture-confirmed values; purpose Unresolved |

For CQ7, `0x1915` matched the following structure in 3,157/3,157 normal main responses:

- bytes 0–1 = `0x1905` bytes 0–1;
- bytes 2–3 = `0x1908` bytes 0–1;
- byte 4 = `floor(0x1907` second LE32 `/ 10)`;
- byte 5 = `floor(0x1907` first LE32 `/ 10)`; and
- bytes 6–7 = `0`.

This CQ7 layout does not apply to EP12 Plus. Frames `0x190A`–`0x190F` were not observed in the audited captures; no payload, meaning, or scheduler position is assigned to them.

### Individual virtual-unit status

#### `0x0C05` — individual unit data

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `int16` LE | `0.1 A/count` | Negative of generic current while power path active; otherwise zero; saturated to `int16` | Individual-unit current | Native EP12/EP12 Plus sign Capture-confirmed; v67 sign implementation-confirmed | Native non-zero values: positive charging, negative discharging. Frozen v67 sends the opposite sign. CQ7 unit currents remained zero in the audited capture. |
| 2 | `uint8` offset | `1 °C/count`, offset `+50` | `temperature_max_dC / 10 + 50`, clamped to `[0, 255]` | Maximum unit temperature | Capture-confirmed | Whole-degree conversion truncates toward zero. |
| 3 | `uint8` offset | `1 °C/count`, offset `+50` | `temperature_min_dC / 10 + 50`, clamped to `[0, 255]` | Minimum unit temperature | Capture-confirmed | — |
| 4 | `uint8` | `1 %/count` | Same capped unit SOC used by `0x1878` byte 1 | Individual-unit SOC | Capture-confirmed | — |
| 5 and low nibble of 6 | Packed unsigned 12-bit | `1 mV/count` | `cell_max_voltage_mV`, capped at `0xFFF` | Maximum unit cell voltage | Capture-confirmed | Decode as `b5 \| ((b6 & 0x0F) << 8)`. |
| High nibble of 6 and byte 7 | Packed unsigned 12-bit | `1 mV/count` | `cell_min_voltage_mV`, capped at `0xFFF` | Minimum unit cell voltage | Capture-confirmed | Decode as `((b6 >> 4) & 0x0F) \| (b7 << 4)`. |

Frozen v67 represents the complete generic battery as one virtual EP unit. This frame is sent alone in response to `0x1871` byte 0 `0x01`, byte 4 `0x01`.

### Detailed cell-voltage frames

All 36 frames below use the same payload format:

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–1 | `uint16` LE | `1 mV/count` | First virtual position assigned to the frame | Virtual cell voltage 1 of 4 | Capture-confirmed format | — |
| 2–3 | `uint16` LE | `1 mV/count` | Second virtual position | Virtual cell voltage 2 of 4 | Capture-confirmed format | — |
| 4–5 | `uint16` LE | `1 mV/count` | Third virtual position | Virtual cell voltage 3 of 4 | Capture-confirmed format | — |
| 6–7 | `uint16` LE | `1 mV/count` | Fourth virtual position | Virtual cell voltage 4 of 4 | Capture-confirmed format | — |

For zero-based virtual index `v` and usable physical cell count `N`, v67 chooses physical zero-based index `floor(v × N / 144)`. `N` is capped at Battery-Emulator's `MAX_AMOUNT_CELLS`. A selected zero reading is replaced by the final physical cell's non-zero voltage when available, otherwise by `3300 mV`. If `N` is zero, every virtual position uses `3300 mV`.

That 144-position remapping is v67's generic implementation. It must not be presented as a requirement that the source battery itself contain 144 cells, and no Leaf-specific cell-layout assumption is made.

For `N > 144`, the frozen sampling formula does not preserve the final physical-cell endpoint; for example, with 192 physical cells, virtual position 144 samples physical position 191. This is a documented v67 implementation limitation, not a Fox protocol semantic and not a redesign proposal.

#### `0x0C1D` — virtual cells 1–4

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `1, 2, 3, 4` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C21` — virtual cells 5–8

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `5, 6, 7, 8` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C25` — virtual cells 9–12

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `9, 10, 11, 12` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C29` — virtual cells 13–16

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `13, 14, 15, 16` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C2D` — virtual cells 17–20

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `17, 18, 19, 20` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C31` — virtual cells 21–24

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `21, 22, 23, 24` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 0. |

#### `0x0C35` — virtual cells 25–28

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `25, 26, 27, 28` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 1. |

#### `0x0C39` — virtual cells 29–32

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `29, 30, 31, 32` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 1. |

#### `0x0C3D` — virtual cells 33–36

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `33, 34, 35, 36` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 1. |

#### `0x0C41` — virtual cells 37–40

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `37, 38, 39, 40` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 1. |

#### `0x0C45` — virtual cells 41–44

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `41, 42, 43, 44` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 1. |

#### `0x0C49` — virtual cells 45–48

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `45, 46, 47, 48` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 2. |

#### `0x0C4D` — virtual cells 49–52

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `49, 50, 51, 52` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 2. |

#### `0x0C51` — virtual cells 53–56

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `53, 54, 55, 56` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 2. |

#### `0x0C55` — virtual cells 57–60

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `57, 58, 59, 60` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 2. |

#### `0x0C59` — virtual cells 61–64

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `61, 62, 63, 64` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 2. |

#### `0x0C5D` — virtual cells 65–68

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `65, 66, 67, 68` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 3. |

#### `0x0C61` — virtual cells 69–72

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `69, 70, 71, 72` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 3. |

#### `0x0C65` — virtual cells 73–76

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `73, 74, 75, 76` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 3. |

#### `0x0C69` — virtual cells 77–80

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `77, 78, 79, 80` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 3. |

#### `0x0C6D` — virtual cells 81–84

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `81, 82, 83, 84` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 3. |

#### `0x0C71` — virtual cells 85–88

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `85, 86, 87, 88` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 4. |

#### `0x0C75` — virtual cells 89–92

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `89, 90, 91, 92` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 4. |

#### `0x0C79` — virtual cells 93–96

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `93, 94, 95, 96` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 4. |

#### `0x0C7D` — virtual cells 97–100

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `97, 98, 99, 100` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 4. |

#### `0x0C81` — virtual cells 101–104

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `101, 102, 103, 104` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 4. |

#### `0x0C85` — virtual cells 105–108

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `105, 106, 107, 108` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 5. |

#### `0x0C89` — virtual cells 109–112

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `109, 110, 111, 112` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 5. |

#### `0x0C8D` — virtual cells 113–116

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `113, 114, 115, 116` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 5. |

#### `0x0C91` — virtual cells 117–120

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `117, 118, 119, 120` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 5. |

#### `0x0C95` — virtual cells 121–124

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `121, 122, 123, 124` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 5. |

#### `0x0C99` — virtual cells 125–128

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `125, 126, 127, 128` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 6. |

#### `0x0C9D` — virtual cells 129–132

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `129, 130, 131, 132` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 6. |

#### `0x0CA1` — virtual cells 133–136

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `133, 134, 135, 136` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 6. |

#### `0x0CA5` — virtual cells 137–140

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `137, 138, 139, 140` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 6. |

#### `0x0CA9` — virtual cells 141–144

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0–7 | Four `uint16` LE values | `1 mV/count` | Virtual positions `141, 142, 143, 144` | Four virtual cell voltages | Capture-confirmed format | Voltage batch 6; final detailed-voltage frame. |

### Detailed temperature frames

v67 has only generic minimum and maximum battery temperatures, not 16 independent sensor values. Before encoding, it swaps the generic values if necessary to restore minimum ≤ maximum. It then alternates minimum and maximum across the 16 virtual positions. Each byte is `temperature_dC / 10 + 50`, with C++ truncation toward zero and saturation to `[0, 255]`.

This mapping provides the native frame shape without inventing physical sensor identities.

#### `0x0D21` — virtual temperature positions 1–8

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0, 2, 4, 6 | Four offset `uint8` values | `1 °C/count`, offset `+50` | Generic minimum temperature repeated | Virtual temperature positions 1, 3, 5, 7 | Capture-confirmed encoding; Provisional synthetic mapping | No claim that these are four distinct physical sensors. |
| 1, 3, 5, 7 | Four offset `uint8` values | `1 °C/count`, offset `+50` | Generic maximum temperature repeated | Virtual temperature positions 2, 4, 6, 8 | Capture-confirmed encoding; Provisional synthetic mapping | — |

#### `0x0D22` — virtual temperature positions 9–16

| Bytes | Type / endian | Scale | v67 source/value | Best-supported meaning | Evidence | Notes |
|---|---|---|---|---|---|---|
| 0, 2, 4, 6 | Four offset `uint8` values | `1 °C/count`, offset `+50` | Generic minimum temperature repeated | Virtual temperature positions 9, 11, 13, 15 | Capture-confirmed encoding; Provisional synthetic mapping | No claim that these are four distinct physical sensors. |
| 1, 3, 5, 7 | Four offset `uint8` values | `1 °C/count`, offset `+50` | Generic maximum temperature repeated | Virtual temperature positions 10, 12, 14, 16 | Capture-confirmed encoding; Provisional synthetic mapping | — |

The two temperature frames are transmitted together in response to `0x1871` byte 0 `0x01`, byte 4 `0x04`.

## Cross-frame relationships

| Relationship | Mapping / implementation | Evidence status |
|---|---|---|
| `0x1873` bytes 6–7 and `0x1900` bytes 2–3 | Same nominal-energy value at `10 Wh/count` | Capture-confirmed |
| Native `0x1903` bytes 6–7 | `floor(U16(0x1873,b6-7) × 0x1873 byte4 / 1000)`, at `0.1 kWh/count` | Capture-confirmed, 14,296/14,296 comparable responses |
| Frozen v67 `0x1903` bytes 6–7 | Nominal energy divided by 200 | Frozen implementation; disproved as a universal native interpretation |
| `0x187B` bytes 4–5 and `0x1900` bytes 0–1 | v67 sends the same rated-capacity value | v67 implementation; native `0x1900` appears per-unit while `0x187B` can be system-total |
| `0x1878` bytes 4–7 | Charged Wh plus discharged Wh | Capture-confirmed; Firmware-supported |
| `0x1879` and `0x187A` | Directional capacity and directional energy respectively | `0x1879` both halves Capture-confirmed; charged path and both `0x187A` halves Hardware/app-confirmed where stated above |
| `0x1875` bytes 6–7 | Floors total bidirectional energy throughput divided by `2 × datalayer.battery.info.total_capacity_Wh` | v67 source-confirmed calculation; Hardware/app-confirmed field/display and first 0 → 1 transition; native Fox calculation formula not independently proven |
| `0x1905` byte 4 | Coarse copy of `0x1906` bytes 4–5 divided by 10 | Capture-confirmed relationship |
| `0x1905` byte 5 | Strongly associated with `floor(0x1907` second LE32 `/ 10)` | Not universal: one EP12 Plus matched-response exception; CQ7 matches strongly; exact in v67 by implementation |
| `0x1905` byte 6 | Strongly associated with `floor(0x1908` second LE32 `/ 10)` | Not universal: 42 EP12 Plus matched-response exceptions and an older startup exception; CQ7 matches strongly; exact in v67 by implementation |

## Rejected / disproved interpretations

- `0x1878` byte 1 is **not** a `0x31`/`0x32` charge/discharge state flag. Those hexadecimal byte values are decimal 49 and 50 and corresponded to 49% and 50% individual-unit SOC.
- `0x1878` byte 4 is **not** a fixed charging flag. It is the least-significant byte of the 32-bit little-endian throughput field in bytes 4–7.
- `0x1878` bytes 4–7 are **not** Fox's independent Total Charged source. They carry absolute bidirectional throughput in Wh.
- The v66 charged-only `0x1878` experiment did **not** fix Total Charged and is not the v67 mapping.
- Fox reverting to its approximately 92% fallback does **not** prove that `0x1879` failed. The fallback path and the successful decoding of the direct counter are separate questions.
- `0x1906` bytes 4–5 values such as `970` and `977` are **not** a simple charge/discharge state flag. Captures show slow model-like variation rather than direction switching.
- Native `0x1903` bytes 6–7 are **not** a universal fixed nominal-energy field at `200 Wh/count`; fixed 50% SOC in the old captures created that ambiguity.
- `0x1905` raw `320` is **not** universally a 3.20 V nominal-cell value; CQ7 uses `1240`.
- Native `0x1908` bytes 0–3 are **not** universally a simple `0`/`14`/`16` enum, and its second LE32 does **not** universally follow v67's capacity-adjusted formula.

This document deliberately does not reproduce the full Fox energy-counter fallback analysis, persistence discussion, commissioning sequence, or test chronology. Those subjects belong in [`ENERGY-COUNTERS.md`](ENERGY-COUNTERS.md) and other dedicated project documents.
