# FoxESS EP-Series Commissioning Guide

## Scope

This guide covers practical first-start commissioning of the standalone FoxESS EP-Series protocol running under Battery-Emulator.

The frozen hardware-proven reference for this document is **v67**.

The protocol is intended to remain battery-agnostic. The principal real-hardware validation used a Nissan Leaf battery pack, but that battery is a test platform rather than a protocol requirement. Users should rely on the battery state, limits, permissions and safety data supplied by their own supported Battery-Emulator battery integration.

This guide assumes that the battery, inverter and Battery-Emulator installation have already been installed and commissioned safely.

It does not replace:

- FoxESS inverter installation or service documentation;
- Battery-Emulator setup documentation;
- battery-specific safety requirements;
- electrical or high-voltage installation procedures; or
- the detailed protocol evidence documents in this repository.

The purpose here is to establish the Fox-facing battery data and cumulative histories cleanly enough to verify that the protocol is behaving as expected.

---

## Before first start

Before enabling normal power flow, confirm that:

- [ ] the Battery-Emulator battery integration is already operating correctly;
- [ ] battery CAN communication is live;
- [ ] reported battery voltage is sensible for the connected battery;
- [ ] reported SOC is sensible;
- [ ] charge and discharge limits are sensible;
- [ ] battery temperature data is sensible;
- [ ] there are no relevant active battery or system faults;
- [ ] the battery integration permits contactor closing;
- [ ] the inverter-side FoxESS battery configuration is appropriate for use with the EP protocol; and
- [ ] you understand that v67's locally generated cumulative energy, capacity and cycle-related counters are currently volatile across Battery-Emulator restart or reflash.

Do not copy unverified inverter settings from another installation.

Where an inverter or Battery-Emulator setting is not explicitly documented by this project, use the relevant FoxESS or Battery-Emulator documentation instead.

---

## Why first-start SOC matters

Frozen v67 begins a fresh local counter epoch with its generated cumulative charged and discharged histories at zero.

During hardware testing, Fox's displayed Total Charged value did not always use the direct charged history immediately. When the genuine charged history was too small relative to discharged energy, Fox could instead display a discharged-energy-derived fallback.

The observed fallback relationship was approximately:

```text
fallback charged ≈ discharged / 0.9203
```

The fallback behaviour itself is hardware/app-confirmed, but the exact Fox comparison threshold, hysteresis, state logic and internal calculation remain unresolved.

This creates an important first-start consideration.

If substantial discharge is accumulated while the direct charged history is still near zero, Fox's fallback result can become dominant before the genuine charged path has established a plausible history.

For that reason, the recommended commissioning approach is:

> Start with the battery at a relatively low SOC where practical, then establish meaningful genuine charging before allowing substantial discharge.

This is a **commissioning mitigation only**.

It is not a replacement for persistent counters.

See [ENERGY-COUNTERS.md](ENERGY-COUNTERS.md) for the full Total Charged and fallback evidence.

---

## Recommended first-start sequence

### 1. Start from a relatively low SOC where practical

Where normal battery operation allows it, begin commissioning with enough available charging headroom to establish a useful direct charged history.

There is **no hardware-proven universal SOC percentage** that must be used.

"Relatively low" means only that the battery has useful room to accept genuine charging.

Do not:

- deliberately exceed the battery integration's permitted SOC range;
- violate voltage or safety limits;
- deeply discharge a battery merely to satisfy this recommendation; or
- treat any particular SOC percentage as a protocol requirement.

Battery safety and battery-integration limits remain authoritative.

### 2. Start Battery-Emulator and allow the protocol to initialise

Start Battery-Emulator normally and allow the FoxESS EP protocol to establish communication with the inverter.

Confirm that:

- battery CAN communication remains active;
- no relevant system or battery fault is present;
- contactor closing is permitted by the battery integration;
- the inverter recognises the virtual EP battery; and
- expected battery information begins to populate.

Do not force contactors or bypass a readiness condition to make the inverter recognise the battery.

No universal startup delay is specified by this project.

### 3. Confirm basic live values before significant power flow

Before deliberately moving substantial energy, check that the live battery presentation is sensible.

