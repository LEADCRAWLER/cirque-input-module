# Cirque Pinnacle Driver Power Optimization Plan

**Repository:** `https://github.com/alee0729/cirque-input-module`
**Upstream:** `geeksville/cirque-input-module` → `petejohanson/cirque-input-module`
**Date:** 2026-03-20

---

## Context

This repo is a Zephyr module providing a driver for Cirque Pinnacle 1CA027 ASIC-based trackpads (TM035035, TM040040, etc.) used in ZMK keyboards. The geeksville fork already adds shutdown mode during ZMK deep-sleep. This plan identifies further power optimizations rooted in the Pinnacle ASIC register spec and Zephyr PM framework.

### Key Files to Modify

| File | Purpose |
|------|---------|
| `drivers/input/input_pinnacle.c` | Main driver — all register I/O, PM callbacks, data reporting |
| `dts/bindings/cirque,pinnacle*.yaml` | Devicetree bindings — exposes config properties to board overlays |
| `Kconfig` | Build-time configuration flags |

### Pinnacle Register Reference (from GT-AN-090620 v1.6)

| Register | Name | Power-Relevant Bits |
|----------|------|---------------------|
| 0x02 | Status1 | Bit[3] SW_CC, Bit[2] SW_DR — must clear to deassert HW_DR |
| 0x03 | SysConfig1 | Bit[2] Sleep Enable, Bit[1] Shutdown, Bit[0] Reset |
| 0x04 | FeedConfig1 | Bit[1] Data Mode (abs/rel), Bit[0] Feed Enable |
| 0x09 | Sample Rate | Configurable samples/sec |
| 0x0A | Z-Idle | Number of empty packets when no touch (default 0x1E = 30) |
| 0x0C | Sleep Interval | Wake interval during sleep mode |
| 0x0D | Sleep Timer | Timer for entering sleep |

### Power Consumption Reference (from TM035035 spec v1.2)

| Mode | Current @ 3.3V |
|------|----------------|
| Active (finger moving) | 2.9 mA |
| Idle (no finger) | 1.7 mA |
| Sleep (after ~5s, configurable) | 40 µA |
| Shutdown | 0.23 µA |

---

## Phase 1: Quick Wins (Low Risk, High Impact)

### 1.1 Enable Native Sleep Mode by Default

**What:** Assert Bit[2] of Register 0x03 (SysConfig1) during init so the Pinnacle enters its built-in 40 µA low-power sleep after ~5 seconds of no touch.

**Why:** Saves ~1.66 mA during the gap between "user stopped touching" and "ZMK deep-sleep kicks in" (typically minutes). The 300ms wake latency is acceptable for trackpad input.

**Implementation:**
```c
// In the init or configure function, after clearing flags:
// Read current SysConfig1, set bit 2 (Sleep Enable)
uint8_t syscfg;
pinnacle_read(dev, PINNACLE_REG_SYS_CONFIG1, &syscfg);
syscfg |= BIT(2); // Sleep Enable
pinnacle_write(dev, PINNACLE_REG_SYS_CONFIG1, syscfg);
```

**DT binding addition** (cirque,pinnacle.yaml):
```yaml
sleep-mode-enable:
  type: boolean
  description: |
    Enable sleep mode, allowing the hardware to enter a lower power
    mode (~40 µA) after 5 seconds with no finger detected.
    Wake latency is approximately 300ms.
```

**Note:** The upstream Zephyr driver already has this property. Verify the geeksville fork honors it, and consider defaulting it to `true` in the binding or documenting it prominently.

### 1.2 Reduce Z-Idle Packet Count

**What:** Set Register 0x0A to 0 or 1 (instead of default 30) when in absolute mode.

**Why:** Each Z-idle packet triggers HW_DR → MCU interrupt → SPI/I2C transaction → flag clear. Default of 30 means 300ms of unnecessary wakeups every time a finger lifts. Setting to 0 eliminates this entirely.

**Implementation:**
```c
// After configuring absolute mode:
if (cfg->data_mode == PINNACLE_DATA_MODE_ABSOLUTE) {
    pinnacle_write(dev, PINNACLE_REG_Z_IDLE,
                   cfg->idle_packets_count); // DT-configurable, default to 0 or 1
}
```

**DT binding:** Already exists as `idle-packets-count`. Change the default from unset to `0` in the binding YAML.

### 1.3 Disable Data Feed When Not Needed

**What:** Add a helper to clear/set Bit[0] of Register 0x04 (FeedConfig1) to stop/start the measurement system.

**Why:** When the trackpad is logically disabled (e.g., ZMK layer change, or no active consumers), the measurement system can be halted to drop below Idle current.

**Implementation:**
```c
static int pinnacle_set_feed_enable(const struct device *dev, bool enable)
{
    uint8_t feedcfg;
    int ret = pinnacle_read(dev, PINNACLE_REG_FEED_CONFIG1, &feedcfg);
    if (ret) return ret;

    if (enable) {
        feedcfg |= BIT(0);
    } else {
        feedcfg &= ~BIT(0);
    }
    return pinnacle_write(dev, PINNACLE_REG_FEED_CONFIG1, feedcfg);
}
```

