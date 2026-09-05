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

![Register-level Frecuency measurement](recursos/imgs/Registerlevel_frecuency.jpeg)

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

---

# Session 4 

**Goal session 3 (one sentence):** Complete 4 different level exercices that reproduce four classic pattern animations with four LEDs

## Setup

- Pin map: GP10 → PIN_D, GP11 → PIN_C, GP12 → PIN_B, GP13 → PIN_A (4-bit bus, one LED + resistor per pin); pushbutton on the breadboard for reset.

- Photo of the raspberry Pi Pico setup:


---

## What I did

1. Connected the LEDs to GP10-GP13.
2. Reused the register-level configuration validated in Session 3 (gpio_init per pin + a single sio_hw->gpio_oe_set with a 4-bit mask) instead of the SDK calls.
3. Wrote the coude for each exercise.
4. Run each program and recordered it as evidence.

- Setuo code we used in the exercises:
```python
#define PIN_A 13
#define PIN_B 12
#define PIN_C 11
#define PIN_D 10

const uint32_t MASK =
    (1u << PIN_A) | (1u << PIN_B) | (1u << PIN_C) | (1u << PIN_D);

gpio_init(PIN_A);
gpio_init(PIN_B);
gpio_init(PIN_C);
gpio_init(PIN_D);

sio_hw->gpio_oe_set = MASK;   // OE = Output Enable
``` 

## Exercise 1

### 4 bit binary counter
Goal is to use the LEDs as a 4-bit-binary number wich counts from 0 to 15 in a cycle.
```python
int counter = 0;

while (true) {
    sio_hw->gpio_clr = MASK;
    sio_hw->gpio_set = (counter << 10);
    sleep_ms(500);

    counter++;
    if (counter > 15) {
        counter = 0;
        sleep_ms(1000);
    }
}
```
 ### Evidence
 

---

## Exercise 2

### Bouncing light
The goal for this exercise was to move the LED across four positions (0001,0010,0100,1000,0100,0010,0001) and back.

```python
int counter = 0;

while (true) {
    sio_hw->gpio_clr = MASK;
    sio_hw->gpio_set = (1u << (counter + 10));
    sleep_ms(500);

    counter++;
    if (counter > 3) {
        while (counter > 0) {
            counter--,
            sio_hw->gpio_clr = MASK;
            sio_hw->gpio_set = (1u << (counter + 10));
            sleep_ms(500);
        }
    }
}
```
### Evidence

---

## Exercise 3
### Fill and empty animation
The goal was to turn LEDs on one by one starting from right and going to the left and then turning them off, we did this using a gpio_togl.

```python
int counter = 0;

while (true) {
    sio_hw->gpio_togl = (1u << (counter + 10));
    sleep_ms(500);

    counter++;
    if (counter > 3) {
        counter = 0;
    }
}
```
### Evidence

---


## Exercise 4
### Fill from the outside inward
The goal was to light the two outer LEDs, then the two inner ones and then turning them off in the inverse sequence

```python
int counter = 0;

while (true) {
    sio_hw->gpio_togl = (1u << (counter + 10));
    sio_hw->gpio_togl = (1u << (13 - counter));
    sleep_ms(500);

    counter++;
    if (counter > 1) {
        counter = 0;
    }
}
```
### Evidence

---

## What went wrong
We made a mistake with code 2 at the moment of writing the code, and in some of the code we could have optimeze them more.

## Open question
How could I know how to optimize my code before someone else spots it. 