# FoxESS EP-Series Energy Counters and Total Charged Behaviour

## Scope

This document describes the energy, cumulative-capacity, Total Charged fallback and equivalent-cycle architecture used by the frozen hardware-proven v67 FoxESS EP-Series Battery-Emulator implementation.

It covers:

- `0x187A` directional cumulative charged and discharged energy;
- `0x1878` individual-unit SOC and absolute bidirectional energy throughput;
- `0x1879` directional cumulative charged and discharged capacity;
- `0x1875` equivalent-cycle reporting;
- Fox Daily Charged and Daily Discharged behaviour;
- Fox Total Charged and Total Discharged behaviour;
- the observed Fox Total Charged fallback/plausibility behaviour; and
- the current volatile-counter limitation across restart and reflash.

This is not a duplicate of [`CAN-FRAME-MAP.md`](CAN-FRAME-MAP.md). That document defines the frame-level byte map. This document instead explains how the energy-related fields interact, how their meanings were established, which conclusions came from genuine EP12 traffic, which came from manager-firmware analysis, which were proved on the real inverter/app, and which questions remain unresolved.

The evidence terms used here are:

| Label | Meaning |
| --- | --- |
| **Hardware/app-confirmed** | Demonstrated by controlled behaviour on the physical inverter, battery and/or FoxESS app. |
| **Capture-confirmed** | Directly observed in genuine EP12 CAN traffic. |
| **Firmware-supported** | Manager-firmware analysis supports the field boundary, decoding or use. |
| **Strongly inferred** | Multiple observations support the interpretation, but decisive direct native proof remains incomplete. |
| **Provisional** | A useful current interpretation with limited supporting evidence. |
| **Unresolved** | Available evidence does not establish the exact behaviour or semantic meaning. |

Evidence is weighted in this order:

1. real hardware/app behaviour;
2. genuine EP12 CAN captures;
3. manager-firmware analysis;
4. frozen v67 source;
5. historical comments and earlier assumptions.

The distinction matters throughout this document. What v67 deliberately transmits is not automatically a claim that a genuine EP12 BMS internally calculates the same quantity in the same way.

---

## Counter overview

The energy-related frames expose several related but distinct quantities.

| Frame | Bytes | Wire quantity | Scale | v67 source | Best evidence |
| --- | --- | --- | --- | --- | --- |
| `0x187A` | 0–3 | Cumulative charged energy, `uint32` LE | `0.1 kWh/count` | `foxess_installation_charged_energy_Wh / 100` | **Hardware/app-confirmed** |
| `0x187A` | 4–7 | Cumulative discharged energy, `uint32` LE | `0.1 kWh/count` | `foxess_installation_discharged_energy_Wh / 100` | **Hardware/app-confirmed** |
| `0x1878` | 1 | Individual virtual-unit SOC | `1 %/count` | Generic reported SOC divided by 100 | **Capture-confirmed** |
| `0x1878` | 4–7 | Cumulative absolute energy throughput, `uint32` LE | `1 Wh/count` | charged Wh + discharged Wh | **Capture-confirmed**, **Firmware-supported** |
| `0x1879` | 0–3 | Cumulative charged capacity, `uint32` LE | `0.1 Ah/count` | `foxess_charged_capacity_dAh` | Charged path **Hardware/app-confirmed**; native progression **Capture-confirmed** |
| `0x1879` | 4–7 | Matching cumulative discharged capacity, `uint32` LE | `0.1 Ah/count` | `foxess_discharged_capacity_dAh` | **Strongly inferred** |
| `0x1875` | 6–7 | Whole equivalent-cycle value, `uint16` LE | `1 cycle/count` | Bidirectional Wh throughput divided by `2 × datalayer.battery.info.total_capacity_Wh` | Field/display and first threshold **Hardware/app-confirmed**; exact v67 calculation established by frozen source |

These quantities are not interchangeable.

`0x187A` is a coarse directional energy pair.

`0x1878` is a higher-resolution absolute-throughput quantity with no direction encoded in the counter itself.

`0x1879` is directional cumulative capacity rather than energy.

`0x1875` carries a whole-number cycle value derived by v67 from the installation energy pair.

Most importantly, `0x1878` bytes 4–7 are **not** Fox's independent Total Charged source. The v66 hardware experiment specifically tested and rejected that interpretation.

---

## v67 local accounting

### Authoritative installation-energy pair

The frozen v67 implementation maintains two authoritative local energy totals:

```text
foxess_installation_charged_energy_Wh
foxess_installation_discharged_energy_Wh
```

Both are unsigned 64-bit values.