Call `pinnacle_set_feed_enable(dev, false)` in the PM suspend callback and `true` in resume.

---

## Phase 2: Zephyr PM Framework Integration

### 2.1 Implement `pm_device_action_cb_t` Callback

**What:** Add a proper Zephyr device PM action callback that handles SUSPEND and RESUME actions by putting the Pinnacle into Shutdown/Active modes.

**Why:** This is the standard Zephyr pattern. It enables both system-managed PM (the MCU suspends all devices before deep sleep) and runtime PM (the driver can be individually suspended when unused). Currently, the geeksville fork likely handles this ad-hoc via ZMK-specific hooks.

**Implementation:**
```c
#ifdef CONFIG_PM_DEVICE
static int pinnacle_pm_action(const struct device *dev,
                              enum pm_device_action action)
{
    struct pinnacle_data *data = dev->data;

    switch (action) {
    case PM_DEVICE_ACTION_SUSPEND:
        /* Disable feed first to stop data generation */
        pinnacle_set_feed_enable(dev, false);
        /* Enter shutdown mode: 0.23 µA */
        pinnacle_write(dev, PINNACLE_REG_SYS_CONFIG1, BIT(1));
        break;

    case PM_DEVICE_ACTION_RESUME:
        /* Clear shutdown bit, restore config */
        pinnacle_write(dev, PINNACLE_REG_SYS_CONFIG1, 0x00);
        /* Wait for POR-equivalent startup */
        k_msleep(50);
        /* Clear CC flag from wakeup */
        pinnacle_write(dev, PINNACLE_REG_STATUS1, 0x00);
        /* Re-run full configuration sequence */
        pinnacle_configure(dev);
        /* Re-enable feed */
        pinnacle_set_feed_enable(dev, true);
        break;

    default:
        return -ENOTSUP;
    }
    return 0;
}
#endif /* CONFIG_PM_DEVICE */
```

**Device definition macro update:**
```c
PM_DEVICE_DT_INST_DEFINE(inst, pinnacle_pm_action);

DEVICE_DT_INST_DEFINE(inst,
    pinnacle_init,
    PM_DEVICE_DT_INST_GET(inst),  /* was NULL */
    &pinnacle_data_##inst,
    &pinnacle_config_##inst,
    POST_KERNEL, CONFIG_INPUT_INIT_PRIORITY,
    NULL);
```

### 2.2 Enable Runtime PM Support

**What:** Add `CONFIG_PM_DEVICE_RUNTIME` support so the Pinnacle can be independently suspended when no input subsystem consumer is using it.

**Why:** On split keyboards, the trackpad half may be idle while keys are being typed on the other half. Runtime PM lets the driver suspend automatically based on usage count.

**DT property to add:**
```yaml
# Users can enable per-device:
zephyr,pm-device-runtime-auto: true
```

**Kconfig addition:**
```kconfig
config INPUT_PINNACLE_PM
    bool "Enable power management for Pinnacle trackpad"
    default y
    depends on PM_DEVICE
    help
      Enable device power management support for the Cirque Pinnacle
      trackpad. When enabled, the trackpad will enter shutdown mode
      during system suspend, reducing current to ~0.23 µA.
```

---

## Phase 3: Hardware-Level Optimizations

### 3.1 Supply GPIO Power Gating

**What:** If the board design includes a `supply-gpios` (load switch / MOSFET on VDD), toggle it in the PM callback for true zero-current draw.

**Why:** Even Pinnacle Shutdown mode draws 0.23 µA. Cutting VDD achieves 0 µA and eliminates bus leakage.

**Implementation:**
```c
static int pinnacle_pm_action(const struct device *dev,
                              enum pm_device_action action)
{
    const struct pinnacle_config *cfg = dev->config;

    switch (action) {
    case PM_DEVICE_ACTION_SUSPEND:
        pinnacle_set_feed_enable(dev, false);
        pinnacle_write(dev, PINNACLE_REG_SYS_CONFIG1, BIT(1));

        /* Cut power if supply GPIO is configured */
        if (cfg->supply_gpio.port != NULL) {
            gpio_pin_set_dt(&cfg->supply_gpio, 0);
        }
        break;

    case PM_DEVICE_ACTION_RESUME:
        if (cfg->supply_gpio.port != NULL) {
            gpio_pin_set_dt(&cfg->supply_gpio, 1);
            k_msleep(50); // Allow power stabilization + POR
        }
        pinnacle_write(dev, PINNACLE_REG_SYS_CONFIG1, 0x00);
        k_msleep(50);
        pinnacle_write(dev, PINNACLE_REG_STATUS1, 0x00);
        pinnacle_configure(dev);
        pinnacle_set_feed_enable(dev, true);
        break;

    default:
        return -ENOTSUP;
    }
    return 0;
}
```

### 3.2 Batch Register Reads

**What:** Use auto-increment reads (SPI filler byte 0xFC / I2C CRA auto-increment) for position data instead of individual register reads.

**Why:** Reading registers 0x12–0x17 (6 bytes) as a batch takes one SPI transaction instead of 6, reducing bus active time and MCU wakeup duration.

