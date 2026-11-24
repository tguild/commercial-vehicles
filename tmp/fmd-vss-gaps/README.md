# Fleet data points not covered by VSS: define or derive?

Status: working draft, 23 Nov 2026. A proposal for the COVESA Commercial and Fleet Vehicle Group. Not adopted.

## Sources

- [Fleet Management Data (FMD) sheet](https://docs.google.com/spreadsheets/d/1FMtSLhJy1REgadlnSwAqPX7r57rqdLYyw4W7ZsWyg8E), tab "Pillar -> UseCase -> Data Needs", v4.6.
- An OEM partner's VSS mapping of 158 signals (Nov 2026), and Geotab's review of it.
- VSS 6.1, and VSS master after 6.1, which adds ACEA FMS v5 (PR #932; checked at 9dd09ad8).

## Approach

For each data point that has no VSS signal, the first question is whether a consumer can derive it from existing signals. A new signal is proposed only when at least one of these holds:

1. **ECU-only information.** The ECU computes the value from information that isn't exposed elsewhere in VSS.
2. **Lifetime counter.** A consumer installed later, or one that loses connectivity, can't rebuild it.
3. **Regulated meaning.** Regulation already fixes what it means, so vehicles already implement it.
4. **Safety- or claim-critical.** A heuristic isn't acceptable.
5. **Expensive to derive.** Deriving it needs high-rate or bulky inputs.

Everything else becomes a **derivation rule**. These rules belong with the FMD sampling guidance, not in the VSS tree.

## Proposed signals (`fmd-gaps.vspec`)

The overlay holds 10 proposals: 15 signals and 1 branch. It validates against VSS master with vss-tools 1.9.9.

| Proposal | Signal(s) | Source of the need | Rules |
|---|---|---|---|
| Collision type | `Safety.CollisionType` (FRONT/REAR/LEFT/RIGHT/ROLLOVER/MULTIPLE) | OEM partner (crash type); FNOL; emergency dispatch | 1, 4, 5 |
| Battery thermal event | `TractionBattery.ThermalEventState` (NONE/RISK/DETECTED) | OEM partner (battery fire alarm); UN GTR 20 warning | 1, 3, 4 |
| DPF regeneration | `DieselParticulateFilter.RegenerationStatus`, `.SootLoad` | FMD row 75; OEM partner (DPF warning) | 1 |
| DEF consumed | `DieselExhaustFluid.TotalConsumption` | FMD "aftertreatment fuel used" | 1, 2 |
| DEF inducement | `DieselExhaustFluid.Inducement.{Level, Reasons, DistanceToRestriction, TimeToRestriction}` | OEM partner (12 AdBlue text warnings) | 1, 3 |
| Idle fuel | `FuelSystem.TotalIdleConsumption` (J1939 SPN 236), `.TotalIdleConsumptionGaseous` | FMD rows 76, 99 | 2 |
| V2G/V2L energy out | `TractionBattery.Charging.AccumulatedDischargedEnergy` | FMD row 88 | 2 |
| Tire monitoring status | `Tire.PressureStatus` (LOW/CRITICALLY_LOW/HIGH/FAST_LEAK/SENSOR_ERROR) | OEM partner (TPMS malfunction) | 1, 3, 5 |
| Cell temperature extremes | `TractionBattery.Temperature.IdMax`, `.IdMin` (mirror `CellVoltage.IdMax/IdMin`) | OEM partner (cell temperature index) | 5 |

Naming follows existing VSS patterns:
- The state enums use the `Safety.Rollover` and `RoadIcingState` set: UNKNOWN/NONE/RISK/DETECTED.
- Lifetime counters follow `TotalConsumption` and `AccumulatedChargedEnergy`.
- The cell index follows `CellVoltage.IdMax`.

`Tire.PressureStatus` is shown on one wheel only. Open VSS PR 940 refactors wheel designation, so the upstream form should be decided after that.

### Not proposed, and why

| Data point | Instead |
|---|---|
| Satellite count (FMD row 3; OEM partner) | `CurrentLocation.HorizontalAccuracy` and `.VerticalAccuracy` (m) plus `GNSSReceiver.FixType`. Accuracy serves the "validate GPS" purpose better. |
| Idle EV energy out/in (FMD 93–94; OEM partner) | A derivation rule (below). EVs don't usually track idle energy separately. |
| Airbag and parking brake fault (FMD 63) | **On hold.** FMS tell-tales come in the next release, and VSS draft PR #951 (trouble codes) may cover these. |
| Onboard charger output (OEM partner) | A derivation rule (below); the approximation is adequate for fleet use. |
| Heartbeat, device power and memory (FMD 4, 35–36) | Device-side, outside VSS scope. |

## Derivation rules

These are for FMD sampling guidance. Here, "idle" means `PropulsionSystem.IsReady` (or `CombustionEngine.IsRunning`) is true and `Vehicle.IsMoving` is false.

| Value | Rule | Needs |
|---|---|---|
| Trip value of any lifetime counter | Reading at trip end minus reading at trip start | The counter |
| Idle fuel (if a fleet-wide idle definition is wanted) | Add up the change in `HighResTotalConsumption` across idle periods, or integrate `FuelRate` (l/h) while idle | Next release (FMS v5). On 6.1 it isn't possible, because `InstantConsumption` is l/100 km and has no value at standstill. |
| Idle gaseous fuel | Add up the change in `TotalConsumptionGaseous` across idle periods | Next release; there is no gaseous fuel rate |
| Idle EV energy out | Add up the change in `AccumulatedConsumedEnergy` across idle periods, or integrate negative `CurrentPower` | 6.1 (with `IsReady` from the next release) |
| Idle EV energy in | ≈ 0 at standstill; plugged-in energy counts as charging | — |
| Onboard charger output | `TractionBattery.CurrentVoltage` and `CurrentCurrent` while AC charging (`ChargeCurrent.PhaseN` > 0). Check: AC input power minus `Charging.PowerLoss`. Reads low by the high-voltage auxiliary load. | 6.1 |
| Cell temperature index (until IdMax/IdMin exist) | argmax or argmin over `Temperature.CellTemperature` | The full array |
| AC vs DC charging | `ChargeCurrent.DC` > 0 means DC; `ChargeCurrent.PhaseN` > 0 means AC | 6.1 |
| Fast tire leak (until PressureStatus exists) | Rate of fall of `Tire.Pressure` above a threshold | Closely spaced samples |
| DEF consumed (until TotalConsumption exists) | Fall in `DieselExhaustFluid.Level` × `Capacity`, ignoring refill jumps | Coarse: 1% of tank |
| Heartbeat | Report `Vehicle.CurrentDateTime` (next release) on a fixed interval | — |
| Accident reconstruction | Event-triggered high-rate capture of `Acceleration.*`, `AngularVelocity.*`, `Speed`, `Brake.PedalPosition`, `SteeringWheel.Angle` | A sampling rule, not a signal |

## Corrections made in FMD sheet v4.6

These signals exist in VSS, but the sheet mapped the data point to the wrong one:

- **Total fuel used:** was `ConsumptionSinceStart`, which counts only since ignition on. Now `FuelSystem.TotalConsumption` (lifetime, next release).
- **Charging energy in:** was `Charging.ChargeRate`, which is km of range added per hour. Now `AccumulatedChargedEnergy`.
- **EV charging power:** was `GrossCapacity`, which is battery size. Now `CurrentPower`, read while charging.
- **Hydrogen level and used:** now `FuelSystem.RelativeLevel` (SupportedFuelTypes H2) and `TotalConsumptionGaseous`. No Hydrogen branch is needed.
- **Earlier proposals:** malformed or swapped names replaced (DPF, idle EV energy, H2 idle).
- **Door status:** `isOpen` → `IsOpen`.

## Under discussion: diagnostics (VSS draft PR #951)

> **Status: under development. Not merged.** Findings that rely on `Diagnostics.DTCCount` or `DTCList` are provisional.

- **Issue [#936](https://github.com/COVESA/vehicle_signal_specification/issues/936)** (Cox Automotive, Aug 2026) says `DTCList` only fits light-vehicle OBD-II codes, formatted per SAE J2012. It cannot hold heavy-duty J1939 faults: SPN, FMI, occurrence count and source address.
- **Draft PR [#951](https://github.com/COVESA/vehicle_signal_specification/pull/951)** (opened 10 Nov 2026, last updated 20 Nov) adds `Diagnostics.ActiveTroubleCodes`, an array of the protocol-neutral `VehicleDataTypes.TroubleCode`. The record holds:
  - `ECU`, `CodeSystem` and `Code`
  - `Status`: ACTIVE, PENDING, INACTIVE or UNKNOWN
  - `OccurrenceCount` and `LastObserved`
  - `Details`: defined by a versioned `DetailsSchema` URI
- **Open review points:**
  - Typed J1939 SPN/FMI.
  - COVESA-owned Details schemas, for example a J1939 DM1 schema with lamp status.
  - A schema digest for offline edge caching.

If #951 is adopted as drafted:
- `Status = ACTIVE` stops `DTCCount > 0` over-reporting when it is used as the engine warning light.
- J1939 lamp status would give heavy-duty MIL, amber and red-stop state directly.
- Grouping faults by code letter would apply only to `SAE-J2012` codes.
- The held airbag and parking brake fault proposals may become unnecessary.

## Open issues to raise with COVESA

1. **Tell-tale naming.** `HMI.TellTales.TellTale[1..75].Status` (next release) is numbered by FMS1 PGN FD7D and has no lamp names. A consumer needs the ACEA FMS table to know which number is airbag, brake, TPMS or MIL.
2. **Collision severity.** Should `CollisionType` come with a severity measure, such as delta-V or peak acceleration? Insurance FNOL would use it.
3. **J1939 cross-references.** SPNs 236, 3700 and 3719 in the comments should be checked by a J1939 owner before submission.
4. **Joint authorship.** The OEM partner raised collision type, thermal event, DEF inducement, TPMS status and cell temperature index independently. Propose co-authoring.

## Files

- `fmd-gaps.vspec`: the overlay with the proposed signals.
- `README.md`: this file.