They accumulate energy measured by Battery-Emulator from the generic battery datalayer while the FoxESS power path is active.

The inputs are:

```text
datalayer.battery.status.voltage_dV
datalayer.battery.status.reported_current_dA
elapsed time in milliseconds
```

Battery-Emulator's generic current convention is:

```text
positive current = physical charging
negative current = physical discharging
```

This is the sign used to decide which installation counter receives an increment.

It must not be confused with the live Fox current representation in frames such as `0x1873`, where v67 deliberately reverses the sign to match genuine Fox traffic:

```text
negative Fox current = charging
positive Fox current = discharging
```

That live-current sign inversion is not applied to the cumulative installation counters.

### Energy integration

v67 first takes the absolute current magnitude and calculates:

```text
energy numerator =
    voltage_dV
    × absolute_current_dA
    × elapsed_ms
```

Because:

```text
voltage_dV = volts × 10
current_dA = amps × 10
```

their product is power in units of watts × 100.

The exact v67 conversion to watt-hours is therefore:

```text
completed Wh =
    accumulated(
        voltage_dV
        × |current_dA|
        × elapsed_ms
    )
    / 360000000
```

or equivalently:

```text
Wh =
    voltage_dV
    × |current_dA|
    × elapsed_ms
    / 360000000
```

v67 does not discard the fractional part after each update. It keeps separate 64-bit charge and discharge remainders:

```text
foxess_charged_energy_remainder
foxess_discharged_energy_remainder
```

Only complete watt-hours are moved into the corresponding installation counter. The remaining sub-Wh numerator is retained for the next update.

This prevents repeated truncation of small integration intervals from systematically losing energy.

### Direction selection

If:

```text
reported_current_dA > 0
```

the completed energy is added to:

```text
foxess_installation_charged_energy_Wh
```

If:

```text
reported_current_dA < 0
```

it is added to:

```text
foxess_installation_discharged_energy_Wh
```

No energy is directionally accumulated at exactly zero current.

The integration interval itself is active only while the generic FoxESS power-path condition is satisfied. If the battery is not on the active power path, v67 refreshes its integration timestamp rather than integrating the inactive period.

This prevents elapsed time while contactors are open from being multiplied by a later current sample.

### Capacity integration

Directional capacity is accumulated separately from energy.

The input is current × elapsed time rather than voltage × current × elapsed time.

For charging:

```text
charged dAh =
    accumulated(
        positive current_dA
        × elapsed_ms
    )
    / 3600000
```

For discharging:

```text
discharged dAh =
    accumulated(
        |negative current_dA|
        × elapsed_ms
    )
    / 3600000
```

where one `dAh` count represents `0.1 Ah`.

Separate remainder accumulators preserve fractions below one `0.1 Ah` count.

This creates:

```text
foxess_charged_capacity_dAh
foxess_discharged_capacity_dAh
```

for `0x1879`.

### Derived throughput

Although the source retains a compatibility throughput accumulator, the actual v67 value placed into `0x1878` bytes 4–7 is derived from the authoritative directional energy pair:

```text
absolute throughput Wh =
    foxess_installation_charged_energy_Wh
    +
    foxess_installation_discharged_energy_Wh
```

The wire value is saturated at `UINT32_MAX`.

Conceptually, the v67 energy architecture is therefore:

```text
physical charging
    │
    ├── integrated energy ──> charged Wh ──> 0x187A bytes 0–3
    │                                  │
    │                                  └──> cycle throughput
    │
    └── integrated current ─> charged dAh ─> 0x1879 bytes 0–3


physical discharging
    │
    ├── integrated energy ──> discharged Wh ─> 0x187A bytes 4–7
    │                                     │
    │                                     └──> cycle throughput
    │
    └── integrated current ─> discharged dAh ─> 0x1879 bytes 4–7


charged Wh + discharged Wh
    │
    ├──> 0x1878 bytes 4–7, absolute throughput Wh
    │
    └──> 0x1875 equivalent-cycle calculation
```

---

## Native evidence for `0x1878`

`0x1878` was initially one of the more misleading EP frames because short captures allowed several apparently plausible but incorrect interpretations.

### Early interpretation: byte 1 as a state flag

Short capture examples included values such as:

```text
00 32 00 00 20 00 00 00
```

during charging and:

```text
00 31 00 00 00 00 00 00
```

during a separate discharge capture.

This originally made it tempting to interpret:

```text
0x32 = charging/idle state
0x31 = discharging state
```

That interpretation was disproved.

`0x31` hexadecimal is decimal `49`.

`0x32` hexadecimal is decimal `50`.

The genuine captures showed the same values in the unit SOC paths.

