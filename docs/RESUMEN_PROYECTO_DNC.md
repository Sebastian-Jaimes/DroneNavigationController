# DNC (Drone Navigation Controller): resumen de diseño, decisiones y auditoría

> **Propósito de este documento:** servir de contexto inicial para un chat o proyecto nuevo. Resume el estado del diseño electrónico del DNC tras la sesión de revisión del 6 al 11 de octubre de 2026: qué se hizo, qué se decidió y por qué, y qué queda pendiente.
> **Fuente de verdad:** los archivos KiCad del repositorio. Este documento puede quedar desactualizado; ante una diferencia, manda el esquemático.

---

## 1. Contexto del proyecto

| Ítem | Valor |
|---|---|
| Repositorio | `DroneNavigationController` (rama de trabajo `dev`; rama principal `main`) |
| Autor | Sebastian Jaimes |
| Producto | Controlador de vuelo para dron, **una sola placa** que integra todo (no es un stack) |
| Herramienta EDA | **KiCad 10.0.5** (`C:\Program Files\KiCad\10.0\bin\kicad-cli.exe`) |
| Proyecto KiCad | `electronics/DNC/DNC.kicad_pro` + hojas en `electronics/DNC/modules/` |
| Diagrama de arquitectura | `docs/Arquitectura.drawio` (respaldo del original: `docs/Arquitectura.backup.drawio`) |
| Idioma de trabajo | Español |

### Requisitos y preferencias del usuario (aplican a toda selección de componentes)

1. **Compra en LCSC** (ensamblaje en JLCPCB), pero cada parte crítica debe estar también en **distribuidores oficiales (DigiKey o Mouser)** como segunda fuente.
2. **Grado industrial:** temperatura de operación **−40 a +85 °C** como mínimo.
3. **Protección industrial:** ESD **IEC 61000-4-2 nivel 4** (±8 kV contacto / ±15 kV aire) en todo lo que sale de la placa.
4. **Sin BGA** (encarece la fabricación). Se aceptan QFN, WSON, LGA y SON con pads perimetrales.
5. **Solo partes activas** (no NRND, EOL ni "Last Time Buy").
6. **Considerar siempre el consumo de energía.**
7. **Los pasivos (R, C) se seleccionan al final del esquemático;** por ahora sus campos y footprints pueden quedar vacíos.
8. El usuario dibuja en KiCad; el asistente **revisa con netlist y ERC** después de cada cambio. Hay que **guardar (Ctrl+S)** antes de pedir una revisión.

---

## 2. Arquitectura del sistema (decisión vigente)

Plasmada en `docs/Arquitectura.drawio` (título "Arquitectura DNC v1").

```
Batería (XT60) → Protección (TVS) + Medición (shunt + INA169) → VIN
VIN → Buck TPS54331 → 5V_SYS   (USB VBUS → diodo OR B5819W → 5V_SYS)
5V_SYS → AP2112K-3.3 → 3V3_SYS (F405, G0B1, MicroSD, LEDs)
5V_SYS → LDO        → 3V3_SEN (BMI270, BMP581; riel limpio)

STM32F405 (control de vuelo)
 ├─ SPI1 + INT1 ──► IMU BMI270
 ├─ I2C1 + INT ───► Barómetro BMP581 (bus propio)
 ├─ SDIO 4 bits ──► MicroSD (blackbox), con TPD6F002 (ESD + EMI)
 ├─ TIM5 ×4 ──────► ESC (DShot/PWM)
 ├─ USB FS ───────► USB-C (USBLC6-2)
 ├─ ADC ──────────► VBAT_SENSE, CURR_SENSE
 ├─ PA8 TIM1_CH1 ─► Buzzer piezo (PWM 4 kHz)
 ├─ PB4/PB5 ──────► LEDs de estado
 └─ SPI2 + DMA ◄──► STM32G0B1 (planificado)

STM32G0B1 (comunicaciones)
 ├─ UART ─► Receptor ELRS/CRSF (telemetría)
 ├─ UART ─► GPS
 ├─ UART/I2C ─► LiDAR
 └─ I2C2/UART ─► Conector de expansión (OLED de depuración opcional)
```

### Decisiones de arquitectura confirmadas por el usuario