Confirm, where available:

- SOC;
- battery voltage;
- current direction;
- charge permission;
- discharge permission;
- charge and discharge power/current limits;
- battery temperature; and
- SOH.

For Fox-facing current values, the convention used by the tested EP protocol is:

```text
charging    = negative current
discharging = positive current
```

This is the Fox-facing representation.

It must not be confused with Battery-Emulator's internal generic datalayer convention, where positive current represents physical charging and negative current represents physical discharging.

### 4. Charge first

Allow the battery to perform genuine charging before deliberately accumulating substantial discharge.

The objective is not to reach a particular SOC and it is not necessary to charge to 100%.

The objective is to establish real charged history.

During genuine charging, v67 builds:

- cumulative charged energy for `0x187A`; and
- cumulative charged capacity for the direct `0x1879` charged path.

Building this history gives Fox a genuine charged source from which it can derive or accept its independent Total Charged result.

This reduces the chance that a discharged-energy-derived fallback immediately dominates a new zero-start installation.

There is currently **no independently proven universal minimum kWh or Ah value** that guarantees Fox will select the direct path.

### 5. Confirm charged counters behave sensibly

Where the FoxESS app and Battery Details page are available, allow enough time for their displayed values to refresh and check the result.

During physical charging, expected behaviour includes:

- Daily Charged Energy increasing as sufficient energy accumulates;
- charged cumulative history progressing;
- Total Charged becoming consistent with the established genuine charged history once Fox accepts the direct path;
- Total Discharged not materially increasing during a controlled pure-charging test; and
- reported status and current direction agreeing with the physical battery state.

The Fox app/cloud path may update later than the physical CAN event, so an immediate screen update should not be required for commissioning to be considered valid.

The `0x187A` energy quantities are exposed at `0.1 kWh/count`, so very small energy transfers may not immediately produce a visible step.

### 6. Perform a controlled discharge check

After a meaningful genuine charged history has been established, allow a normal controlled discharge within the battery integration's permitted limits.

Confirm that:

- battery status indicates discharging;
- Fox-facing current is positive;
- Daily Discharged and Total Discharged progress appropriately once sufficient energy has accumulated; and
- Total Charged does not increase as though the battery were physically charging while Fox continues to accept the direct charged path.

One of the decisive v67 hardware tests showed:

```text
Total Discharged: 11.20 → 11.50 kWh
Total Charged:    19.40 → 19.40 kWh
```

while the battery was physically discharging.

That hold demonstrated that Fox had accepted a charged result independent of continuing discharge.

Do not expect every short discharge to move a visible energy value immediately. `0x187A` uses `0.1 kWh` increments, and app/cloud presentation can add further delay.

### 7. Confirm the installation is ready for normal operation

Before considering initial commissioning complete, confirm that:

- charging operates normally;
- discharging operates normally;
- contactors behave normally;
- battery charge and discharge limits remain respected;
- battery details are sensible;
- charged and discharged energy counters respond in the correct physical direction; and
- a genuine charged history has been established before substantial ongoing discharge.

Normal operation can then begin within the limits and safety rules of the supported battery integration and inverter installation.

---

## What normal behaviour looks like

| Observation | Expected interpretation |
| --- | --- |
| Status shows Charging | Battery is physically charging |
| Fox-facing current is negative | Expected charging-current convention |
| Charged counters progress | Expected during sufficient physical charging |
| Status shows Discharging | Battery is physically discharging |
| Fox-facing current is positive | Expected discharging-current convention |
| Discharged counters progress | Expected during sufficient physical discharge |
| Energy value does not move after a very small transfer | Can be normal because `0x187A` updates in `0.1 kWh` steps |
| Total Charged remains fixed during pure discharge | Expected when Fox is continuing to use the accepted direct charged path |
| Total Charged rises during discharge while Daily Charged does not | Can indicate Fox's fallback/model-derived Total Charged behaviour |
| Battery Cycles remains at zero | Can be normal until sufficient cumulative bidirectional throughput reaches the first whole-cycle threshold |
| Fox app/cloud values update after the physical event | Can be normal; user-visible updates need not be instantaneous |

