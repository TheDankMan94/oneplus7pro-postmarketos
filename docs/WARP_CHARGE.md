# Warp Charge on the OnePlus 7 Pro (guacamole): research notes

Status: **not implemented**. Written 2026-11-07 after reading the vendor kernel
(LineageOS/android_kernel_oneplus_sm8150, GPL-2.0).

## What the hardware is

- An STM8S microcontroller ("stm8s_fastcg", I2C address 0x26) on the same QUP I2C
  bus as the fuel gauge (`i2c@a80000`). The device tree intentionally does not
  describe it.
- Four GPIO lines from the stock overlay: `charging_switch1`, `charging_reset`,
  `charging_clock`, `charging_data`.
- When Warp Charge is active the MCU, not the PM8150B charger, feeds the battery
  directly (the "switch" GPIO moves the battery path from the PMIC to the MCU).
  The PMIC charger's current and temperature limits therefore no longer apply.

## What the vendor code does (about 3000 lines before the charger integration)

- `drivers/power/oplus/vooc_ic/oplus_stm8s.c` (about 1000 lines): MCU firmware
  check, firmware update over I2C, shutdown sequence.
- `drivers/power/oplus/oplus_vooc.c` (about 2500 lines): the state machine. It
  does the adapter handshake on the data/clock lines with microsecond timing,
  steps the current, applies per-temperature current tables (the stock overlay has
  about 30 `qcom,vooc_strategy*` temperature and current values), watches the
  battery, detects shorts, and returns control to the PMIC charger.
- It is tied into `oplus_charger.c`, the gauge and the adapter code, and into
  userspace services.

## Why it was not ported

- A mistake in the temperature or current handling can overheat the cell, and the
  PMIC's own protections are bypassed while the MCU charges.
- It cannot be tested safely without a charger/battery test setup, and a wrong
  handshake can also leave the MCU in a bad state.
- The existing mainline-style path already charges at 5 V / 3 A over USB-PD
  (see CLAUDE.md patches 0018 to 0024). Do not re-add 9 V / 12 V PDOs without new
  evidence.

## If someone wants to continue

1. Capture a real Warp Charge session on stock Android first (logcat/dmesg of
   `oplus_vooc`), to see the exact GPIO and I2C sequence.
2. Port the state machine with every temperature and over-current check kept, and
   default it to off (module parameter), with the PMIC charger as the fallback.
3. Test with the battery temperature visible and a hard current limit, one step
   at a time.
