# Arquitectura

Una sola placa (no es un stack), lógica de 3,3 V en todo el sistema.
Diagrama: [Arquitectura.drawio](Arquitectura.drawio) (abrir con draw.io / diagrams.net). Pines: [pinout.md](pinout.md).

```
XT60 → TVS + shunt/INA169 → VIN ─┬─ VIN_MTR → ESC ×4
                                 └─ SSB44 → TPS54331 → 5V_SYS ← B5819W ← USB VBUS
5V_SYS → TPS62162 → 3V3_SYS (MCUs, SD, LEDs) ─PG→ LP5907 → 3V3_SEN (IMU, baro)
F405 ⇄ SPI2 + DMA + IRQ ⇄ G0B1
```

## MCUs
| MCU | Núcleo / memoria | Función |
|---|---|---|
| STM32F405RG (principal) | M4F 168 MHz · 1 MB / 192 KB | PID, actitud, IMU (SPI1), baro (I2C1), blackbox SDIO, DShot, ADC, USB, buzzer/LEDs |
| STM32G0B1RET6 (secundario) | M0+ 64 MHz · 512 KB / 144 KB | ELRS/CRSF, GPS, LiDAR, expansión I2C2/UART |
| Enlace | — | SPI2 del F405 maestro → G0B1 esclavo, full-duplex con DMA + IRQ "dato listo" |

## Energía
| Riel / bloque | Parte | Notas |
|---|---|---|
| Entrada | XT60 + TVS SMBJ18A + 470 µF/35 V Low-ESR | MVP1 = **4S** |
| 5V_SYS | Buck TPS54331 (5 V, 3 A, 570 kHz) | Desde VIN vía diodo SSB44 |
| 3V3_SYS | Buck TPS62162 (3,3 V, 1 A) | F405, G0B1, MicroSD, LEDs |
| 3V3_SEN | LDO LP5907 + ferrita | BMI270 y BMP581; habilitado por PG del TPS62162 |
| USB | VBUS → B5819W → 5V_SYS; USBLC6-2 en D+/D− | |
| Corriente | Shunt 0,5 mΩ + INA169 (RL 27 kΩ + 10 nF) | 13,5 mV/A, ~150 A fondo de escala |
| Tensión | Divisor 10k/1k | VBAT_SENSE |
| Límite de corriente | XT60: ~30 A cont. / ~60 A pico | Por confirmar |

## Sensores y periféricos
| Bloque | Parte | Bus / notas |
|---|---|---|
| IMU | Bosch BMI270 | SPI1 exclusivo + INT1 |
| Barómetro | Bosch BMP581 | I2C1 propio + INT; 0x47 por defecto (jumper BARO_ADDR) |
| Blackbox | MicroSD Molex 472192001 | SDIO 4 bits; TPD6F002 (ESD/EMI) + ESD5Z3.3 en VDD |
| Buzzer | Piezo CPT-12123 + FDV303N | PWM 4 kHz, desde 5V_SYS |
| LEDs | 2 de estado (error, status) | 680 Ω, activos en bajo |
| ESC ×4 | Pads MTR + VIN_MTR/MGND (THT) | 220 Ω en serie + ESD USBLC6-4SC6; [DShot](interfaces/dshot.md) |
| Radio RC | ELRS/CRSF | UART del G0B1 |
| GPS + mag, LiDAR, VTX HD | Externos | UART/I2C del G0B1 |
| Expansión | Conector I2C2/UART + 3V3 + GND | Reemplaza al OLED |

## Mecánica / DFM
- Señales críticas: JST-SH 1,0 mm (top) + pads redundantes (bottom).
- Pads directos: batería, ESC y radio RC.
- Huecos de montaje aislados de la red de potencia (frame de carbono conductor).
- ESD IEC 61000-4-2 nivel 4 en todo lo que sale de la placa.
- Ensamble JLCPCB; stackup de 4 capas (TBD en Fase 6).

## Pendiente
- Esquemático del G0B1 (alimentación, SPI2, UARTs, SWD) y conector ELRS (hoja Receptor aún con nRF24L01P).
- Conectores GPS/LiDAR/expansión con ESD.
- Huellas por verificar: TPD6F002 (DSV 3,0 × 1,35 mm), BMP581, CPT-12123.
- LEDs (color/Vf), pasivos y BOM.
