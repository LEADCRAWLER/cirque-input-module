# cirque-input-module (tweaked)

A Zephyr module providing a driver for Cirque Pinnacle 1CA027 ASIC-based trackpads (TM035035, TM040040, etc.) used in ZMK keyboards.

This is a fork of the original [cirque-input-module](https://github.com/petejohanson/cirque-input-module) via [geeksville's fork](https://github.com/geeksville/cirque-input-module), with power optimization, additional operating modes, and reliability improvements.

## Features

### Operating Modes

- **Relative mode** — Standard mouse-like relative X/Y deltas (default)
- **Absolute mode** — Reports absolute X/Y positions with configurable scaling and clamping (`absolute-mode`)
- **Abs-rel mode** — Reads absolute position internally but reports relative deltas, with `BTN_TOUCH` events for finger on/off detection (`abs-rel-divisor`) — useful for automatic layer switching

### Power Management

| Mode | Current @ 3.3V | How to enable |
|------|----------------|---------------|
| Active (finger moving) | 2.9 mA | Default when feed is enabled |
| Idle (no finger) | 1.7 mA | Default when no touch detected |
| Sleep (hardware) | ~40 µA | `sleep` DT property |
| Shutdown | ~0.23 µA | Automatic on PM suspend (`CONFIG_INPUT_PINNACLE_PM`) |
| Power gated | 0 µA | `supply-gpios` DT property |

- **Hardware sleep mode** — Pinnacle enters ~40 µA low-power sleep after ~5 seconds of no touch, with configurable scan interval (`sleep-interval`) controlling the wake latency vs. power tradeoff
- **PM device callbacks** — Full Zephyr `PM_DEVICE_ACTION_SUSPEND`/`RESUME` support. On suspend: disables feed, enters shutdown mode, optionally cuts VDD via supply GPIO. On resume: restores power, performs software reset, recalibrates, and re-enables feed
- **Supply GPIO power gating** — If `supply-gpios` is configured, VDD is cut entirely on suspend for true 0 µA draw
- **Adaptive sample rate** — Dynamically reduces sample rate during low-movement periods to save active-mode current (`adaptive-sample-rate`, `high-sample-rate`, `low-sample-rate`)
- **Z-idle packet reduction** — Configurable idle packet count (`idle-packets-count`) to reduce unnecessary MCU wakeups after finger lift in absolute mode
- **ZMK idle sleeper** — Optional integration that links Pinnacle sleep to ZMK activity state (`CONFIG_ZMK_INPUT_PINNACLE_IDLE_SLEEPER`)

### Configuration

- **ADC sensitivity** — 1x (most sensitive) to 4x (least sensitive, lowest power) via `sensitivity` property
- **Edge sensitivity tuning** — Per-axis minimum Z thresholds (`x-axis-z-min`, `y-axis-z-min`) via Extended Register Access (ERA)
- **Axis inversion** — `x-invert`, `y-invert`
- **90-degree rotation** — `rotate-90` swaps X and Y axes
- **Tap control** — Disable all taps (`no-taps`) or just secondary taps (`no-secondary-tap`)
- **Absolute mode scaling** — Scale output to arbitrary width/height with configurable min/max clamping

### Reliability

- **Guaranteed interrupt recovery** — DR interrupt is always re-enabled in the work callback regardless of which code path executes, preventing permanent loss of data-ready notifications
- **Bounded polling timeouts** — ERA register access and calibration loops have maximum iteration counts to prevent infinite hangs
- **Software reset on PM resume** — Full reset sequence on wake (matching init) ensures registers return to known defaults before reconfiguration
- **Feed disable during reconfiguration** — Data feed is disabled before modifying configuration registers per Pinnacle datasheet recommendations

## Devicetree Properties

All properties are defined in `dts/bindings/input/cirque,pinnacle-common.yaml`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `dr-gpios` | phandle-array | — | Data ready GPIO (required) |
| `supply-gpios` | phandle-array | — | Optional VDD supply control GPIO for power gating |
| `sleep` | boolean | false | Enable hardware sleep mode (~40 µA after 5s idle) |
| `sleep-interval` | int | 50 | Sleep mode scan interval (higher = lower current, longer wake latency) |
| `sensitivity` | string | — | ADC attenuation: `"1x"`, `"2x"`, `"3x"`, `"4x"` |
| `x-axis-z-min` | int | 5 | X-axis edge sensitivity threshold |
| `y-axis-z-min` | int | 4 | Y-axis edge sensitivity threshold |
| `absolute-mode` | boolean | false | Enable absolute position reporting |
| `abs-rel-divisor` | int | 0 | If non-zero, use absolute mode internally but report relative events with touch |
| `idle-packets-count` | int | 0 | Z-idle packets after finger lift (absolute mode only) |
| `adaptive-sample-rate` | boolean | false | Enable dynamic sample rate adjustment |
| `high-sample-rate` | int | 100 | Active sample rate (samples/sec) |
| `low-sample-rate` | int | 25 | Idle sample rate (samples/sec) |
| `rotate-90` | boolean | false | Swap X and Y axes |
| `x-invert` | boolean | false | Invert X axis |
| `y-invert` | boolean | false | Invert Y axis |
| `no-taps` | boolean | false | Disable all tap detection |
| `no-secondary-tap` | boolean | false | Disable secondary (right-click) tap only |
| `absolute-mode-scale-to-width` | int | 1024 | Scale absolute X to this range |
| `absolute-mode-scale-to-height` | int | 1024 | Scale absolute Y to this range |
| `absolute-mode-clamp-min-x` | int | 128 | Minimum X for absolute mode |
| `absolute-mode-clamp-max-x` | int | 1920 | Maximum X for absolute mode |
| `absolute-mode-clamp-min-y` | int | 64 | Minimum Y for absolute mode |
| `absolute-mode-clamp-max-y` | int | 1472 | Maximum Y for absolute mode |

## Kconfig Options

| Option | Default | Description |
|--------|---------|-------------|
| `INPUT_PINNACLE` | y | Enable the Cirque Pinnacle driver |
| `INPUT_PINNACLE_INIT_PRIORITY` | `INPUT_INIT_PRIORITY` | Device initialization priority |
| `INPUT_PINNACLE_PM` | y (if `PM_DEVICE`) | Enable PM suspend/resume callbacks (shutdown on suspend) |
| `ZMK_INPUT_PINNACLE_IDLE_SLEEPER` | n | Link Pinnacle sleep to ZMK idle/active state (requires `ZMK_MOUSE`) |

## Credits

- [Pete Johanson](https://github.com/petejohanson) — Original cirque-input-module driver
- [geeksville](https://github.com/geeksville) — Shutdown mode, absolute mode, abs-rel mode fork
