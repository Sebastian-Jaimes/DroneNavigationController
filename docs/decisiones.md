# Decisiones

Una línea por decisión; la más reciente arriba. Detalle y descartes: historial de git (`git log`, versión antigua de `docs/RESUMEN_PROYECTO_DNC.md`).
"6–11 oct" = sesión de revisión del 6 al 11 de octubre de 2026.

| Fecha | Decisión | Por qué |
|---|---|---|
| 2026-10-12 | Pads de potencia ESC THT + ESD (USBLC6-4SC6) en MTR1–4 | Cable de 16 AWG (~15 A por ESC en el MVP1); las señales MTR salen de la placa |
| 2026-10-12 | TVS SMBJ18A unidireccional (antes SMBJ26CA) | Clamp de 29,2 V, por debajo del máximo absoluto de 30 V del TPS54331 |
| 2026-10-12 | INA169 alimentado desde 3V3_SYS | Su salida no debe superar 3,3 V (entrada ADC del F405) |
| 2026-10-12 | 3V3_SYS con buck TPS62162 (antes LDO AP2112K) | El LDO disipaba ~0,5 W en SOT-23-5 |
| 2026-10-12 | 3V3_SEN (LP5907) habilitado por el PG del TPS62162 | Secuencia: sensores después de 3V3_SYS |
| 2026-10 | MVP1 solo 4S; 6S pospuesto | El TPS54331 llega a 28 V; 6S exige buck de 60 V, TVS de ~28 V y capacitores de 50 V |
| 2026-10 | Huecos de montaje aislados de la red de potencia | El frame de carbono es conductor (riesgo de corto) |
| 2026-10 | Conectores JST-SH + pads redundantes; batería, ESC y RC solo en pads | Modularidad sin perder confiabilidad en vuelo agresivo |
| 6–11 oct | Receptor ELRS/CRSF por UART del G0B1; fuera nRF24L01, ATmega328P y LM358 | Estándar, con telemetría |
| 6–11 oct | Se mantiene el STM32G0B1 como MCU de comunicaciones | Descarga al F405 de periféricos lentos (decisión del autor) |
| 6–11 oct | OLED eliminado → conector de expansión I2C2/UART | No aporta en vuelo |
| 6–11 oct | Barómetro en I2C propio | Evitar latencia y bloqueos del bus compartido |
| 6–11 oct | MicroSD por SDIO 4 bits | Blackbox más rápido; elimina el 74AHC1G125 del bus SPI |
| 6–11 oct | Motores en TIM5 (PA0–PA3) | Liberar PC8/PC9 para el SDIO |
| 6–11 oct | Barómetro Bosch BMP581 | Activo, bajo ruido, en LCSC; DPS310/BMP388/LPS22/SPL06 descartados (stock u obsolescencia) |
| 6–11 oct | Protección SD: TPD6F002 + ESD5Z3.3 | ESD + EMI + R serie en un solo WSON, −40…125 °C, en LCSC |
| 6–11 oct | Buzzer piezo SMD CPT-12123 en placa | 5 mA, menos ruido en 5V_SYS y menos vibración hacia la IMU |
| 6–11 oct | Reglas de compra: LCSC + DigiKey/Mouser, −40…85 °C, sin BGA, solo partes activas | Ensamble JLCPCB con segunda fuente y grado industrial |
