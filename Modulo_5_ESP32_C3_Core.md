# Módulo 5: Núcleo ESP32-C3 (ESP32-C3-MINI-1-N4) — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-17
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 5 es el **corazón digital** del Smart Relay: un subsistema basado en el SoC **ESP32-C3-MINI-1-N4** (Espressif) que concentra el procesamiento, la conectividad inalámbrica (Wi-Fi 4 + BLE 5) y la lógica de control de todos los demás módulos del producto.

Sus funciones son:

1. **Hospedar** el microcontrolador ESP32-C3 (núcleo RISC-V monoprocesador a 160 MHz, 400 KB SRAM, 4 MB flash SPI interna) que ejecuta el firmware IDF/Arduino con soporte nativo de Matter/Home Assistant.
2. **Orquestar** las señales de control del módulo de potencia: conmutación del relé (M3), lectura del switch de pared vía optoacoplador (M8), sensado térmico por NTC en ADC (M9) y LED de estado (M9).
3. **Exponer** una interfaz **USB-C nativa** (GPIO18/19 *hardwired* en silicio al controlador USB Serial/JTAG) para flasheo, logs y depuración sin necesidad de puente USB-UART externo (M6).
4. **Garantizar un boot determinístico** mediante los strapping pins requeridos por Espressif: GPIO2 con pull-up externo obligatorio (diferencia crítica vs ESP32-C6), GPIO8 con pull-up interno suficiente, y GPIO9 como selector de modo BOOT/DOWNLOAD.
5. **Proveer** antena Wi-Fi PCB *inverted-F* integrada en el módulo (≈ 2.5 dBi) con zona de keepout de sólo **10 mm** (33 % más compacta que el C6-WROOM-1), habilitando PCBs más pequeñas.
6. **Entregar** certificación RF heredada: el módulo ya está pre-certificado **FCC (2AC7Z-ESPC3MINI1)**, **CE/RED** e **IC**, lo que permite certificación modular del producto final sin repetir pruebas de radiación.

**Entradas del módulo:**
- `+3.3V` — rail regulado provisto por el **Módulo 4 (LDO ME6211C33M5G-N)**.
- `GND_DC` — plano de tierra común DC compartido con M2/M4/M6/M7/M8/M9.

**Salidas del módulo (señales lógicas y USB):**
- **Módulo 3** — GPIO4 hacia driver BJT del relé HF115F.
- **Módulo 6** — USB D−/D+ (GPIO18/19), UART TX/RX fallback (GPIO21/20).
- **Módulo 8** — GPIO5 recibe la señal del fototransistor del optoacoplador PC817.
- **Módulo 9** — GPIO3 lee el divisor NTC (ADC1_CH3); GPIO6 maneja el LED de estado.

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Arquitectura Interna del ESP32-C3-MINI-1-N4

El ESP32-C3-MINI-1-N4 es un **módulo castellated** de 13.2 × 16.6 × 2.4 mm que empaqueta todos los bloques funcionales necesarios para un nodo IoT Wi-Fi + BLE en una sola PCB sellable por reflow:

```
          ESP32-C3-MINI-1-N4 — BLOQUES INTERNOS
          ═══════════════════════════════════════

  ┌────────────────────────────────────────────────────┐
  │    ANTENA PCB inverted-F (en el borde)             │
  │           │                                        │
  │           ▼                                        │
  │    ┌─────────────┐     ┌─────────────────────┐     │
  │    │ Balun +     │◄───►│ RF 2.4 GHz: Wi-Fi 4 │     │
  │    │ Matching    │     │ (b/g/n) + BLE 5.0   │     │
  │    └─────────────┘     └──────────┬──────────┘     │
  │                                    │                │
  │                         ┌──────────▼──────────┐     │
  │                         │ ESP32-C3 SoC        │     │
  │                         │  • RISC-V 32-bit    │     │
  │                         │    single-core      │     │
  │                         │    @ 160 MHz        │     │
  │                         │  • 400 KB SRAM      │     │
  │                         │  • 384 KB ROM       │     │
  │                         │  • USB Serial/JTAG  │     │
  │                         │  • ADC 12-bit 6 ch  │     │
  │                         │  • Timers, SPI, I²C │     │
  │                         └──────────┬──────────┘     │
  │                                    │                │
  │                         ┌──────────▼──────────┐     │
  │                         │ Flash SPI 4 MB      │     │
  │                         │ (interna, GPIO11-17)│     │
  │                         └─────────────────────┘     │
  │                                                     │
  │    XTAL 40 MHz interno (no requiere cristal ext.)   │
  │                                                     │
  │    GND pad inferior + pads castellated laterales    │
  └────────────────────────────────────────────────────┘
```

**Puntos clave de la arquitectura:**

- **RISC-V 32-bit** — ISA abierta (RV32IMC). Diferencia vs el ESP32/ESP32-S que usan Xtensa LX6/LX7. El compilador GCC y el IDF soportan ambos sin cambios en el código de usuario.
- **Cristal 40 MHz interno** — no requiere cristal externo en la PCB portadora. Menor BOM, menor riesgo de layout RF.
- **Flash SPI de 4 MB integrada** — los pines SPI (GPIO11–17) están internamente conectados a la flash y **no son accesibles** en los pads castellated. Esto previene errores de diseño donde el usuario intente usar esos GPIOs.
- **USB Serial/JTAG hardwired** — GPIO18 y GPIO19 están conectados directamente al controlador USB en silicio. No pueden reasignarse ni usarse como GPIO normal cuando USB está en uso.

