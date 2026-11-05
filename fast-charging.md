# Fast charging on the RG DS Plus: plan

Status: idea, not started. Written 2026-10-30. Nothing here is decided upstream yet. The first step is an issue on
JorreFog/BananaLinux, since it changes behaviour the manufacturer chose.

## Why

BananaOS charges the RG DS Plus at about **7.5 W**. GammaOS charges it noticeably faster. The goal is an opt-in
**Settings › System › Fast charging** toggle, off by default.

## What we found

### Where the 7.5 W comes from

The mainline RK817 charger driver (Linux 7.2.7, `drivers/power/supply/rk817_charger.c`) hard-codes the USB input
current limit to **1.5 A** (`RK817_USB_ILIM_SEL` = `0x03`). It writes this at probe and again in both the plug-in
and plug-out handlers (`rk817_plug_in_isr` / `rk817_plug_out_isr`). 5 V × 1.5 A = 7.5 W.

The battery charge current comes from the device tree's `constant-charge-current-max-microamp`, which is a
placeholder of **10 A**. The DTS comment says it's "likely incorrect". The driver clamps it to the chip's 3.5 A
maximum. At 1.5 A input this never matters.

The driver exposes nothing writable on the charger device (online, USB type and voltages only). On the battery the
only writable property is `charge_full`. Writing the PMIC register from userspace wouldn't stick, because the next
plug-in event writes 1.5 A again.

### Stock firmware vs GammaOS

Taken from the `rk817,charger` node of the device trees in the stock Anbernic image
(`RG-DS-PLUS-EN16GB-V1.0-20260915.IMG`) and GammaOS Core 1.4.4:

| | Input current | Battery charge current | Min. input voltage |
|---|---|---|---|
| Stock Anbernic | 1500 mA (7.5 W) | 2000 mA (~0.5C) | 4.5 V |
| GammaOS 1.4.4 | 3000 mA (15 W) | 3500 mA (~0.8C) | 4.2 V |
| BananaOS today | 1500 mA (7.5 W) | 10 A placeholder, clamped to 3.5 A | VLIM on |

**BananaOS already matches the manufacturer's input limit.** GammaOS goes beyond the stock config on all three
values. The battery is 4247 mAh (`design_capacity`).

### Input current vs charge current

- **Input current** is drawn from the USB charger at 5 V. It sets how much power comes in.
- **Charge current** goes into the battery at its own voltage (~3.6–4.4 V). It sets how hard the battery is pushed,
  which is what matters for heat and wear.

The charger steps 5 V down to the battery's voltage, and the input also has to run the handheld. Roughly, at
~90% efficiency:

    5 V × input current × 0.9 ≈ handheld's own use + battery voltage × charge current

Whichever limit is reached first sets the speed:

- Stock, 1.5 A in, so ~6.75 W usable. Off or asleep: ~1.8 A into the battery, below the 2 A charge limit, so the
  **input** limit binds. Playing at ~2 W: ~1.25 A left for the battery.
- GammaOS, 3 A in, so ~13.5 W usable. Off: ~3.5 A into the battery, exactly its charge limit, so the two are matched.

Near full the battery switches to constant voltage and the current tapers off, so the last 10–20% is slow in every
mode.

## Risk

- Faster charging goes **beyond what Anbernic configured**. The likely cost is battery wear and heat over time,
  not immediate damage.
- Raising the input limit lets the battery side actually reach its limit. So the charge current must be set to a
  real value, not left at the 10 A placeholder (3.5 A after the clamp).
- GammaOS also lowers the input-voltage floor to 4.2 V, which makes a weak charger or cable work harder. Keep stock's
  4.5 V.

## Design

### 1. Kernel patch (`buildroot/board/rgds/linux-patches/`)

Around 40 lines in `rk817_charger.c`:

- Store the input current limit and the charge current in `struct rk817_charger`, starting from the current
  behaviour (1.5 A in).
- Make `POWER_SUPPLY_PROP_INPUT_CURRENT_LIMIT` writable on the charger, and
  `POWER_SUPPLY_PROP_CONSTANT_CHARGE_CURRENT` on the battery. Validate them against the register tables (input:
  the `RK817_USB_ILIM_SEL` steps; charge: `rk817_chg_cur_to_reg`, 0.5–3.5 A).
- Apply the stored values at probe and in both plug handlers, instead of the hard-coded `0x03`.

This is the form that could also go to mainline Linux: standard sysfs properties, no new DT bindings needed.

### 2. Device tree

Set the battery's `constant-charge-current-max-microamp` to a real value instead of 10 A: **2 A**, matching stock.
That becomes the "off" charge current.

### 3. Setting

- `settings.conf` key, e.g. `fast_charging = off` (default `off`; add it to `settings_defaults.cpp`).
- A row in Settings › System with help text saying what it does and that it's beyond the stock setting.
- Strings added to `ui/assets/lang/zh-cn.tsv`.

### 4. Applying it

Write the two sysfs values from `settings.conf` at boot (init script) and whenever the toggle changes. `bananad`
already watches the charger's online state, so it's the other candidate. The kernel keeps the values across plug
events.

| Mode | Input | Charge |
|---|---|---|
| Off (stock) | 1.5 A | 2.0 A |
| On | **TBD**: 2.5 A / 2.5 A (gentler) or 3 A / 3.5 A (GammaOS) | **TBD** |

## Open questions for the maintainer

1. Is a fast-charging option wanted at all, or should BananaOS stay at stock?
2. The "on" values: the GammaOS ones, or something in between?
3. Heat protection: drop back to stock above a battery temperature? Only if the RK817 gauge reports temperature
   reliably on this board. Check first.
4. Whether the RG DS (non-Plus) has the same charger and limits. Only the Plus matters for us.

## Steps

1. Open an issue upstream with the findings and the design above. Wait for the maintainer's call.
2. Branch off `main` (e.g. `feature/fast-charging`).
3. Kernel patch, then build `rgds-plus` and check the sysfs files appear and the values stick across plug/unplug.
4. Measure on the Plus with a USB power meter, in both modes, with the handheld off and while playing. Watch the
   battery temperature.
5. Setting, applying it at boot and on change, and translations.
6. `make test`, then an update package to test on the device, then a PR.

## How the device trees were pulled (to repeat)

Both images were scanned for flattened-device-tree blobs (magic `d00dfeed`) that mention `rk817`, reading in chunks
(memory-mapping the 15 GB stock image crashed). They were decompiled with the `dtc` built with the kernel:
`output/rgds-plus/build/linux-custom/scripts/dtc/dtc -I dtb -O dts`. The container has no system `dtc`.