| Decisión | Elección | Motivo |
|---|---|---|
| Receptor RC | **ELRS/CRSF por UART** | Estándar, con telemetría. **Se eliminan nRF24L01, ATmega328P y LM358** |
| STM32G0B1 | **Se mantiene** como MCU de comunicaciones | Elección del usuario (se le recomendó eliminarlo para la v1; lo decidió mantener) |
| MicroSD | **SDIO 4 bits** | Más rápido para el blackbox |
| OLED | **Eliminado de la placa** | No aporta en vuelo; se reemplaza por un conector de expansión |
| Buzzer | **Piezo SMD en placa** (en lugar del estándar FPV externo) | Integración en una sola placa; bajo consumo |
| Barómetro | Bus I²C **propio**, no compartido con el OLED | Evitar latencia y bloqueos del bus |

---

## 3. Asignación de pines del STM32F405RGT6 (vigente, verificada en el netlist)

> Validar en **STM32CubeMX** antes del ruteo (timers, DMA, funciones alternativas).

| Función | Pin | Señal / red | Estado |
|---|---|---|---|
| Motores (TIM5_CH1–CH4, propuesto) | PA0, PA1, PA2, PA3 | MTR1–MTR4 | ✅ Conectado (se movieron de PC6–PC9 por el conflicto con SDIO) |
| SPI1: IMU | PA4 CS, PA5 SCK, PA6 MISO, PA7 MOSI | CSB, SCK, MISO, MOSI | ✅ |
| INT de la IMU | PC4 (EXTI4) | INT1 | ✅ |
| INT del barómetro | PC5 (EXTI5) | INT | ✅ |
| I2C1: barómetro | PB8 SCL, PB9 SDA | SCL, SDA | ✅ |
| ADC | PB0, PB1 | VBAT_SENSE, CURR_SENSE | ✅ |
| SDIO | PC8 D0, PC9 D1, PC10 D2, PC11 D3, PC12 CLK, PD2 CMD | DAT0–DAT3, CLK, CMD | ✅ |
| Buzzer | PA8 (TIM1_CH1) | BUZZER | ✅ |
| LEDs | PB4, PB5 | LED_ERROR, LED_STATUS | ⚠️ Etiquetados en la hoja del F405, **falta el pin en el bloque de la raíz** |
| UART1 (debug) | PA9 TX, PA10 RX | conector DEBUG | ✅ |
| USB FS | PA11, PA12 | USB_STM32_DN/DP | ✅ |
| SWD | PA13, PA14 | conector PROG | ✅ |
| Boot | BOOT0, PB2 (BOOT1) | pull-down de 10 k | ✅ |
| **Libres** | PC0–PC3, PC6/PC7 (USART6), PC13, PA15, PB3, PB6/PB7, PB10/PB11 (USART3), **PB12–PB15 (SPI2, reservado para el enlace con el G0B1)** | — | — |

Regla: las interrupciones (INT1, INT) deben usar **líneas EXTI distintas**; por ejemplo PC4 y PB4 comparten EXTI4.

---

## 4. Estado por hoja del esquemático

Hojas jerárquicas: `ESC`, `USB_POWER_IN`, `STM32G0B1`, `MicroSD`, `Sensores`, `Receptor`, `STM32F405`, `PMS` (`POWER_MANAGEMENT_SYSTEM.kicad_sch`), `Indicadores`.

### 4.1 PMS: gestión de energía

| Bloque | Implementación | Notas |
|---|---|---|
| Entrada | XT60 → TVS (en el último netlist, **SMBJ26CA**; antes SMAJ33A) | |
| Medición de corriente | Shunt de 0.5 mΩ + **INA169**, RL = 47 k (≈ 23.5 mV/A, ~140 A de fondo de escala) | En el último netlist el shunt aparece en la red `VIN_MTR`, junto con un diodo SSB44 |
| Medición de batería | Divisor 10 k / 1 k → VBAT_SENSE (relación 11:1) | |
| 5V_SYS | **Buck TPS54331**: VSENSE 10 k / 1.91 k (≈ 5.0 V), UVLO por EN (332 k / 68.1 k), diodo B340A, L 6.8 µH, 2 × 33 µF | **VIN máximo de 28 V → batería hasta 4S.** Para 6S usar un buck de 60 V (TPS54360, LMR16030) |
| 3V3_SYS | **AP2112K-3.3 (600 mA)**; reemplazó al LP5907 de 250 mA, que no alcanzaba con la SD | ✅ Resuelto |
| 3V3_SEN | LDO dedicado a los sensores | Verificar qué LDO quedó (LP5907 o AP2112K) |
| LEDs de encendido | Verde (en 3V3_SYS, antes en 3V3_SEN) y rojo (5V_SYS) | Verificar que el verde quedó en 3V3_SYS |