### 2.2 Por Qué ESP32-C3-MINI-1 en Vez de ESP32-C6-WROOM-1

La PCB original del proyecto fue especificada para ESP32-C6 (Wi-Fi 6 + Thread/Zigbee). Para esta revisión de 1 canal se prefiere el C3 por cinco razones concretas:

| Criterio | ESP32-C6-WROOM-1 | **ESP32-C3-MINI-1** |
|---|---|---|
| Footprint | 25.5 × 18.0 × 3.1 mm | **13.2 × 16.6 × 2.4 mm** |
| Zona de keepout de antena | 15 mm | **10 mm** |
| Wi-Fi | Wi-Fi 6 (ax) + Thread + Zigbee | Wi-Fi 4 (n) + BLE 5.0 |
| Núcleo | RISC-V 160 MHz + LP-core | RISC-V 160 MHz |
| Flash interna | 4 MB | 4 MB |
| Precio LCSC (qty ≥ 10) | ~$3.40 | **~$2.10** |
| Certificación FCC/CE modular | Sí | Sí |
| Strapping pin crítico | GPIO8 (pull-up interno) | **GPIO2 (requiere pull-up externo)** |

**Conclusión:** para un relé residencial de 1 canal que sólo necesita reportar estado y recibir comandos, Wi-Fi 4 es suficiente (hasta 54 Mbps reales, latencia < 50 ms a router LAN). El ahorro del 30 % en BOM y del 45 % en área de PCB justifica usar el C3. Si en el futuro se requiere Thread/Matter nativo, el diseño se puede portar al C6 cambiando el módulo (ambos son castellated, footprints distintos pero migración simple).

### 2.3 Strapping Pins y Secuencia de Boot — Diferencia Crítica C3 vs C6

Los **strapping pins** son GPIOs cuya lectura al momento del reset determina el modo de arranque del chip. Espressif define tres strapping pins en la familia C3:

| Pin | Función de strapping | Estado requerido para boot normal | Mecanismo |
|---|---|---|---|
| **GPIO2** | Selección SPI boot mode | HIGH | **Pull-up externo 10 kΩ obligatorio** — no hay pull-up interno fiable en el C3 |
| **GPIO8** | Controla ROM serial output (log de boot) | HIGH | Pull-up interno suficiente (~45 kΩ). Dejar flotante es aceptable pero se recomienda no usar el pin |
| **GPIO9** | Selección BOOT/DOWNLOAD mode | HIGH = SPI flash, LOW = download | Pull-up interno activo; botón a GND lo lleva a LOW para entrar a download mode |

**⚠ Diferencia clave vs ESP32-C6:** en el C6, el strapping principal es **GPIO8** que sí tiene pull-up interno confiable, y no requiere resistor externo. En el **C3, GPIO2 requiere un resistor de 10 kΩ externo a +3.3V (R_STRAP)**. Omitir este componente es la causa más común de boots intermitentes o fallos de arranque tras un power cycle — el chip puede bootear una vez, y fallar al siguiente reset.

**Secuencia de boot del ESP32-C3 (simplificada):**

1. Power-on Reset (POR) — el chip espera VDD > 3.0 V y EN = HIGH.
2. Lectura de strapping pins en el primer ciclo de reloj post-reset:
   - GPIO2 = HIGH → bootloader ROM arranca SPI flash normal.
   - GPIO2 = LOW → bootloader entra a SDIO/JTAG mode (no usado en este diseño).
3. GPIO9 = HIGH → ejecuta firmware desde flash. GPIO9 = LOW → espera comando por USB/UART (modo download para flasheo).
4. Transferencia de control al bootloader de segunda etapa (en flash) → firmware de usuario.

**Después del boot, GPIO2, GPIO8 y GPIO9 quedan libres** para ser usados como GPIOs normales, pero en este diseño se dejan dedicados a su función de strapping para evitar interferencia (excepto GPIO9, que se reusa como botón BOOT manual).

### 2.4 Circuito de Reset (Pin EN) — Robustez Anti-EMI

El pin **EN** (Chip Enable) es el reset asíncrono del ESP32-C3. Debe mantenerse en HIGH durante operación normal y en LOW para forzar reset. El circuito de reset del Módulo 5 consta de tres elementos:

```
                +3.3V
                  │
              [R_EN: 10 kΩ]
                  │
                  ├────────────► Pin EN (módulo)
                  │
             [C_EN: 100 nF]
                  │
                  ├──── GND
                  │
          [SW2: Tactile 6×6 mm]
                  │
                  └──── GND
```

**Función de cada componente:**

- **R_EN (10 kΩ)** — pull-up débil que mantiene EN en HIGH cuando SW2 no está presionado. Valor estándar recomendado por Espressif para evitar consumo excesivo (Iq = 330 µA durante reset).
- **C_EN (100 nF)** — filtro de paso bajo con constante de tiempo τ = R_EN × C_EN = 10 kΩ × 100 nF = **1 ms**. Suprime:
  - Glitches EMI inducidos por la conmutación del relé K1 (di/dt de 50 A/ms durante el cierre de contactos) que pueden acoplar a la traza EN por capacitancia parásita.
  - Picos de transmisión WiFi (350 mA pulsos de 1 ms) que caen momentáneamente el rail +3.3V.
  - Rebote mecánico del botón SW2 al presionar (típico 100–500 µs).
