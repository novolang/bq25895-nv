# bq25895-nv

A driver for TI's BQ25895, an I2C-controlled single-cell lithium charger
with a 5 A switching stage, an input from 3.9 V to 14 V, a boost mode that
powers the input pin from the cell, and a battery monitor.  It is specified
in TI's data sheet "BQ25895 I2C Controlled Single Cell 5-A Fast Charger
with MaxCharge for High Input Voltage and Adjustable Voltage 3.1-A Boost
Operation", SLUSC88C, March 2015, revised October 2022 (the brief of this
package names it SLUSC88).  The driver implements the charger side of the
`PowerManager` trait and the `Identifiable` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv).  Zephyr has no
driver for this part.

## Install

```sh
novo pkg add bq25895-nv
```

## Example

A device program sets up the charger and watches the charge over I2C.  The
breakout's SDA and SCL go to the board's I2C pins and the charger answers at
0x6A.  The program checks the part number, disables the watchdog (`begin`),
limits the input to 1500 mA and the fast charge to 1024 mA at 4208 mV,
turns the charge on, and once a second runs the battery monitor and prints
one line over RTT.  The phase is 0 idle, 1 precharge, 2 constant current, 3
constant voltage, 4 done, 5 fault and 6 charging (the part's fast charge,
which it does not split into the constant current and constant voltage
phases).

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_power.{ ChargeState, MilliAmps, MilliVolts, PowerManager }
use bq25895
use bq25895.{ Bq25895 }

// The charger's address.
const ADDR = 0x6A

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

// The number a phase prints as.
fn phase_number(s: ChargeState) -> Int
    match s
        ChargeState.ChargeIdle            => 0
        ChargeState.ChargePrecharge       => 1
        ChargeState.ChargeConstantCurrent => 2
        ChargeState.ChargeConstantVoltage => 3
        ChargeState.ChargeDone            => 4
        ChargeState.ChargeFault           => 5
        ChargeState.ChargeActive          => 6

@tier(embedded)
fn main() [hw]
    let charger = Bq25895.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(50)
    if not charger.begin()
        hal.uart.write("bq25895: no answer\n")
        return
    let _ = charger.input_set_current_limit(MilliAmps { ma: 1500 })
    let _ = charger.charge_set_current(MilliAmps { ma: 1024 })
    let _ = charger.charge_set_voltage_limit(MilliVolts { mv: 4208 })
    let _ = charger.set_charging(true)
    loop
        match charger.measure()
            Some(a) =>
                hal.uart.write("bq25895")
                field(" mV=", a.vbat_mv)
                field(" mA=", a.ichg_ma)
                field(" phase=", phase_number(charger.charge()))
                match charger.faults()
                    Some(f) => field(" fault=", f.now)
                    None    => hal.uart.write(" fault=?")
                hal.uart.write("\n")
            None =>
                hal.uart.write("bq25895: no result\n")
        hal.timer.delay_ms(1000)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 17,000 bytes | 6,358 bytes | 3,252 bytes |
| nRF54LM20 DK | 17,984 bytes | 6,528 bytes | 3,236 bytes |
| FRDM-MCXN947 | 19,332 bytes | 8,416 bytes | 2,748 bytes |

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), on the
pins `docs/hal/README.md` names under "The Nordic DKs' buses", so the
images carry the driver's reading path.  The driver's own code allocates
nothing: every buffer is a fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries a BQ25895 yet.  The program
builds, runs under QEMU to its refusal (the emulated board has no I2C
device), and the driver's functions run over a double of the charger that
follows the register map of the data sheet.  A reply from a charger over a
board's bus has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The steps of the fields are pure.  The driver performs no
input or output of its own and allocates nothing of its own.  The bus
functions of devices-nv build one short frame per transaction inside a
`system` block and release it when the call returns.  The module checks at
every tier, `embedded` included; the exceptions are the host-scheduler
async impl at the end of the module, which carries `@tier(app)`, and
`int_arm`, which takes the effect of the pin-edge source it is handed.  The
waits are the port's: the board's type waits through its bus, `[hw]`.