Battery Cycles in v67 are transmitted as a whole integer. The hardware-proven test showed the displayed value move from `0` to `1` around the first expected v67 equivalent-cycle threshold.

The exact v67 cycle calculation and the distinction from Fox's unresolved native cycle algorithm are documented in [ENERGY-COUNTERS.md](ENERGY-COUNTERS.md).

---

## Recognising Total Charged fallback

A practical sign of Fox's Total Charged fallback behaviour is:

> The battery is physically discharging, Total Charged rises, but Daily Charged does not behave as though physical charging is occurring.

During testing, fallback-region observations included a very stable relationship between Total Discharged and the displayed Total Charged result.

The observed ratio was approximately:

```text
discharged / charged ≈ 0.9203
```

or equivalently:

```text
fallback charged ≈ discharged / 0.9203
```

This can be useful as a diagnostic clue, but it is not an official Fox formula and must not be treated as one.

The important accompanying observations are:

- the inverter still correctly reports physical discharge;
- Fox-facing current direction remains correct;
- Daily Discharged continues to increase;
- Daily Charged does not falsely indicate genuine physical charging; and
- Total Charged can track a discharged-derived result instead of the previously established direct charged value.

Fallback behaviour does **not** by itself mean that the protocol or `0x1879` has failed.

v67 hardware testing proved that the direct charged path works. After sufficient genuine charging, Fox accepted that path and Total Charged remained fixed during subsequent physical discharge. Further discharge could later cause Fox to return to fallback behaviour.

The best-supported interpretation is therefore that Fox applies some form of plausibility or source-selection logic between its direct charged path and a discharged-derived fallback.

The exact selection threshold and algorithm remain unresolved.

See [ENERGY-COUNTERS.md](ENERGY-COUNTERS.md) for the complete evidence.

---

## If fallback appears during commissioning

If Total Charged appears to be following the fallback during initial commissioning:

1. Confirm that the battery is physically charging or discharging in the state you expect.
2. Confirm that Fox-facing current direction and status agree with that physical state.
3. Confirm that charged and discharged energy counters are progressing in the correct direction.
4. If this is a fresh zero-counter installation, establish additional genuine charging before deliberately accumulating substantial discharge.
5. Allow enough time for Fox app/cloud values to refresh before drawing conclusions from one screen update.
6. Avoid unnecessary emulator reboot or reflash while establishing the initial charged history.

Do not:

- falsify cumulative counters;
- manually invent starting totals;
- defeat Fox plausibility behaviour;
- change protocol source code simply because fallback appears; or
- interpret fallback alone as proof that a CAN mapping has changed.

If behaviour still appears inconsistent, capture the relevant CAN and app evidence and compare the observation with [TESTING-AND-EVIDENCE.md](TESTING-AND-EVIDENCE.md) and [CAN-FRAME-MAP.md](CAN-FRAME-MAP.md).

---

## Reboot and firmware-update warning

> **Important: frozen v67 does not yet preserve its locally generated cumulative histories across Battery-Emulator reboot or reflash.**

The v67 cumulative installation counters are RAM-only.

A Battery-Emulator reboot or firmware reflash can therefore reset the local sources used for:

- `0x187A` cumulative charged energy;
- `0x187A` cumulative discharged energy;
- `0x1878` absolute throughput derived from the local energy pair;
- `0x1879` cumulative charged capacity;
- `0x1879` cumulative discharged capacity; and
- the locally derived and transmitted equivalent-cycle value.

Fox can retain history from before the local reset.

Observed or expected consequences of that loss of local continuity include:

- negative Daily Charged Energy;
- negative Daily Discharged Energy;
- Total Charged fallback behaviour returning; and
- the locally derived/transmitted Battery Cycles value returning to a lower value or zero after its source history is lost.

Real hardware testing directly observed negative daily values after a local counter reset. The result is consistent with Fox retaining a higher previous cumulative reference while Battery-Emulator restarted from lower local totals.

This project does **not** claim that Fox Cloud's own internally stored cycle count has been proved to reset.

The value known to reset in frozen v67 is the locally generated and transmitted equivalent-cycle source because the cumulative local history feeding it is volatile.