- **SW2 (tactile 6×6 mm)** — pulsador manual a GND para reset del usuario.

**⚠ Sin C_EN, el ESP32 puede auto-resetearse aleatoriamente** durante la conmutación del relé en productos cercanos a cargas inductivas (motores, balastros de iluminación). Este es un fallo conocido que sólo aparece en ambiente real, no en bench.

### 2.5 Desacoplo de Alimentación — Esquema Dual C_vdd1 + C_vdd2

El ESP32-C3 demanda corrientes de hasta **350 mA en pulsos de 1–2 ms** durante la transmisión Wi-Fi. El esquema de decoupling debe proveer corriente local en dos bandas de frecuencia:

```
+3.3V ──┬──[C_vdd1: 10 µF X5R 0805]──┬──[C_vdd2: 100 nF X7R 0402]──┬── Pin 3V3 módulo
         │                              │                              │
        GND                             GND                           GND
```

| Capacitor | Valor | Banda que cubre | Función |
|---|---|---|---|
| **C_vdd1** | 10 µF / 25 V X5R 0805 | DC – 500 kHz | Reservorio de carga local — absorbe pulsos TX WiFi de 350 mA sin colapsar el rail |
| **C_vdd2** | 100 nF / 16 V X7R 0402 | 1 MHz – 100 MHz | Filtro de alta frecuencia — suprime el switching interno del SoC (~40 MHz) y spikes del regulador buck interno del C3 |

**Reglas de colocación:**
- **C_vdd2 debe estar a ≤ 1.5 mm** del pin 3V3 del módulo (su ESL domina > 10 MHz; cualquier traza larga degrada su efectividad).
- **C_vdd1 a ≤ 3 mm** del pin 3V3.
- **Orden desde el pin:** C_vdd2 primero (HF), luego C_vdd1 (bulk). Esto asegura que los transitorios rápidos sean interceptados por el capacitor de baja ESL antes de viajar a la trayectoria del bulk.
- Ambos comparten un par de vías a GND que bajan directamente al plano GND_DC sin thermal relief (conducción máxima).

### 2.6 Antena PCB Integrada y Zona de Keepout

El módulo ESP32-C3-MINI-1 incorpora una antena PCB **inverted-F** en uno de sus bordes. Esta antena fue caracterizada y certificada por Espressif asumiendo que el integrador respeta la **zona de keepout**:

```
           BORDE DEL PCB PORTADOR (recomendado)
           ════════════════════════════════════

    ┌──── 10 mm ────►
    │                │
    │  ╔════════════════════╗
    │  ║ Antena PCB módulo  ║ ◄── en el borde corto del módulo
    │  ╚════════════════════╝
    │  │                    │
    │  │  ESP32-C3-MINI-1   │
    │  │  13.2 × 16.6 mm    │
    │  │                    │
    │  └────────────────────┘
    │
    ZONA DE KEEPOUT — 10 mm × ancho del módulo
    • Sin cobre en ninguna capa (top, inner1, inner2, bottom)
    • Sin componentes dentro de 5 mm del extremo
    • Sin vías, fills, planos GND
    • Preferible: que coincida con el borde exterior del PCB
```

**Reglas Espressif (Hardware Design Guidelines ESP32-C3-MINI-1 v1.2, sección 5):**

1. **10 mm de zona libre** más allá del borde de la antena, sin cobre en **ninguna capa** (la regla aplica a las 4 capas en PCBs de 4 capas, y a las 2 capas en PCBs de 2 capas).
2. **Sin componentes** metálicos (capacitores cerámicos, inductores, LEDs) dentro de 5 mm del extremo de la antena — estos absorben energía RF y desintonian la antena.
3. **Orientación preferente:** la antena debe apuntar hacia la **pared exterior del gabinete plástico**, alejada de masas metálicas (relé K1, bornes de tornillo).
4. **Distancia mínima a estructuras metálicas** (relé, inductor del snubber, carcasa metálica si existiera): 15 mm.

**Ventaja del C3-MINI-1 vs C6-WROOM-1:** el keepout del C3 es de 10 mm; el del C6 es de 15 mm. En una PCB de 50 × 40 mm, 5 mm de diferencia en el borde libre representan un 10 % más de área utilizable para lógica de potencia.

### 2.7 Disipación Térmica del Pad GND Inferior

El pad GND inferior del módulo (≈ 6 × 8 mm) cumple doble función:

1. **Retorno de corriente** de alta frecuencia RF hacia el plano GND (loops cortos = menos EMI radiada).
2. **Disipación térmica** — durante transmisión WiFi continua, el SoC puede disipar hasta **0.8 W** que debe evacuarse por el pad inferior.

**Regla de layout:**
- **≥ 4 vías térmicas de Ø 0.3 mm** bajo el pad GND, conectadas al plano GND_DC en la capa inferior.
- Thermal relief **deshabilitado** en estas vías (queremos máxima conductividad térmica, aunque dificulte soldadura por reflow — compensar con perfil de soldadura más largo).
- Plano GND inferior ≥ 100 mm² continuo bajo el módulo.