**Riesgos abiertos:**
- **Huecos de montaje "M12V" conectados a la red de potencia** (VIN / VIN_MTR). Con un frame de fibra de carbono (conductora) hay riesgo de corto. **No resuelto.**
- Retroalimentación: con USB conectado y sin batería, 5V_SYS puede volver a VIN a través del diodo de cuerpo del FET alto del buck. Evaluar.
- La salida `CURR_SENSE` ya llega al ADC (PB1).

### 4.2 USB_POWER_IN

USB-C con CC1/CC2 a 5.1 k, **USBLC6-2SC6** en D+/D−, diodo OR **B5819W** de VBUS a 5V_SYS. ✅ Hay etiquetas duplicadas en D+/D− (`USB_CONN_DP` frente a `D+`), solo un aviso del ERC.

### 4.3 Sensores ✅ (completa y conectada al F405)

| Componente | Detalle |
|---|---|
| **IMU BMI270** | Símbolo `Sensor_Motion:BMI160` (misma distribución de pines), huella `Bosch_LGA-14_3x2.5mm`. SPI; CSB con pull-up de 10 k (mantiene el modo SPI); 100 nF en VDD y VDDIO; riel 3V3_SEN |
| Pines sin usar del BMI270 | ASDx, ASCx, INT2, OCSB, OSDO con marcador de "no conectar". **Pendiente:** confirmar en la hoja de datos de Bosch si ASDx/ASCx deben ir a GND o VDDIO |
| **Barómetro BMP581** (LCSC **C5362283**) | Símbolo de la librería externa `MSS_Sensor`. I²C con pull-ups de 4.7 k a 3V3_SEN; CSB a VDDIO (modo I²C); 100 nF en VDD y VDDIO; INT a PC5 |
| Dirección del barómetro | Jumper **JP `BARO_ADDR`** (`SolderJumper_3_Bridged12`): por defecto SDO a VDDIO → **0x47** (0x46 con SDO a GND; confirmar en la hoja de datos) |
| **Pendientes** | (a) La huella del BMP581 es `ST_HLGA-10_2x2mm_P0.5mm_LayoutBorder3x2y` (la del LPS22): **verificar contra el patrón de soldadura de Bosch** o importar con `easyeda2kicad` desde C5362283. (b) Los pines VSS 8 y 9 están ocultos, lo que genera el aviso "GND y VSS en la misma red"; conviene hacerlos visibles. (c) Configurar las salidas INT de ambos sensores como **push-pull** en el firmware (no tienen pull-up) |

### 4.4 MicroSD ✅ (completa salvo la huella del filtro)

| Componente | Detalle |
|---|---|
| Socket | Molex **472192001** (LCSC **C164170**), huella `microSD_HC_Molex_47219-2001` |
| Filtro ESD/EMI | **TI TPD6F002QDSVRQ1** (LCSC **C2860634**): filtro pi de 100 Ω y 17 pF + 17 pF, ±20 kV contacto, −40 a +125 °C, AEC-Q101, ancho de banda de 100 MHz |
| Canales | CH1 DAT2, CH2 DAT3, CH3 CMD, CH4 CLK, CH5 DAT0, CH6 DAT1 (en el orden de los pines del socket). `In` al F405, `Out` al socket. **Según la hoja de datos, los dos lados están protegidos**, así que la orientación es indiferente. Pad central (13) a GND ✅ |
| Pull-ups | 47 k a 3V3_SYS en **CMD y DAT0–DAT3**; CLK **sin** pull-up |
| Resistencia en serie | La de 22 Ω de CLK se **eliminó**, porque el TPD6F002 ya trae 100 Ω |
| Alimentación del socket | 3V3_SYS + 10 µF + 100 nF + TVS **onsemi ESD5Z3.3T1G** (LCSC **C61148**, SOD-523, unidireccional, cátodo a 3V3_SYS). Respaldo de la misma huella: MSKSEMI ESD5Z3.3T1G-MS (C3008119) |
| **Pendiente** | El símbolo y la huella del TPD6F002 son los del **TPD6F003** (`Texas_R-PWSON-N12_EP0.4x2mm`, paso de 0.4 mm). El TPD6F002 viene en **DSV de 3.0 × 1.35 mm**. **Verificar la huella con el plano DSV0012A** o importarla desde LCSC. La distribución de pines del símbolo sí coincide |

