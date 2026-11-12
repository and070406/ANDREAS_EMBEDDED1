# Exam 1 — Blackout 3x3

## Practical Exam — 3x3 Lights-Out Puzzle with Bitwise State and Interrupts

**Goal (one sentence):** Build a Lights-Out–style puzzle on a 3×3 LED/button grid, where pressing a button toggles a plus-shaped set of neighboring LEDs, the whole board is tracked as a single bit-packed integer, and a dedicated button restarts the game at any moment via a hardware interrupt.

---

### Setup

- Pin mapping: GP2–GP10 drive the 9-LED grid (`MASK_LED = 0x1FF << 2`); GP11–GP19 are the 9 grid buttons (`MASK_BTN = 0x1FF << 11`); GP20 is the restart button. All ten buttons use internal **pull-up** resistors.
- Unlike the roulette lab's rising-edge interrupts, here the interrupt is set on `GPIO_IRQ_EDGE_FALL` — the moment the button goes down — since the win/lose check needs to react instantly to the press itself, not to its release.
- The exam required: the board state in a single integer, every board operation done with bitwise operators only, no timer peripherals, restart served by interrupt (never polled), a self-written RNG (shifts + XOR, no `rand()`), every interrupt-shared variable marked `volatile`, and restart working from any state — including mid-victory-blink.

### What I did

1. Wired the 9-LED grid on GP2–GP10, 9 grid buttons on GP11–GP19, and a restart button on GP20, all with internal pull-ups.
2. Packed the entire board into one `uint16_t`, where bit `i` represents LED `i` in row-major order (`0 1 2 / 3 4 5 / 6 7 8`).
3. Precomputed a `PLUS_MASK[9]` array so that applying a move is a single XOR against the pressed button's mask — this also makes every randomly generated starting board guaranteed solvable, since XOR is its own inverse.
4. Wrote a self-contained `xorshift32` RNG (only shifts and XOR) seeded at boot and re-mixed with the exact microsecond timestamp on every press, and used it to generate a starting board with `generar_tablero()`.
5. Wrote one shared interrupt callback that routes all ten pins: grid buttons XOR their `PLUS_MASK` into `board`, while the restart pin regenerates the board and clears the win flag unconditionally, with per-pin debounce done by comparing `time_us_64()` timestamps.

### Exercise — Bit-packed board with interrupt-driven moves and restart

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include "hardware/gpio.h"
#include <stdio.h>

#define LED_BASE     2
#define N_LEDS       9
#define BTN_BASE     11
#define N_BTN        9
#define RESTART_PIN  20

// Tiempo minimo entre dos pulsaciones del mismo boton para
// considerarlas distintas (filtra el rebote mecanico). Solo lee
// el contador libre de tiempo, no configura ningun timer.
#define DEBOUNCE_US  30000

const uint32_t MASK_LED = 0x1FF << LED_BASE;
const uint32_t MASK_BTN = 0x1FF << BTN_BASE;

// Estado del tablero: un solo entero, bit i = LED i encendido
static volatile uint16_t board = 0;
static volatile bool gane = false;

static uint32_t rng_state = 1;
static uint64_t ultimo_evt[N_BTN] = {0};
static uint64_t ultimo_evt_restart = 0;

// RNG propio (xorshift): solo shifts y XOR, nada de rand()
static inline uint32_t xorshift32(uint32_t *state)
{
    uint32_t x = *state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    *state = x;
    return x;
}

// Mascaras "plus" precalculadas: aplicar un movimiento = una XOR
static const uint16_t PLUS_MASK[N_LEDS] = {
    0x00B, 0x017, 0x026,
    0x059, 0x0BA, 0x134,
    0x0C8, 0x1D0, 0x1A0
};

static void dibujar_tablero(void)
{
    sio_hw->gpio_clr = MASK_LED;
    sio_hw->gpio_set = ((uint32_t)board) << LED_BASE;
}