Con estas vías, la resistencia térmica efectiva θ_JA del módulo baja de ~45 °C/W (sin vías) a ~25 °C/W. Para una disipación continua de 0.5 W: ΔT = 12.5 °C — perfectamente aceptable en ambiente de 35 °C (temperatura de caja de interruptor residencial en Bogotá a 2640 m de altitud).

---

## 3. Asignación Completa de GPIOs

| GPIO | Función asignada | Dirección | Módulo destino | Strapping | Notas |
|---|---|---|---|---|---|
| **GPIO4** | Control relé K1 | Output | M3 | No | → R1 (1 kΩ) → base SS8050. Salida digital limpia |
| **GPIO5** | Entrada switch AC | Input (pull-up) | M8 | No | ← Colector del PC817 con R5 (10 kΩ) pull-up a +3.3V |
| **GPIO3** | Sensor NTC temperatura | ADC Input | M9 | No | ADC1_CH3. Lectura analógica del divisor NTC 10k |
| **GPIO6** | LED de estado | Output | M9 | No | Active-low vía R_LED (470 Ω). MTCK/JTAG pero seguro en uso normal |
| **GPIO9** | Botón BOOT | Input (pull-up interno) | M5 (SW1) | **Sí** | Pull-up interno suficiente. SW1 a GND → download mode |
| **GPIO18** | USB D− | USB nativo | M6 | — | **Hardwired en silicio** al controlador USB Serial/JTAG. No reasignable |
| **GPIO19** | USB D+ | USB nativo | M6 | — | **Hardwired en silicio**. No reasignable |
| **GPIO21** | UART TX fallback | Output | M6 | No | UART0 TXD por defecto. Header J_UART 1.27 mm para flasheo secundario |
| **GPIO20** | UART RX fallback | Input | M6 | No | UART0 RXD por defecto. Header J_UART |
| **EN** | Reset (Chip Enable) | Input | M5 (SW2) | — | R_EN 10 kΩ pull-up + C_EN 100 nF + SW2 |
| **GPIO2** | *Strapping (pull-up externo)* | — (boot) | M5 (R_STRAP) | **Sí ⚠** | **R_STRAP 10 kΩ externo obligatorio** — diferencia clave vs C6 |
| **GPIO8** | *Strapping (pull-up interno)* | — (boot) | M5 | Sí | Controla ROM serial output. Pull-up interno suficiente |
| GPIO0, 1, 7, 10 | **Reservados / libres** | — | — | — | Disponibles para expansión (I²C, SPI, sensores adicionales) |

**Justificación de la selección:**
- **GPIO4 para relé** — no es strapping pin, no es ADC, salida push-pull limpia.
- **GPIO5 para switch** — soporta pull-up interno, no es strapping, no interfiere con boot.
- **GPIO3 para NTC** — ADC1_CH3 con mejor SNR que los ADC2 (que comparten recursos con el radio Wi-Fi y se ven degradados durante TX).
- **GPIO6 para LED** — aunque es MTCK (JTAG), no afecta el boot normal; en este diseño no se usa JTAG externo (se usa USB Serial/JTAG integrado).
- **GPIO9 para BOOT** — designación Espressif idéntica en C3 y C6.
- **GPIO2 para strapping** — reservado exclusivamente para pull-up; no se usa para ninguna otra función, aunque el chip lo permita, para evitar interferencia con el valor de strapping al reset.

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama General del Módulo

```
      MÓDULO 5 — ESP32-C3-MINI-1-N4 (U1)
      ═════════════════════════════════════

        +3.3V (desde M4)                       GND_DC (plano común)
            │                                       │
            ├──[C_vdd1: 10 µF X5R 0805]────────────┤
            │                                       │
            ├──[C_vdd2: 100 nF X7R 0402]───────────┤
            │                                       │
            ▼                                       ▼
      ┌───────────────────────────────────────────────────┐
      │                                                   │
      │   Pin 3V3  ◄── +3.3V                              │
      │   Pin GND  ──► GND_DC (+ pad inferior, 4 vías)    │
      │                                                   │
      │   ┌─────────────────────────────────────────┐     │
      │   │ Circuito de reset:                      │     │
      │   │                                         │     │
      │   │  +3.3V ──[R_EN 10 kΩ]──┐                │     │
      │   │                         ├──► Pin EN     │     │
      │   │  GND ──[C_EN 100 nF]──┤                 │     │
      │   │                         │                │     │
      │   │  GND ──[SW2 tactile]───┘                │     │
      │   └─────────────────────────────────────────┘     │
      │                                                   │
      │   ┌─────────────────────────────────────────┐     │
      │   │ Strapping GPIO2 (obligatorio en C3):    │     │
      │   │  +3.3V ──[R_STRAP 10 kΩ]──► GPIO2       │     │
      │   └─────────────────────────────────────────┘     │
      │                                                   │
      │   ┌─────────────────────────────────────────┐     │
      │   │ Botón BOOT (GPIO9):                     │     │
      │   │  GPIO9 ──[SW1 tactile]── GND            │     │
      │   │  (pull-up interno — sin pull-up ext.)   │     │
      │   └─────────────────────────────────────────┘     │
      │                                                   │
      │   GPIO4  ──────► M3 (R1 1 kΩ → base Q1 SS8050)    │
      │   GPIO5  ◄────── M8 (colector PC817 + R5 pull-up) │
      │   GPIO3  ◄────── M9 (divisor NTC + R_NTC_PU)      │
      │   GPIO6  ──────► M9 (R_LED 470 Ω → ánodo LED1)    │
      │   GPIO18 ◄─────► M6 (USB D− + R_DM 22 Ω)          │
      │   GPIO19 ◄─────► M6 (USB D+ + R_DP 22 Ω)          │
      │   GPIO20 ◄────── M6 (UART RX, header J_UART)      │
      │   GPIO21 ──────► M6 (UART TX, header J_UART)      │
      │                                                   │
      │             [ ANTENA PCB ↑ ]                      │
      │        Keepout 10 mm hacia el borde exterior      │
      │                                                   │
      └───────────────────────────────────────────────────┘
```