### 4.5 Indicadores (buzzer ✅, LEDs ⚠️)

**Buzzer** (completo):

```
5V_SYS ─┬─ BZ1 + (CPT-12123-81-SMT-TR) ─┐
        ├─ R 1 k (descarga del piezo) ───┤
        └─ D 1N4148W (cátodo a 5V_SYS) ──┤
                                         └─ Drain de Q FDV303N
PA8 (BUZZER) ─ R 330 Ω ─┬─ Gate          Source → GND
                        └─ R 47 k → GND (pull-down)
```

| Componente | Parte | Notas |
|---|---|---|
| BZ1 | Same Sky **CPT-12123-81-SMT-TR** (LCSC **C3812179**) | Piezo pasivo; 5 Vp-p nominal (máx. 25 Vp-p); **5 mA máx.**; **81 dB** a 10 cm; **4 kHz**; 16 nF; **−40 a +120 °C**; 12 × 12 × 3 mm; **no lavable**; soldar dentro de 48 h tras abrir el empaque al vacío |
| Q | onsemi **FDV303N** (LCSC **C80498**), símbolo `Transistor_FET:BSS138` (1 = G, 2 = S, 3 = D), SOT-23 | Corregido: el símbolo `Device:Q_NMOS` tenía pines D/G/S sin número |
| D | 1N4148W-E3-08 (Vishay) | Recorta los picos que genera el piezo ante golpes |
| Firmware | PWM en TIM1_CH1 a **4 kHz, ciclo del 50 %**; tonos de 2 a 5 kHz; para silenciarlo, PA8 en bajo | |

**Pendientes del buzzer:**
- **Huella de BZ1 vacía:** usar la huella oficial de Same Sky y verificar los pads 1 (+) y 2 (−).
- **Disponibilidad:** en una captura LCSC lo mostraba agotado y DigiKey con ~1 860 unidades; un distribuidor (Electro Sonic) mencionaba un aviso de fin de vida. **Confirmar stock y estado.**

**LEDs de estado:**
- LED_STATUS (verde): 3V3_SYS → LED → 680 Ω → PB5.
- LED_ERROR (rojo): 3V3_SYS → LED → 680 Ω → PB4.
- Se encienden con el pin en bajo.

**Pendientes de los LEDs:**
- **Importar los pines `LED_ERROR` y `LED_STATUS` en el bloque STM32F405 de la raíz y cablearlos.** Hoy son redes de un solo nodo.
- Elegir los LEDs. Con 680 Ω, el verde debe ser **verde amarillento (Vf ≈ 2.0–2.2 V)**; un verde puro (InGaN, ~3 V) necesitaría unos 150–220 Ω.

**Opcional recomendado:** footprint de un conector **JST-GH de 2 pines, marcado DNP,** en paralelo con BZ1 para un buzzer externo. Si se monta: TVS de 5 V en BZ− y PTC (~200 mA) en BZ+.

### 4.6 STM32F405

Alimentación, VCAP de 2.2 µF ×2, VDDA con ferrita + 1 µF + 100 nF, cristales con resistencias de 0 Ω, NRST con capacitor y pulsador, conectores PROG (SWD) y DEBUG (UART1). Todos los periféricos de la sección 3 están conectados.

**Avisos del ERC:**
- **VDDA sin fuente:** la ferrita no cuenta como fuente; poner un `PWR_FLAG` en la red `VDDA_F4`.
- **LED_ERROR / LED_STATUS:** etiquetas jerárquicas sin pin en la raíz (ver 4.5).

### 4.7 ESC

Resistencias de 220 Ω en serie en MTR1–4, conector a los ESC, alimentación `VIN_MTR`, huecos de montaje (ver el riesgo en 4.1).

### 4.8 STM32G0B1 y Receptor: ⚠️ obsoletas respecto de la arquitectura

