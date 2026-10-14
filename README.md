# DNC: Drone Navigation Controller

Controlador de vuelo en una sola placa con **STM32F405** (control) + **STM32G0B1** (comunicaciones).
Objetivo: correr ArduPilot (misiones autónomas con GPS y barómetro) y Betaflight/INAV (acrobático, loop times bajos).

**Estado:** esquemático en curso (KiCad 10). MVP1 para batería 4S. Seguimiento de tareas en Notion.

| Fase | Contenido | Estado |
|---|---|---|
| 1 | Arquitectura y esquemático base | ✅ |
| 2 | Energía y protecciones | ⏳ |
| 3 | G0B1, receptor y conectores | ⏳ |
| 4 | Verificación del esquemático (CubeMX, huellas, ERC) | ⏳ |
| 5 | Cierre del esquemático (BOM, revisión) | ⏳ |
| 6 | PCB layout (4 capas, JLCPCB) | ⏳ |
| 7 | Firmware (bring-up, drivers, DShot) | ⏳ |

## Documentación
- [Arquitectura](docs/arquitectura.md): bloques, energía, sensores y periféricos
- [Pinout](docs/pinout.md): tabla única de pines
- [Decisiones](docs/decisiones.md): qué se decidió y por qué
- [Interfaces](docs/interfaces/): [DShot](docs/interfaces/dshot.md)

## Estructura
| Carpeta | Contenido |
|---|---|
| `docs/` | Documentación técnica (fuente de verdad) |
| `electronics/` | Proyecto KiCad del DNC |
| `hardware/` | Modelos 3D (frame, motores, componentes) |
| `firmware/` | Firmware (Fase 7) |

## Licencia
MIT, ver [LICENSE](LICENSE). Contacto: sebastian.jaimes.chacon@gmail.com