At startup and during charging, unit SOC was 50% and `0x1878` byte 1 was `0x32`.

During the discharge capture, unit SOC was 49% and byte 1 was `0x31`.

Therefore:

```text
0x1878 byte 1 = individual virtual-unit SOC in whole percent
```

**Evidence: Capture-confirmed.**

### Early interpretation: byte 4 as a charging flag

The charging example:

```text
00 32 00 00 20 00 00 00
```

also encouraged an early interpretation of `0x20` in byte 4 as some type of charging flag.

That interpretation was also disproved.

Manager-firmware analysis supports the structural decoding:

```text
byte 0      = separate byte
byte 1      = separate byte
bytes 2–3   = one separate 16-bit field
bytes 4–7   = one little-endian 32-bit field
```

The four bytes beginning at byte 4 are therefore a numeric field, not four independent status bytes.

**Evidence: Firmware-supported.**

### Long-capture throughput evidence

The longer continuous dual-EP12 capture was decisive.

The test sequence transferred approximately:

```text
physical charge       ~223 Wh
physical discharge     ~23 Wh
absolute movement     ~246 Wh
```

During that sequence, `0x1878` bytes 4–7 increased toward approximately:

```text
240
```

The agreement is close to the measured total absolute transferred energy.

The earlier:

```text
20 00 00 00
```

therefore decodes as decimal `32`, or approximately `32 Wh`, rather than a fixed `0x20` operating flag.

The important feature of the longer capture is that the 32-bit field continued accumulating across energy movement rather than identifying one direction.

The best-supported mapping is:

```text
0x1878 bytes 4–7
    = cumulative absolute energy throughput
    = charged energy + discharged energy
    = uint32 little-endian
    = 1 Wh/count
```

**Evidence: Capture-confirmed, Firmware-supported.**

The exact persistence/reset epoch of the genuine EP12 counter remains **Unresolved** because the available native evidence does not span a controlled BMS power-cycle with a known pre-reset counter value.

### v66 hardware experiment

After the early investigation, v66 deliberately transmitted charged-only Wh in `0x1878` bytes 4–7 as an isolated hardware experiment.

If `0x1878` were Fox's independent Total Charged source, this should have caused the Fox Total Charged value to follow genuine charging independently.

It did not.

Fox continued exhibiting the discharged-energy-derived fallback relationship.

Therefore:

```text
0x1878 bytes 4–7 ≠ independent Fox Total Charged
```

and:

```text
charged-only Wh in 0x1878 does not solve Total Charged
```

**Evidence: Hardware/app-confirmed rejection.**

v67 consequently restored `0x1878` to absolute bidirectional throughput.

---

## `0x187A` — Fox directional energy path

`0x187A` contains two separate 32-bit little-endian cumulative energy fields:

```text
bytes 0–3 = charged energy
bytes 4–7 = discharged energy
```

Both use:

```text
0.1 kWh/count
```

or:

```text
100 Wh/count
```

v67 generates them as:

```text
charged wire count =
    foxess_installation_charged_energy_Wh / 100

discharged wire count =
    foxess_installation_discharged_energy_Wh / 100
```

with saturation to `UINT32_MAX`.

### Charged side

The charged half was correlated with physical charging in genuine EP12 traffic and subsequently demonstrated in the Fox app.

Increasing the charged-side counter moves Fox's charged-energy presentation.

**Evidence: Hardware/app-confirmed.**

### Discharged side

The discharged half was hardware-tested successfully in Battery-Emulator operation and drives the corresponding discharged-energy behaviour.

The short available genuine EP12 discharge sample did not contain enough physical discharge to cross a `0.1 kWh` boundary, which is why the native bytes remained zero in that particular capture.

This lack of a native transition does not override the later isolated hardware/app proof.

**Evidence: Hardware/app-confirmed.**

### Daily values are not transmitted as separate daily counters

The Fox app exposes fields named:

```text
Daily Charged Energy
Daily Discharged Energy
```

but `0x187A` itself should not therefore be described as a pair of daily-since-midnight wire counters.

The stronger interpretation is that `0x187A` supplies cumulative directional counters and Fox retains a previous baseline from which shorter-period values can be derived.

This interpretation became particularly important when the local Battery-Emulator counters were reset by reboot/reflash while Fox retained its previous cloud/app baseline. Negative Daily Charged/Discharged values could then appear because Fox's retained reference was higher than the newly reset local cumulative value.

That behaviour is strong real-system evidence that at least part of the Fox daily presentation is calculated from differences between cumulative values rather than being transmitted directly as an independent daily counter.