### 4.2 Tabla Pin a Pin Exhaustiva

La numeración de pines sigue el **datasheet ESP32-C3-MINI-1 v1.2, figura 3** (top view).

| Pin # | Nombre | Dirección | Red (net) | Conexión / Componente | Zona |
|---|---|---|---|---|---|
| 1 | GND | — | `GND_DC` | Pad lateral → plano GND | DC |
| 2 | 3V3 | Power | `+3.3V` | Entrada desde M4, con C_vdd1 + C_vdd2 locales | DC |
| 3 | EN | Input | `EN_MCU` | R_EN 10 kΩ a +3.3V, C_EN 100 nF a GND, SW2 a GND | DC |
| 4 | GPIO4 | Output | `RELAY_CTRL` | → M3 (R1 1 kΩ → base SS8050) | DC |
| 5 | GPIO5 | Input | `SW_IN` | ← M8 (colector PC817, R5 pull-up 10 kΩ, C_deb 100 nF) | DC |
| 6 | GPIO6 | Output | `LED_STATUS` | → M9 (R_LED 470 Ω → ánodo LED1) | DC |
| 7 | GPIO7 | — | `GPIO7_NC` | Flotante / libre para expansión futura | DC |
| 8 | GND | — | `GND_DC` | Pad lateral → plano GND | DC |
| 9 | GPIO8 | Strap | `GPIO8_STRAP` | Flotante (pull-up interno); strapping boot ROM output | DC |
| 10 | GPIO9 | Input | `BOOT_BTN` | SW1 tactile a GND (pull-up interno del chip) | DC |
| 11 | GPIO10 | — | `GPIO10_NC` | Flotante / libre para expansión | DC |
| 12 | VDD_SPI | Power | — | **NO USAR** (interno a flash SPI) | DC |
| 13–19 | GPIO11–17 | — | — | **NO ACCESIBLES** (dedicados a flash SPI interna) | DC |
| 20 | GPIO18 | USB D− | `USB_DN` | → M6 (R_DM 22 Ω → USBLC6 → conector J_USB) | DC |
| 21 | GPIO19 | USB D+ | `USB_DP` | → M6 (R_DP 22 Ω → USBLC6 → conector J_USB) | DC |
| 22 | GPIO20 | Input | `UART_RX` | → M6 (header J_UART pin 3) | DC |
| 23 | GPIO21 | Output | `UART_TX` | → M6 (header J_UART pin 2) | DC |
| 24 | GND | — | `GND_DC` | Pad lateral → plano GND | DC |
| 25 | GPIO0 | — | `GPIO0_NC` | Flotante / libre | DC |
| 26 | GPIO1 | — | `GPIO1_NC` | Flotante / libre | DC |
| 27 | GPIO2 | Strap | `GPIO2_STRAP` | **R_STRAP 10 kΩ a +3.3V (OBLIGATORIO)** | DC |
| 28 | GPIO3 | ADC | `NTC_SENSE` | ← M9 (divisor R_NTC_PU 10 kΩ + NTC_temp 10 k) | DC |
| — | Pad GND inferior | — | `GND_DC` | Pad central → plano GND con ≥ 4 vías térmicas Ø 0.3 mm | DC |

**Nota:** los números de pin exactos pueden variar según revisión del datasheet; usar siempre la **footprint oficial de EasyEDA Pro** generada automáticamente al importar el símbolo con LCSC **C2838502**.

### 4.3 Detalle de los Circuitos de Soporte

```
R_EN — Pull-up EN (10 kΩ 0402)
  Pad 1 ◄──── +3.3V
  Pad 2 ──── Pin EN de U1 + C_EN pad 1 + SW2 terminal

C_EN — Filtro EN (100 nF 0402 X7R)
  Pad 1 ◄──── Pin EN de U1 (mismo nodo que R_EN pad 2)
  Pad 2 ──── GND_DC
  [COLOCAR A ≤ 2 mm DEL PIN EN]

SW2 — Botón reset (tactile 6×6 mm THT, 4-pin SPST)
  Terminales superiores ◄──── Pin EN de U1
  Terminales inferiores ──── GND_DC
  [NORMALMENTE ABIERTO — cierra al presionar]

R_STRAP — Pull-up GPIO2 (10 kΩ 0402)
  Pad 1 ◄──── +3.3V
  Pad 2 ──── Pin GPIO2 de U1
  [CRÍTICO — sin este resistor, boot intermitente]

SW1 — Botón BOOT (tactile 6×6 mm THT, 4-pin SPST)
  Terminales superiores ◄──── Pin GPIO9 de U1
  Terminales inferiores ──── GND_DC
  [NO AGREGAR CAPACITOR en paralelo a SW1 — puede inducir download mode espurio]

C_vdd1 — Bulk decouple 3V3 (10 µF 0805 X5R)
  Pad 1 ◄──── +3.3V
  Pad 2 ──── GND_DC
  [COLOCAR A ≤ 3 mm DEL PIN 3V3]

C_vdd2 — HF decouple 3V3 (100 nF 0402 X7R)
  Pad 1 ◄──── +3.3V
  Pad 2 ──── GND_DC
  [COLOCAR A ≤ 1.5 mm DEL PIN 3V3 — MÁS CERCA QUE C_vdd1]
  [ORDEN DESDE PIN 3V3: primero C_vdd2 (HF), luego C_vdd1 (bulk)]
```