- **STM32G0B1:** el MCU está **completamente desconectado** (ni siquiera VDD/VSS). La hoja todavía contiene el **ATmega328P, los LM358, el ICSP y LEDs** del diseño anterior. Hay que limpiarla y diseñar el G0B1 según la arquitectura: enlace SPI2 con el F405, UART a ELRS, GPS, LiDAR y expansión, y su propio SWD.
- **Receptor:** todavía tiene el **nRF24L01P** con red de RF. Se reemplaza por un **conector UART para ELRS/CRSF** (con ESD).

---

## 5. Selección de componentes: historial y descartes

### Barómetro (elegido: **Bosch BMP581**, LCSC C5362283)

| Candidato | Resultado |
|---|---|
| Infineon DPS310 | No está en LCSC |
| Bosch BMP388 (C779278) | Obsoleto en DigiKey |
| ST LPS22DF (C3253714) | Sin stock |
| ST LPS22HB (C94049) | Generación anterior; stock inconsistente |
| ST LPS22HH (C2827824) | Stock inconsistente |
| Goertek SPL06-001 (C2684428) | Obsoleto |
| TDK ICP-20100 | En fin de vida |
| Bosch BMP390 (C5124834) | Aparecía agotado |
| Goertek SPA06-003 (C30589048) | **Alternativa económica** (unos 0.43 USD, mucho stock); menor soporte en firmware |
| **Bosch BMP581** | **Elegido:** activo, bajo ruido, mismo fabricante que la IMU, en LCSC |

Nota: se refutó una recomendación externa que proponía el LPS22HB por "stock garantizado" y afirmaba que bastaba un comando de consola en Betaflight; un driver no se activa por comando. La fotosensibilidad aplica a **todos** los barómetros MEMS: hay que alejarlos de los LEDs y cubrirlos con espuma de celda abierta.

### Protección de la MicroSD (elegido: **TPD6F002QDSVRQ1** + **ESD5Z3.3T1G**)

| Candidato | Resultado |
|---|---|
| 74AHC1G125 | Solo servía para aislar el MISO de la SD en un bus SPI compartido. **Innecesario con SDIO**; eliminado |
| onsemi CM1624 | En fin de vida ("Last Shipments") |
| ST EMIF06-MSD02N16 | Activo e integra todo, pero no está en LCSC (30 semanas en Farnell) |
| ST EMIF06-HSD03F3 (C3152872) | **BGA** y desde −30 °C: descartado |
| Nexperia IP4252 / IP4253 / IP4856 | Descontinuados |
| TI TPD6E05U06 (C962978) | Solo ESD (±12 kV contacto, 0.47 pF); alternativa discreta válida |
| **TI TPD6F002QDSVRQ1** | **Elegido:** ESD + EMI + resistencias en serie, −40 a +125 °C, WSON, en LCSC |
| TVS del VDD: PESD3V3U1UA (C403915), TSD03DYFR, CDSOD323-T03S | Alternativas en SOD-323 |
| **onsemi ESD5Z3.3T1G (C61148)** | **Elegido** por el usuario. DigiKey lo marca "Active" pero con backorder |

### Buzzer (elegido: **Same Sky CPT-12123-81-SMT-TR**)

| Candidato | Resultado |
|---|---|
| Murata PKLCS1212E24A0-R1 | Toda la serie PKLCS1212 se descontinúa (09/2027) |
| Same Sky CPT-1203-78-SMT-TR (C3812197) | Sin stock en LCSC |
| PUI SMT-1240-S-HT | No está en LCSC |
| Same Sky CMT-8504-100-SMT-TR (C3811795 / C22359707) | 100 dB, pero **150 mA promedio / ~330 mA pico** y sin stock en LCSC |
| Same Sky CMI-9705-0380/0580-SMT-TR (C5438433) | Alternativa activa magnética de 30–50 mA |
| Same Sky CMI-1295-0585T | Opción externa (through-hole, 5 V activo, 85 dB, −40 a +85 °C), no está en LCSC |
| **Same Sky CPT-12123-81-SMT-TR** | **Elegido:** piezo, 5 mA, 81 dB, −40 a +120 °C |

Criterio clave: el **consumo** favorece al piezo (~25 mW frente a ~0.75 W del magnético potente), que además genera menos ruido en el riel de 5 V y menos vibración hacia la IMU.