The exact Fox day-boundary mechanism remains **Unresolved**. The current evidence does not establish whether Fox's daily baseline follows site local time, inverter time, account time, server time or another epoch.

No separate confirmed daily charged/discharged CAN field has been identified.

### Relationship to Total Charged

The fact that `0x187A` contains charged energy does **not** mean that Fox's displayed Total Charged field simply mirrors `0x187A` bytes 0–3 under all conditions.

The hardware tests showed that Fox can substitute a fallback estimate for displayed Total Charged while the direct charged path is considered implausible.

This distinction was one of the central findings of the project.

---

## `0x1879` — independent cumulative capacity path

`0x1879` provided the missing evidence needed to explain Fox's independent Total Charged behaviour.

### Genuine native capture

In the long dual-EP12 charging capture, bytes 0–3 did not remain zero.

They progressed:

```text
0 → 1 → 2 → 3 → 4
```

during physical charging.

The observed transitions occurred at approximately:

```text
79 Wh
82 Wh
158 Wh
164 Wh
```

of system charging.

The capture used two EP12 units in parallel.

At approximately 395–410 V, one `0.1 Ah` increment for one battery corresponds to roughly:

```text
0.1 Ah × 395–410 V
≈ 39.5–41 Wh
```

With two parallel batteries sharing current, paired transitions are therefore expected after roughly:

```text
~79–82 Wh
```

of total system energy for the first pair and:

```text
~158–164 Wh
```

for the second pair.

That is the pattern observed.

This also explains why the earlier single-EP12 short charging capture, with only about `32 Wh` represented in `0x1878`, did not yet show a non-zero `0x1879` value.

The best-supported charged-side mapping is therefore:

```text
0x1879 bytes 0–3
    = cumulative charged capacity
    = uint32 little-endian
    = 0.1 Ah/count
```

The fact that the field is charging-only is **Capture-confirmed**. The `0.1 Ah/count` interpretation is extremely strongly supported by the threshold pattern and was subsequently used successfully in the hardware-proven v67 implementation.

### Discharged half

The matching v67 mapping is:

```text
0x1879 bytes 4–7
    = cumulative discharged capacity
    = uint32 little-endian
    = 0.1 Ah/count
```

However, the available genuine discharge portion transferred only approximately `23 Wh`.

That was insufficient to cross the expected first `0.1 Ah` threshold.

The native discharged field therefore remained zero.

The mirrored two-`uint32` structure, directional symmetry and v67 architecture strongly support the discharged interpretation, but a decisive genuine native discharge transition is still missing.

**Evidence: Strongly inferred.**

### v67 hardware proof of the charged path

The most important `0x1879` result came from the real inverter/app test rather than from the native capture alone.

With v67 transmitting directional cumulative capacity, enough genuine charging was accumulated for Fox to stop relying on the previous fallback relationship.

At that point the displayed Total Charged value followed the independently established charged path.

A subsequent physical discharge produced approximately:

```text
Total Discharged: 11.20 → 11.50 kWh
Total Charged:    19.40 → 19.40 kWh
```

Total Discharged increased.

Total Charged remained fixed.

This is the decisive behavioural proof that Fox had accepted an independent charged source rather than continuing to derive Total Charged mechanically from discharged energy.

**Evidence: Hardware/app-confirmed.**

The available evidence therefore supports a manager path in which `0x1879` charged capacity contributes to Fox's independently derived charged-energy result.

The exact internal conversion from cumulative Ah to the displayed kWh value remains **Unresolved**.

It may involve voltage, model values, averaged quantities, calibration parameters or a combination of these. No public documentation in this repository should claim an exact conversion algorithm unless it is separately proved.

---

## Fox Total Charged fallback discovery

The fallback behaviour was identified through repeated hardware observations rather than through a single firmware decode.

### 1. Total Charged behaved incorrectly during discharge

Before the independent charged path had been established, physical discharge caused Fox's displayed Total Charged value to increase despite there being no physical charging.

At the same time:

- the inverter correctly reported discharging;
- Daily Discharged increased;
- Total Discharged increased; and
- Daily Charged did not behave as though the battery were physically charging.

This ruled out a simple reversal of the battery current sign or a swap of the `0x187A` charge/discharge fields.

### 2. The charged value tracked discharged energy at a very stable ratio

Longer observations included:

```text
Total Discharged = 8.90 kWh
Total Charged    = 9.67 kWh

8.90 / 9.67 = 0.920372
```

and later:

```text
Total Discharged = 9.10 kWh
Total Charged    = 9.89 kWh

9.10 / 9.89 = 0.920121
```

The behaviour is approximately equivalent to:

```text
fallback charged ≈ discharged / 0.9203
```

This was too stable to be explained by random display lag or ordinary app rounding.

**Evidence for the fallback behaviour: Hardware/app-confirmed.**

### 3. v66 ruled out `0x1878` as the missing direct source

v66 changed `0x1878` bytes 4–7 to charged-only Wh.

The fallback continued.

Therefore the missing independent Total Charged source was not simply charged Wh in `0x1878`.

**Evidence: Hardware/app-confirmed.**

### 4. `0x1879` was restored from genuine capture evidence

Re-analysis of the genuine dual-EP12 capture showed that `0x1879` bytes 0–3 were not always zero. They formed the charging-only cumulative sequence:

```text
0 → 1 → 2 → 3 → 4
```

with transition spacing matching `0.1 Ah` increments.

v67 therefore restored the native `0x1878` throughput interpretation and transmitted directional cumulative capacity in `0x1879`.

### 5. Fox accepted the genuine charged path

After sufficient charging, the direct charged path became accepted by Fox.

The decisive discharge test then showed:

```text
Total Discharged increasing
Total Charged fixed
```

rather than Total Charged continuing to follow the fallback.

**Evidence: Hardware/app-confirmed.**

### 6. The fallback could later return

Further discharge eventually caused Fox to resume its fallback relationship when the direct charged history once again fell outside the region Fox considered plausible relative to discharged energy.

This showed that acceptance of the direct charged source is not necessarily permanent when the transmitted cumulative histories start from a fresh zero epoch.

**Evidence: Hardware/app-confirmed.**

The most cautious description is therefore:

> Fox uses a plausibility or model-selection mechanism that can choose between a direct charged-energy path and a discharged-energy-derived fallback.

An observed selection can resemble:

```text
larger/plausible of:
    direct charged result
    fallback charged estimate
```

but there is **no proof** that the firmware literally performs:

```text
max(real_charged, fallback)
```

The exact comparison, threshold, hysteresis and state logic remain **Unresolved**.

---

## The approximately `0.9203` relationship

The fallback behaviour itself was established from hardware/app observations.

The measured ratio was approximately:

```text
0.9203
```

so the displayed fallback behaved approximately as:

```text
fallback charged ≈ discharged / 0.9203
```

Separate genuine Fox data contained model-related values including approximately:

```text
0.942
0.977
```

Their product is:

```text
0.942 × 0.977 = 0.920334
```

Comparison:

| Source | Ratio/factor |
| --- | ---: |
| Hardware observation: `8.90 / 9.67` | `0.920372` |
| Hardware observation: `9.10 / 9.89` | `0.920121` |
| Observed Fox coefficients: `0.942 × 0.977` | `0.920334` |

The numerical agreement is notable.

The relevant values occur in Fox's model-oriented `0x1902` / `0x1906` data, and manager-firmware analysis supports those areas as model parameters rather than independent cumulative-energy counters.

However, the correct public conclusion is:

> The hardware-observed fallback factor of approximately `0.9203` is notably consistent with the product of the observed Fox model coefficients `0.942 × 0.977`.

It must **not** be written as:

> Fox is proven to calculate Total Charged using `0.942 × 0.977`.

That exact internal multiplication has not been proved.

The physical meanings of the relevant `0x1902` and `0x1906` model parameters are also not completely resolved.

The approximately `0.9203` relationship is therefore **Strongly inferred** as being associated with the model coefficients, while the fallback behaviour itself is **Hardware/app-confirmed**.

---

## Why fallback does not mean `0x1879` failed

This distinction is essential.

A later return to the fallback originally risked being misinterpreted as evidence that the new `0x1879` mapping had failed.

The hardware sequence disproves that conclusion.

The sequence was:

1. v67 transmitted the direct cumulative charged-capacity path.
2. Sufficient genuine charging was accumulated.
3. Fox accepted the direct charged result.
4. During subsequent physical discharge, Total Discharged increased.
5. Total Charged remained fixed.
6. Later, after further discharge changed the plausibility relationship, Fox could return to the discharged-derived fallback.

Step 5 is decisive.

If the `0x1879` charged path had never worked, there would be no reason for Total Charged to become independently stationary while discharge continued.

The later fallback therefore demonstrates a Fox selection/plausibility behaviour, not failure of the `0x1879` mapping.

> **A return to the approximately 92% fallback must not be described as `0x1879` failing.**

The direct path had already been experimentally isolated and accepted.

---

## Total Discharged versus Total Charged

The protocol should not be simplified into an assumption that Fox treats Total Charged and Total Discharged identically internally.

The evidence supports several distinct quantities:

