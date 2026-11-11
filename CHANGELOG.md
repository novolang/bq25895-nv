# Changelog

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
