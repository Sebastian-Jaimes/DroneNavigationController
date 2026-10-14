# Pinout

Única tabla de pines del DNC. El resto de los documentos enlaza aquí.
Fuente: esquemático (`electronics/DNC/`). **Validar en STM32CubeMX** antes del ruteo (AF, timers, DMA).

| Señal | Pin | Periférico | MCU | Notas |
|---|---|---|---|---|
| MTR1 | PA0 | TIM5_CH1 | F405 | DShot, ver [interfaces/dshot.md](interfaces/dshot.md) |
| MTR2 | PA1 | TIM5_CH2 | F405 | |
| MTR3 | PA2 | TIM5_CH3 | F405 | |
| MTR4 | PA3 | TIM5_CH4 | F405 | Movidos desde PC6–PC9 por el conflicto con SDIO |
| IMU_CS (CSB) | PA4 | SPI1_NSS | F405 | BMI270 |
| IMU_SCK | PA5 | SPI1_SCK | F405 | |
| IMU_MISO | PA6 | SPI1_MISO | F405 | |
| IMU_MOSI | PA7 | SPI1_MOSI | F405 | |
| INT1 (IMU) | PC4 | EXTI4 | F405 | Push-pull en FW (sin pull-up) |
| INT (baro) | PC5 | EXTI5 | F405 | Push-pull en FW (sin pull-up) |
| SCL (baro) | PB8 | I2C1_SCL | F405 | BMP581, bus propio |
| SDA (baro) | PB9 | I2C1_SDA | F405 | |
| VBAT_SENSE | PB0 | ADC | F405 | Divisor 10k/1k |
| CURR_SENSE | PB1 | ADC | F405 | INA169 |
| SD DAT0–DAT3 | PC8–PC11 | SDIO | F405 | 4 bits |
| SD CLK | PC12 | SDIO_CK | F405 | |
| SD CMD | PD2 | SDIO_CMD | F405 | |
| BUZZER | PA8 | TIM1_CH1 | F405 | PWM 4 kHz |
| LED_ERROR | PB4 | GPIO | F405 | Activo en bajo. Falta el pin en el bloque de la raíz |
| LED_STATUS | PB5 | GPIO | F405 | Activo en bajo. Ídem |
| UART1_TX / RX | PA9 / PA10 | USART1 | F405 | Conector DEBUG |
| USB D− / D+ | PA11 / PA12 | USB OTG FS | F405 | |
| SWDIO / SWCLK | PA13 / PA14 | SWD | F405 | Conector PROG |
| BOOT0 / BOOT1 | BOOT0 / PB2 | — | F405 | Pull-down 10 k |
| Enlace G0B1 | PB12–PB15 | SPI2 (maestro) | F405 | Reservado, aún sin cablear |
| IRQ "dato listo" | TBD | EXTI | F405 ← G0B1 | |
| Enlace F405 | TBD | SPI (esclavo) | G0B1 | |
| ELRS/CRSF | TBD | UART | G0B1 | |
| GPS | TBD | UART + I2C | G0B1 | |
| LiDAR | TBD | UART/I2C | G0B1 | |
| Expansión | TBD | I2C2/UART | G0B1 | |
| SWD G0B1 | TBD | SWD | G0B1 | |

**Libres en el F405:** PC0–PC3, PC6/PC7 (USART6), PC13, PA15, PB3, PB6/PB7, PB10/PB11 (USART3).

**Regla:** cada interrupción en una línea EXTI distinta (p. ej. PC4 y PB4 comparten EXTI4).