```text
0x187A
    directional energy
    charged and discharged
    0.1 kWh/count

0x1879
    directional capacity
    charged and discharged
    0.1 Ah/count

0x1878
    absolute bidirectional throughput
    1 Wh/count
```

Hardware testing established that the discharged-energy path works independently and that Fox's displayed Total Charged can use a separate direct charged-capacity path subject to its plausibility/fallback behaviour.

This means:

- directional energy and directional capacity are different CAN quantities;
- `0x1878` cannot substitute for either directional pair;
- Fox's displayed totals need not come from symmetrical internal calculation paths;
- the independent charged-capacity path has been experimentally isolated; and
- the matching native discharged-capacity path should not be assigned additional application behaviour without evidence.

The current project does not claim to know Fox's complete internal lifetime-energy architecture.

---

## Equivalent cycles

### v67 implementation

v67 sends the equivalent-cycle value in:

```text
0x1875 bytes 6–7
```

as a `uint16` little-endian whole-cycle count.

The frozen v67 implementation calculates:

```text
equivalent cycles =
    (
        foxess_installation_charged_energy_Wh
        +
        foxess_installation_discharged_energy_Wh
    )
    /
    (
        2
        × datalayer.battery.info.total_capacity_Wh
    )
```

This is integer arithmetic, so incomplete cycles are truncated.

For example, a calculated value of:

```text
0.99
```

still transmits:

```text
0
```

and the value changes to:

```text
1
```

only once the first complete threshold is crossed.

The result is saturated at:

```text
UINT16_MAX = 65535
```

### v67 readiness condition

The cycle calculation is performed only when:

```text
foxess_ep_battery_ready == true
```

and:

```text
2 × datalayer.battery.info.total_capacity_Wh > 0
```

Otherwise the calculated value remains zero for that update.

The energy totals feeding the calculation themselves accumulate only while the power path is active, as described earlier.

### Capacity denominator

The denominator uses:

```text
datalayer.battery.info.total_capacity_Wh
```

This is the generic Battery-Emulator capacity input used by v67.

It should not be described universally as strict manufacturer "rated energy", because the exact source and semantics of `total_capacity_Wh` can vary between battery integrations. Depending on the integration it may reflect a configured capacity, model capacity, BMS-derived capacity or another generic base-capacity value.

The important v67 implementation fact is simply that this exact generic field is used.

### Hardware proof

The real test installation eventually showed approximately:

```text
Total Charged:    35.90 kWh
Total Discharged: 33.70 kWh
Combined:         69.60 kWh
```

for a battery represented at approximately:

```text
35 kWh
```

The first expected v67 equivalent-cycle threshold is therefore approximately:

```text
2 × 35 kWh
≈ 70 kWh
```

At approximately that combined throughput, Fox Battery Cycles physically moved:

```text
0 → 1
```

**Evidence: Hardware/app-confirmed.**

This proves two important things:

1. Fox accepts `0x1875` bytes 6–7 as the Battery Cycles field.
2. The first real v67 threshold behaved as expected from the v67 bidirectional-throughput calculation.

It does **not** independently prove that a genuine Fox EP12 BMS calculates its own native cycle counter using the identical formula.

The exact v67 formula is known because it is present in the frozen source.

Fox's native internal cycle-calculation algorithm remains **Unresolved**.

---

## Restart/reflash behaviour

The cumulative installation counters in frozen v67 are volatile.

The relevant energy and capacity state is held in ordinary RAM variables including:

```text
foxess_installation_charged_energy_Wh
foxess_installation_discharged_energy_Wh
foxess_charged_capacity_dAh
foxess_discharged_capacity_dAh
```

They initialise from zero and frozen v67 does not restore them from persistent storage.

A normal restart or firmware reflash therefore starts a new local zero epoch.

### Effect on `0x187A`

Before restart:

```text
0x187A = non-zero cumulative charged/discharged values
```

After restart:

```text
local v67 source counters = zero
```

Fox can retain the previous cloud/app cumulative baseline.

The resulting backwards jump can therefore cause negative Daily Charged or Daily Discharged presentation because the newly transmitted cumulative value is below the retained Fox baseline.

**Evidence: Hardware/app-confirmed.**

### Effect on Total Charged fallback

Resetting the genuine local charged history to zero also destroys the plausibility relationship that had allowed Fox to accept the direct charged path.

The fallback can therefore become active again after restart/reflash.

**Evidence: Hardware/app-confirmed behaviour associated with the zero-counter epoch.**

### Effect on `0x1878`

Because v67 derives absolute throughput from the local directional Wh pair:

```text
0x1878 throughput
    =
    charged Wh
    +
    discharged Wh
```

that value also restarts from zero.

### Effect on `0x1879`

The local charged and discharged `dAh` accumulators are also RAM-only, so both directional capacity fields restart from zero.

### Effect on equivalent cycles

The v67 cycle field is calculated from:

```text
local charged Wh + local discharged Wh
```

so restarting those source counters also resets the locally derived/transmitted equivalent-cycle value.

This wording is deliberate: the project has proved that the **v67 transmitted value** resets because its local source counters reset. It should not be broadened into a claim that Fox Cloud itself necessarily erases its historical cycle data according to the same mechanism.

### Persistence status

Persistence is the principal remaining release-quality blocker, but persistence architecture is outside the scope of this document.

Its implementation is intentionally deferred for discussion with Dala and the Battery-Emulator project.

No NVM design is proposed here.

---

## Commissioning implication

The zero-start behaviour creates a practical commissioning consideration even before persistence is implemented.

For a fresh installation whose local counters begin at zero:

1. start with the battery at a relatively low SOC where practical;
2. establish a meaningful amount of genuine charging; and
3. avoid substantial discharge until the genuine charged history has become established.

The purpose is not to manipulate Fox's reported energy.

It is to avoid a situation where:

```text
direct genuine charged history ≈ zero
```

while:

```text
discharged history = already substantial
```

because that relationship allows Fox's discharged-derived fallback estimate to overtake the newly established direct charged path.

Charging first gives the direct path enough history to enter Fox's plausible region before significant discharge is accumulated.

This is only a commissioning mitigation.

It is **not** a substitute for persistent counters across restart/reflash.

The detailed installation procedure belongs in [`COMMISSIONING.md`](COMMISSIONING.md).

---

## Evidence chronology

The interpretation developed through a sequence of controlled observations and failed hypotheses.

| Stage | Observation/test | Result | Evidence impact |
| --- | --- | --- | --- |
| Early short captures | `0x1878` contained `0x31`, `0x32` and `0x20` in positions that appeared state-like | Initially suggested state/charging flags | **Provisional** early theory only |
| SOC comparison | `0x31 = 49`, `0x32 = 50`, matching unit SOC | Byte 1 identified as unit SOC rather than direction state | **Capture-confirmed** |
| Longer continuous EP12 capture | `0x1878` 32-bit field continued toward ~240 while measured absolute transfer was ~246 Wh | Established absolute-throughput interpretation | **Capture-confirmed** |
| `0x1879` native re-analysis | Charged half progressed `0 → 1 → 2 → 3 → 4` around ~79, ~82, ~158 and ~164 Wh in a dual-EP12 charge | Strong evidence for directional charged capacity at `0.1 Ah/count` | **Capture-confirmed** quantity; scale exceptionally strongly supported |
| Fallback observations | `8.90 / 9.67 = 0.920372`; `9.10 / 9.89 = 0.920121` | Total Charged identified as following a discharged-derived fallback in this region | **Hardware/app-confirmed** |
| Model comparison | `0.942 × 0.977 = 0.920334` | Near-exact consistency with observed Fox model coefficients | **Strongly inferred** relationship, not proven formula |
| v66 charged-only `0x1878` test | Total Charged continued following fallback | Rejected `0x1878` as independent Total Charged source | **Hardware/app-confirmed** rejection |
| v67 directional `0x1879` test | Direct charged history became established and was accepted | Identified functional independent charged path | **Hardware/app-confirmed** |
| v67 discharge after acceptance | Total Discharged `11.20 → 11.50 kWh`; Total Charged stayed `19.40 kWh` | Direct charged path proved independent during discharge | **Hardware/app-confirmed** |
| Further discharge | Fox could later return to fallback | Established plausibility/selection behaviour rather than permanent direct-source selection | **Hardware/app-confirmed** behaviour; exact threshold **Unresolved** |
| Cycle threshold test | Around `35.90 + 33.70 ≈ 69.60 kWh` total app energy, Battery Cycles changed `0 → 1` with ~35 kWh represented capacity | Proved field/display and first v67 threshold | **Hardware/app-confirmed** |

No date has been assigned to a test in this table unless the evidence record itself requires it. The purpose is interpretation chronology rather than a release timeline.

---

## Rejected interpretations

The following theories have been disproved or remain explicitly unproven and must not be reintroduced as established mappings without new evidence.

### `0x1878` byte 1 is a charge/discharge state flag — disproved

`0x31` and `0x32` were decimal SOC values 49% and 50%.

The byte tracks individual-unit SOC.

### `0x1878` byte 4 is a charging flag — disproved