## What it is

A charger takes power from an input (a USB port or an adapter) on the pin
VBUS, supplies the system on SYS and charges the cell on BAT.  A lithium
cell is charged in phases: at a low precharge current while it is deeply
discharged, at a constant fast-charge current up to the charge voltage, at
that constant voltage while the current falls, until the current is below
the termination current.  The part runs these by itself with the settings
in its registers, and the host changes them.

| Register | What it holds |
|---|---|
| REG00 | the input current limit (IINLIM, 100 mA and 50 mA a step to 3.25 A), the ILIM pin enable, the high-impedance mode |
| REG02 | the ADC conversion start and rate, the boost frequency, the input current optimizer, the adapter detections |
| REG03 | the watchdog reset, the boost mode (OTG), the charge enable, the minimum system voltage (3.0 V to 3.7 V) |
| REG04 | the fast charge current (64 mA a step to 5056 mA) |
| REG05 | the precharge and termination currents (64 mA and 64 mA a step to 1024 mA) |
| REG06 | the charge voltage (3.840 V and 16 mV a step to 4.608 V), the recharge threshold, the precharge threshold |
| REG07 | the charge termination, the watchdog period, the safety timer |
| REG09 | the battery FET and ship mode |
| REG0A | the boost voltage (4.55 V and 64 mV a step to 5.51 V) |
| REG0B | the status: the input type, the charge phase, power good |
| REG0C | the faults |
| REG0D | the absolute input voltage limit |
| REG0E to REG12 | the battery monitor: battery, system and VBUS voltage, the thermistor, the charge current |
| REG13 | the input current limit in effect |
| REG14 | the register reset, the optimizer status, the part number and revision |

The bus is I2C at up to 400 kHz, address 0x6A.  A write is the register and
the byte in one transaction, and a read is the register, a repeated start
and the data.  Several registers can be read in one transaction, except
REG0C.

## What the package contains

| Module | What is in it |
|---|---|
| `bq25895` | The type `Bq25895`, the trait `Bq25895Port` and the host-scheduler `Bq25895Host`; `Bq25895Adc` and `Bq25895Faults`; the bus framing, the steps of the fields, the input, the charge, the watchdog, the status and the faults, the battery monitor, the boost mode and the rail, the ship mode |

## How to choose an entry point

A program on a board uses `Bq25895`.  It is built by `attach` with the
board's bus, its GPIO and the wiring, as a board's device table builds it.
The waits go through the bus (`I2cBus.delay_ms`), so there is no timer slot.  It implements `PowerManager` (the charger side) and
`Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()` | makes the bus ready, checks the part number and disables the watchdog |
| `set_charging(on)` | the charge on or off |
| `charge_current()`, `charge_set_current(i)` | `PowerManager`: the fast charge current, the step at or below the request |
| `charge_voltage_limit()`, `charge_set_voltage_limit(v)` | `PowerManager`: the charge voltage, the step at or below the request |
| `input_current_limit()`, `input_set_current_limit(i)` | `PowerManager`: the input limit register, the step at or below the request |
| `set_precharge_current(ma)`, `set_termination_current(ma)` | the two currents that bound the charge |
| `input_current()` | the input limit in effect, which follows input source detection |
| `battery_health()` | `PowerManager`: over-voltage, cold or hot thermistor from REG0C, else good |
| `measure()` | one battery monitor conversion, waited for, and its five readings |
| `battery()` | `PowerManager`: the battery voltage and the charge current after one conversion, the phase, no percent |
| `charge()`, `faults()` | the phase now, and the faults since the last read and the present ones |
| `feed_watchdog()` | resets the watchdog timer |
| `set_boost(mv, khz)`, `enable_boost(on)` | the boost output |
| `rail_count()` and the rail methods | the boost output is rail 0 |
| `power_source()` | the input, from REG0B |
| `device_id()` | `Identifiable`: the part number field of REG14, 7, with the revision as `rev` |
| `int_level()` | the level of the INT line, -1 for a wiring with none |