---

## 5. Reglas de Layout PCB

### 5.1 Reglas Críticas de Colocación

| Regla | Valor | Razón |
|---|---|---|
| Distancia C_vdd2 ↔ Pin 3V3 | ≤ 1.5 mm | ESL del 0402 domina > 10 MHz; cualquier traza larga degrada HF decoupling |
| Distancia C_vdd1 ↔ Pin 3V3 | ≤ 3 mm | Bulk decoupling para picos TX WiFi; loop de corriente corto |
| Orden desde pin 3V3: C_vdd2 → C_vdd1 | Obligatorio | HF primero intercepta transitorios rápidos |
| Distancia C_EN ↔ Pin EN | ≤ 2 mm | Filtro de glitches EMI; loop de 1 ms debe ser estable |
| Distancia R_STRAP ↔ Pin GPIO2 | ≤ 5 mm | Pull-up weak — cualquier capacitancia parásita grande puede inducir glitch al boot |
| Vías térmicas bajo pad GND inferior | ≥ 4 × Ø 0.3 mm | θ_JA de 45 → 25 °C/W; crítico durante TX WiFi continuo |
| Ancho de traza +3.3V (entrada) | ≥ 0.4 mm | 350 mA pico; 10 °C rise en cobre 1 oz |
| Zona de keepout antena | 10 mm sin cobre | Datasheet Espressif — garantiza certificación FCC/CE heredada |
| Distancia antena ↔ relé K1 | ≥ 15 mm | Masa metálica del relé desintoniza la antena inverted-F |
| Plano GND bajo módulo | ≥ 100 mm² continuo | Retorno RF + disipación térmica |
| Traza USB D−/D+ | 90 Ω diferencial, longitud match ±0.5 mm | Cumplimiento USB 2.0 Full-Speed eye diagram |
| Traza USB D−/D+ (longitud total) | ≤ 50 mm desde U1 hasta J_USB | Minimizar reflexiones y pérdida |

### 5.2 Checklist de Layout

- [ ] C_vdd2 colocado a ≤ 1.5 mm del pin 3V3 del U1.
- [ ] C_vdd1 colocado a ≤ 3 mm del pin 3V3 del U1, **después** de C_vdd2 en la ruta.
- [ ] C_EN colocado a ≤ 2 mm del pin EN del U1.
- [ ] **R_STRAP presente y conectado entre GPIO2 y +3.3V** (verificación prioritaria — fallo común).
- [ ] SW1 (BOOT) conectado entre GPIO9 y GND sin capacitor en paralelo.
- [ ] Zona de keepout de antena de 10 mm libre de cobre en **todas las capas**.
- [ ] Pad GND inferior con ≥ 4 vías térmicas Ø 0.3 mm al plano GND_DC.
- [ ] Antena orientada hacia el borde exterior del PCB, alejada ≥ 15 mm del relé K1.
- [ ] Plano GND_DC continuo bajo el módulo (sin cortes por trazas).
- [ ] GPIO18/19 ruteados como par diferencial con impedancia 90 Ω y R_DP/R_DM de 22 Ω en M6.
- [ ] Botones SW1 y SW2 accesibles mecánicamente desde la carcasa.
- [ ] Sin vías atravesando el footprint del U1 (interferirían con disipación térmica y con la antena si están cerca).

---

## 6. BOM Completo — Módulo 5 con Códigos LCSC

Todos los componentes están verificados para importación directa desde LCSC / JLCPCB Standard Parts Library y son compatibles con **EasyEDA Pro Edition**. Todos los códigos LCSC fueron verificados contra [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv).

### 6.1 Componentes Primarios

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 1 | **U1** | ESP32-C3-MINI-1-N4 | SoC WiFi 4 + BLE 5.0, RISC-V 160 MHz, 400 KB SRAM, 4 MB flash, antena PCB, pre-certif. FCC/CE/IC | Módulo castellated 13.2 × 16.6 × 2.4 mm | **C2838502** | Espressif Systems | ESP32-C3-MINI-1-N4 | $2.10 |
| 2 | **C_vdd1** | Capacitor MLCC bulk decouple | 10 µF / 25 V, ±10 %, X5R | 0805 SMD | **C15850** | Samsung Electro-Mechanics | CL21A106KAYNNNE | $0.02 |
| 3 | **C_vdd2** | Capacitor MLCC HF decouple | 100 nF / 16 V, ±10 %, X7R | 0402 SMD | **C14663** | YAGEO | CC0402KRX7R9BB104 | $0.01 |
| 4 | **C_EN** | Capacitor MLCC filtro EN | 100 nF / 16 V, ±10 %, X7R | 0402 SMD | **C14663** | YAGEO | CC0402KRX7R9BB104 | $0.01 |
| 5 | **R_EN** | Resistor pull-up EN | 10 kΩ, ±5 %, 1/16 W | 0402 SMD | **C25744** | UNI-ROYAL (Uniroyal Elec) | 0402WGF1002TCE | $0.005 |
| 6 | **R_STRAP** | Resistor pull-up GPIO2 (strapping) | 10 kΩ, ±5 %, 1/16 W | 0402 SMD | **C25744** | UNI-ROYAL (Uniroyal Elec) | 0402WGF1002TCE | $0.005 |
| 7 | **SW1** | Tactile switch BOOT | 6 × 6 × 4.3 mm, SPST, 4-pin through-hole | THT | **C318884** | XKB Connectivity | TS-1187A-B-A-B | $0.04 |
| 8 | **SW2** | Tactile switch RESET | 6 × 6 × 4.3 mm, SPST, 4-pin through-hole | THT | **C318884** | XKB Connectivity | TS-1187A-B-A-B | $0.04 |