**Implementation:**
```c
/* Replace individual reads with batch */
static int pinnacle_read_abs_data(const struct device *dev,
                                  struct pinnacle_abs_data *data)
{
    uint8_t buf[6]; /* Registers 0x12 through 0x17 */
    int ret = pinnacle_seq_read(dev, PINNACLE_REG_PACKET_BYTE_0, buf, 6);
    if (ret) return ret;

    data->btn = buf[0] & 0x3F;
    data->x = buf[2] | ((buf[4] & 0x0F) << 8);
    data->y = buf[3] | ((buf[4] & 0xF0) << 4);
    data->z = buf[5] & 0x3F;
    return 0;
}
```

**Verify** the existing driver already does this — if using petejohanson's base it likely does, but confirm no regressions from the geeksville fork.

### 3.3 Tune Sleep Interval Register

**What:** Expose Register 0x0C (Sleep Interval) as a DT property to let users trade wake latency for power.

**Why:** Longer sleep scan intervals reduce average current below the 40 µA baseline. The TM035035 spec's 40 µA figure was measured at a 3-second scan interval.

**DT binding:**
```yaml
sleep-interval:
  type: int
  default: 50
  description: |
    Sleep mode scan interval in multiples of the base period.
    Higher values reduce average sleep current but increase
    wake-from-sleep latency. Default provides ~300ms wake.
    Maximum value 255.
```

---

## Phase 4: Adaptive Power (Advanced)

### 4.1 Adaptive Sample Rate

**What:** Dynamically adjust Register 0x09 (Sample Rate) based on touch activity — full rate during active movement, reduced rate during near-idle.

**Why:** Lower sample rates reduce Active mode current from the 2.9 mA baseline.

**Implementation sketch:**
```c
/* After reading position data, if delta is very small: */
if (abs(dx) < MOVEMENT_THRESHOLD && abs(dy) < MOVEMENT_THRESHOLD) {
    if (data->activity_counter++ > IDLE_THRESHOLD) {
        pinnacle_write(dev, PINNACLE_REG_SAMPLE_RATE, LOW_SAMPLE_RATE);
    }
} else {
    data->activity_counter = 0;
    pinnacle_write(dev, PINNACLE_REG_SAMPLE_RATE, HIGH_SAMPLE_RATE);
}
```

**Risk:** Medium — adds bus transactions for rate changes. Only implement if profiling shows Active mode is a significant battery contributor.

### 4.2 Sensitivity Auto-Adjustment

**What:** Use 4x ADC attenuation (least sensitive, lowest power) by default and only switch to higher sensitivity when overlay thickness requires it.

**Why:** The sensitivity DT property already exists. Ensure the default (`4x`) is preserved and document that `1x` increases analog front-end power.

---

## Implementation Order (Recommended)

| Priority | Item | Effort | Power Savings |
|----------|------|--------|---------------|
| P0 | 1.1 Native sleep mode | 1 hour | 1.66 mA → 40 µA during idle |
| P0 | 2.1 PM action callback | 2-3 hours | 1.7 mA → 0.23 µA during system suspend |
| P1 | 1.2 Z-Idle reduction | 30 min | Eliminates 30 MCU wakeups per finger-lift |
| P1 | 1.3 Feed disable on suspend | 30 min | Clean shutdown sequence |
| P1 | 3.2 Batch register reads | 1 hour | Reduces SPI bus active time ~6x per read cycle |
| P2 | 2.2 Runtime PM | 2 hours | Auto-suspend when trackpad unused |
| P2 | 3.1 Supply GPIO gating | 1 hour | True 0 µA (board-design dependent) |
| P2 | 3.3 Sleep interval tuning | 30 min | Sub-40 µA sleep current |
| P3 | 4.1 Adaptive sample rate | 3 hours | Reduced Active mode current |

---

## Testing Checklist

- [ ] Verify trackpad wakes from sleep mode within 300ms of touch
- [ ] Verify trackpad wakes from shutdown mode after system resume
- [ ] Measure current in each mode with multimeter/PPK2
- [ ] Confirm no data loss during mode transitions
- [ ] Test with both SPI and I2C configurations
- [ ] Test with `supply-gpios` power gating (if applicable)
- [ ] Verify HW_DR interrupt still fires correctly after sleep/wake cycles
- [ ] Test split keyboard scenario: trackpad half idle while key half active
- [ ] Confirm ZMK deep-sleep → wake → trackpad functional within 1 second

---

## Claude Code Usage

To use this plan with Claude Code, clone your fork and run:

```bash
cd cirque-input-module
claude
```

Then paste or reference this file:

```
@PINNACLE_POWER_OPTIMIZATION_PLAN.md Implement Phase 1 changes (items 1.1, 1.2, 1.3)
starting with the native sleep mode enable in the init function.
Review the existing driver code first and show me what you plan to change.
```

For Phase 2:
```
@PINNACLE_POWER_OPTIMIZATION_PLAN.md Now implement the PM action callback (item 2.1).
Make sure to handle the full suspend/resume lifecycle including re-running
the configuration sequence on wake.
```
