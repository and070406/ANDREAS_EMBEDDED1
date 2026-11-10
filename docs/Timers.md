# Timers

## Session — Repeating Timers and Timing Measurements

**Goal (one sentence):** Generate a periodic signal with a Pico SDK repeating timer (without blocking `main()`) and measure how the callback duration and the sign of the delay affect the real timing.

---

### Setup

- Pin mapping: GP15 is the timed output (oscilloscope CH1), GP14 marks the callback duration (CH2), GND is the probe ground.
- Toggling a GPIO every `TOGGLE_MS` gives a waveform period of `2 * TOGGLE_MS`, so `f = 1000 / (2 * TOGGLE_MS)`.
- A **negative** delay schedules callbacks relative to their start time; a **positive** delay waits until the callback finishes.

### What I did

1. Wrote a repeating timer that toggles GP15 every 250 ms and predicted period, frequency and duty cycle before measuring.
2. Changed only `TOGGLE_MS` (250 → 100 → 50 → 10 ms) and compared predicted vs. measured frequency.
3. Added `busy_wait_us_32(WORK_US)` inside the callback and compared `-TIMER_DELAY_MS` vs `+TIMER_DELAY_MS`.

### Exercise — Measure a repeating timer

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "pico/time.h"

#define SIGNAL_PIN          15
#define CALLBACK_MARK_PIN   14
#define TIMER_DELAY_MS      100
#define WORK_US             20000   // 0 for the baseline
#define USE_NEGATIVE_DELAY  1       // 1 = start-to-start, 0 = end-to-start

static repeating_timer_t timer;

static bool timer_callback(repeating_timer_t *rt) {
    static bool state = false;

    gpio_put(CALLBACK_MARK_PIN, 1);
    state = !state;
    gpio_put(SIGNAL_PIN, state);
    busy_wait_us_32(WORK_US);       // intentional workload
    gpio_put(CALLBACK_MARK_PIN, 0);

    return true;                    // keep repeating
}

int main() {
    stdio_init_all();

    gpio_init(SIGNAL_PIN);
    gpio_set_dir(SIGNAL_PIN, GPIO_OUT);
    gpio_init(CALLBACK_MARK_PIN);
    gpio_set_dir(CALLBACK_MARK_PIN, GPIO_OUT);

    int32_t delay_ms = USE_NEGATIVE_DELAY ? -TIMER_DELAY_MS : TIMER_DELAY_MS;
    add_repeating_timer_ms(delay_ms, timer_callback, NULL, &timer);

    while (true) {
        printf("Main is still running\n");
        sleep_ms(1000);
    }
}
```

| Configuration | Expected interval | Measured interval |
| ------------- | ----------------- | ----------------- |
| `-100 ms`, `WORK_US = 0` |  |  |
| `-100 ms`, `WORK_US = 20000` |  |  |
| `+100 ms`, `WORK_US = 20000` |  |  |

### AI Assistance

**Prompt used:**

> _Paste the prompt here_

**AI response (summary):**

> _Paste the response here_

**What I verified / changed:**

- _Write here_

### Evidence

_Paste oscilloscope screenshots / video here_

### What went wrong

_Write here_

### Open question

- _Write here_