---

## 6. Recomendaciones de diseño acordadas (aplicar en el PCB y en el firmware)

1. **IMU:** lejos del buzzer, de los LEDs y de las fuentes conmutadas. Verificar con blackbox el espectro del giroscopio con el buzzer encendido y apagado: el piezo a 4 kHz con el giroscopio muestreando a ~3.2 kHz puede producir aliasing.
2. **BMP581:** lejos del buck, del inductor y de los LEDs; cubrirlo con espuma de celda abierta; no tapar su orificio con conformal coating.
3. **TPD6F002 y ESD5Z3.3:** pegados al socket de la SD; pad central con varias vías a GND; pistas cortas y rectas, sin esquinas agudas.
4. **Buzzer:** Q, D y la resistencia de 1 k pegados a BZ1; sin lavado ni coating sobre él; orificio de sonido despejado.
5. **Batería:** máximo **4S** por el TPS54331.
6. **Conector de expansión** (I2C2/UART + 3V3 + GND) en lugar del OLED fijo.
7. **Todo lo que sale de la placa** lleva TVS: USB (hecho), ESC, receptor, GPS, expansión, buzzer externo.
8. **Hoja de alimentación:** poner `PWR_FLAG` en GND (junto al XT60, en la raíz) y en VDDA_F4.

---

## 7. Auditoría: pendientes priorizados

### Críticos (bloquean el PCB)
- [ ] Hojas **STM32G0B1** y **Receptor**: eliminar ATmega328P, LM358, ICSP y nRF24; diseñar el G0B1 (alimentación, enlace SPI2 con el F405, UARTs, SWD) y el conector ELRS.
- [ ] Huellas verificadas: **TPD6F002** (DSV de 3.0 × 1.35 mm), **BMP581** (patrón de Bosch), **CPT-12123** (Same Sky).
- [ ] Pines `LED_ERROR` / `LED_STATUS` en el bloque F405 de la raíz.
- [ ] Huecos de montaje "M12V" en la red de potencia: decidir si se aíslan.
- [ ] Validar el mapa de pines en STM32CubeMX (TIM5 para motores, DMA de SDIO, SPI1, SPI2 e I2C1).

### Importantes
- [ ] **Anotar** el esquemático completo (todo sigue como `U?`, `R?`, `C?`, y hay referencias duplicadas).
- [ ] `PWR_FLAG` en GND y en VDDA_F4.
- [ ] Elegir los LEDs (Vf y color) y confirmar la resistencia.
- [ ] Confirmar stock y estado del CPT-12123-81-SMT-TR (LCSC y DigiKey/Mouser).
- [ ] BMI270: comprobar la recomendación de Bosch para ASDx/ASCx sin usar.
- [ ] Verificar qué LDO alimenta 3V3_SEN y dónde quedó el LED verde de encendido.
- [ ] Actualizar la nota de pines de `docs/Arquitectura.drawio`: motores en **PA0–PA3** (no PA2/PA3 en TIM2), buzzer en PA8 y LEDs en PB4/PB5.

### Al final del esquemático
- [ ] Seleccionar pasivos (MPN, LCSC y footprint). Sugerencias: resistencias 0402 al 1 %, 100 nF en 0402 X7R, 10 µF en 0603/0805 X5R/X7R de ≥ 10 V.
- [ ] Llenar los campos que faltan (diodo 1N4148W: número de LCSC).

### Higiene del repositorio y del proyecto KiCad
- [ ] **Librerías externas fuera del repo:** `Gen_devices`, `digikey-footprints` y `MSS_Sensor` (`${MS4M_SYMBOL}`). Crear librerías del proyecto (`electronics/DNC/libs/` + `sym-lib-table` / `fp-lib-table`).
- [ ] Agregar `electronics/DNC/.history/` (historial local de KiCad 10, que es un repo git propio) al `.gitignore`.
- [ ] Los README de `docs/`, `firmware/`, `software/`, `tests/` y `electronics/` son plantillas que describen carpetas inexistentes; el README raíz menciona pantalla LCD y ESP32/Arduino.
- [ ] `hardware/3d_models` tiene el frame ZMR250 tres veces y un archivo de bloqueo de SolidWorks versionado (`~$Brazo.SLDPRT`).
- [ ] Considerar **Git LFS** para STEP, SLDPRT, PDF y DOCX (`.git` ≈ 53 MB).
- [ ] El PDF del certificado del curso probablemente no debería estar en el repo.
- [ ] Nada de esta sesión se ha commiteado. Los cambios de KiCad, `docs/Arquitectura.drawio`, `docs/Arquitectura.backup.drawio` y este archivo están sin commit.

