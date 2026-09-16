# Digital Inputs

## Session 6 — Two-Button Logic Gates and a Shifting LED

**Goal (one sentence):** Configure two push-buttons with internal pull-downs as digital inputs, use them to emulate OR, AND, and XOR logic on a single LED, and then reuse the same buttons to shift a lit LED across four positions.

---

### Setup

- Pin mapping: GP2 drives the LED used for the OR/AND/XOR exercises; GP6 and GP7 are the two push-buttons, both configured as inputs through `gpio_pull_down()`.
- For the shifting exercise, the LED output expands to a 4-bit bus on GP2–GP5 (`MASK_LED = 0xF << 2`), while the buttons stay on GP6 and GP7.
- Important detail: since these buttons rely on internal **pull-down** resistors, pressing a button drives the line **high (1)**, unlike the pull-up wiring used in the previous session — no `!` negation is needed here.
- `stdio_init_all()` and a `printf()` call are used in the OR/AND/XOR versions to print the raw button-mask value to the serial console on each loop iteration, which was useful for confirming the readings while debugging.

### What I did

1. Wired a single LED on GP2 (plus GP3–GP5 for the shifting exercise) and two push-buttons on GP6 and GP7, each configured as an input with `gpio_pull_down()`.
2. Built three variations of essentially the same program — same setup, only the condition evaluating the two button states changes — so the LED behaves as an OR, an AND, and an XOR gate.
3. Wrote a fourth program that uses one button to advance a position counter and the other to move it back, lighting only the LED at the current position and wrapping around at both ends of the 4-bit bus.
4. Captured each behavior on video as supporting evidence.

### Exercise 1 — AND gate

The LED only turns on when **both** buttons are pressed at the same time (with pull-downs, "pressed" reads as 1, so no negation is required).

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;

    
        int resultado = (entaadas & MASK_BOTONES);
        printf("Botones: %d\n", resultado);
        if (entaadas & 1<<6 && entaadas & 1<<7) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```
<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/AND_Gate.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

### Exercise 2 — OR gate

The LED turns on if button A **or** button B (or both) are pressed.

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;

    
        int resultado = (entaadas & MASK_BOTONES);
        printf("Botones: %d\n", resultado);
        if (entaadas & MASK_BOTONES) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```
<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/OR_gate.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>
### Exercise 3 — XOR gate

The LED turns on when **exactly one** of the two buttons is pressed, using boolean variables `a` and `b` combined with the `^` operator instead of masking the raw register directly.

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;
        int a=(entaadas & (1u << 6))!=0;
        int b=(entaadas & (1u << 7))!=0;

        int resultado = (a ^ b);
        printf("Botones: %d\n", resultado);
        if (resultado) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/XOR_gate.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

### Exercise 4 — Shift the lit LED left/right

One button (GP6) advances a `counter` variable and the other (GP7) decreases it; only the LED at position `counter` is lit on the 4-bit bus (GP2–GP5). Two separate flags, `f1` and `f2`, latch each button independently so a press only increments/decrements once instead of repeating while held, and the counter wraps at both edges (0–3). A short `sleep_ms(15)` at the end of the loop acts as simple debounce.

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 0xF << 2;// bits 2,3,4,5 = 0b111100
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(3);
    gpio_init(4);
    gpio_init(5);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    int counter = 0;
    int f1= 0;
    int f2= 0;
while (true) {
        uint32_t entradas = sio_hw->gpio_in;
        int a = (entradas & (1u << 6)) != 0;
        int b = (entradas & (1u << 7)) != 0;

        // --- Botón 1: avanza ---
        if (a && !f1) {
            counter++;
            if (counter > 3) counter = 0;   // wrap 0..3
            f1 = 1;
        } else if (!a && f1) {
            f1 = 0;
        }

        // --- Botón 2: retrocede ---
        if (b && !f2) {
            counter--;
            if (counter < 0) counter = 3;   // wrap 0..3
            f2 = 1;
        } else if (!b && f2) {
            f2 = 0;
        }

        // --- Refresca LEDs cada vuelta ---
        sio_hw->gpio_clr = MASK_LED;               // apaga todos
        sio_hw->gpio_set = (1u << (2 + counter));  // prende solo el actual

        sleep_ms(15); // debounce sencillo
    }
}
```

<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/exercise2.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

### What went wrong

- When we were trying to do the AND exercise we ended up doing the OR exercise so we had to reprogamm it again.
- We also had problems with the hardware, we did not connect right the buttons, so at first it did not work.

### Open question

- Right now `f1` and `f2` are separate flags checked one after another, so if both buttons were pressed at the exact same instant, only the first `if` block (button 1 / advance) would run that cycle. Is that acceptable behavior, or should simultaneous presses be handled some other way (e.g., ignored entirely, like the AND/OR exercises implicitly do)?