**Total componentes primarios:** 8 referencias (6 part numbers únicos: U1, C_vdd1, C_vdd2/C_EN compartidos, R_EN/R_STRAP compartidos, SW1/SW2 compartidos). **Subtotal BOM: ≈ $2.23 USD** (sin ensamblaje).

### 6.2 Componentes Alternativos / Segundas Fuentes

Todos los componentes primarios tienen código LCSC vigente en JLCPCB Standard Library (abril 2026). Aun así, se listan alternativas verificadas como respaldo ante out-of-stock temporales o sustitución por criterio de costo.

#### 6.2.1 Alternativas para U1 (módulo ESP32)

| Opción | LCSC | MPN | Fabricante | Diferencia vs primario | Drop-in? |
|---|---|---|---|---|---|
| **Primario** | **C2838502** | ESP32-C3-MINI-1-N4 | Espressif | — | — |
| Alt. 1 | **C3013450** | ESP32-C3-WROOM-02-N4 | Espressif | 18 × 20 mm, antena PCB mayor, 8 MB flash opcional | **No** — footprint distinto |
| Alt. 2 | **C2934569** | ESP32-C3-MINI-1U-N4 | Espressif | Variante con conector U.FL para antena externa | **No** — requiere antena externa + conector |

#### 6.2.2 Alternativas para C_vdd1 (10 µF / 25 V / 0805 / X5R)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C15850** | CL21A106KAYNNNE | Samsung | — |
| Alt. 1 | **C96446** | GRM21BR61E106KA73L | Murata | X5R, 25 V, equivalente funcional |
| Alt. 2 | **C45783** | CL21A106KPFNNNE | Samsung | Misma línea, variante de proveedor |

#### 6.2.3 Alternativas para C_vdd2 / C_EN (100 nF / 16 V / 0402 / X7R)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C14663** | CC0402KRX7R9BB104 | YAGEO | — |
| Alt. 1 | **C1525** | 0402B104K160CT | Walsin | X7R, 16 V, mismo fit |
| Alt. 2 | **C49678** | CC0805KRX7R9BB104 | YAGEO | **⚠ 0805** — sólo si la PCB se rediseña para 0805 |

#### 6.2.4 Alternativas para R_EN / R_STRAP (10 kΩ / 0402)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C25744** | 0402WGF1002TCE | UNI-ROYAL | ±5 %, 1/16 W |
| Alt. 1 | **C25804** | RC0402FR-0710KL | YAGEO | ±1 %, 1/16 W (mejor tolerancia) |
| Alt. 2 | **C131676** | ERJ-2GEJ103X | Panasonic | ±5 %, 1/10 W, stock masivo |

#### 6.2.5 Alternativas para SW1 / SW2 (tactile 6×6×4.3 mm, 4-pin THT)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C318884** | TS-1187A-B-A-B | XKB Connectivity | Altura 4.3 mm, DIP 4-pin |
| Alt. 1 | **C221992** | TS-1088-AR02026 | XKB Connectivity | Altura 5.0 mm, mismo footprint 4-pin |
| Alt. 2 | **C720477** | PTS645SM43SMTR92 LFS | C&K | SMT equivalente (requiere rediseño de footprint) |

---

## 7. Checklist de Verificación del Módulo

Antes de enviar el diseño a fabricación JLCPCB:

- [ ] **R_STRAP (10 kΩ) presente** entre GPIO2 y +3.3V — **verificar en el esquemático antes de hacer el routing**. Este es el fallo más común que causa boots intermitentes.
- [ ] **SW1 (BOOT) sin capacitor externo** en paralelo — cualquier capacitor > 10 nF en GPIO9 puede inducir entrada espuria a download mode al power-on.
- [ ] **C_vdd1 y C_vdd2 a ≤ 3 mm** del pin 3V3 del módulo, con C_vdd2 más cerca que C_vdd1.
- [ ] **C_EN (100 nF) presente** entre EN y GND, a ≤ 2 mm del pin EN.
- [ ] **R_EN (10 kΩ) presente** entre EN y +3.3V (sin este, EN queda flotante → reset aleatorio).
- [ ] **Zona de keepout de antena de 10 mm libre en todas las capas** (top, bottom, y cualquier capa interna si aplica).
- [ ] **Pad GND inferior con ≥ 4 vías térmicas Ø 0.3 mm** conectadas al plano GND_DC.
- [ ] **Antena orientada hacia el borde exterior del PCB**, alejada ≥ 15 mm del relé K1 y ≥ 20 mm de la zona AC (M1/M2).
- [ ] **GPIO18/19 (USB)** con resistores R_DP/R_DM de 22 Ω en M6 (ubicados fuera de M5 pero parte de la cadena USB).
- [ ] **Botones SW1/SW2 accesibles mecánicamente** desde la carcasa plástica (verificar en el modelo 3D del gabinete).
- [ ] **GPIO11–17 NO CONECTADOS** en el esquemático (son la flash SPI interna — Espressif requiere que no se usen).
- [ ] **Plano GND continuo** bajo todo el módulo y bajo el área de los capacitores de decoupling.
- [ ] **Verificación ERC** en EasyEDA Pro: ningún pin de strapping flotante, ningún pin de power sin conexión, ningún net con un solo nodo.
- [ ] **Verificación DRC** con reglas Espressif: distancia antena a cobre ≥ 10 mm, ancho de trazas de potencia ≥ 0.4 mm.

