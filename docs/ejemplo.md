# Session 3 & 4- Register level & GPIO


**Goal session 3 (one sentence):** Compare the signal and execution time between SDK GPIO timing instructions and registrer-Level.

**Prediction:** The SDK GPIO has a slower execution so it takes a few extra cycles to run the code. This is why I think the frecuency levels of the register level instructions will be higher  


## Setup

- Pin map: GP18 was connected to the LED output and the scope CH1 probe was on GP18

---

## What I did

1. I connected my Pico 2 to a breadboard and opened a new project from "Blink" in the VS Code extension.
2. Ran SDK instructions.
3. Connected and callibrated the oscilloscope  to measure the frequency.
4. Deleted the SDK and replaced it with register-level instructions.
5. Use the oscilloscope again with the same settings.

## Evidence
![SDK Frecuency measurement](recursos/imgs/SDK_frecuency.jpeg)

This image shows the SDK frecuency that was at 792.4 Hz.

![Register-level Frecuency measurement](recursos/imgs/registerlevel_frecuency.jpeg)

This image shows the Register-level frecuency measurement and it was at 1.317 kHz.

---

## What went wrong


- I forgot to  solder my Pi Pico 2 and couldn't use it, so I did it with my classmate.
- When we ran the code we forgot to add 
``` codigo
sio_hw->gpio_oe_set = LED_MASK;
```

---

## Code
### SDK 

``` codigo

gpio_init(LED);
gpio_set_dir(LED, GPIO_OUT);

while (true) {
    gpio_put(LED, 1);
    gpio_put(LED, 0);
}
``` 
### Register-level
```python
const uint32_t LED_MASK = 1u << LED;

gpio_init(LED);
sio_hw->gpio_oe_set = LED_MASK;

while (true) {
    sio_hw->gpio_set = LED_MASK;
    sio_hw->gpio_clr = LED_MASK;
}

```