Persistence is a known deferred upstream requirement and the principal remaining release-quality blocker.

The charge-first commissioning procedure described in this guide can reduce problems associated with an initial zero-counter start.

It **does not solve restart or reflash persistence**.

---

## If the emulator has already been rebooted or reflashed

After a reboot or reflash of frozen v67, do not assume that the new local zero-start histories match whatever cumulative reference Fox retained from the previous session.

Check the Battery Details presentation for symptoms including:

- negative Daily Charged Energy;
- negative Daily Discharged Energy;
- Total Charged returning to fallback behaviour; or
- the locally transmitted cycle value returning to a lower value.

These symptoms should not immediately be interpreted as evidence that the CAN field mapping has changed or stopped working.

A more likely explanation, where they appear immediately after a v67 restart or reflash, is loss of the volatile local cumulative history.

Establish fresh controlled charging and discharging observations before diagnosing a mapping fault.

Do not attempt to repair this by entering arbitrary starting totals or fabricating counters.

A proper persistence and migration mechanism remains deferred upstream work and is outside the scope of this guide.

---

## Battery-agnostic commissioning

The FoxESS EP protocol is intended to consume the generic data and limits supplied by the selected Battery-Emulator battery integration.

Do not copy Nissan Leaf-specific values from the hardware-validation installation as though they were EP protocol requirements.

In particular, do not copy battery-specific:

- cell count;
- battery capacity;
- temperature assumptions;
- SOC limits;
- voltage limits;
- current limits;
- resistance values; or
- safety settings.

The Nissan Leaf installation demonstrates real interoperability with a non-Fox battery through Battery-Emulator.

It does not define the required configuration for other battery integrations.

Each supported battery integration remains responsible for presenting appropriate generic battery state, permissions, limits and safety information to Battery-Emulator.

---

## Safety boundaries

This commissioning guide concerns protocol communication and establishment of battery data history.

Battery and inverter protections remain authoritative.

During commissioning:

- respect the charge and discharge limits supplied through the supported battery integration;
- do not force contactors closed;
- do not bypass BMS or system faults;
- do not exceed battery voltage limits;
- do not exceed battery current or power limits;
- do not exceed battery temperature limits; and
- do not alter safety behaviour merely to obtain a desired FoxESS app value.

The EP protocol must operate inside the normal Battery-Emulator safety path.

High-voltage installation, wiring, isolation, protection and regulatory requirements are outside the scope of this document.

---

## Commissioning checklist

- [ ] Battery integration healthy
- [ ] No relevant system or battery faults
- [ ] Battery CAN communication live
- [ ] Battery voltage plausible
- [ ] Battery SOC plausible
- [ ] Battery temperature plausible
- [ ] Charge and discharge limits plausible
- [ ] Inverter recognises the virtual EP battery
- [ ] Contactors close normally
- [ ] Charging state and Fox-facing current direction correct
- [ ] Genuine charged history established
- [ ] Charged counters progress during charging
- [ ] Controlled discharge operates normally
- [ ] Discharging state and Fox-facing current direction correct
- [ ] Discharged counters progress during discharge
- [ ] Total Charged behaviour checked
- [ ] User understands that v67 cumulative counters are volatile across reboot/reflash
- [ ] Normal operation started only after basic communication and power-flow validation

---

## Related documentation

For deeper protocol detail and the evidence behind this guide, see:

- [README.md](../README.md) — project overview, status and limitations
- [CAN-FRAME-MAP.md](CAN-FRAME-MAP.md) — detailed FoxESS EP CAN frame and field map
- [ENERGY-COUNTERS.md](ENERGY-COUNTERS.md) — cumulative energy, capacity, Total Charged fallback and cycle behaviour
- [TESTING-AND-EVIDENCE.md](TESTING-AND-EVIDENCE.md) — real-hardware validation and evidence chronology
- [FIRMWARE-REVERSE-ENGINEERING.md](FIRMWARE-REVERSE-ENGINEERING.md) — manager-firmware-derived structural evidence and remaining unresolved questions
- [KNOWN-LIMITATIONS.md](KNOWN-LIMITATIONS.md) — known limitations, evidence boundaries and deferred upstream work
