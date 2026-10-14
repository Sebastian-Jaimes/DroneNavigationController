# DShot (F405 → ESC)

| | |
|---|---|
| Variante | TBD (DShot300/600, ¿bidireccional?) |
| Pines | TIM5_CH1–CH4 → PA0–PA3 (ver [../pinout.md](../pinout.md)) |
| HW | 220 Ω en serie por línea |
| FW | Timer PWM + DMA |

**Trama:** 11 bits throttle · 1 bit telemetría · 4 bits CRC
- 0 = desarmado · 1–47 = comandos · 48–2047 = acelerador
- CRC = `(v ^ v>>4 ^ v>>8) & 0xF`

**Pendiente:** definir si es bidireccional (afecta la elección de pines/timer por la captura de entrada) y verificar los streams DMA de TIM5 en el F405.

**Verificado:** — (agregar captura de osciloscopio al probar)

Ref: [DShot and Bidirectional DShot (brushlesswhoop.com)](https://brushlesswhoop.com/dshot-and-bidirectional-dshot/)