`0x20` was the least-significant byte of a 32-bit little-endian counter equal to decimal 32.

The full field continued increasing beyond that value.

### `0x1878` bytes 4–7 are independent Total Charged — disproved

Native evidence shows absolute bidirectional throughput.

The v66 charged-only hardware experiment also failed to produce independent Total Charged behaviour.

### Sending charged-only Wh in `0x1878` solves Total Charged — disproved

This was directly tested in v66 and failed.

### Fox returning to fallback means `0x1879` failed — disproved

v67 first proved the direct charged path by showing Total Charged remain fixed during subsequent discharge.

A later return to fallback therefore reflects Fox's plausibility/model-selection behaviour.

### `970` / `977` in `0x1906` are simple direction-state values — disproved

The available captures and model behaviour do not support treating these numbers as simple charge/discharge flags.

They behave as model-related quantities and show more nuanced state-dependent variation.

Their exact official semantics remain unresolved.

### `0.942 × 0.977` is definitively Fox's internal Total Charged formula — not proven

The numerical match is extremely close:

```text
0.942 × 0.977 = 0.920334
```

and the hardware fallback observations cluster around the same factor.

That establishes a strong relationship worth documenting.

It does **not** establish the exact sequence of mathematical operations used inside Fox firmware.

---

## Remaining unknowns

The following questions remain genuinely unresolved.

### Exact Fox Total Charged selection threshold

Fox has been observed to move from fallback to the direct charged path and later return to fallback.

The exact numerical threshold, hysteresis, state conditions and selection algorithm remain **Unresolved**.

### Exact internal conversion from `0x1879` charged capacity to Total Charged kWh

The charged-capacity source has been functionally proved, but the exact voltage/model/calibration calculation used to turn cumulative Ah into displayed charged energy remains **Unresolved**.

### Exact physical semantics of the relevant `0x1902` / `0x1906` model coefficients

Their numerical relationship to fallback behaviour is strong, but their official physical names and complete model roles remain **Unresolved**.

### Native `0x1879` discharged-side transition

The matching `uint32` discharged-capacity field at `0.1 Ah/count` is **Strongly inferred**.

A genuine native EP12 discharge containing enough transferred capacity to cross a clear threshold would provide the missing direct proof.

### Native persistence epoch

The exact reset/persistence epochs of genuine EP12 cumulative energy/capacity counters remain **Unresolved**.

The available evidence does not establish whether the native counters are tied to BMS lifetime, service replacement, battery power cycle, manager session or another internal epoch.

### Native Fox equivalent-cycle algorithm

Hardware proves that Fox accepts and displays `0x1875` bytes 6–7 as Battery Cycles and that the v67 value physically moved `0 → 1` around the expected first equivalent-cycle threshold.

The frozen source proves the exact v67 formula.

It remains **Unresolved** whether a genuine Fox EP12 BMS internally derives its own cycle value using the identical calculation.

---

## Summary

The frozen v67 energy architecture is based on two independent forms of directional accumulation:

```text
energy:
    charged Wh
    discharged Wh

capacity:
    charged 0.1 Ah
    discharged 0.1 Ah
```

Those values are exposed in three distinct cumulative CAN representations:

```text
0x187A
    directional energy
    0.1 kWh/count

0x1879
    directional capacity
    0.1 Ah/count

0x1878
    absolute bidirectional energy throughput
    1 Wh/count
```

and the local directional energy pair also feeds:

```text
0x1875
    whole equivalent cycles
```

The major hardware discovery was that Fox's displayed Total Charged value is not simply another view of `0x1878`.

When the genuine direct charged history is too low relative to discharged energy, Fox can display a fallback charged value derived from discharged energy and its battery model.

That fallback followed approximately:

```text
charged fallback ≈ discharged / 0.9203
```

and the observed ratio is notably consistent with:

```text
0.942 × 0.977 ≈ 0.9203
```

without proving that Fox internally performs that exact multiplication.

v67 then established the independent charged path through `0x1879`: once Fox accepted the direct charged history, Total Charged remained fixed during subsequent physical discharge while Total Discharged continued rising.

That test also proved why a later return to fallback must not be mistaken for failure of `0x1879`.

Finally, the successful Battery Cycles `0 → 1` transition demonstrated that Fox accepts the cycle field produced by v67's bidirectional installation-throughput calculation. The formula is proven as a v67 implementation choice; Fox's native equivalent-cycle algorithm remains a separate unresolved question.

The main remaining architectural limitation is not the energy mapping itself but continuity: all current v67 installation energy and capacity counters are volatile across restart/reflash.
