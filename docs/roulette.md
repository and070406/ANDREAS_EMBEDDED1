# LED Roulette with Interrupts

## Session — 5-LED Roulette Controlled by GPIO Interrupts

**Goal (one sentence):** Build a 5-LED roulette that lights one LED at a time, controlled by hardware interrupts instead of polling, where one button checks for a win on the middle LED and a second button cycles the roulette speed.

---

### Setup

- Pin mapping: GP2–GP6 drive the 5-LED bus (`MASK_LED = 0x1F << 2`); GP7 is the win/restart button; GP8 is the speed button. Both buttons use the internal **pull-up** (`gpio_pull_up()`), so each pin reads `1` when idle and drops to `0` on press.
- Unlike polling-based labs, both buttons are read through **hardware interrupts** set on `GPIO_IRQ_EDGE_RISE`, so each callback fires the instant the button is released back up, not while it's held down.
- Three variables are shared between the interrupts and the main loop — `gane`, `counter`, and `vel` — and all three are declared `volatile`, since a value written inside an interrupt could otherwise be cached and never show up in `main()`.

### What I did

1. Wired the 5-LED bus on GP2–GP6 and two push-buttons on GP7 and GP8, both with internal pull-ups.
2. Registered a single interrupt callback (the Pico SDK only allows one global callback per core) and wrote a router function, `escoge_boton`, that checks which pin fired and forwards it to the right handler.
3. Wrote the main loop to run the roulette by polling only a `counter` variable — lighting `1u << (2 + counter)` and advancing every `vel` milliseconds — while all button logic lives entirely in the interrupt handlers.

### Exercise — Win detection and speed control

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include "hardware/gpio.h"
#include <stdio.h>


#define BOTON_PIN 7  
#define BOTON_PIN2 8  


const uint32_t MASK_LED = 0x1F << 2;


volatile bool gane = false;  
volatile int counter = 0;  
volatile int vel = 500;  


void stop_callback(uint gpio, uint32_t events)
{

    if (gpio == BOTON_PIN && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de ganar presionado\n");

        if (!gane)
        {

            if (counter == 2)
            {
                printf("¡Ganaste! Empieza el parpadeo\n");
                gane = true;
            }
            else
            {

                printf("Fallaste, sigue jugando\n");
            }
        }
        else
        {
            printf("Reiniciando el juego\n");
            gane = false;  
            counter = 0;  
        }
    }


    gpio_acknowledge_irq(gpio, events);
}


void boton_velocidad(uint gpio, uint32_t events)
{
    if (gpio == BOTON_PIN2 && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de velocidad presionado\n");


        if (vel == 500)
        {
            vel = 250;
        }
        else if (vel == 250)
        {
            vel = 100;
        }
        else if (vel == 100)
        {
            vel = 500;
        }
    }

    gpio_acknowledge_irq(gpio, events);
}

void escoge_boton(uint gpio, uint32_t events)
{
    if (gpio == BOTON_PIN)
    {
        stop_callback(gpio, events);
    }
    else if (gpio == BOTON_PIN2)
    {
        boton_velocidad(gpio, events);
    }
}

int main(void)
{
    stdio_init_all();


    gpio_init(2);
    gpio_init(3);
    gpio_init(4);
    gpio_init(5);
    gpio_init(6);
    gpio_init(BOTON_PIN);
    gpio_init(BOTON_PIN2);


    sio_hw->gpio_oe_set = MASK_LED;


    sio_hw->gpio_oe_clr = (1u << BOTON_PIN) | (1u << BOTON_PIN2);

    // Pull-up: el pin normalmente está en "1" (alto), y baja a "0"
    // cuando presionas el botón (conecta a GND)
    gpio_pull_up(BOTON_PIN);
    gpio_pull_up(BOTON_PIN2);


    gpio_set_irq_enabled_with_callback(BOTON_PIN, GPIO_IRQ_EDGE_RISE, true, &escoge_boton);


    gpio_set_irq_enabled(BOTON_PIN2, GPIO_IRQ_EDGE_RISE, true);


    while (true)
    {
        if (gane)
        {

            sio_hw->gpio_set = MASK_LED;
            sleep_ms(200);
            sio_hw->gpio_clr = MASK_LED;
            sleep_ms(200);

        }
        else
        {

            sio_hw->gpio_clr = MASK_LED;  
            sio_hw->gpio_set = (1u << (2 + counter));  
            sleep_ms(vel);  

            printf("GPIO: %d \n", sio_hw->gpio_in);

            counter++;  
            if (counter > 4)  
                counter = 0;  
        }
    }
}
```
### Evidence

<video width="65%" controls preload="metadata">
  <source src="../recursos/imgs/roulette.mp4" type="video/mp4">
  Tu navegador no soporta la reproducción de video HTML5.
</video>

### What went wrong

We did not use the `gpio_set_irq_enabled_with_callback` so when we ran the code and we stopped the light it did not matter what LED it was you will always win

### Open question

- Without software debounce, a single physical press can sometimes register as more than one edge — would a time-based check (comparing against `time_us_32()`) be the right fix, similar to what the exam later required?