// Genera un tablero siempre resoluble: XOR es su propio inverso
static uint16_t generar_tablero(void)
{
    uint16_t nuevo = 0;
    int n = 5 + (xorshift32(&rng_state) % 6);
    for (int i = 0; i < n; i++)
    {
        int idx = xorshift32(&rng_state) % N_LEDS;
        nuevo ^= PLUS_MASK[idx];
    }
    if (nuevo == 0) nuevo = PLUS_MASK[4];
    return nuevo;
}

static void gpio_callback(uint gpio, uint32_t events)
{
    uint64_t ahora = time_us_64();
    rng_state ^= (uint32_t)ahora;

    if (gpio == RESTART_PIN)
    {
        // El restart funciona en cualquier momento, incluso en victoria
        if ((events & GPIO_IRQ_EDGE_FALL) &&
            (ahora - ultimo_evt_restart > DEBOUNCE_US))
        {
            ultimo_evt_restart = ahora;
            board = generar_tablero();
            gane = false;
            dibujar_tablero();
            printf("RESTART -> tablero 0x%03X\n", board);
        }
        gpio_acknowledge_irq(gpio, events);
        return;
    }

    int idx = gpio - BTN_BASE;
    if (idx < 0 || idx >= N_BTN)
    {
        gpio_acknowledge_irq(gpio, events);
        return;
    }

    // Mientras gane == true, los botones de la grid ya no alteran el tablero
    if ((events & GPIO_IRQ_EDGE_FALL) && !gane &&
        (ahora - ultimo_evt[idx] > DEBOUNCE_US))
    {
        ultimo_evt[idx] = ahora;
        board ^= PLUS_MASK[idx];
        printf("Boton %d -> 0x%03X\n", idx, board);

        if (board == 0)
        {
            gane = true;
            printf("BLACKOUT! Ganaste\n");
        }
        dibujar_tablero();
    }

    gpio_acknowledge_irq(gpio, events);
}

int main(void)
{
    stdio_init_all();

    rng_state = (uint32_t)time_us_32() | 1u;

    for (int i = 0; i < N_LEDS; i++)
        gpio_init(LED_BASE + i);
    sio_hw->gpio_oe_set = MASK_LED;

    for (int i = 0; i < N_BTN; i++)
    {
        gpio_init(BTN_BASE + i);
        gpio_pull_up(BTN_BASE + i);
    }
    gpio_init(RESTART_PIN);
    gpio_pull_up(RESTART_PIN);
    sio_hw->gpio_oe_clr = MASK_BTN | (1u << RESTART_PIN);

    board = generar_tablero();
    dibujar_tablero();

    gpio_set_irq_enabled_with_callback(BTN_BASE, GPIO_IRQ_EDGE_FALL, true, &gpio_callback);
    for (int i = 1; i < N_BTN; i++)
        gpio_set_irq_enabled(BTN_BASE + i, GPIO_IRQ_EDGE_FALL, true);
    gpio_set_irq_enabled(RESTART_PIN, GPIO_IRQ_EDGE_FALL, true);

    while (true)
    {
        if (gane)
        {
            sio_hw->gpio_set = MASK_LED;
            sleep_ms(150);
            if (!gane) continue;
            sio_hw->gpio_clr = MASK_LED;
            sleep_ms(150);
        }
        else
        {
            tight_loop_contents();
        }
    }
}
```
### Evidence -game restarting

<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/Exam1_restart.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

---
### Evidence -winning the game
<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/Exam1_win.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

### What went wrong

I had trouble with setting the logic first and we had some troubles with de hardware and connection of the buttons

### Open question

- The debounce timestamps (`ultimo_evt[]`, `ultimo_evt_restart`) are only ever touched inside the interrupt handler itself, never read from `main()` — is it correct that they don't need `volatile` the way `board` and `gane` do, since nothing outside the interrupt context ever reads them?