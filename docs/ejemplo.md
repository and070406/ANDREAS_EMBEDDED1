# Session 3 & 4- Register level & GPIO


**Goal session 3 (one sentence):** Compare the signal and execution time between SDK GPIO timing instructions and registrer-Level.

**Prediction:** The SDK GPIO has a slower execution so it takes a few extra cycles to run the code. This is why I think the frecuency levels of the register level instructions will be higher  

!!! tip "Consejo"
    Mantén este resumen corto (máx. 5 líneas). Lo demás va en secciones específicas.

---

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

## 4) Requisitos

**Software**
- _SO compatible (Windows/Linux/macOS)_
- _Python 3.x / Node 18+ / Arduino IDE / etc._
- _Dependencias (p. ej., pip/requirements, npm packages)_

**Hardware (si aplica)**
- _MCU / Sensores / Actuadores / Fuente de poder_
- _Herramientas (multímetro, cautín, etc.)_

**Conocimientos previos**
- _Programación básica en X_
- _Electrónica básica_
- _Git/GitHub_

---

## 5) Instalación

```bash
# 1) Clonar
git clone https://github.com/<usuario>/<repo>.git
cd <repo>

# 2) (Opcional) Crear entorno virtual
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# 3) Instalar dependencias (ejemplos)
pip install -r requirements.txt
# o, si es Node:
npm install


```