---

## 8. Conexión con Otros Módulos

Tabla resumen de todas las señales que cruzan la frontera del Módulo 5. Esta tabla es la **interfaz de integración** — cuando se dibuje el esquemático jerárquico en EasyEDA Pro, estos son los *hierarchical sheet pins* del bloque M5.

| Señal / Net | Dirección desde M5 | Destino | Componente en el camino | Zona |
|---|---|---|---|---|
| `+3.3V` | Entrada (VDD) | M4 (VOUT ME6211) | C_vdd1 + C_vdd2 locales | DC |
| `GND_DC` | Plano común | M2, M4, M6, M7, M8, M9 | Vías térmicas + plano inferior | DC |
| `RELAY_CTRL` (GPIO4) | Salida | M3 | R1 (1 kΩ) → base Q1 SS8050 | DC |
| `SW_IN` (GPIO5) | Entrada | M8 | R5 (10 kΩ pull-up), C_deb (100 nF), colector PC817 | DC |
| `NTC_SENSE` (GPIO3) | Entrada ADC | M9 | R_NTC_PU (10 kΩ) + NTC_temp (10 k B=3435) | DC |
| `LED_STATUS` (GPIO6) | Salida | M9 | R_LED (470 Ω) → ánodo LED1 | DC |
| `USB_DN` (GPIO18) | Bidireccional | M6 | R_DM (22 Ω) → USBLC6-2SC6 → J_USB | DC |
| `USB_DP` (GPIO19) | Bidireccional | M6 | R_DP (22 Ω) → USBLC6-2SC6 → J_USB | DC |
| `UART_RX` (GPIO20) | Entrada | M6 | J_UART header 1.27 mm (pin 3) | DC |
| `UART_TX` (GPIO21) | Salida | M6 | J_UART header 1.27 mm (pin 2) | DC |

**Todas las señales son zona DC** — ninguna cruza la barrera de aislamiento galvánica 3000 VAC que separa el primario AC (M1/M2) del secundario. El aislamiento galvánico es manejado por el HLK-5M05 (M2), el optoacoplador PC817 (M8) y el relé HF115F (M3).

---

## 9. Estimación de Consumo del Módulo 5

| Estado | Corriente desde +3.3V | Comentario |
|---|---|---|
| **Deep-sleep** (RTC on, wake on timer) | 5 µA | Módulo casi apagado; sólo RTC y retención de SRAM lite |
| **Light-sleep** (WiFi modem-sleep) | 130 µA | Mantiene asociación WiFi, no descarga datos |
| **Idle conectado a WiFi** (AP associated) | 15–25 mA | Beacon listening cada DTIM (100 ms) |
| **RX WiFi** | 80 mA | Recepción de paquete |
| **TX WiFi continuo** | 240 mA | Transmisión sostenida (raro en IoT de telemetría) |
| **TX WiFi pico** | **350 mA** | Pulso de 1–2 ms durante el envío de un paquete 802.11n |

**Consumo promedio estimado en operación real (smart relay residencial):**
- 95 % del tiempo en light-sleep o idle.
- 5 % del tiempo en ciclos de TX/RX cortos (beacon, keepalive, reporte de estado).
- **Promedio: ~20 mA @ 3.3 V = 66 mW**.

Este consumo es compatible con la capacidad del LDO ME6211 (500 mA) con margen > 20×, y con la fuente HLK-5M05 (1000 mA @ 5V) con margen > 40×.

---

## 10. Referencias

- **ESP32-C3-MINI-1 Datasheet v1.2** — Espressif Systems, 2023.
- **ESP32-C3 Hardware Design Guidelines v1.3** — Espressif Systems, secciones "Power Supply", "Strapping Pins", "Antenna Keepout".
- **ESP32-C3 Technical Reference Manual** — Espressif Systems, capítulos "GPIO Matrix" y "USB Serial/JTAG Controller".
- **IEEE 802.11n-2009** — Wi-Fi 4 baseband reference.
- **USB 2.0 Full-Speed Compliance** — USB-IF specification, sección 7.1.1 (differential impedance 90 Ω ±15 %).
- [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) — documento maestro del proyecto, sección "Módulo 5: ESP32-C3 Core" (líneas 507–603).
- [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) — fuente del rail +3.3V.
- [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) — lista maestra de materiales con códigos LCSC verificados.

---

**Fin del documento — Módulo 5: ESP32-C3 Core v1.0**
