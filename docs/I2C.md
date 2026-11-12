# ADC — Photoresistor (LDR) with EMA Filter + Electret Microphone Threshold

**Goal (one sentence):** Read two analog sensors on the Pico 2 W through ADC0 (GP26): an LDR, filtered with an exponential moving average (EMA) and converted to volts and a 0–100 % light level, and an electret microphone, compared against a fixed raw threshold to flag "high noise".

---

## Prediction

The RP2350 ADC is 12-bit with VREF = 3.3 V, so:

- `1 LSB = 3.3 V / 4096 ≈ 0.806 mV`  `V = raw · 3.3 / 4095`
- **LDR level:** `Nivel = (y − LDR_MIN) · 100 / (LDR_MAX − LDR_MIN)`, with `LDR_MIN = 9` and `LDR_MAX = 3600` taken from the darkest and brightest raw values I saw before writing the code. Covered → **≈ 0 %**; flashlight on top → **high, but < 100 %** (the flashlight was never pointed perfectly into the LDR).
- **EMA filter:** `y = α·x + (1 − α)·y` with `α = 1/16`. At one sample every 100 ms, the time constant is ≈ `Ts / α = 0.1 s · 16 = 1.6 s`, so after a sudden light change the filtered value should only reach ~63 % of the jump after ~1.6 s and ~95 % after ~4.8 s. **I expect `Filtro` to visibly lag `Raw`.**
- **Electret threshold:** the mic signal is biased around mid-scale (`3.3 V / 2 = 1.65 V ≈ 2048`). The threshold `2110` is `2110 · 3.3 / 4095 ≈ 1.70 V`, i.e. ~62 LSB (~50 mV) above the bias point, so only a reasonably loud sound should print "Ruido alto".

## Setup

- Pin map: `LDR_GPIO_PIN` = GP26 → `LDR_ADC_INPUT` = ADC0. The same pin/channel is used for the electret (one sensor at a time, separate programs).
- **LDR:** voltage divider between 3.3 V and GND, midpoint to GP26. The reading **goes up with light**, so the LDR is on the 3.3 V side and the fixed resistor on the GND side.
  ![LDR divider on the breadboard, midpoint wired to GP26](../img/adc_ldr_setup.jpg)
- **Electret:** electret capsule with its bias resistor, a coupling capacitor and a resistor divider that centers the signal around mid-supply before GP26.
  ![Electret microphone front end on the breadboard, output to GP26](../img/adc_electret_setup.jpg)
- Non-default: none. `adc_gpio_init(26)` disables the digital input buffer and pulls on the pin; ADC in free single-shot mode (`adc_read()`), default 150 MHz `clk_sys`. USB serial monitor at 115200 baud.

## What I did

1. Initialized the ADC (`adc_init`, `adc_gpio_init(26)`, `adc_select_input(0)`) and printed raw values to find the dark/bright extremes of the LDR → used them as `LDR_MIN = 9` and `LDR_MAX = 3600`.
2. Added the EMA filter from the course notes (`α = 0.0625`), seeded with the first sample so it doesn't start from 0.
3. Converted the filtered value to volts and to a percentage, and printed `Raw | Filtro | Voltaje | Nivel` every 100 ms.
4. Recorded a demo: hands covering the LDR, then uncovered (room light), then a phone flashlight on top and moving away → `img/adc_ldr_demo.mp4` (audio removed).
5. Swapped the LDR for the electret front end and flashed `adcmicrosencillo.c`, which prints the raw value every 100 ms and labels it "Ruido alto" when `raw > 2110`, otherwise "Ruido normal".

---

## Evidence

| #  | File | What it shows | Reading on the serial monitor |
| --- | --- | --- | --- |
| E1 | `img/adc_ldr_demo.mp4` | ~0:00–0:04: LDR covered with both hands | Raw = 37, Filtro = 37, 0.03 V, Nivel = 0 % |
| E2 | `img/adc_ldr_demo.mp4` | ~0:06: hands removed, room light | Raw ≈ 782–787, **Filtro ≈ 488–489**, 0.39–0.40 V, Nivel = 13 % (filter still climbing) |
| E3 | `img/adc_ldr_demo.mp4` | ~0:09: light increasing | Raw = 799–1028, Filtro = 742–765, 0.60 V, Nivel = 20 % |
| E4 | `img/adc_ldr_demo.mp4` | ~0:12: flashlight on the LDR | **Raw = 3039 / 2906, Filtro = 2397 / 2429**, 1.93–1.96 V, Nivel = 66–67 % |
| E5 | `img/adc_ldr_demo.mp4` | ~0:15: flashlight moving away | **Raw = 1072, Filtro = 2653**, 2.14 V, Nivel = 73 % (filter still high) |

<video controls muted width="100%" src="recursos/imgs/adc_ldr_demo.mp4"></video>

Spot-check of the conversion with E4: `2397 · 3.3 / 4095 = 1.93 V` ✔ and `(2429 − 9) · 100 / 3591 = 67 %` ✔ — the printed values match the formulas.

## What went wrong

- **Wiring error on the temperature sensor:** we made a mistake when connecting the temperature sensor, so it did not give a valid reading at first and we had to recheck the wiring before it worked. *(Evidence to be added.)*
- **No instrumented electret, so only a simple noise detector:** we didn't buy the electret module with its own amplifier, so we only had the bare capsule with a basic bias/divider front end. Its signal is small, so instead of actually measuring sound level we built a simple noise detector: one raw sample compared against a fixed threshold (`2110`, ≈ 1.70 V) that prints "Ruido alto" / "Ruido normal".
- **Calibrating the potentiometer reading took some effort:** getting the potentiometer measurement to map correctly to its full range took several tries, because the real minimum and maximum raw values were not exactly 0 and 4095. *(Evidence to be added.)*

## Code

```c
// LDR — EMA filter + conversion (Fotorresistencia.c)
uint16_t x = adc_read();                       // 0..4095
if (!y_init) { y = (float)x; y_init = true; }  // seed with first sample
else          y = alpha * (float)x + (1.0f - alpha) * y;   // alpha = 1/16

uint16_t y_u16 = (uint16_t)(y + 0.5f);
float voltaje  = (y_u16 * 3.3f) / 4095.0f;
int porcentaje = ((y_u16 - LDR_MIN) * 100) / (LDR_MAX - LDR_MIN);

// Electret — fixed threshold (adcmicrosencillo.c)
uint16_t m = adc_read();
if (m > 2110) printf("Raw=%u | Ruido alto\n", m);    // ~1.70 V, ~62 LSB above mid-scale
else          printf("Raw=%u | Ruido normal\n", m);
sleep_ms(100);
```

Full files at `src/adc/Fotorresistencia.c` and `src/adc/adcmicrosencillo.c`.

## Open question
The Pico's internal pull-ups are weak and both modules may already have their own pull-up resistors. Are the resulting rise times acceptable at 100 kHz, and could I check them on the oscilloscope?