The module's functions take a `Bq25895Port`, the trait a type that holds a
bus implements, so a test runs them over a double.  They hold more than the
methods: the safety timer, the thresholds, the optimizer, the detections,
the continuous conversion, the ship mode and the absolute input voltage
limit.

The charger's limits are `PowerManager` methods.  A setter sets the step of
the field at or below the request and answers the value the part holds, and
`None` when the request is below the field's range (100 mA for the input
limit, 3840 mV for the charge voltage, a negative current) or the part did
not take it; a request above the range sets the top of it.  The package's
own functions (`charge_current_set`, `charge_voltage_set`,
`input_current_set`) answer `-1` for a refusal and set the bottom of the
range for a request below it.

### The watchdog

After a power-on reset the part is in default mode with its watchdog
expired, and charges on its register defaults.  Any write moves it to host
mode, where the watchdog counts 40 s by default and, when it expires, puts
every register back at its default except IINLIM, VINDPM, VINDPM_OS and the
battery FET bits.  `begin` writes WATCHDOG = 00, so the part keeps the
program's settings with no feeding, and the safety timer still ends a
charge.  A program that wants the watchdog sets it with
`watchdog_set(secs)` (40, 80 or 160) and calls `watchdog_feed` before the
period ends.

### The faults

REG0C keeps the faults since its last read, and a second read answers the
present state; the part sends no INT for a new fault until the register has
been read.  `faults_read` reads it twice, alone, since REG0C does not
support a multi-byte read, and answers both: `past` and `now`.  The
decoders are `fault_watchdog`, `fault_boost`, `fault_charge` (1 input, 2
thermal shutdown, 3 safety timer), `fault_battery` and `fault_ntc` (the
temperature sensor's state is always the present one).  `charge()` answers
`ChargeFault` when `now` holds a watchdog, charge, battery or NTC fault.

### The input limit

IINLIM is changed by the part after it identifies the input (a USB host
500 mA, a CDP 1.5 A, a DCP 3.25 A, an unknown adapter 500 mA), and the
input current optimizer may lower it again.  The limit in effect is REG13's
IDPM_LIM, which `input_current_effective` reads.  With EN_ILIM set the limit
is the lower of the register and the ILIM pin.

### The battery monitor

A one-shot conversion is started by CONV_START and the part clears the bit
when it is done; the data sheet's maximum time is 1 second.  `adc_measure`
starts it, pauses 10 ms at a time until the bit clears, and reads REG0E to
REG12 in one transaction.  In continuous mode (CONV_RATE) the part converts
every second and `adc_measure` reads the latest.  The battery voltage is
2.304 V and 20 mV a step; the system voltage the same; VBUS 2.6 V and 100 mV
a step; the thermistor 21 % and 0.465 % a step of REGN; the charge current
50 mA a step.  The charge current reads 0 when the battery is below
V(BATSHORT), and the monitor works with only the battery for a battery
voltage above the minimum system voltage.  The charge current is the only
current it measures: `battery()` answers a discharge as 0 mA.

### The boost mode

The boost output (OTG) powers VBUS from the cell, 4.55 V to 5.51 V in 64 mV
steps.  It is configuration: `boost_set(mv, khz)` sets the voltage and the
frequency, 500 or 1500 kHz, and `boost_enable(on)` sets OTG_CONFIG.  The
mode starts when the OTG pin is high and no input is detected.  The
frequency bit is written first, since the part ignores a write to it when
OTG_CONFIG is enabled.  The rail methods of `PowerManager` act on this
output: rail 0 is on when OTG_CONFIG is set and REG0B says OTG, faulted when
the boost fault bit is set; `rail_set_voltage` takes the highest step at or
below the request.

## Async forms

The charger completes a battery monitor conversion in the background, and
reports events on INT.

On a device, `int_arm(edge, w, f)` arms the INT pin on a pin-edge source
(`AsyncEdge` of embedded-hal-nv) for the falling edge, the start of the
part's 256 microsecond pulse, and resolves the Future `f` that the caller
awaits with `core.rt.future_await`; the caller then reads `faults_read` and
`status_read`.  `int_disarm` takes the arm back.  No board's GPIO
implements `AsyncEdge` today, so the source is a program's own interrupt
glue; the suite runs the function over a double.

On the host scheduler, `host_of(port)` answers a `Bq25895Host`, which
implements `PowerManagerAsync`: `battery_measure` starts a conversion, parks
the task 10 ms at a time until the part clears CONV_START (at most 120
times), and answers the battery.  The struct is not a `@value` struct, since
an `async fn` takes none, and its impl is `@tier(app)`, since the embedded
tier refuses an `async fn`.

## The rules a user needs

1. Settings are masked updates.  A setter reads its register, changes its own
   bits and writes it back, so one setting does not undo another.  The bits
   that clear themselves (CONV_START, WD_RST, FORCE_ICO, REG_RST) are written
   as 0 by an update of another bit of the same register.
2. A program that does not feed the watchdog must have it disabled, or the
   part returns to its defaults every 40 s.  `begin` does.
3. The input current limit read from REG00 is not the limit in effect.
4. Read REG0C twice.  `faults_read` does.
5. The charge current is measured in charge mode only, and reads 0 below
   V(BATSHORT).
6. `reset` writes REG_RST and waits for it to clear; it also resets the
   safety timer.  The charge settings go back to the part's defaults, which
   are for a 2 A charge at 4.208 V.
7. Ship mode (`ship_mode`) removes the system's power.  The part leaves it
   when an input is plugged, QON is pulled low, or BATFET_DIS is cleared.
8. The charge also needs the CE pin low.  `charge_enable` sets the register
   bit only.

## What is not included

- The thermal and temperature thresholds (BHOT, BCOLD, TREG), the IR
  compensation (REG08), the current pulse control (EN_PUMPX, PUMPX_UP and
  PUMPX_DN) and the MaxCharge adapter's own settings beyond the three
  detection enables; `reg_update` reaches the bits.
- The two fast-charge phases apart: the part reports fast charging for both
  the constant-current and the constant-voltage phase, and `charge()`
  answers `ChargeActive` for both.
- A dead cell in `battery_health()`, and a battery temperature: the part has
  no flag for the first and reads its thermistor as a percent of REGN, not
  as a temperature.
- A discharge current, which the part does not measure.
- A percent of charge: a charger has no gauge.  `battery().percent` is
  `None`.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` (which also waits), `GpioOut` and `AsyncEdge` traits and the
  board handles.
- axp2101-nv drives a PMIC with a charger, and bq27441-nv and
  max17048-nv drive fuel gauges, on the same `PowerManager` trait.

## Tests

`novo test` runs the module over a double of the charger that answers on an
I2C bus with the reset values of the data sheet's tables, a REG0B that the
test sets, a REG0C whose first read answers the faults since the last read
and whose later reads answer the present state, a battery monitor that loads
its readings when a one-shot conversion ends, the bits that clear
themselves, and a reset that holds REG_RST for a few reads.  It checks the
framing of every entry point, the steps and limits of every field, the
masked updates and their refusals, the watchdog, the status, the fault
register read twice, the monitor and its bound, the boost mode in the order
the part needs, the rail, the awaitable line over a pin-edge double, the
host-scheduler form and the type over the host's board handles.  The suite
covers every line of `src/`.

The steps are checked against the data sheet's ranges and defaults, for
instance IINLIM 001000 is 500 mA, ICHG 0100000 is 2048 mA, VREG 010111 is
4.208 V and BOOSTV 1001 is 5.126 V.  `tests/coverage.sh` merges the
suite's coverage over `src/`.

Two mutations were checked and the suite turned red: reading REG0C once and
answering the same byte for the past and the present faults, and writing the
boost voltage before the frequency.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the
data sheet above and the `bq25895` crate (DanNixon/bq25895, 0.0.5).  Zephyr
has no driver for the part, and Embassy and esp-hal hold none.  Each fix
kept is named in `src/bq25895.nv` where it is kept: one transaction for a
register write, a masked update for every field, and the fault register
read twice.

## Licence

Apache-2.0.