---

## 8. Cómo se hicieron las revisiones (método reproducible)

KiCad no puede exportar un netlist útil mientras el esquemático no esté anotado (todas las referencias `U?` se mezclan). El flujo usado:

1. Copiar `DNC.kicad_sch`, `DNC.kicad_pro` y `modules/` a una carpeta temporal.
2. En la **copia**, reemplazar con un script cada `(reference "X?")` por una referencia única (`X901`, `X902`, …). **No se tocan los archivos originales.**
3. Exportar el netlist y correr el ERC sobre la copia:
   ```bash
   K="/c/Program Files/KiCad/10.0/bin/kicad-cli.exe"
   "$K" sch export netlist --format kicadsexpr -o dnc.net copy/DNC.kicad_sch
   "$K" sch erc --severity-all -o erc.rpt copy/DNC.kicad_sch
   "$K" pcb drc -o drc.rpt DNC.kicad_pcb
   ```
4. Leer el netlist con un parser de S-expresiones (Python) para listar pin → red por componente y nodos por red, filtrando por hoja (`sheetpath`).
5. Ignorar como ruido `lib_symbol_mismatch` y `footprint_link_issues`; centrarse en `pin_not_connected`, `hier_label_mismatch`, `label_dangling`, `power_pin_not_driven` y `pin_to_pin`.

---

## 9. Historial de la sesión (cronológico)

1. **Revisión general del repo y de KiCad.** Se detectó: esquemático sin anotar, rieles `+3.3V` / `+5V` sin fuente, F405 casi sin periféricos conectados, G0B1 desconectado, SD que recibiría 5 V desde el ATmega, conflicto SDIO–motores, huecos M12V en la red de potencia, librerías externas, 276 violaciones de ERC y PCB vacío.
2. **Barómetro:** DPS310 → BMP388 → LPS22DF → evaluación de una recomendación externa → BMP581 frente a SPA06-003 → **BMP581**.
3. **Hoja Sensores:** se corrigió la idea de "componentes duplicados" (eran la caché de símbolos de la librería); cableado del BMP581, jumper `BARO_ADDR` (Bridged12), pull-ups y exportación de SPI/I²C/INT al F405.
4. **OLED:** se analizó y se **eliminó**; se reemplaza por un conector de expansión.
5. **Arquitectura:** el usuario decidió ELRS por UART, mantener el G0B1 y SDIO. Se reescribió `Arquitectura.drawio` (con respaldo).
6. **Mapa de pines:** los motores pasaron a PA0–PA3 para liberar el SDIO; ADC en PB0/PB1.
7. **MicroSD:** SDIO, eliminación del 74AHC1G125, pull-ups, búsqueda de protección (CM1624 → … → **TPD6F002**), TVS del VDD (**ESD5Z3.3T1G**) y verificación con la hoja de datos de TI (ambos lados protegidos; la huella no coincide).
8. **Buzzer:** se corrigió el diseño de referencia (R9 de 100 Ω en serie y alimentación de VCC_MCU eran errores); análisis de externo frente a SMD y del estándar FPV; varias rondas de disponibilidad; análisis de consumo; **CPT-12123-81-SMT-TR**; corrección del símbolo del MOSFET.
9. **LEDs de estado:** dibujados; falta la conexión en la raíz.

---

## 10. Siguientes pasos sugeridos (orden)

1. Cerrar `Indicadores` (pines de los LEDs en la raíz).
2. Rediseñar **STM32G0B1** y **Receptor** según la arquitectura.
3. Conector de expansión y ESD en todos los conectores externos.
4. Validar los pines en CubeMX y actualizar `Arquitectura.drawio`.
5. Corregir las huellas pendientes, anotar, poner `PWR_FLAG` y dejar el ERC en cero.
6. Seleccionar los pasivos y completar la BOM con LCSC + DigiKey/Mouser.
7. PCB: ubicación según la sección 6.
