# Diseño PCB: Smart Relay ESP32-C3 de 1 Canal para Instalación Residencial

Relé inteligente de un canal basado en **ESP32-C3-MINI-1-N4** con fuente AC-DC aislada **HLK-5M05**, relé **Hongfa HF115F-I/005-1HS3A** de 16 A, detección de interruptor físico por **optoacoplador PC817C** y flasheo seguro por **USB-C nativo**. Diseñado para instalarse dentro de cajas de interruptor doméstico en redes 110/220 V AC, con integración a Home Assistant vía **ESPHome**.

Este diseño usa **ESP32-C3** por su madurez de ecosistema (Arduino + ESP-IDF en ESPHome, Tasmota estable, comunidad extensa). La arquitectura de módulos es compatible pin-a-pin con **ESP32-C6-WROOM-1** para migración futura cuando Matter/Thread y WiFi 6 maduren en ESPHome (ver [sección de migración a C6](#18-ruta-de-migración-a-esp32-c6)). Aplica las lecciones de ingeniería inversa de la Shelly Plus 1 y la ESP32-C6_Relay_X1 documentadas en este mismo directorio.

---

## Tabla de contenidos

1. [Decisiones de diseño y respuestas técnicas](#1-decisiones-de-diseño-y-respuestas-técnicas)
2. [Arquitectura de módulos](#2-arquitectura-de-módulos)
3. [Módulo 1: Entrada AC y protecciones](#3-módulo-1-entrada-ac-y-protecciones)
4. [Módulo 2: Fuente AC-DC aislada (HLK-5M05)](#4-módulo-2-fuente-ac-dc-aislada-hlk-5m05)
5. [Módulo 3: Relé de potencia + snubber de salida](#5-módulo-3-relé-de-potencia--snubber-de-salida)
6. [Módulo 4: Regulador 3.3 V (ME6211C33)](#6-módulo-4-regulador-33-v-me6211c33)
7. [Módulo 5: ESP32-C3 Core](#7-módulo-5-esp32-c3-core)
8. [Módulo 6: USB-C (flasheo y programación)](#8-módulo-6-usb-c-flasheo-y-programación)
9. [Módulo 7: Selector de fuente AC/USB (OR-Diode)](#9-módulo-7-selector-de-fuente-accusb-or-diode)
10. [Módulo 8: Detección de interruptor físico (PC817C)](#10-módulo-8-detección-de-interruptor-físico-pc817c)
11. [Módulo 9: Sensores, LEDs y botones](#11-módulo-9-sensores-leds-y-botones)
12. [Diagrama de bloques completo](#12-diagrama-de-bloques-completo)
13. [BOM completo (Bill of Materials)](#13-bom-completo-bill-of-materials)
14. [Firmware ESPHome](#14-firmware-esphome-para-home-assistant)
15. [Especificaciones del PCB y layout](#15-especificaciones-del-pcb-y-layout)
16. [Pasos de elaboración](#16-pasos-de-elaboración-de-la-pcb)
17. [Normativas y certificación](#17-normativas-y-certificación)
18. [Ruta de migración a ESP32-C6](#18-ruta-de-migración-a-esp32-c6)

---

## 1. Decisiones de diseño y respuestas técnicas

### 1.1 Modelo de ESP: ESP32-C3-MINI-1-N4 (con ruta de migración a C6)

| Criterio | ESP32 clásico (U4WDH) | **ESP32-C3-MINI-1 (elegido)** | ESP32-C6-WROOM-1 (futuro) |
|---|---|---|---|
| Arquitectura | Xtensa LX6 single/dual | **RISC-V 160 MHz** | RISC-V HP 160 MHz + LP 20 MHz |
| WiFi | 802.11 b/g/n (WiFi 4) | **802.11 b/g/n (WiFi 4)** | 802.11ax (WiFi 6) con TWT |
| BLE | 4.2 | **5.0** | 5.0 con Direction Finding |
| 802.15.4 | No | **No** | Zigbee 3.0 / Thread 1.3 / Matter |
| USB nativo | No (necesita CH340/CP2102) | **Sí (GPIO18/19)** | Sí (GPIO12/13) + JTAG |
| Secure Boot | v1 | **v2** | v2 RSA-3072 + TEE |
| Deep sleep | ~10 µA | **~5 µA** | ~7 µA con RTC + LP-core |
| Pre-certificado FCC/CE | Sí | **Sí (FCC 2AC7Z-ESPC3MINI1)** | Sí (FCC 2AC7Z-C6WROOM1) |
| Precio módulo (qty 100) | ~$2.50 | **~$1.00–1.50** | ~$2.00 |
| SRAM | 520 KB | **400 KB** | 512 KB HP + 16 KB LP |
| Flash | 4 MB (embebida) | **4 MB** | 4 MB (N4) |
| Antena | Depende del módulo | **PCB integrada** | PCB integrada |
| ESPHome framework | Arduino o IDF | **Arduino o IDF (ambos)** | Solo IDF |
| Tasmota | Maduro | **Maduro** | Maduro |
| Comunidad/tutoriales | Extensa (10+ años) | **Extensa (3+ años)** | Reciente (~10 meses en ESPHome) |

**Razón de la elección del C3 sobre el C6**:
1. **Madurez de ecosistema**: ESPHome soporta C3 con framework Arduino **e** IDF. El C6 solo soporta IDF dentro de ESPHome (agregado en junio 2025, ~10 meses de maduración vs. años del C3).
2. **Costo**: $1.00–1.50 vs $2.00 — ahorro de ~$0.50–1.00 por unidad significativo en volumen.
3. **Arduino compatible**: permite usar librerías Arduino estándar en lambdas de ESPHome y componentes custom. Con C6 se requiere API ESP-IDF exclusivamente.
4. **Comunidad**: 10× más tutoriales, ejemplos y respuestas en foros para C3.
5. **Suficiente para el caso de uso**: WiFi 4 + BLE 5.0 + USB nativo cubre 100 % de las necesidades de un relé con Home Assistant.

**Variante MINI-1**: módulo compacto de 13.2 × 16.6 × 2.4 mm con pads castellated en los bordes (soldables a mano) y pad GND inferior. 22 GPIOs disponibles, 4 MB flash (N4). Pre-certificado FCC/CE/IC — herencia modular al producto final.

**Ruta de migración a C6**: la arquitectura de 9 módulos es idéntica. Solo cambia el Módulo 5 (core) y los pines en M3/M6/M8/M9. Ver [sección 18](#18-ruta-de-migración-a-esp32-c6) para la tabla de equivalencia GPIO.

### 1.2 Control de 110 V sin quemar el ESP32

La clave es **aislamiento galvánico total** entre la red AC y la electrónica digital. A diferencia de la Shelly Plus 1, que usa un buck offline no aislado (probablemente LNK304) donde el GND del ESP32 está eléctricamente conectado al retorno de la red AC —haciendo letal conectar un USB mientras la placa está energizada—, este diseño interpone un **módulo HLK-5M05 con 3000 VAC de rigidez dieléctrica** entre la red y el ESP32. El microcontrolador jamás ve voltajes de red. El relé se activa mediante un transistor driver NPN (SS8050) alimentado desde el rail aislado de 5 V DC.

### 1.3 Funcionamiento simultáneo con el interruptor físico

Se implementa un **optoacoplador PC817C** que detecta la presencia de tensión AC en el terminal SW (conectado al interruptor de pared existente). El optoacoplador preserva el aislamiento galv��nico —su LED está en la zona AC, su fototransistor en la zona DC— con 5000 Vrms de separación. El firmware ESPHome en el ESP32-C3 lee el GPIO del fototransistor y conmuta el relé **localmente, sin necesidad de red WiFi ni servidor**. El interruptor físico funciona incluso si el WiFi está caído o Home Assistant no responde.

**Modos de operación del switch** (configurables por firmware, idénticos a Shelly):
- **Toggle**: el relé replica el estado del interruptor (switch cerrado = relé ON)
- **Edge**: cada cambio de estado del switch (abrir o cerrar) alterna el relé — ideal para reemplazar interruptores basculantes existentes sin recablear
- **Momentary**: pulsación breve alterna el relé — para pulsadores momentáneos
- **Detached**: el switch solo reporta estado a HA sin accionar el relé — para escenarios de smart bulbs

### 1.4 Aislamiento correcto AC/DC

Cuatro barreras de aislamiento independientes:

| Componente | Aislamiento | Función |
|---|---|---|
| HLK-5M05 | 3000 VAC | Separa la red AC del rail DC 5 V |
| PC817C | 5000 Vrms | Separa la señal del switch (AC) del GPIO (DC) |
| Relé HF115F-I | 5000 Vrms bobina↔contactos (creepage 10 mm) | Separa el circuito de potencia del driver |
| PCB slot fresado | 2 mm ancho | Barrera física entre trazas AC y DC |

**Creepage y clearance para Bogotá (2640 m)**:
- IEC 62368-1, aislamiento reforzado, pollution degree 2, material group IIIa (FR-4)
- Base (nivel del mar): clearance 4.0 mm, creepage 6.3 mm
- Factor de corrección por altitud (2000–3000 m): ×1.14 para clearance
- **Diseño**: clearance >= 5 mm, creepage >= 6.5 mm con slot fresado

### 1.5 Protección contra sobrecargas y descargas acumuladas

**En la entrada AC:**
- **F1**: fusible slow-blow 1 A 250 V — desconecta fallas sostenidas
- **MOV1**: varistor 14D431K (MCOV 275 VAC) — clampa transientes > 430 V, absorbe surges hasta 4.5 kA (8/20 µs), cumple IEC 61000-4-5 nivel 2 residencial
- **NTC1**: termistor 5 Ω — limita corriente de inrush a ~31 A a 110 V (vs. ilimitado sin él)
- **CX1**: capacitor X2 100 nF 275 VAC — filtra ruido diferencial EMI
- **R_disc**: 1 MΩ 1 W — descarga CX1 en < 1 s cuando se desconecta la alimentación (requerido por IEC 62368-1)
- **TF1**: fusible térmico 115 °C 2 A — respaldo contra incendio si el HLK-5M05 falla catastróficamente

**En la salida del relé:**
- **Snubber RC**: 100 Ω 1 W + 10 nF X2 275 VAC en paralelo con los contactos — absorbe arco al abrir en cargas inductivas
- Se usa 10 nF (no 100 nF) para evitar corriente de fuga de 4.15 mA que causa parpadeo fantasma en LEDs

**En el lado DC:**
- **C_bulk**: 100 µF/10 V electrolítico — absorbe picos de corriente del ESP32 en TX WiFi
- **Protección térmica**: NTC 10 kΩ B=3350 en GPIO3 (ADC) con apagado automático del relé a 80 °C

**En USB:**
- **USBLC6-2SC6**: TVS de protección ESD IEC 61000-4-2 nivel 4 (8 kV contacto, 15 kV aire)
- **PTC1**: fusible resettable 500 mA protege contra cortocircuito USB

### 1.6 Durabilidad y normativas

**Vida útil del relé:**
- Hongfa HF115F-I: 10 millones de ciclos mecánicos, 100 000 ciclos eléctricos a 16 A
- Contactos AgSnO₂ resistentes a soldadura por inrush de lámparas LED (120 A / 20 ms rating explícito)
- A 20 operaciones/día: vida eléctrica > 13 años

**Normativas aplicables:**
- **RETIE Resolución 40117/2024** (Colombia) — obligatorio para comercialización. Evaluación contra NTC 1337 / IEC 60669-2-1
- **IEC 62368-1** — seguridad de equipos electrónicos (reemplaza IEC 60950-1 desde 2020)
- **IEC 61000-4-5** — inmunidad a surges
- **EN 55032 clase B** — emisiones conducidas (el CMC y CX1 ayudan a cumplir)
- **FCC Part 15 B SDoC** — para exportación a EE. UU. (hereda del módulo Espressif pre-certificado)

**Encapsulado:**
- ABS o policarbonato UL 94 V-0
- IP20 mínimo para interior seco, IP54 para ambientes húmedos (baños, lavaderos)
- Recubrimiento conformal de silicona IPC-CC-830B (25–50 µm) sobre PCB ensamblada, excluyendo contactos del relé, tornillos y conector USB-C

### 1.7 Medio de flasheo: USB-C nativo

El ESP32-C3 integra un controlador USB 2.0 Full-Speed con clase **CDC-ACM (Serial/JTAG)** en silicio, en GPIO18 (D−) y GPIO19 (D+). Esto elimina la necesidad de un chip USB-UART externo (CH340, CP2102, FT232). **esptool.py** maneja la comunicación directamente por USB Serial.

**Seguridad de flasheo**: como la fuente HLK-5M05 proporciona aislamiento galvánico de 3000 VAC, conectar un cable USB al PC mientras la placa está energizada por AC **es seguro**. Esto es una ventaja fundamental sobre la Shelly Plus 1, donde hacer lo mismo destruye el adaptador UART y puede electrocutar al operador.

**Header UART fallback**: GPIO21 (TX) y GPIO20 (RX) expuestos en un header de 4 pines (GND, RX, TX, 3V3) de paso 1.27 mm para flasheo manual si el USB falla. Procedimiento manual: GPIO9 a GND + pulsar RESET → entra en download mode → flashear por UART.

### 1.8 Integración con Home Assistant

**ESPHome** con framework **Arduino o ESP-IDF** es la elección. El C3 soporta ambos frameworks dentro de ESPHome, dando máxima flexibilidad.

**Ventajas de ESPHome:**
- **API nativa** con auto-descubrimiento mDNS — no necesita MQTT broker
- Las entidades (relay, switch, temperatura, botón) aparecen automáticamente en HA
- **Operación local-first**: el interruptor físico y la protección térmica funcionan sin WiFi ni HA
- **OTA integrado** para actualizaciones remotas sin tocar hardware
- Configuración completa en ~80 líneas de YAML
- **Framework Arduino disponible**: acceso a librerías Arduino estándar para componentes custom

**Sobre Matter**: el C3 no soporta Matter/Thread (requiere 802.15.4). Si Matter se vuelve un requisito, migrar a ESP32-C6 usando la [tabla de equivalencia GPIO](#18-ruta-de-migración-a-esp32-c6). La arquitectura de módulos no cambia.

---

## 2. Arquitectura de módulos

El diseño se divide en **9 módulos funcionales** separados por la barrera de aislamiento:

```
┌──────────────────────────────────┐   ┌──────────────────────────────────┐
│       ZONA AC (PELIGROSA)        │   │       ZONA DC (SELV, SEGURA)     │
│                                  │   │                                  │
│  M1: Entrada AC y protecciones   │   │  M4: Regulador 3.3 V            │
│  M2: Fuente AC-DC (HLK-5M05) ───┼──>│  M5: ESP32-C6 Core              │
│  M3: Relé + snubber de salida    │   │  M6: USB-C (flasheo)            │
│  M8: Detección SW (PC817 LED) ───┼──>│  M7: Selector AC/USB            │
│                                  │   │  M8: Detección SW (PC817 foto)  │
│                                  │   │  M9: Sensores y LEDs            │
└──────────────────────────────────┘   └──────────────────────────────────┘
            ^^^^^^^^ SLOT FRESADO 2 mm (3000 VAC) ^^^^^^^^
```

Los módulos M2, M3 y M8 **cruzan la barrera de aislamiento**: el HLK-5M05 tiene sus pines AC en la zona caliente y sus pines DC en la zona segura; el PC817 tiene su LED en zona AC y su fototransistor en zona DC; el relé tiene sus contactos en zona AC y su bobina en zona DC.

---

## 3. Módulo 1: Entrada AC y protecciones

### Función

Recibir 110/220 VAC de la red doméstica, proteger contra transientes de red (surges, spikes), limitar la corriente de inrush, filtrar EMI conducido y proporcionar un respaldo contra incendio.

### Esquema de conexión

```
L (terminal) ──[F1]──[TF1]──[NTC1]──┬──[L1 bobinado A]── AC_L_OUT ──> HLK-5M05 pin 1
                                     │
                                  [MOV1]   (L a N)
                                  [CX1]    (L a N)
                                  [R_disc] (L a N)
                                     │
N (terminal) ────────┬───────────────┴──[L1 bobinado B]── AC_N_OUT ──> HLK-5M05 pin 2
```

### Componentes — conexiones pin a pin

| Ref | Componente | Valor | Encapsulado | Pin 1 conecta a | Pin 2 conecta a | Notas |
|---|---|---|---|---|---|---|
| J_AC | Terminal block | KF128-7.5-2P, 300 V/15 A | THT 7.5 mm pitch | Línea (L) de la vivienda | Neutro (N) de la vivienda | Creepage 5.5 mm entre tornillos |
| F1 | Fusible slow-blow | 1 A 250 V, 5×20 mm | Portafusible PCB (BLX-A) | Salida de J_AC pin 1 (L) | Entrada de TF1 | Littelfuse 0215001.MXP. LCSC C142715 |
| TF1 | Fusible térmico | 115 °C, 2 A, 250 V | Axial, montado en contacto con HLK-5M05 | Salida de F1 | Entrada de NTC1 | SETsafe T115. Actúa como última defensa contra incendio |
| NTC1 | Termistor inrush | 5 Ω, 2 A, 9 mm disco | Radial THT | Salida de TF1 | Nodo común L (antes de L1) | RUILON 5D-9. Limita inrush a ~31 A @ 110 V, ~62 A @ 220 V |
| MOV1 | Varistor | 14D431K (MCOV 275 VAC) | 14 mm disco radial | Nodo común L (post-NTC1) | N (neutro) | Bourns. Clamp ~700 V, surge 4.5 kA. **Para 110 V-only**: usar 10D241K (MCOV 150 VAC) |
| CX1 | Capacitor X2 | 100 nF 275 VAC MKP | Film, pitch 15 mm | Nodo común L (post-NTC1) | N (neutro) | Filtro diferencial EMI. Self-healing. Jimson MKP104K275A07 |
| R_disc | Resistencia descarga | 1 MΩ 1 W | Axial THT | Paralelo a CX1 (terminal L) | Paralelo a CX1 (terminal N) | Descarga CX1 en < 1 s (τ = R×C = 1M × 100nF = 100 ms, 5τ = 500 ms) |
| L1 | Choke modo común | 10 mH, 0.5 A | DIP-4 UU9.8 | Pin 1 (dot) = nodo L_in, Pin 2 = AC_L_OUT | Pin 4 (dot) = N_in, Pin 3 = AC_N_OUT | **Opcional en prototipo** — puentear pins 1↔2 y 3↔4. FH UU9.8-10mH |

### Nota sobre el MOV para 110 V vs 220 V

- **Solo 110 V (Colombia doméstico)**: usar **10D241K** — MCOV 150 VAC, V_varistor 240 V @ 1 mA, clamp ~395 V. Protección más agresiva a 110 V.
- **Dual 110/220 V (exportación)**: usar **14D431K** — MCOV 275 VAC, V_varistor 430 V @ 1 mA, clamp ~700 V. Compatible con ambas tensiones.
- El MOV **siempre va después del fusible F1**: si el MOV falla en cortocircuito (modo de falla esperado), F1 se sacrifica y desconecta la línea.

---

## 4. Módulo 2: Fuente AC-DC aislada (HLK-5M05)

### Función

Convertir 85–265 VAC (50/60 Hz) a 5 VDC aislado con 3000 VAC de rigidez dieléctrica. Esta es la pieza central de la seguridad del diseño — es lo que permite que el USB y los GPIOs del ESP32 estén a potencial seguro SELV.

### Por qué HLK-5M05 y no otras opciones

| Opción | Potencia | Aislamiento | Margen sobre pico 1.65 W | Seguridad USB | Precio |
|---|---|---|---|---|---|
| HLK-PM01 (3 W / 0.6 A) | 3 W | 3000 VAC | 82 % | Seguro | $1.50–2.50 |
| **HLK-5M05 (5 W / 1 A)** | **5 W** | **3000 VAC** | **203 %** | **Seguro** | **$2.00–3.50** |
| MeanWell IRM-05-5 (5 W / 1 A) | 5 W | 3000 VAC | 203 % | Seguro | $5.00–9.00 |
| SMPS no aislada (LNK304) | ~1.5 W | Ninguno | — | **PELIGROSO** | $0.50–1.00 |
| Capacitive dropper | ~0.3 W | Ninguno | — | **PELIGROSO** | $0.20 |

El **HLK-PM01 es justo** (82 % de margen sobre el pico) — el ESP32-C3 en TX WiFi consume hasta 350 mA @ 3.3 V que se transforman en ~530 mA @ 5 V a través del LDO (eficiencia 66 %), más 80 mA de la bobina del relé = 610 mA pico. Con el PM01 a 600 mA **no hay margen**. El **HLK-5M05 a 1 A** da 390 mA de margen (~64 %), opera a ~60 % de carga en el peor caso, se calienta menos y dura más.

Para **producción certificada UL**, sustituir por **MeanWell IRM-05-5** (mismo footprint DIP-4, certificado UL/TÜV/CB).

### Presupuesto de potencia detallado

| Consumidor | Condición | Corriente @ 5 V | Potencia |
|---|---|---|---|
| ESP32-C3 TX WiFi (pico 2 ms) | Via LDO 66 % eff: 350 mA @ 3.3 V ÷ 0.66 | 530 mA | 2.65 W |
| ESP32-C3 modem-sleep (promedio) | Via LDO: 30 mA @ 3.3 V ÷ 0.66 | 45 mA | 0.23 W |
| Relé HF115F-I bobina (activado) | 5 V / 62.5 Ω | 80 mA | 0.40 W |
| LED de estado | Via LDO: 2.3 mA @ 3.3 V | ~4 mA | 0.01 W |
| PC817 fototransistor + NTC | Despreciable | ~1 mA | < 0.01 W |
| **Total pico** (TX + relé ON) | | **~615 mA** | **~3.07 W** |
| **Total promedio** (modem-sleep + relé ON) | | **~130 mA** | **~0.65 W** |
| **Capacidad HLK-5M05** | | **1000 mA** | **5.00 W** |

### Conexiones pin a pin

```
HLK-5M05 (módulo DIP-4, 34 × 20 × 15 mm):

         ┌──────────────────┐
  AC ────┤ 1              4 ├──── +Vo (+5 VDC)
         │    HLK-5M05      │
  AC ────┤ 2              3 ├──── −Vo (GND DC)
         └──────────────────┘

Pin 1 (AC)  <── AC_L_OUT desde Módulo 1 (salida del choke L1 o nodo L post-NTC1)
Pin 2 (AC)  <── AC_N_OUT desde Módulo 1 (no polarizado, L/N intercambiables)
Pin 3 (−Vo) ──> GND_DC rail (aislado galvánicamente de AC)
Pin 4 (+Vo) ──> +5V_AC rail ──> hacia Módulo 7 (OR-diode)
```

**Posición en el PCB**: el módulo cruza la barrera de aislamiento. Pins 1 y 2 en la zona AC, pins 3 y 4 en la zona DC. El slot fresado de 2 mm pasa debajo del cuerpo del módulo.

### Filtrado de salida (zona DC)

| Ref | Componente | Valor | Conexión | Banda de frecuencia |
|---|---|---|---|---|
| C_out1 | Electrolítico radial | 100 µF / 16 V (5×7 mm) | +5V_AC (pin 4) a GND_DC (pin 3). Lo más cerca posible del módulo. | Baja freq (DC – 1 kHz). Reserva bulk, transitorios relé/WiFi TX. |
| C_out2 | Cerámico MLCC X5R | 10 µF / 25 V 0805 | +5V_AC a GND_DC. Paralelo a C_out1, filtra rizado MF. | Media freq (1 kHz – 100 kHz). ESR ultra-bajo absorbe rizado flyback 65-100 kHz. |
| C_out3 | Cerámico MLCC X7R | 100 nF / 50 V 0805 | +5V_AC a GND_DC. Paralelo a C_out1/C_out2, filtra HF. | Alta freq (>100 kHz). Suprime spikes de conmutación y VHF. |

---

## 5. Módulo 3: Relé de potencia + snubber de salida

### Selección del relé: Hongfa HF115F-I/005-1HS3A

| Parámetro | **HF115F-I (elegido)** | Songle SRD-05 | Panasonic AJVN5241F |
|---|---|---|---|
| Tensión bobina | **5 VDC** | 5 VDC | 12 VDC |
| Resistencia bobina | **~62.5 Ω** | 70 Ω | 720 Ω |
| Corriente bobina | **80 mA** | 71 mA | 16.7 mA |
| Potencia bobina | **0.40 W** | 0.36 W | 0.20 W |
| Configuración contactos | **SPST-NO (1 Form A) bifurcado** | SPDT (1 Form C) | SPST-NO (1 Form A) |
| Rating contactos | **16 A / 250 VAC @ 85 °C** | 10 A / 125 VAC | 16 A / 250 VAC |
| Material contactos | **AgSnO₂** | AgCdO (o AgSnO₂ RoHS) | AgSnO₂ |
| **Rating de inrush** | **120 A / 20 ms (explícito)** | No especificado | TV-8 (implícito para lighting) |
| Rigidez dieléctrica bobina↔contacto | **5000 Vrms** (10 mm creepage) | 1500 Vrms | 2500 Vrms |
| Vida mecánica | **10⁷ ciclos** | 10⁷ ciclos | 2 × 10⁷ ciclos |
| Vida eléctrica @ carga nominal | **10⁵ ciclos** | 10⁵ ciclos | 10⁴ ciclos |
| Certificaciones | **UL, TÜV, CQC** | Ninguna confiable | UL, CSA, VDE |
| Dimensiones | **29.0 × 12.7 × 15.7 mm** | 19.0 × 15.5 × 15.3 mm | 20.0 × 10.0 × 10.9 mm |
| Precio (qty 100) | **$2.00–3.00** | $0.30–0.50 | $3.00–5.00 |

**Razón de elección sobre el Panasonic AJVN**: bobina de 5 V permite alimentación directa desde el HLK-5M05 sin rail adicional de 12 V. Esto elimina 3–5 componentes del BOM (módulo 12 V o buck converter + capacitores). El rating de inrush de 120 A / 20 ms está explícitamente publicado en el datasheet, mientras que el AJVN solo indica TV-8. Y los **5000 Vrms** de aislamiento (datasheet Hongfa) superan ampliamente los 2500 Vrms de Shelly.

**Razón de elección sobre el Songle SRD-05**: el Songle es un relé de prototipado sin rating de inrush documentado, con contactos AgCdO que se sueldan con inrush de LED (ver análisis detallado en la ingeniería inversa de la ESP32-C6_Relay_X1). Para producto residencial, inaceptable.

### Circuito driver del relé — conexiones pin a pin

El HF115F-I tiene **6 pines** (1, 2 bobina; 5, 6, 7, 8 contactos bifurcados — no existen pines 3 ni 4). La bobina es **no polarizada**: cualquiera de los pines 1 o 2 puede ir al +5V. Convención aquí: pin 2 = lado alto, pin 1 = lado conmutado por Q1.

```
                          +5V rail
                            │
                     K1 Coil A2 (pin 2)
                            │
                     D1 (SS14) cátodo ──── +5V rail
                            │
                     D1 (SS14) ánodo
                            │
                     K1 Coil A1 (pin 1)
                            │
                     Q1 Collector (SS8050 SOT-23 pin 2)
                            │
          ┌─────────────────┤
          │                 │
ESP32     │          Q1 Base (pin 1)
GPIO4 ──[R1: 1.0 kΩ]───────┤
                            │
                       [R2: 10 kΩ]
                            │
                           GND
                            │
                     Q1 Emitter (pin 3)
                            │
                           GND
```

### Cálculos del driver

**Corriente de bobina:**
- I_coil = V_rail / R_coil = 5.0 V / 62 Ω ≈ **80 mA**
- V_CE(sat) del SS8050 a 80 mA ≈ 0.2 V → V_coil real = 4.8 V > **3.5 V (pickup, datasheet)** ✓

**Corriente de base necesaria:**
- hFE mínimo del SS8050 @ 80 mA = 100 (datasheet)
- Overdrive factor = 3 (para saturación segura en todo el rango de temperatura)
- I_base_min = 80 mA / 100 × 3 = **2.4 mA**

**Resistencia de base R1:**
- R1 = (V_GPIO − V_BE) / I_base = (3.3 − 0.7) / 2.4 mA = 1083 Ω → **1.0 kΩ (valor estándar E24)**
- I_base real con R1=1.0k: considerando el divisor con R2...

**Pull-down de boot R2 (crítico para seguridad):**
- Durante el boot del ESP32, GPIO4 está en alta impedancia (tri-state). Sin R2, ruido o corriente de fuga podrían activar el relé momentáneamente.
- R2 = 10 kΩ mantiene la base a 0 V cuando GPIO4 no está driving.
- **Durante operación normal** (GPIO4 = HIGH, 3.3 V):
  - V_base = 3.3 × (10k / (1k + 10k)) = **3.0 V** (suficiente, > 0.7 V)
  - I_base = (3.0 − 0.7) / 1.0k = **2.3 mA** (factor de saturación = 2.3 × 100 / 80 = 2.88x) ✓
- **Durante boot** (GPIO4 = Hi-Z):
  - V_base = 0 V (R2 a GND). Relé apagado. ✓

**Diodo flyback D1 (SS14 Schottky):**
- Cuando Q1 se apaga, la bobina del relé genera un pico de tensión inversa (V = −L × dI/dt).
- D1 (SS14, 40 V / 1 A, VF = 0.5 V) clampa este pico a +5.5 V (rail + VF), protegiendo Q1.
- Energía almacenada en la bobina: E = ½ × L × I² ≈ ½ × 50 mH × (80 mA)² = 0.16 mJ — trivial para el SS14.
- El SS14 Schottky tiene menor VF que el 1N4148 (0.5 V vs 0.7 V), dando un clamp más limpio.

### Pinout del relé HF115F-I (vista inferior, DIP-6)

```
  Vista inferior (bottom view) — datasheet Hongfa

  ┌─────────────────────────────────────────────┐
  │                                             │
  │    ●                                  ●  ●  │
  │   (2)                                (6)(8) │  ← zona AC (contactos)
  │                                             │
  │          HF115F-I/005-1HS3A                 │
  │          5V 16A 1 Form A bifurcado          │
  │                                             │
  │    ●                                  ●  ●  │
  │   (1)                                (5)(7) │
  │                                             │
  └─────────────────────────────────────────────┘
   ↑ zona DC
   (bobina)

 Pin 1 = Coil A1   ──> Collector Q1 (+ D1 ánodo)              [ZONA DC]
 Pin 2 = Coil A2   ──> +5V rail (+ D1 cátodo)                 [ZONA DC]
 Pin 5 = NO1       ──┐
                     ├── unir externamente en el PCB → LOAD_OUT → J_LOAD [ZONA AC]
 Pin 7 = NO2       ──┘
 Pin 6 = COM1      ──┐
                     ├── unir externamente en el PCB → AC_L (línea)      [ZONA AC]
 Pin 8 = COM2      ──┘
```

**Nota**: el relé cruza la barrera de aislamiento. Pins 5, 6, 7, 8 (contactos) están en la zona AC; pins 1 y 2 (bobina) están en la zona DC. El aislamiento de **5000 Vrms** (creepage interno 10 mm, datasheet Hongfa) del propio relé proporciona la separación.

**Contactos bifurcados**: el HF115F-I tiene dos contactos SPST-NO en paralelo dentro del mismo encapsulado, accionados por la misma armadura. Para aprovechar el rating de 16 A y el inrush de 120 A/20 ms es **obligatorio unir externamente pin 5 con pin 7, y pin 6 con pin 8**. Si solo se cablea un par, la capacidad de corriente se reduce a la mitad.

### Snubber en contactos de salida

```
     nodo COM bifurcado (K1 pin 6 + pin 8)
            │
            ├────── [R_snub: 100 Ω 1 W] ──────┐
            │                                  │
            ├────── [C_snub: 10 nF X2] ───────┤
            │                                  │
            │                                  │
     Línea AC (L)                   nodo NO bifurcado (K1 pin 5 + pin 7)
                                            │
                                            └── J_LOAD (a la carga)
```

**Por qué 10 nF y no 100 nF:**
- Con 100 nF, impedancia del capacitor a 60 Hz: Z = 1/(2π × 60 × 100 nF) = 26.5 kΩ
- Corriente de fuga con relé abierto: I = 110 V / 26.5 kΩ = **4.15 mA**
- Muchos drivers LED económicos tienen un umbral de corriente mínima de ~1–3 mA para mantener el LED apagado. Con 4.15 mA, el LED **parpadea** (ghost flickering).
- Con 10 nF: Z = 265 kΩ, I = 110 V / 265 kΩ = **0.4 mA** — por debajo del umbral. Sin parpadeo.

### Lista de componentes del módulo 3

| Ref | Componente | Valor / Part Number | Encapsulado | Qty |
|---|---|---|---|---|
| K1 | Hongfa HF115F-I/005-1HS3A | Relé 5 V 16 A 1 Form A bifurcado, AgSnO₂, 5000 Vrms isol. | THT 6 pines (29 × 12.7 × 15.7 mm) | 1 |
| Q1 | SS8050 NPN BJT | V_CE=25 V, I_C=1.5 A, hFE>=100 | SOT-23 | 1 |
| D1 | SS14 Schottky | 40 V / 1 A, V_F=0.5 V | SMA (DO-214AC) | 1 |
| R1 | Resistencia base | 1.0 kΩ ±5 % 1/16 W | 0402 SMD | 1 |
| R2 | Pull-down anti-boot | 10 kΩ ±5 % 1/16 W | 0402 SMD | 1 |
| R_snub | Snubber resistor | 100 Ω ±5 % 1 W MFR | Axial THT (tolerancia a 110/220 V) | 1 |
| C_snub | Snubber capacitor | 10 nF X2 310 VAC (KYET PX103K2C0702) | Box film P=7.5 mm | 1 |
| J_LOAD | Terminal block salida | KF128-7.5-2P, 300 V / 15 A | THT 7.5 mm pitch | 1 |

---

## 6. Módulo 4: Regulador 3.3 V (ME6211C33)

### Función

Convertir el rail de +5 V a 3.3 V estable para alimentar el ESP32-C3 y la electrónica digital de señal (optoacoplador secundario, NTC pull-up, LED).

### Selección: ME6211C33M5G-N

| Parámetro | AMS1117-3.3 (rechazado) | AP2112K-3.3 (alternativa) | **ME6211C33 (elegido)** |
|---|---|---|---|
| Tipo | LDO | LDO | **LDO** |
| Dropout voltage | 1.3 V | 250 mV | **100 mV** |
| Corriente máx salida | 1 A | 600 mA | **500 mA** |
| Corriente quiescente | 5–11 mA | 55 µA | **40 µA** |
| PSRR @ 1 kHz | 65 dB | 70 dB | **75 dB** |
| Ruido de salida | ~250 µVrms | ~55 µVrms | **~36 µVrms** |
| Estable con MLCC cerámicos | No (necesita tantalio) | Sí | **Sí** |
| Pin de enable | No | Sí | **Sí** |
| Encapsulado | SOT-223 (grande) | SOT-23-5 | **SOT-23-5** |
| Precio | $0.08 | $0.16 | **$0.12** |

**Razón de rechazo del AMS1117**: con dropout de 1.3 V, si el HLK-5M05 baja a 4.5 V bajo carga (posible con los picos de TX WiFi), la tensión de salida sería 4.5 − 1.3 = 3.2 V — por debajo de los 3.3 V nominales del ESP32. Además, el AMS1117 es inestable con capacitores cerámicos MLCC y requiere tantalio, y su consumo quiescente de 5–11 mA es 100× superior al ME6211.

### Análisis térmico

- **Disipación pico**: (5.0 − 3.3) × 0.35 A = **0.595 W** (durante TX WiFi, ~2 ms)
- En SOT-23-5 con plano GND y 4 vías térmicas (θ_JA ≈ 150 °C/W):
  - ΔT_rise = 0.595 × 150 = 89 °C
  - Pero el pico dura ~2 ms — la masa térmica del encapsulado y el cobre absorben. T_junction real << cálculo estacionario.
- **Disipación promedio**: (5.0 − 3.3) × 0.15 A = **0.255 W**
  - ΔT_rise = 0.255 × 150 = 38 °C
  - A 35 °C ambiente (caja de interruptor): T_junction = 73 °C — **cómodo**, muy por debajo del shutdown térmico.

### Circuito de aplicación — conexiones pin a pin

```
ME6211C33M5G-N (SOT-23-5):

Pin 1 (VIN)  ──── +5V rail ──┬──[C4: 1 µF / 25 V X7R 0603]── GND
                              │
Pin 2 (VSS)  ──── GND (con 4 vías térmicas de 0.3 mm a plano inferior)
                              
Pin 3 (CE)   ──── VIN (enable permanente — conectar a pin 1)

Pin 4 (NC)   ──── Flotante (no conectar)

Pin 5 (VOUT) ──── +3.3V rail ──┬──[C5: 10 µF / 10 V X7R 0805]── GND
                                │
                           [C6: 100 nF / 16 V X7R 0402]── GND
```

**Notas de layout:**
- C4 lo más cerca posible de pin 1 (VIN), con retorno corto a GND
- C5 y C6 lo más cerca posible de pin 5 (VOUT)
- 4 vías térmicas de 0.3 mm debajo del pad GND (pin 2) conectando al plano de cobre inferior

### Lista de componentes del módulo 4

| Ref | Componente | Valor | Encapsulado | Qty |
|---|---|---|---|---|
| U2 | ME6211C33M5G-N | LDO 3.3 V / 500 mA, dropout 100 mV | SOT-23-5 | 1 |
| C4 | Cerámico entrada | 1 µF / 25 V X7R | 0603 | 1 |
| C5 | Cerámico salida (bulk) | 10 µF / 10 V X7R | 0805 | 1 |
| C6 | Cerámico salida (HF) | 100 nF / 16 V X7R | 0402 | 1 |

---

## 7. Módulo 5: ESP32-C3 Core

### Módulo: ESP32-C3-MINI-1-N4

- **Cristal 40 MHz** integrado en el módulo (no se necesita externo en la PCB portadora)
- **Antena PCB** integrada (inverted-F, ~2.5 dBi) con zona de keepout definida por Espressif
- **22 GPIOs** disponibles (GPIO0–10, 18–21); GPIO11–17 reservados para flash SPI interna
- **13.2 × 16.6 × 2.4 mm** — más compacto que el WROOM-1 del C6 (25.5 × 18 mm)
- **Pre-certificado** FCC (2AC7Z-ESPC3MINI1), CE (RED), IC — herencia modular al producto final
- **Pads castellated** en los bordes: soldable a mano. Pad GND inferior requiere vía térmica o reflow.

### Asignación completa de GPIOs

| GPIO | Función asignada | Dirección | Módulo | Notas |
|---|---|---|---|---|
| **GPIO4** | Control relé | Output | M3 | Al base del SS8050 via R1 (1 kΩ). No es strapping pin. Seguro. |
| **GPIO5** | Entrada switch (SW) | Input (pull-up) | M8 | Collector del PC817 con R5 (10 kΩ) pull-up a 3.3 V. No es strapping. |
| **GPIO3** | Sensor NTC temperatura | ADC Input | M9 | ADC1_CH3. No es strapping. Lectura analógica limpia. |
| **GPIO6** | LED de estado | Output | M9 | Active-low via R_LED (470 Ω). No es strapping. MTCK (JTAG) pero no afecta boot normal. |
| **GPIO9** | Botón BOOT | Input (pull-up interno) | M9 | **Strapping pin principal**. Pull-up interno → SPI flash boot. Botón a GND → download mode. |
| **GPIO18** | USB D− | USB | M6 | **Hardwired en silicio** al controlador USB Serial/JTAG. No reasignable. |
| **GPIO19** | USB D+ | USB | M6 | **Hardwired en silicio**. No reasignable. |
| **GPIO21** | UART TX (fallback) | Output | M6 | UART0 TXD por defecto. Header secundario de flasheo. |
| **GPIO20** | UART RX (fallback) | Input | M6 | UART0 RXD por defecto. Header secundario. |
| **EN** | Reset (chip enable) | Input | M5 | Pull-up externo 10 kΩ + 100 nF a GND + botón reset. |
| **GPIO2** | *Strapping (pull-up 10 kΩ)* | — | M5 | Debe ser HIGH al boot. Pull-up 10 kΩ externo obligatorio. Pin libre después del boot. |
| **GPIO8** | *Strapping (pull-up interno)* | — | M5 | Controla ROM serial output. Pull-up interno suficiente. Pin libre después del boot. |
| GPIO0, 1, 7, 10 | **Reservados / libres** | — | — | Disponibles para expansión futura (I²C, SPI, sensores adicionales). |

**Justificación de la selección de pines:**
- **GPIO4 para relé**: no es strapping pin, no es ADC, no tiene efectos secundarios al boot. Salida digital limpia. Mismo pin usado en el diseño previo con C3 (`smart_relay/`).
- **GPIO5 para switch**: no es strapping pin, soporta pull-up interno. Mismo pin del diseño previo.
- **GPIO3 para NTC**: es ADC1_CH3 (necesario para lectura analógica). No es strapping pin — más limpio que GPIO4 del diseño C6 que era MTMS.
- **GPIO6 para LED**: no es strapping. MTCK (JTAG) pero no afecta boot normal. Limpio para salida digital.
- **GPIO9 para BOOT**: strapping pin designado por Espressif para selección de modo de boot en C3 (idéntico al C6).
- **GPIO2 (strapping)**: debe ser HIGH al boot para operación normal. Pull-up 10 kΩ externo obligatorio. No se usa para otra función para evitar interferencia.

### Circuito de soporte del ESP32-C3

**Alimentación y decoupling:**
```
+3.3V rail ──┬──[C_vdd1: 10 µF X7R 0805]──┬──[C_vdd2: 100 nF X7R 0402]──┬── Pin 3V3 (módulo)
              │                              │                              │
             GND                            GND                           GND
```
- C_vdd1 (10 µF) absorbe los transientes de corriente del TX WiFi (hasta 350 mA pico)
- C_vdd2 (100 nF) filtra ruido de alta frecuencia del switching interno
- Ambos colocados a < 3 mm del pin 3V3 del módulo

**Circuito de reset (pin EN):**
```
+3.3V ──[R_EN: 10 kΩ]──┬── Pin EN (módulo)
                         │
                    [C_EN: 100 nF]──── GND
                         │
                    [SW2: Tactile 6×6 mm]── GND
```

**Strapping pin GPIO2 (obligatorio):**
```
+3.3V ──[R_STRAP: 10 kΩ]── GPIO2
```
- Mantiene GPIO2 = HIGH durante boot para selección de modo SPI flash normal
- Sin este pull-up, el boot puede fallar intermitentemente

**Botón BOOT (GPIO9):**
```
GPIO9 ──[SW1: Tactile 6×6 mm]── GND
```
- Pull-up interno suficiente — **no agregar pull-up externo**
- **No agregar capacitor grande a GPIO9** — podría causar entrada espuria a download mode

**Pins GND del módulo (pads laterales + pad inferior):**
- Todos conectados al plano GND con múltiples vías
- Pad inferior GND: 4 vías térmicas de 0.3 mm mínimo

### Zona de keepout de antena

Per Espressif Hardware Design Guidelines para C3-MINI-1:
- **10 mm** de zona libre más allá del borde de la antena del módulo
- **Sin cobre** (trazas, planos de GND, fill) en ninguna capa dentro de la zona de keepout
- **Sin componentes** dentro de 5 mm del extremo de la antena
- **Orientación**: la antena debe apuntar hacia la pared exterior de la caja plástica, alejada de la zona AC y del relé metálico
- El keepout del C3-MINI-1 es más compacto que el del C6-WROOM-1 (10 mm vs 15 mm), ventaja en PCBs pequeñas

### Lista de componentes del módulo 5

| Ref | Componente | Valor | Encapsulado | Qty |
|---|---|---|---|---|
| U1 | ESP32-C3-MINI-1-N4 | 4 MB flash, antena PCB, RISC-V 160 MHz | Módulo castellated 13.2 × 16.6 × 2.4 mm | 1 |
| C_vdd1 | Cerámico decouple 3.3 V | 10 µF / 10 V X7R | 0805 | 1 |
| C_vdd2 | Cerámico HF decouple | 100 nF / 16 V X7R | 0402 | 1 |
| R_EN | Pull-up EN | 10 kΩ ±5 % | 0402 | 1 |
| R_STRAP | Pull-up GPIO2 (strapping) | 10 kΩ ±5 % | 0402 | 1 |
| C_EN | Filtro EN | 100 nF / 16 V X7R | 0402 | 1 |
| SW1 | Botón BOOT | Tactile 6×6 mm | THT | 1 |
| SW2 | Botón RESET | Tactile 6×6 mm | THT | 1 |

---

## 8. Módulo 6: USB-C (flasheo y programación)

### Función

Interfaz primaria de flasheo usando el USB Serial/JTAG nativo del ESP32-C3. Como el lado DC está galvánicamente aislado de la red AC, conectar USB con la placa energizada es seguro.

### Circuito completo — conexiones pin a pin

```
Conector USB-C (16 pines):

  VBUS (A4, B4) ──[PTC1: 500 mA resettable]── +5V_USB ──> Módulo 7 (D_USB)
  
  CC1 (A5) ──[R_CC1: 5.1 kΩ ±1 %]── GND
  CC2 (B5) ──[R_CC2: 5.1 kΩ ±1 %]── GND
  
  D− (A7, B7) ──┬──[U_TVS: USBLC6-2SC6]──[R_DM: 22 Ω]── GPIO18 (USB_D−)
                 │
  D+ (A6, B6) ──┴──[U_TVS: USBLC6-2SC6]──[R_DP: 22 Ω]── GPIO19 (USB_D+)
  
  GND (A1, B1, A12, B12) ── GND
  SHELL ──[R_shell: 1 MΩ]──┬── GND
                            │
                     [C_shell: 4.7 nF]
                            │
                           GND
```

### Detalle de conexiones del USBLC6-2SC6 (SOT-23-6)

```
USBLC6-2SC6:
  Pin 1 (I/O1)  ── D− del conector USB-C (después de junction D−)
  Pin 2 (GND)   ── GND
  Pin 3 (I/O2)  ── D+ del conector USB-C
  Pin 4 (I/O2)  ── D+ hacia R_DP (22 Ω) → GPIO19
  Pin 5 (VBUS)  ── +3.3V (o VBUS 5V, según datasheet; 3.3V recomendado para clamp más bajo)
  Pin 6 (I/O1)  ── D− hacia R_DM (22 Ω) → GPIO18
```

### Por qué R_CC1 y R_CC2 son obligatorios

Sin los pull-downs de 5.1 kΩ en CC1 y CC2, los cables USB-C a USB-C no negocian y no enumeran el dispositivo. Esto es exactamente el bug documentado en la ingeniería inversa de la ESP32-C6_Relay_X1 china: ahorraron estos dos resistores y la placa solo funciona con cables USB-A a USB-C. Con R_CC1 y R_CC2, la enumeración funciona con cualquier cable USB-C.

### Auto-reset en silicio

El ESP32-C3 maneja la secuencia auto-reset **en el propio controlador USB Serial/JTAG integrado**. No necesita el circuito clásico de dos transistores NPN (DTR/RTS → EN/GPIO0) que usa el ESP32 original. esptool.py envía comandos USB para comunicarse directamente. El botón BOOT (GPIO9 a GND) se usa como fallback manual: mantener BOOT presionado mientras se pulsa RESET para entrar en download mode.

### Lista de componentes del módulo 6

| Ref | Componente | Valor / Part Number | Encapsulado | Qty |
|---|---|---|---|---|
| J_USB | Conector USB-C 16-pin | Mid-mount SMD (GCT USB4085-GF-A o equiv.) | SMD | 1 |
| PTC1 | Fusible resettable | 500 mA, V_hold=6 V | 1206 SMD | 1 |
| R_CC1 | Pull-down CC1 | 5.1 kΩ ±1 % | 0402 | 1 |
| R_CC2 | Pull-down CC2 | 5.1 kΩ ±1 % | 0402 | 1 |
| R_DP | Terminación serie D+ | 22 Ω ±5 % | 0402 | 1 |
| R_DM | Terminación serie D− | 22 Ω ±5 % | 0402 | 1 |
| U_TVS | ESD protection USB | USBLC6-2SC6 (STMicro) | SOT-23-6 | 1 |
| C_USB1 | VBUS filtro | 10 µF / 10 V X7R | 0805 | 1 |
| C_USB2 | VBUS decouple | 100 nF / 16 V X7R | 0402 | 1 |

---

## 9. Módulo 7: Selector de fuente AC/USB (OR-Diode)

### Función

Permitir alimentación desde AC (HLK-5M05, operación normal) o desde USB (para flasheo/debug en mesa), sin retroalimentación de corriente entre las dos fuentes. Los diodos Schottky en configuración OR previenen que la fuente AC envíe 5 V al host USB y viceversa.

### Circuito — conexiones pin a pin

```
+5V_AC (HLK-5M05 pin 4) ── D_AC (SS14) ánodo → cátodo ──┐
                                                            ├── +5V_COMBINED
+5V_USB (via PTC1) ──────── D_USB (SS14) ánodo → cátodo ──┘
                                                            │
                                                     [C_bulk: 100 µF / 10 V electrolítico]
                                                            │
                                                           GND
                                                            │
                                                     +5V rail ──> ME6211 (M4) + K1 Coil A2 pin 2 (M3)
```

### Modos de operación

| Modo | Fuente | D_AC | D_USB | Rail +5V | Aplicación |
|---|---|---|---|---|---|
| Solo AC | HLK-5M05 energizado | Conduce (V_F≈0.45V) | Bloqueado | ~4.55 V | Operación normal instalado en pared |
| Solo USB | Cable USB conectado | Bloqueado | Conduce (V_F≈0.45V) | ~4.55 V | Flasheo en mesa de trabajo |
| Ambos | AC + USB simultáneos | Ambos conducen parcialmente | — | ~4.55 V | Tolerable pero NO recomendado |

**¿Por qué SS14 Schottky y no 1N4007 silicio?**
- 1N4007: V_F ≈ 1.0 V → rail cae a 4.0 V → ME6211 con dropout 100 mV produce 3.3 V pero con < 600 mV de margen
- SS14: V_F ≈ 0.45 V → rail queda en 4.55 V → ME6211 tiene 1.15 V de headroom. Mucho más robusto.

### Lista de componentes del módulo 7

| Ref | Componente | Valor | Encapsulado | Qty |
|---|---|---|---|---|
| D_AC | SS14 Schottky | 40 V / 1 A, V_F=0.45 V | SMA | 1 |
| D_USB | SS14 Schottky | 40 V / 1 A | SMA | 1 |
| C_bulk | Electrolítico bulk | 100 µF / 10 V low-ESR | Radial 6.3 mm | 1 |

---

## 10. Módulo 8: Detección de interruptor físico (PC817C)

### Función

Detectar el estado del interruptor de pared existente (toggle o momentáneo) manteniendo aislamiento galvánico completo entre la zona AC y la zona DC.

### Por qué optoacoplador y NO divisor resistivo (como Shelly)

Shelly usa un divisor resistivo directo de la red al GPIO porque su GND **ya está al potencial de red** (fuente no aislada). En este diseño, el lado DC está galvánicamente aislado vía HLK-5M05. Usar un divisor resistivo desde la red hasta un GPIO **rompería ese aislamiento** y anularía toda la inversión en seguridad. El optoacoplador PC817C mantiene **5000 Vrms de separación** entre su LED (zona AC) y su fototransistor (zona DC).

### Circuito completo — conexiones pin a pin

**Lado AC (primario, zona caliente):**

```
Terminal SW (interruptor de pared) ──┐
                                      │
                                 [R3: 68 kΩ 1 W MFR]
                                      │
                                 [R4: 68 kΩ 1 W MFR]
                                      │
                                 [D2: 1N4007] ánodo hacia R4, cátodo hacia PC817 pin 1
                                      │
                                PC817 Pin 1 (ánodo LED)
                                      │
                                PC817 Pin 2 (cátodo LED)
                                      │
                                 Neutro (N) ── retorno por camino AC
```

**Lado DC (secundario, zona SELV):**

```
+3.3V ──[R5: 10 kΩ]──┬── GPIO5 (ESP32-C3)
                       │
              PC817 Pin 4 (Collector fototransistor)
                       │
              PC817 Pin 3 (Emitter fototransistor)
                       │
                      GND

GPIO5 ──[C_deb: 100 nF X7R]── GND    (filtro RC hardware debounce, τ = 10k × 100nF = 1 ms)
```

### PC817C — pinout y posición en PCB

```
PC817C (DIP-4):
┌─────────────────┐
│  1 (Anode)   4 (Collector) │
│  ●  LED  ●  ●  Photo  ●   │
│  2 (Cathode) 3 (Emitter)  │
└─────────────────┘

Pins 1, 2 ── ZONA AC (lado del slot donde están las trazas de red)
Pins 3, 4 ── ZONA DC (lado del slot donde está el ESP32)
```

El cuerpo del PC817 **cruza el slot fresado** de 2 mm — el aislamiento de 5000 Vrms del optoacoplador es la barrera funcional en esta sección.

### Cálculos para 110 V y 220 V

Se usan dos resistores de 68 kΩ en serie (R_total = 136 kΩ) en vez de uno solo de 136 kΩ para:
1. Repartir la disipación de potencia (cada uno disipa la mitad)
2. Repartir la caída de voltaje (cada uno ve ~77 V pico máx a 220 V, bien dentro del rating)

**A 110 V RMS (155 V pico) — Colombia doméstico:**
- I_LED_peak = (155 − 1.0_diodo − 1.2_LED) / 136k = **1.125 mA**
- I_LED_avg (media onda, factor 0.318) = 1.125 × 0.318 = **0.358 mA**
- Potencia por resistor: V_rms² / (2 × R) = 110² / (2 × 68k) = **0.089 W** (seguro a 1 W rating)
- PC817 grado C (CTR >= 200 % a 5 mA; ~100 % a 0.36 mA por curvas del datasheet):
  - I_collector ≈ 0.358 mA × 1.0 = **0.358 mA**
  - V_GPIO6 = 3.3 − (0.358 × 10k) = 3.3 − 3.58 = **~0 V (saturado). Logic LOW sólido.** ✓

**A 220 V RMS (311 V pico) — exportación o circuitos 220 V:**
- I_LED_peak = (311 − 2.2) / 136k = **2.27 mA**
- I_LED_avg = 2.27 × 0.318 = **0.722 mA**
- Potencia por resistor: 220² / (2 × 68k) = **0.356 W** (seguro a 1 W)
- CTR a 0.72 mA ≈ 150 % → I_collector ≈ 1.08 mA. V_GPIO6 = 3.3 − 10.8 = **~0 V. Logic LOW sólido.** ✓

**Máxima corriente LED a 265 V (peor caso high-line):**
- I_LED_peak = (265 × √2 − 2.2) / 136k = **2.75 mA** — bien por debajo del máximo del PC817 (50 mA). ✓

### Lógica GPIO

| Estado del interruptor | PC817 LED | Fototransistor | GPIO6 | Lógica ESPHome |
|---|---|---|---|---|
| **Abierto** (sin AC en SW) | OFF | OFF (alta impedancia) | GPIO5 = **HIGH (3.3 V)** via R5 | `inverted: true` → OFF |
| **Cerrado** (AC presente en SW) | ON (pulsando a 50/60 Hz) | ON → pull-down | GPIO5 = **LOW (~0 V)** | `inverted: true` → ON |

El ripple de 50/60 Hz de la media onda se filtra por:
1. **C_deb (100 nF)** con R5 (10 kΩ): τ = 1 ms — suaviza los pulsos en un nivel DC estable
2. **Firmware**: `delayed_on_off: 50ms` en ESPHome — debounce digital para el rebote mecánico del interruptor

### Lista de componentes del módulo 8

| Ref | Componente | Valor / Part Number | Encapsulado | Qty |
|---|---|---|---|---|
| U_OPTO | PC817C | Optoacoplador CTR >= 200 % (grado C) | DIP-4 (cruza slot) | 1 |
| D2 | 1N4007 | Rectificador 1000 V / 1 A | SMA (DO-214AC) | 1 |
| R3 | Resistencia limitadora AC | 68 kΩ ±5 % 1 W MFR | Axial THT | 1 |
| R4 | Resistencia limitadora AC | 68 kΩ ±5 % 1 W MFR | Axial THT | 1 |
| R5 | Pull-up fototransistor | 10 kΩ ±5 % 1/4 W | 0402 SMD | 1 |
| C_deb | Debounce hardware | 100 nF / 16 V X7R | 0402 SMD | 1 |
| J_SW | Terminal interruptor | KF128-5.0-2P o pads de soldadura | THT o SMD pad | 1 |

---

## 11. Módulo 9: Sensores, LEDs y botones

### 9.1 Sensor de temperatura NTC

Réplica del diseño de la Shelly Plus 1 (NTC 10 kΩ B=3350 en GPIO32). Permite protección por sobretemperatura con apagado automático del relé a 80 °C.

```
+3.3V ──[R_NTC_PU: 10 kΩ 1 % MFR]──┬── GPIO3 (ADC1_CH3)
                                       │
                                  [NTC_temp: 10 kΩ B=3350]
                                       │
                                      GND
```

**Configuración downstream** (NTC entre punto de sensado y GND):
- A 25 °C: NTC = 10 kΩ → V_ADC = 3.3 × 10k / (10k + 10k) = **1.65 V** (mid-scale)
- A 80 °C: NTC ≈ 1.26 kΩ → V_ADC = 3.3 × 1.26k / (10k + 1.26k) = **0.37 V**
- Ambos valores dentro del rango del ADC del ESP32-C6 con atenuación 12 dB (0–2.5 V efectivos)

### 9.2 LED de estado

```
+3.3V ──[R_LED: 470 Ω 0402]── LED1 ánodo (+)
                                  │
                              LED1 cátodo (−) ── GPIO6
```

**Active-low**: GPIO6 = LOW → LED encendido. GPIO6 = HIGH → LED apagado.
- LED verde 0805, V_F ≈ 2.2 V
- I_LED = (3.3 − 2.2) / 470 = **2.3 mA** — visible en interior, bajo consumo

### 9.3 Botones (ya descritos en Módulo 5)

- **SW1 (BOOT)**: GPIO9 → Tactile 6×6 mm → GND. Pull-up interno 45 kΩ.
- **SW2 (RESET)**: EN → Tactile 6×6 mm → GND. Pull-up externo R_EN 10 kΩ.

### Lista de componentes del módulo 9

| Ref | Componente | Valor | Encapsulado | Qty |
|---|---|---|---|---|
| NTC_temp | Termistor NTC | 10 kΩ B=3350 | Bead o 0805 SMD | 1 |
| R_NTC_PU | Pull-up NTC | 10 kΩ ±1 % MFR | 0402 | 1 |
| LED1 | LED de estado | Verde, V_F ≈ 2.2 V | 0805 SMD | 1 |
| R_LED | Limitador corriente LED | 470 Ω ±5 % | 0402 | 1 |

---

## 12. Diagrama de bloques completo

```
═══════════════ ZONA AC (POTENCIAL DE RED) ═══════════════

L ──┬── [F1 1A SB]──[TF1 115°C]──[NTC1 5Ω]──┬──[L1 CMC]── HLK-5M05 pin 1 (AC)
    │                                          │
    │                                       [MOV1 14D431K]
    │                                       [CX1 100nF X2]
    │                                       [R_disc 1MΩ]
    │                                          │
N ──┼──────────────────────────────────────────┴──[L1 CMC]── HLK-5M05 pin 2 (AC)
    │
    ├── Interruptor pared ── J_SW ──[R3 68k]──[R4 68k]──[D2 1N4007]
    │                                                       │
    │                                                PC817 LED (pins 1,2)
    │                                                       │
    └─────────────────────────────── Neutro return ─────────┘
    
    COM bifurcado (K1 pin 6 ═══ pin 8) ── L (Línea directa)
    NO  bifurcado (K1 pin 5 ═══ pin 7) ──┬──[R_snub 100Ω + C_snub 10nF X2]──┬── J_LOAD ── Carga ── N
                                          └──────────────────────────────────┘

═══════ SLOT FRESADO 2 mm (3000 VAC AISLAMIENTO GALVÁNICO) ═══════

═══════════════ ZONA DC (SELV, AISLADA, SEGURA) ═══════════════

HLK-5M05 pin 4 (+5V) ──|>|── D_AC (SS14) ──┐
                                              ├── +5V_COMBINED ──[C_bulk 100µF]── +5V rail
USB VBUS ──[PTC1 500mA] ──|>|── D_USB (SS14)┘
                                                       │
                                            +5V rail ──┤
                                                       │
                                              K1 Coil A2 (pin 2)
                                              D1 (SS14) cátodo ─── +5V
                                                       │
                            GPIO4 ──[R1 1k]── Q1 Base  │
                                     [R2 10k]── GND    │
                            Q1 Collector ── K1 Coil A1 (pin 1) ── D1 ánodo
                            Q1 Emitter ── GND
                                                       │
                                              ME6211 VIN ─┤
                                              ME6211 VOUT ── +3.3V rail
                                                              │
                                                  ESP32-C3-MINI-1-N4
                                                  ├── GPIO4:  Relé (Q1 base → K1)
                                                  ├── GPIO5:  Switch (PC817 coll. + R5 10k PU)
                                                  ├── GPIO3:  NTC temp (R_NTC 10k PU + NTC 10k)
                                                  ├── GPIO6:  LED estado (R_LED 470Ω)
                                                  ├── GPIO9:  Botón BOOT (SW1 → GND)
                                                  ├── GPIO18: USB D− (→ R_DM 22Ω → USBLC6 → USB-C)
                                                  ├── GPIO19: USB D+ (→ R_DP 22Ω → USBLC6 → USB-C)
                                                  ├── GPIO21: UART TX (→ J_UART pin 3)
                                                  ├── GPIO20: UART RX (→ J_UART pin 2)
                                                  ├── GPIO2:  Strapping (R_STRAP 10k PU)
                                                  ├── EN:     Reset (R_EN 10k PU + C_EN 100nF + SW2)
                                                  └── Antena ──> hacia pared exterior caja
                                                                 10 mm keepout sin cobre
```

### Cableado de instalación residencial

```
Tablero eléctrico:
  Breaker ──────── L terminal (J_AC pin 1)
  Neutro ─────────── N terminal (J_AC pin 2)

Interruptor de pared existente:
  Un extremo ── L (Línea, mismo breaker)
  Otro extremo ── SW terminal (J_SW)

Carga (lámpara/bombilla):
  J_LOAD ──────── un extremo de la carga
  N (Neutro) ──── otro extremo de la carga
```

---

## 13. BOM completo (Bill of Materials)

Todos los componentes incluyen referencia **LCSC** para importación directa en EasyEDA Pro. El archivo `BOM_Smart_Relay_C6.csv` contiene el BOM completo con Manufacturer y MPN para generar la orden en JLCPCB/LCSC.

### Componentes activos (ICs, módulos, relé)

| # | Ref | Componente | Valor | Encapsulado | LCSC | Qty |
|---|---|---|---|---|---|---|
| 1 | U1 | ESP32-C3-MINI-1-N4 | WiFi 4 + BLE 5, RISC-V 160 MHz, 4 MB | Módulo 13.2×16.6 mm | **C2838502** | 1 |
| 2 | U2 | ME6211C33M5G-N | LDO 3.3 V / 500 mA, dropout 100 mV | SOT-23-5 | **C82942** | 1 |
| 3 | U3 | PC817C (grado C) | Optoacoplador CTR >= 200 %, 5000 Vrms | DIP-4 | **C66463** | 1 |
| 4 | U4 | USBLC6-2SC6 | TVS ESD USB, IEC 61000-4-2 nivel 4 | SOT-23-6 | **C7519** | 1 |
| 5 | U5 | HLK-5M05 | Fuente AC-DC 5 V/1 A, 3000 VAC aislamiento | Módulo DIP-4 34×20×15 mm | **C209907** | 1 |
| 6 | K1 | HF115F-I/005-1HS3A | Relé 5 V 16 A 1 Form A bifurcado, AgSnO₂, 5000 Vrms | THT 6 pines (29×12.7×15.7 mm) | **C2976795** | 1 |
| 7 | Q1 | SS8050-G | NPN BJT, V_CE=25 V, I_C=1.5 A, hFE 120-200 | SOT-23 | **C164886** | 1 |

### Diodos

| # | Ref | Componente | Valor | Encapsulado | LCSC | Qty |
|---|---|---|---|---|---|---|
| 8 | D1 | SS14 Schottky | 40 V / 1 A, V_F=0.5 V (flyback relé) | SMA DO-214AC | **C2480** | 1 |
| 9 | D2 | 1N4007W | 1000 V / 1 A (rectificador opto) | SMA DO-214AC | **C727110** | 1 |
| 10 | D_AC | SS14 Schottky | 40 V / 1 A (OR-diode fuente AC) | SMA DO-214AC | **C2480** | 1 |
| 11 | D_USB | SS14 Schottky | 40 V / 1 A (OR-diode fuente USB) | SMA DO-214AC | **C2480** | 1 |

### Resistencias

| # | Ref | Valor | Potencia | Encapsulado | LCSC | Qty | Función |
|---|---|---|---|---|---|---|---|
| 12 | R1 | 1.0 kΩ ±5 % | 1/4 W | 0402 | **C11702** | 1 | Base transistor driver relé |
| 13 | R2 | 10 kΩ ±5 % | 1/4 W | 0402 | **C25744** | 1 | Pull-down anti-boot relé |
| 14 | R3 | 68 kΩ ±5 % | 1 W MFR | Axial THT | **C176393** | 1 | Corriente LED opto (AC) |
| 15 | R4 | 68 kΩ ±5 % | 1 W MFR | Axial THT | **C176393** | 1 | Corriente LED opto (AC) |
| 16 | R5 | 10 kΩ ±5 % | 1/4 W | 0402 | **C25744** | 1 | Pull-up fototransistor opto |
| 17 | R_LED | 470 Ω ±5 % | 1/4 W | 0402 | **C25117** | 1 | Corriente LED de estado |
| 18 | R_NTC_PU | 10 kΩ ±1 % | 1/4 W | 0402 | **C25744** | 1 | Pull-up sensor NTC |
| 19 | R_EN | 10 kΩ ±5 % | 1/4 W | 0402 | **C25744** | 1 | Pull-up pin EN (reset) |
| 20 | R_STRAP | 10 kΩ ±5 % | 1/4 W | 0402 | **C25744** | 1 | Pull-up GPIO2 strapping (C3) |
| 21 | R_CC1 | 5.1 kΩ ±1 % | 1/4 W | 0402 | **C25905** | 1 | USB-C CC1 compliance |
| 22 | R_CC2 | 5.1 kΩ ±1 % | 1/4 W | 0402 | **C25905** | 1 | USB-C CC2 compliance |
| 23 | R_DP | 22 Ω ±5 % | 1/4 W | 0402 | **C25092** | 1 | Terminación serie USB D+ |
| 24 | R_DM | 22 Ω ±5 % | 1/4 W | 0402 | **C25092** | 1 | Terminación serie USB D− |
| 25 | R_disc | 1 MΩ ±1 % | 1 W MFR | Axial THT | **C433678** | 1 | Descarga capacitor X2 |
| 26 | R_snub | 100 Ω ±1 % | 1 W MFR (YAGEO MFR1WSFTE52-100R) | Axial THT D3.3×L9 mm | **C172915** | 1 | Snubber contactos relé |

### Capacitores

| # | Ref | Valor | Tipo | Encapsulado | LCSC | Qty | Función |
|---|---|---|---|---|---|---|---|
| 27 | C4 | 1 µF / 25 V | X7R MLCC | 0603 | **C15849** | 1 | Entrada ME6211 |
| 28 | C5 | 10 µF / 10 V | X7R MLCC | 0805 | **C15850** | 1 | Salida ME6211 (bulk) |
| 29 | C6 | 100 nF / 16 V | X7R MLCC | 0402 | **C14663** | 1 | Salida ME6211 (HF) |
| 30 | C_vdd1 | 10 µF / 10 V | X7R MLCC | 0805 | **C15850** | 1 | Decouple 3.3 V ESP32 |
| 31 | C_vdd2 | 100 nF / 16 V | X7R MLCC | 0402 | **C14663** | 1 | HF decouple ESP32 |
| 32 | C_EN | 100 nF / 16 V | X7R MLCC | 0402 | **C14663** | 1 | Filtro reset (EN) |
| 33 | C_deb | 100 nF / 16 V | X7R MLCC | 0402 | **C14663** | 1 | Debounce HW optoacoplador |
| 34 | C_out1 | 100 µF / 10 V | Electrolítico | Radial THT 4×7 mm | **C43314** | 1 | Filtro salida HLK-5M05 |
| 35 | C_out2 | 100 nF / 16 V | X7R MLCC | 0805 | **C49678** | 1 | HF filtro salida HLK |
| 36 | C_bulk | 100 µF / 10 V | Electrolítico | Radial THT 4×7 mm | **C43314** | 1 | Bulk rail +5V combinado |
| 37 | C_USB1 | 10 µF / 10 V | X7R MLCC | 0805 | **C15850** | 1 | Filtro VBUS USB |
| 38 | C_USB2 | 100 nF / 16 V | X7R MLCC | 0402 | **C14663** | 1 | Decouple VBUS USB |
| 39 | CX1 | 100 nF / 275 VAC | X2 MKP Film | Film P=15 mm | **C434188** | 1 | Filtro EMI diferencial AC |
| 40 | C_snub | 10 nF / 310 VAC | X2 Film | Film P=7.5 mm | **C2693738** | 1 | Snubber contactos relé (KYET PX103K2C0702) |

### Protecciones y fusibles

| # | Ref | Componente | Valor | Encapsulado | LCSC | Qty |
|---|---|---|---|---|---|---|
| 41 | F1 | Fusible slow-blow | 1 A 250 V 5×20 mm | Vidrio axial | **C142715** | 1 |
| 42 | F1_holder | Portafusible BLX-A | 5×20 mm PCB mount | THT | **C3131** | 1 |
| 43 | TF1 | Fusible térmico | 115 °C 2 A 250 V (SETsafe RT167) | Axial THT | **C7499892** | 1 |
| 44 | PTC1 | Fusible resettable PTC | 500 mA, V_hold=6 V | 1206 SMD | **C369153** | 1 |
| 45 | MOV1 | Varistor | 14D431K (MCOV 275 VAC, 4.5 kA) | 14 mm disco radial | **C273566** | 1 |
| 46 | NTC_inrush | Termistor inrush limiter | 5 Ω 2 A 9 mm (RUILON 5D-9) | Disco radial THT | **C3789** | 1 |

### Conectores, mecánicos y componentes varios

| # | Ref | Componente | Especificación | LCSC | Qty |
|---|---|---|---|---|---|
| 47 | J_AC | Terminal block AC | KF128-7.5-2P, 300 V / 15 A | **C474954** | 1 |
| 48 | J_LOAD | Terminal block carga | KF128-7.5-2P, 300 V / 15 A | **C474954** | 1 |
| 49 | J_SW | Terminal switch | KF128-5.0-2P, 300 V / 10 A | **C474950** | 1 |
| 50 | J_USB | Conector USB-C 16-pin | TYPE-C-31-M-12 (Hroparts) SMD | **C165948** | 1 |
| 51 | J_UART | Header 1.27 mm | 1×50P (cortar a 4 pines) | **C3408** | 1 |
| 52 | SW1 | Botón BOOT | TS-1187A-B-A-B 6×6 mm THT | **C318884** | 1 |
| 53 | SW2 | Botón RESET | TS-1187A-B-A-B 6×6 mm THT | **C318884** | 1 |
| 54 | LED1 | LED verde 0805 | KT-0805G, 520-530 nm | **C2297** | 1 |
| 55 | NTC_temp | NTC 10 kΩ 0805 | B=3435 ±1 % Vishay NTCS0805E3103FHT | **C3195213** | 1 |
| 56 | L1 | Choke modo común (opc.) | 10 mH 0.5 A UU9.8 | **C148474** | 1 |

### Resumen

- **Total**: ~56 componentes (incluyendo portafusible, sin contar L1 opcional)
- **BOM estimado**: $10–15 USD en cantidades de 100+
- **100 % con referencia LCSC** — importable directamente en EasyEDA Pro
- **Componentes THT**: J_AC, J_LOAD, J_SW, F1, F1_holder, TF1, NTC_inrush, MOV1, R3, R4, R_disc, R_snub, CX1, C_snub, K1, L1, SW1, SW2, C_out1, C_bulk
- **Componentes SMD**: U1, U2, U4, Q1, D1, D2, D_AC, D_USB, todas las R 0402, C 0402/0603/0805, PTC1, LED1, NTC_temp
- **Componentes que cruzan la barrera de aislamiento**: U3 (PC817, DIP-4), U5 (HLK-5M05, DIP-4), K1 (relé, DIP-6)

**Nota sobre NTC_temp**: el Vishay NTCS0805E3103FHT tiene B=3435K (vs 3350K del diseño Shelly). En ESPHome, ajustar `b_constant: 3435` en el YAML. La diferencia es < 2 °C a 80 °C — aceptable para protección térmica.

---

## 14. Firmware ESPHome para Home Assistant

```yaml
substitutions:
  device_name: "smart-relay-c6"
  friendly_name: "Smart Relay C6"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

esp32:
  variant: esp32c3
  board: esp32-c3-devkitm-1
  framework:
    type: arduino    # C3 soporta Arduino e IDF (C6 solo IDF)

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${device_name} Fallback"
    password: !secret ap_password

captive_portal:

logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

# ── Relé ──────────────────────────────────
output:
  - platform: gpio
    id: relay_output
    pin: GPIO4

switch:
  - platform: output
    id: relay
    name: "${friendly_name} Relay"
    output: relay_output
    restore_mode: RESTORE_DEFAULT_OFF

# ── Interruptor físico (optoacoplador PC817C en GPIO5) ──
binary_sensor:
  - platform: gpio
    name: "${friendly_name} Switch"
    pin:
      number: GPIO5
      mode:
        input: true
        pullup: true
      inverted: true    # PC817 tira a LOW cuando switch cerrado
    filters:
      - delayed_on_off: 50ms
    on_state:
      then:
        - switch.toggle: relay    # Cada cambio de estado alterna el relé (modo Edge)

  # Botón BOOT como factory reset (presión prolongada >= 5 s)
  - platform: gpio
    name: "${friendly_name} Button"
    pin:
      number: GPIO9
      inverted: true
      mode:
        input: true
        pullup: true
    filters:
      - delayed_on_off: 5ms
    on_multi_click:
      - timing:
          - ON for at least 5s
        then:
          - button.press: factory_reset_btn

button:
  - platform: factory_reset
    name: "${friendly_name} Factory Reset"
    id: factory_reset_btn

# ── LED de estado ─────────────────────────
status_led:
  pin:
    number: GPIO6
    inverted: true    # Active-low

# ── Sensor de temperatura NTC ─────────────
sensor:
  - platform: adc
    id: temp_adc
    pin: GPIO3
    attenuation: 12db
    update_interval: 30s

  - platform: resistance
    id: temp_res
    sensor: temp_adc
    configuration: DOWNSTREAM
    resistor: 10kOhm

  - platform: ntc
    sensor: temp_res
    name: "${friendly_name} Temperature"
    unit_of_measurement: "°C"
    accuracy_decimals: 1
    calibration:
      b_constant: 3350
      reference_resistance: 10kOhm
      reference_temperature: 298.15K
    on_value_range:
      - above: 80.0
        then:
          - switch.turn_off: relay
          - logger.log:
              level: ERROR
              format: "THERMAL SHUTDOWN: Temperature %.1f°C exceeded 80°C threshold"
              args: ['x']
```

### Compilación y flasheo

```bash
# Primera vez (por USB-C, cable conectado al PC):
esphome compile smart-relay-c3.yaml
esphome upload smart-relay-c3.yaml --device /dev/ttyACM0

# Actualizaciones posteriores (OTA por WiFi):
esphome upload smart-relay-c3.yaml --device smart-relay-c3.local
```

### Integración Home Assistant

- ESPHome usa **mDNS** para auto-descubrimiento
- En HA: Settings → Devices & Services → ESPHome → el dispositivo aparece automáticamente
- Pegar la IP del dispositivo y la clave de encriptación API
- Las entidades (relay switch, wall switch sensor, temperature, button) se crean automáticamente

---

## 15. Especificaciones del PCB y layout

### Dimensiones y especificaciones generales

| Parámetro | Valor | Justificación |
|---|---|---|
| Dimensiones | **42 × 50 mm** | Cabe en caja mural colombiana estándar (~51×76×64 mm interno) |
| Capas | 2 (4 si presupuesto permite) | 2 capas suficiente; 4 mejora EMC y facilita ruteo |
| Material | FR-4 UL 94 V-0, Tg >= 150 °C | Resistente al fuego, soporta temperatura operativa |
| Cobre zona AC | 2 oz (70 µm) | Para trazas de potencia (relé, terminales) |
| Cobre zona DC | 1 oz (35 µm) | Estándar para señales y alimentación DC |
| Slot de aislamiento | **2 mm ancho**, borde a borde | Barrera física entre zona AC y DC |
| Creepage mínimo | **>= 6.5 mm** | IEC 62368-1, aislamiento reforzado, Bogotá 2640 m |
| Clearance mínimo | **>= 5 mm** | Con factor de corrección por altitud ×1.14 |
| Ancho traza AC (contactos relé, terminales) | >= 2 mm | Para corrientes hasta 16 A |
| Ancho traza AC (fuente HLK, protecciones) | >= 1.5 mm | Para corrientes hasta 1 A |
| Ancho traza DC (rails de potencia) | >= 0.5 mm | Para corrientes hasta 500 mA |
| Ancho traza DC (señales) | >= 0.25 mm | Señales digitales baja corriente |
| Vías térmicas bajo ME6211 | 4× de 0.3 mm diámetro | Conectan pad GND a plano inferior |
| Acabado superficial | HASL (prototipo) o ENIG (producción) | ENIG mejor para SMD fino |

### Layout — distribución de zonas

```
┌──────────────────────────────────────────────────┐
│                                                    │
│   J_AC [L][N]     F1  TF1  NTC1                  │
│                                                    │
│   MOV1  CX1  R_disc      L1 (CMC opcional)       │
│                                                    │
│   ╔═══ HLK-5M05 ═══╗                              │
│   ║ pin1    pin4 ║──── C_out1, C_out2             │
│   ║ (AC)    (+5V)║                                 │
│   ║ pin2    pin3 ║──── D_AC, D_USB, C_bulk        │
│   ║ (AC)    (GND)║                                 │
│   ╚═══════════════╝                                │
│                                                    │
│   ══ SLOT FRESADO 2 mm ═══════════════════════    │
│                                                    │
│   ╔═ PC817 ═╗     ME6211 + caps                   │
│   ║ 1,2 │3,4║                                     │
│   ╚═════════╝     Q1, R1, R2, D1                  │
│                                                    │
│   R3, R4 (AC)     K1 (relé HF115F-I)             │
│   D2 (AC)          ┌─────────────┐                │
│                     │ coil+(1)  5 │ (coil- DC)    │
│   J_SW [SW]        │ NO(3)    4  │ (COM AC)      │
│                     └─────────────┘                │
│   R_snub, C_snub                                   │
│   J_LOAD [OUT]     ESP32-C3-MINI-1               │
│                     ┌─────────────────────┐        │
│                     │                     │        │
│                     │    [  ANTENA  ]─────│───► 15mm│
│                     │                     │  keepout│
│                     │  GPIO7 GPIO6 GPIO4  │        │
│                     │  GPIO6 GPIO9 EN     │        │
│                     │  GPIO18/19 (USB)    │        │
│                     │  GPIO21/20 (UART)   │        │
│                     └─────────────────────┘        │
│                                                    │
│   LED1  SW1(BOOT)  SW2(RESET)                     │
│   NTC_temp  R_NTC_PU                               │
│                                                    │
│   J_USB [USB-C] (borde accesible)                  │
│   R_CC1 R_CC2 R_DP R_DM                           │
│   USBLC6-2SC6  PTC1                                │
│   J_UART (header 4-pin)                            │
│                                                    │
└──────────────────────────────────────────────────┘
```

### Reglas críticas de layout

1. **Slot fresado**: 2 mm de ancho, de borde a borde del PCB. Pasa debajo del HLK-5M05, del PC817 y entre las zonas de bobina y contactos del relé. Sin cobre en ninguna capa dentro de 1 mm del borde del slot.

2. **Componentes que cruzan el slot**: HLK-5M05 (pines AC de un lado, pines DC del otro), PC817 (LED lado AC, fototransistor lado DC), relé K1 (contactos lado AC, bobina lado DC). El aislamiento lo provee el propio componente.

3. **Planos de GND**: separados — plano AC_GND y plano DC_GND. Se conectan **únicamente** a través del HLK-5M05 (pin 3 es el punto de unión aislado). No deben existir vías que crucen el slot.

4. **Antena del ESP32-C6**: en la esquina más alejada de la zona AC, apuntando hacia el borde exterior del PCB (y hacia la pared de la caja plástica). 15 mm de zona libre sin cobre ni componentes.

5. **USB-C**: en un borde accesible del PCB. En producto terminado puede quedar bajo una etiqueta removible.

6. **Conformal coating**: silicona IPC-CC-830B (25–50 µm) sobre toda la PCB ensamblada, excepto contactos del relé, tornillos de terminales y conector USB-C.

---

## 16. Pasos de elaboración de la PCB

### Fase 1: Esquemático (KiCad 8+)

1. Crear proyecto KiCad nuevo: `smart_relay_c6`
2. Agregar bibliotecas de footprints para: ESP32-C3-MINI-1, HLK-5M05, HF115F-I, PC817, ME6211, USB-C, USBLC6-2SC6
3. Dibujar cada módulo como **hoja jerárquica** separada (9 hojas)
4. Conectar módulos mediante **power flags** (+5V, +3.3V, GND) y **net labels** (AC_L, AC_N, SW_DETECT, RELAY_CTRL)
5. Asignar footprints a cada símbolo
6. Ejecutar **ERC** (Electrical Rules Check) y corregir errores/warnings

### Fase 2: Layout PCB

1. Importar netlist del esquemático
2. Definir contorno de placa: 42 × 50 mm con esquinas redondeadas (radio 1 mm)
3. Dibujar **slot fresado** de 2 mm en la capa Edge.Cuts
4. Definir **reglas de diseño** (design rules):
   - Clearance AC↔DC: >= 6.5 mm (creepage), >= 5 mm (clearance en aire)
   - Ancho traza zona AC potencia: >= 2 mm
   - Ancho traza zona AC señal: >= 1.5 mm
   - Ancho traza zona DC: >= 0.25 mm (señal), >= 0.5 mm (potencia)
5. Colocar componentes respetando la separación de zonas
6. Rutear trazas zona AC primero (más restrictivas)
7. Rutear trazas zona DC
8. Verter planos de cobre (GND_AC, GND_DC) en ambas capas
9. Verificar **DRC** (Design Rules Check)
10. Agregar **silkscreen**: marcas de polaridad, texto "ISOLATION BARRIER 3000V — DO NOT BRIDGE", referencias de componentes

### Fase 3: Verificación pre-fabricación

1. Revisar visualmente el aislamiento AC/DC (sin trazas cruzando el slot)
2. Verificar que no hay vías en la zona del slot
3. Verificar creepage con herramienta de medición de KiCad
4. Generar **vista 3D** y verificar alturas (especialmente relé 15.7 mm vs. profundidad de caja)
5. Generar **Gerbers** y **archivos de perforación** (Excellon drill)
6. Generar **BOM** en formato CSV/LCSC para pedido
7. Generar **CPL** (Component Placement List) para servicio PCBA

### Fase 4: Fabricación de prototipo

1. Enviar Gerbers a **JLCPCB** o **PCBWay**:
   - 2 capas, 1.6 mm FR-4, HASL, verde
   - Especificar slot fresado (Edge.Cuts con línea cerrada de 2 mm)
   - Solicitar 2 oz cobre en capa superior (o especificar por zona si el fab lo soporta)
2. Pedir servicio **PCBA** para componentes SMD (o soldar a mano con estación de aire caliente)
3. Soldar componentes THT manualmente: terminales, relé, fusibles, resistencias de potencia, MOV, electrolíticos
4. Inspección visual con lupa 10× antes de energizar

### Fase 5: Firmware y pruebas funcionales

1. **Flasheo inicial**: conectar cable USB-C al PC → el ESP32-C6 debe enumerarse como CDC-ACM (`/dev/ttyACM0`) → ejecutar `esphome upload`
2. **Verificar WiFi**: el dispositivo debe crear un AP fallback `smart-relay-c6 Fallback` si no encuentra la red configurada
3. **Verificar Home Assistant**: el dispositivo debe aparecer automáticamente en HA vía mDNS
4. **Probar relé desde HA**: activar/desactivar desde la UI de HA
5. **Probar interruptor físico**: conectar un interruptor toggle al terminal SW y verificar que cada cambio alterna el relé
6. **Probar sin WiFi**: desconectar el router y verificar que el interruptor físico sigue funcionando
7. **Probar protección térmica**: calentar el NTC con un secador de pelo hasta > 80 °C → el relé debe apagarse automáticamente
8. **Test con carga real**: conectar una lámpara LED al terminal de salida y verificar operación
9. **Verificar ghost flickering**: con relé abierto y carga LED conectada, la lámpara NO debe parpadear
10. **Medir aislamiento**: con megóhmetro, verificar >= 2 MΩ a 500 VDC entre zona AC y zona DC
11. **Medir temperatura**: tras 1 hora a carga completa (relé ON, WiFi activo), verificar temperatura del ME6211 y HLK-5M05

### Fase 6: Test de durabilidad

1. **10 000 ciclos de relé** con carga LED real (6 bombillas LED de 3 marcas distintas)
2. Medir resistencia de contacto del relé al inicio, a 1 000, 5 000 y 10 000 ciclos
3. Degradación > 20 % = falla de diseño del relé
4. **Surge test** (si se tiene generador): pulso combinado 1.2/50 µs – 8/20 µs, 1 kV L-N, 2 kV L-PE

### Fase 7: Certificación (para producción comercial)

- **RETIE Resolución 40117/2024** esquema 5 "marca continua" vía CIDET o ICONTEC
- Evaluación contra NTC 1337 / IEC 60669-2-1 / IEC 62368-1
- Presupuesto: COP 35–85 millones (~USD 4–10 k)
- El módulo ESP32-C3-MINI-1 ya tiene certificación FCC/CE/IC → herencia modular al producto final

---

## 17. Normativas y certificación

### Colombia

| Norma | Alcance | Obligatoriedad |
|---|---|---|
| **RETIE Resolución 40117/2024** | Seguridad eléctrica de productos para instalación residencial | **Obligatorio** para comercialización |
| **NTC 1337 / IEC 60669-2-1** | Interruptores electrónicos para uso doméstico | Referencia técnica de evaluación |
| **IEC 62368-1** | Seguridad de equipos de audio/video, IT y comunicaciones | Referencia para aislamiento y creepage |
| CRC Resolución 4507/2014 | Equipos de telecomunicaciones | No aplica (WiFi/BLE exento en este rango) |
| RETILAP | Luminarias | No aplica (esto no es una luminaria) |

**Costo estimado de certificación RETIE**: COP 35–85 millones (~USD 4–10 k) vía CIDET o ICONTEC, incluyendo ensayos, certificación, inscripción SIC y asesoría legal.

**Riesgo de comercializar sin certificar**: multas hasta 2 000 SMLMV (~COP 2 848 millones en 2026), responsabilidad civil objetiva por producto defectuoso (Ley 1480/2011), y responsabilidad penal por lesiones/homicidio culposo en caso de incendio.

### Internacional (si se exporta)

| Mercado | Requisito | Método |
|---|---|---|
| EE. UU. | FCC Part 15 B SDoC + UL Listing (voluntario) | FCC: declaración con módulo pre-certificado. UL: USD 15–30 k |
| Europa | CE self-declaration (LVD + EMC + RED + RoHS) | Sin organismo notificado. DoC firmada por fabricante. |
| UK | UKCA | Similar a CE, organismo UK |

---

## 18. Ruta de migración a ESP32-C6

Cuando ESPHome madure su soporte de C6 y/o Matter/Thread sea un requisito del producto, la migración es directa. La arquitectura de 9 módulos **no cambia** — solo se sustituye el Módulo 5 (core) y se remapean los GPIOs en los módulos que tocan el ESP32.

### Tabla de equivalencia GPIO (C3 → C6)

| Función | ESP32-C3 (actual) | ESP32-C6 (futuro) | Nota |
|---|---|---|---|
| Control relé | **GPIO4** | **GPIO7** | Ambos no-strapping, digital out |
| Entrada switch (SW) | **GPIO5** | **GPIO6** | Ambos no-strapping, digital in |
| NTC temperatura | **GPIO3** (ADC1_CH3) | **GPIO4** (ADC1_CH4) | Ambos ADC capables |
| LED de estado | **GPIO6** | **GPIO8** | Ambos no-strapping |
| Botón BOOT | **GPIO9** | **GPIO9** | **Idéntico** en ambos |
| USB D− | **GPIO18** | **GPIO12** | Hardwired en silicio |
| USB D+ | **GPIO19** | **GPIO13** | Hardwired en silicio |
| UART TX (fallback) | **GPIO21** | **GPIO22** | Configurable via GPIO matrix |
| UART RX (fallback) | **GPIO20** | **GPIO23** | Configurable via GPIO matrix |
| EN (reset) | **EN** | **EN** | **Idéntico** |
| Strapping extra | **GPIO2** (pull-up 10k) | No necesario | C6 no requiere pull-up en GPIO2 |

### Cambios necesarios para migrar

1. **Hardware**: sustituir U1 (ESP32-C3-MINI-1 → ESP32-C6-WROOM-1). El WROOM-1 es más grande (25.5×18 vs 13.2×16.6 mm) — verificar que cabe en el PCB. Si no, usar ESP32-C6-MINI-1 (mismo footprint compacto pero requiere reflow).
2. **PCB**: reubicar las trazas de los GPIOs remapeados. El resto del circuito (HLK-5M05, relé, optoacoplador, LDO, USB-C, protecciones) no cambia.
3. **Firmware**: cambiar los números de GPIO en el YAML de ESPHome y cambiar framework a `esp-idf` (Arduino no soportado en C6 dentro de ESPHome).
4. **Eliminar R_STRAP**: el C6 no necesita el pull-up externo en GPIO2.

### Cuándo migrar

- Cuando **Matter sea un requisito** (Apple HomeKit-only, Google Home Thread)
- Cuando ESPHome C6 tenga **>= 2 años de estabilidad** en producción
- Cuando el **volumen justifique** el delta de precio ($0.50–1.00/unidad)

---

## Referencias de diseño

- Ingeniería inversa Shelly Plus 1: `./Ingeniería inversa avanzada de la Shelly Plus 1.md`
- Ingeniería inversa ESP32-C6_Relay_X1: `./ngeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md`
- Diseño previo ESP32-C3: `/home/dev13/projects/Domotica/smart_relay/`
- ESP32-C3-MINI-1 Datasheet: Espressif v1.1
- ESP32-C3 Hardware Design Guidelines: docs.espressif.com
- Hongfa HF115F-I Datasheet: www.hongfa.com
- HLK-5M05 Datasheet: www.hlktech.com
- ME6211 Datasheet: www.microne.com.cn
- ESPHome ESP32-C3 support: esphome.io
