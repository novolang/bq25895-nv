# Changelog

## 0.2.0 — 2026-11-12

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0. The release depends on both.

- `set_charge_current`, `set_charge_voltage` and `set_input_current` on
  `Bq25895` are gone.  `PowerManager.charge_set_current`,
  `charge_set_voltage_limit` and `input_set_current_limit` replace them, with
  the getters `charge_current`, `charge_voltage_limit` and
  `input_current_limit`.  They answer `Option`, not `-1`, and set the step at or
  below the request: a request below the field's range (a negative current,
  under 100 mA for the input limit, under 3840 mV for the charge voltage)
  answers `None` where the package functions raised it to the bottom of the
  range.  `input_current()` stays, the limit in effect.
- `charge_state_of` and `charge()` answer `ChargeActive` for the fast charge,
  where 0.1.0 answered `ChargeConstantCurrent`, a phase the part cannot
  identify.
- `battery_health()` answers `HealthOvervoltage` for BAT_FAULT, `HealthCold` and
  `HealthOverheat` for a cold or hot thermistor, `HealthFault` for any other NTC
  code and `HealthGood` otherwise.
- `attach(bus, pins, w)` has no `BoardTimer`: the waits go through
  `I2cBus.delay_ms`.

## 0.1.0

The first release: a driver for TI's BQ25895 single-cell I2C fast charger.

- `Bq25895`, the type a board's device table builds, implementing the
  charger side of devices-nv's `PowerManager` and `Identifiable`, with its
  own methods for the charge, the currents and voltages, the input limit,
  the battery monitor, the watchdog, the faults and the boost output.  The
  boost output is the one rail.
- The register access as functions over `Bq25895Port`, a trait that a type
  holding a bus implements, with every setting a masked update and the
  self-clearing bits of REG02, REG03 and REG09 written as 0.
- The steps of every field in both directions, rounded down and held to the
  part's range: the input current limit, the fast charge, precharge and
  termination currents, the charge voltage, the minimum system voltage, the
  absolute input voltage limit, the boost voltage and the monitor's
  readings.
- The input (the limit, the ILIM pin, the high-impedance mode, the absolute
  voltage limit, the optimizer, the detections), the charge (the enable,
  the thresholds, the termination, the safety timer), the watchdog (set,
  feed, and disabled by the bring-up) and the ship mode.
- The status of REG0B with its decoders, the fault register of REG0C read
  twice and alone, and the phase for `PowerManager` with a fault overriding
  it.
- The battery monitor: the one-shot conversion with its bit that clears
  itself, the continuous mode, the five readings in one transaction, and
  `adc_measure`, which waits in 10 ms pauses for the conversion.
- The boost mode as configuration: the voltage and the frequency, with the
  frequency written first, and OTG_CONFIG.
- Two async forms: `int_arm` and `int_disarm`, which arm the INT pin on any
  `AsyncEdge` source for the falling edge, and `host_of`, which gives the
  host scheduler's `PowerManagerAsync` and parks until the conversion ends.
- The suite runs over a double of the charger and covers every line of
  `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a BQ25895.
- The data sheet read is SLUSC88C (revised October 2022); the brief names
  SLUSC88.
- `PowerManager` has no charger current or charge voltage limit, so those
  are the package's own functions beside the trait.
- The part reports fast charging for the constant-current and the
  constant-voltage phases alike; `charge()` answers `ChargeConstantCurrent`
  for both.
- The thermal thresholds, the IR compensation and the current pulse control
  are not implemented.
- No board's GPIO implements `AsyncEdge`, so `int_arm` has no source on a
  board yet.
