# Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable

La placa china **ESP32-C6_Relay_X1 (303E32C6DC1)** es un clon SZHJW de bajo costo (USD 7–11) que combina un SoC puntero —el **ESP32-C6 con Wi-Fi 6, Thread/Zigbee/Matter y USB-JTAG nativo**— con una electrónica de potencia deliberadamente austera: un **relé Songle SRD-05VDC-SL-C** de 10 A sin rating de inrush documentado, un buck SOIC-8 (muy probablemente **XL7015**) seguido de un AMS1117-3.3 y protecciones de entrada mínimas. Es un excelente vehículo de desarrollo sobre C6, pero **no es apta como producto residencial 110 V AC en su forma actual**: el relé es marginal para lámparas LED y claramente insuficiente para aires acondicionados, no tiene certificación RETIE y su rigidez dieléctrica (1500 Vrms) está por debajo del relé Panasonic AJVN que usa el Shelly Plus 1 (2500 Vrms). Para su caso de uso —iluminación y aires acondicionados residenciales en Bogotá, con fuente AC/DC externa— la ruta correcta es rediseñar el PCB manteniendo el C6 y el firmware, sustituyendo el Songle por un Hongfa HF115F-I o contactor externo, añadiendo un front-end AC protegido (fusible + MOV + filtro EMI) alimentado por un módulo aislado tipo HLK-5M05 o MeanWell IRM-05-05, y certificando el conjunto bajo **RETIE Resolución 40117/2024 con referencia a NTC 1337 / IEC 60669-2-1**. El Shelly Plus 1 sigue siendo la comparación correcta en seguridad eléctrica e integración, pero el C6 permite una ventaja estratégica real que Shelly Plus 1 no tiene: **Matter-over-Thread, Wi-Fi 6 con TWT y dual-core RISC-V**.

El resto del informe desarrolla cada bloque con detalle de ingeniería.

## 1. Ingeniería inversa de los componentes de la ESP32-C6_Relay_X1

### 1.1 El ESP32-C6 como núcleo del diseño

El **ESP32-C6** (datasheet Espressif v1.5) es un cambio generacional frente al ESP32-U4WDH que monta Shelly. Tiene una arquitectura **dual RISC-V asimétrica**: un núcleo **HP (High-Performance) RV32IMAC a 160 MHz** con PMP de 16 regiones y pipeline de 4 etapas, y un núcleo **LP (Low-Power) RV32IMAC a 20 MHz** con 16 KB de LP-SRAM dedicada que permanece activo durante deep-sleep. Incorpora 320 KB de ROM, **512 KB de HP-SRAM** y 4 KB de RTC memory; las variantes WROOM-1 integran 4 u 8 MB de flash mientras que las MINI-1 son idénticas pero con footprint reducido. No tiene FPU.

La radio es donde el C6 aplasta al ESP32 clásico: **Wi-Fi 6 (802.11ax) 2.4 GHz con 20 MHz**, OFDMA UL/DL, MU-MIMO en downlink, **TWT (Target Wake Time)** y WPA3 obligatorio; **Bluetooth 5 LE** con coded PHY long-range y **AoA/AoD Direction Finding**; y un **IEEE 802.15.4** nativo que habilita Zigbee 3.0, Thread 1.3 y **Matter over Thread o Wi-Fi**. TWT es particularmente relevante para un interruptor always-on: el dispositivo negocia con el AP una ventana de despertar cada N minutos y duerme el resto, bajando el consumo medio a una fracción del Wi-Fi 4 clásico.

Un detalle decisivo para el diseño de la placa: el ESP32-C6 **integra un controlador USB 2.0 Full-Speed con clases CDC-ACM y JTAG-adapter nativos en silicio** (D+ en GPIO13, D− en GPIO12). Esto elimina la necesidad de un CH340C/CP2102 externo y el par de transistores de auto-reset tipo NodeMCU: el propio controlador USB maneja la secuencia `RESET+DOWNLOAD` por comando. La inspección de fotos públicas de la placa 303E32C6DC1 no muestra ningún SOIC-8 adicional junto al USB-C, confirmando que **el flasheo por USB-C se hace a través del USB-JTAG nativo del C6**. Un header auxiliar de 4 pines "Burn Interface" (GND/RX/TX/5V) queda como fallback sobre UART0 con el procedimiento manual GPIO9↔GND.

Los **pines strapping** son MTMS (GPIO4), MTDI (GPIO5), GPIO8, **GPIO9 = BOOT** y GPIO15; nótese que GPIO9 reemplaza al GPIO0 del ESP32 clásico, por lo que el botón BOOT de la placa actúa sobre GPIO9. La periferia incluye 2×UART + 1×LP-UART, 2×SPI, 1×I²C + 1×LP-I²C, I²S, SDIO 2.0, **TWAI (CAN 2.0)**, LEDC 6 canales, MCPWM, RMT, **SAR-ADC 12-bit en 7 canales**, sensor de temperatura interno y un **ETM (Event Task Matrix)** que encadena periféricos sin intervención de CPU. Seguridad: **Secure Boot v2 con RSA-3072**, Flash Encryption XTS-AES-128/256, HMAC, DS peripheral y **TEE** basado en separación M/U mode. En deep-sleep con RTC + LP memory la hoja v1.5 lista **~7 µA típicos** (la cifra comúnmente repetida de 5 µA corresponde a power-off sin RTC).

Frente al **ESP32-C3** —que la industria está reemplazando— el C6 añade Wi-Fi 6, 802.15.4, el LP-core y Direction Finding; frente al ESP32-U4WDH del Shelly Plus 1 es un salto de arquitectura (Xtensa LX6 → RISC-V) y un salto generacional de radio (Wi-Fi 4 BLE 4.2 → Wi-Fi 6 BLE 5 + Thread).

### 1.2 El relé Songle SRD-05VDC-SL-C y por qué es el eslabón débil

El **Songle SRD-05VDC-SL-C** es un relé SPDT 1-Form-C, bobina 5 VDC / 70 Ω / 71 mA / 0.36 W, con pull-in al 75% de Vnom. El datasheet oficial declara **10 A 125 VAC y 7 A 240 VAC resistivos, pero solo 3 A a 120 VAC inductivo** (cosφ=0.4, L/R=7 ms). La rigidez dieléctrica bobina↔contactos es **1500 Vrms**, los contactos son **AgCdO** (o AgSnO₂ en versiones RoHS tardías) y la vida eléctrica declarada a carga nominal es **10⁵ operaciones**. El punto más preocupante es lo que el datasheet **no publica**: ningún número de **inrush admisible**, a diferencia de Hongfa HF115F-I (120 A / 20 ms), Panasonic DW-H (100 A / 5 ms TV-8) u Omron G5Q, que sí caracterizan este parámetro por diseño para *lighting*.

Esto tiene consecuencias medibles. Los drivers LED modernos con puente rectificador y electrolítico de entrada presentan picos documentados de **30–150× la corriente nominal** durante 100 µs–1 ms; ABB mide **78 A en 195 µs**, OliNo ~45 A en 230 V para una luminaria de 500 W, Ametherm cita **100× la corriente steady-state** para LED frente a 6–10× para incandescentes. Durante el cierre mecánico del contacto, ese pulso capacitivo genera un **arco de plasma que funde microscópicamente el AgCdO y suelda los contactos** ("welding") al enfriar. La comunidad técnica (EEVblog, Reef2Reef, All About Circuits, Edaboard) es unánime: el SRD-05VDC-SL-C es **marginalmente aceptable para 2–3 bombillas LED de buena calidad** y **no adecuado para 4 o más bombillas LED económicas** sin NTC o SSR zero-crossing.

Para aires acondicionados la situación es peor. Un mini-split **12 000 BTU en 110 V** tiene FLA de 8–12 A (~1000–1200 W) y un **LRA de 45–65 A durante 100–500 ms**; el subciclo inicial puede llegar a 80–100 A durante 8–16 ms por saturación de bobinados. Comparado con el rating inductivo real del Songle (3 A @ 120 VAC), el margen es negativo:

| Carga | Steady-state | Pico | Ciclos esperados SRD-05 |
|---|---|---|---|
| 1 bombilla LED 10 W | 0.08 A | 5–15 A / 100 µs | >10⁵ |
| 6 LED 10 W baratas | 0.5 A | 60–200 A / 200 µs | <10⁴ (welding probable) |
| 4 incandescentes 100 W | 3.3 A | 30–40 A / 50 ms | 10⁴–10⁵ |
| AC 12 k BTU no-inverter | 9–11 A | 45–65 A / 200 ms | **100–1000 (falla)** |
| AC 12 k BTU inverter | 8–10 A | 30–80 A capacitivo | 1000–5000 |

El **Panasonic AJVN5241F** que monta el Shelly Plus 1 resuelve el problema con material AgSnO₂, 2500 Vrms de aislamiento bobina-contacto, rating UL/CSA/VDE del componente y una clasificación TV-8 para lighting. El precio es ~$3–5 contra $0.30–0.50 del Songle; ese delta de USD 3 es el que separa un producto residencial de un juguete.

### 1.3 Cadena de alimentación: buck no síncrono + LDO

El rango de entrada **7–60 VDC** impreso en la bornera descarta los sospechosos habituales: MP2315 (24 V máx), MP1584 (28 V), LM2596 (40 V) y XL4015 (36 V). Quedan dos candidatos SOIC-8 compatibles: el **XL7015** (XLSEMI, Vin 5–80 V, Iout 0.8 A, 150 kHz fija) y el **MP9486/MP9486A** (MPS, Vin 4.5–100 V, 1 A pico, hasta 1 MHz). En una placa de USD 8 el **XL7015 es con diferencia el más probable** —es el IC *de facto* en módulos chinos de rango extendido— y el hecho de que el inductor marcado "470" sea de **47 µH** es coherente con su frecuencia de 150 kHz; un MP9486 a 1 MHz pediría 22–33 µH.

La topología es **buck no síncrona con diodo Schottky SS34** (40 V / 3 A). Aquí aparece una vulnerabilidad real del diseño chino: con entrada continua de hasta 60 V y transitorios de red, los 40 V de tensión inversa del SS34 tienen margen **insuficiente**; la práctica correcta sería SS54 (40 V/5 A) o SB5100 (100 V/5 A). Un condensador electrolítico grande a la entrada filtra el ripple y un AMS1117-3.3 en SOT-223 baja de 5 V a 3.3 V para el C6. La eficiencia del bloque completo es mediocre: buck ~80–85% a carga media, el LDO fija un techo de **3.3/5 = 66%** por ratio; el total de 12 V → 3.3 V queda en **55–58%**. Para un dispositivo always-on vale la pena sustituir el AMS1117 por un ME6211/SPX3819 (mejor PSRR, menor Iq) o reemplazar toda la cadena por un buck síncrono directo a 3.3 V.

### 1.4 Front-end USB-C y reset

El USB-C lleva los dos diferenciales al USB-JTAG del C6 (GPIO12/13), alimenta el rail de 5 V y cuenta con TVS de protección ESD. No se observan resistencias CC para negociación USB-PD, por lo que con cables USB-C a USB-C sin negociación puede no haber enumeración: es una limitación conocida de placas chinas que ahorran los dos pull-downs de 5.1 kΩ en CC1/CC2. El **botón BOOT pulsa GPIO9 a GND**; el RESET es implícito por el reset USB. No hay par Q1/Q2 de auto-reset DTR/RTS porque el controlador nativo del C6 lo hace en software.

### 1.5 Headers JP1/JP2/JP3 y GPIOs asignados

La inferencia a partir del manual de usuario y confirmación comunitaria (Home Assistant forum, usuario "leopold", nov-2025) da el siguiente mapa funcional:

- **Relé K1**: GPIO19, activo alto, accionado por transistor NPN tipo SS8050 con resistor base 1 kΩ y diodo flyback 1N4148.
- **LED de estado**: GPIO2.
- **Botón BOOT/programable**: GPIO9 (pull-up interno).
- **UART0 TX/RX (header Burn)**: GPIO16/GPIO17.
- **USB D+/D−**: GPIO13/GPIO12.
- **JP1 (lateral izquierdo)**: expone GPIO0, 1, 2, 3, 4 (MTMS), 5 (MTDI), 6 (MTCK), 7 (MTDO), 15, más 3V3 y GND.
- **JP2 (lateral derecho)**: GPIO8, 10, 11, 18, 20, 21, 22, 23 y 5 V.
- **JP3**: cabecera de 4 pines GND/RX/TX/5V como fallback de flasheo.

Antes de conectar periféricos críticos conviene **verificar con multímetro** cada pin; no existe esquemático oficial publicado del fabricante SZHJW para esta variante C6.

## 2. Adaptación para iluminación y aires acondicionados 110 V AC

El diseño residencial real no consiste en "enchufar la placa a una fuente y conectar la carga al NO/COM". Requiere re-arquitectura en tres dimensiones: **capacidad de conmutación, alimentación aislada y protecciones frente a red**.

### 2.1 Dimensionado honesto del relé frente a la carga objetivo

**Iluminación residencial LED**: la regla práctica es que el SRD-05 soporta hasta ~2–3 bombillas LED de buena calidad (con NTC en el driver). Más allá de eso —o con luminarias baratas tipo chino-genérico— el welding aparece en menos de 10 000 ciclos. Si el objetivo son circuitos de iluminación típicos de vivienda (5–10 puntos de luz), la solución correcta es **SSR zero-crossing** (Omron G3MB-202P para <2 A, o G3NE-210T para circuito completo) que cierra exactamente en el cruce por cero de la senoidal, anulando el pico capacitivo de los drivers. Alternativa electromecánica: **Hongfa HF115F-I/005-1HS3A** (16 A / 250 VAC, 120 A / 20 ms de inrush rated, ~USD 3) —reemplazo con footprint similar pero caracterización de inrush explícita.

**Aires acondicionados 12 000 BTU**: no hay escenario en el que el SRD-05 sea la decisión correcta para control directo del compresor. Para **inverter** el steady-state ya excede el derating seguro del Songle (≤7 A); para **no-inverter** el LRA destruye los contactos en 100–1000 ciclos. La topología obligatoria es **piloto**: la placa ESP32-C6 energiza con su relé un **contactor Schneider LC1D09 (9 A AC-3) o LC1D12** cuya bobina es 24 VAC o 120 VAC (~70 mA pick-up, ~10 mA hold); el contactor conmuta la carga. El Songle solo ve una carga inductiva muy suave y puede incluso sobrevivir ahí. Costo adicional ~USD 30–50 para el contactor, imprescindible para seguridad y vida útil. La alternativa es un **SSR industrial zero-cross de 20–25 A** tipo Omron G3NE-220T con disipador dimensionado para ~10–15 W (1–1.5 V × I), más caro pero con ventaja silenciosa y sin desgaste mecánico.

### 2.2 Fuente AC/DC externa aislada: la diferencia fundamental con Shelly

Aquí reside la oportunidad del diseño. El Shelly Plus 1 usa un **buck offline no aislado LNK304** que referencia el GND del ESP32 a la fase AC —diseño legal solo porque toda la electrónica va encapsulada en plástico V-0 sin terminales de baja tensión accesibles al usuario. Esto hace que **flashear un Shelly conectado a red queme el USB-UART y potencialmente al usuario**. En un diseño con fuente externa se puede —y se debe— mantener el GND DC **galvánicamente aislado de la red** con 3000 VAC de rigidez dieléctrica, lo que vuelve seguro cualquier interfaz expuesta (USB, GPIO, Ethernet).

Los candidatos recomendados son:

| Módulo | Vout / Iout | Aislamiento | Certificaciones | No-load | Precio | Uso |
|---|---|---|---|---|---|---|
| **Hi-Link HLK-5M05** | 5 V / 1 A / 5 W | 3000 VAC | CE, FCC, UL (algunos lotes) | <0.1 W | ~USD 1.5–3 | **Prototipo y serie corta** |
| **MeanWell IRM-05-05** | 5 V / 1 A / 5 W | 3000 VAC | UL, TÜV, CB, CE, EN55032 B | <0.1 W | ~USD 5–9 | **Producción certificada** |
| HLK-PM01 | 5 V / 0.6 A / 3 W | 3000 VAC | CE, RoHS | <0.1 W | ~USD 1.4–2.5 | No recomendado (poco margen) |
| IRM-05-12 / HLK-PM12 | 12 V | 3000 VAC | UL / CE | <0.1 W | USD 2–9 | Si se usan relés 12 V más robustos |

Con el consumo total calculado —**240 mA pico del C6 en TX Wi-Fi 6 + 71 mA del Songle + 20 mA LEDs = ~0.35 A @ 5 V = 1.75 W pico**— la regla de margen 100% recomienda **≥ 3.5 W**; un **HLK-5M05 de 5 W** queda con 185% de margen, temperatura baja y mejor eficiencia a carga parcial. Para certificación UL seria, el **IRM-05-05** es la elección directa.

### 2.3 Protecciones AC de entrada

El front-end de red mínimo aceptable para 110 VAC residencial es:

- **Fusible 1 A fast-blow 5×20 mm** (Littelfuse 0215 / Schurter OMT) en la fase, con poder de corte 35 A. Para cargas hasta ~5 W en primario queda con margen; 2 A si el consumo puntual supera los 100 mA RMS.
- **MOV 14D471K** (Bourns, 470 V clamping, 4.5 kA surge) entre L y N, **después del fusible** para que el fusible corte si el MOV falla en corto.
- **NTC inrush limiter** tipo Ametherm SCK-052 solo si se añade bulk electrolítico externo; los HLK-5M05 y IRM-05 ya llevan slow-start interno.
- **Filtro EMI**: condensador **X2 de 100–470 nF / 275 VAC** (Kemet R46) entre L y N; un **choke de modo común 10–30 mH** (Würth 744843101). Los módulos HLK/IRM ya incluyen CMC interno, por lo que puede omitirse externamente en producción pequeña si se acepta un margen EMC más ajustado.
- **Fusible térmico 115–130 °C / 2 A** (Panasonic EYP-2BN) como backup no rearmable, exigido por UL para equipos sin supervisión.

En la salida del relé, **snubber RC 100 Ω 1 W + 100 nF X2 630 VAC** en paralelo con los contactos NO/COM para cargas inductivas. La fórmula Panasonic (R ≈ 0.5·Vpk/Isw, C = 0.1–1 µF/A) respalda los valores empíricos. En cargas muy inductivas, combinar con MOV V275LA20A en paralelo con la carga. En la bobina del relé, **1N4148 flyback** con cátodo a +5 V; para tiempos de release más rápidos, zener 12 V en serie con el diodo. Driver recomendado: transistor SS8050 o array **ULN2003A** si se planea crecer a múltiples canales.

### 2.4 Aislamiento en PCB y encapsulado

Por **IEC 62368-1** (sustituto de IEC 60950-1 desde 2020), para 110 VAC, OVC II, pollution degree 2, aislamiento básico, material group IIIa, los mínimos son **clearance 2.0 mm / creepage 2.5 mm**; para **aislamiento reforzado** accesible al usuario sube a **5.5 mm / 6.4 mm**. La recomendación pragmática de diseño es **6 mm de separación mínima** entre toda traza primaria (L, N, bulk, MOV, fusible, primario del módulo) y cualquier traza secundaria, con **slot fresado de 1 mm de ancho** debajo del módulo HLK/IRM, isla de GND DC separada y conexión a chasis metálico —si lo hay— únicamente mediante Y-cap de 2.2 nF 250 VAC. Ancho de pista ≥ 1.5 mm para 1 A en primario.

Encapsulado: **ABS o policarbonato UL 94 V-0** (Hammond 1551, Bopla EUROMAS, Gainta G3). IP20 para interior limpio, **IP54 mínimo** para condiciones residenciales típicas (baños, lavaderos, garajes, instalación mural expuesta a polvo). Separación física interior entre zona AC y DC mediante tabique plástico como segunda barrera; tornillo M4 de tierra de seguridad si el chasis es metálico.

### 2.5 Diagrama de bloques del diseño recomendado

```
──────── ZONA AC (caliente) ────────
L → Fusible 1A → NTC 5Ω → [MOV 14D471K]→ X2 100nF → CMC 10mH →┐
                                                                ├→ PRIMARIO HLK-5M05
N ──────────────────────────────── X2 100nF ────────────────────┘
      Fusible térmico 130°C 2A (backup)      Aislamiento 3 kVAC
────── 6 mm milled slot / creepage ──────
──────── ZONA DC (SELV, flotante) ────────
5V → C100µF → AMS1117-3.3 → ESP32-C6 (GPIO19 → SS8050 → bobina SRD-05 con 1N4148 flyback)
                         → GPIO2 LED, GPIO9 BOOT
──────── CARGA AC ────────
NO ─┬─ Snubber RC 100Ω + 100nF X2 ─┬─ L_carga
    └── MOV opcional en paralelo ──┘
COM → L de red
```

La antena del C6 debe quedar a >10 mm del plano de masa y alejada de la zona AC para no degradar la ganancia.

## 3. Flasheo USB-C frente a header UART: dos filosofías de producto

### 3.1 El USB-C nativo del C6 es superior para desarrollo

El controlador USB-JTAG integrado en silicio del ESP32-C6 implementa CDC-ACM + JTAG adapter sin chip externo. **esptool.py** maneja el handshake RESET+DOWNLOAD vía comandos USB; no hace falta pulsar BOOT ni RESET. Velocidades de 921 600 a 1 500 000 baud flashean 1.5 MB en 10–15 s. Aislamiento eléctrico completo con el PC (GND USB sobre 5 V SELV). La desventaja principal del USB-C en productos chinos es la **implementación incorrecta sin los dos pull-downs de 5.1 kΩ en CC1/CC2**, que causa que algunos cables USB-C a USB-C (especialmente los de MacBook) no negocien y no enumeren el dispositivo; suele solucionarse con cable USB-A a USB-C.

### 3.2 El header UART del Shelly es el único método seguro para Shelly, pero cero aislamiento

El header interno del Shelly Plus 1 es **6 pines de paso 1.27 mm** (GND, 3V3, TX, RX, GPIO0, RESET —no 7 como dice la tarea; la fuente KB de Shelly y RevK lo confirman). Requiere USB-UART externo a 3.3 V TTL, puentear GPIO0 a GND **antes** de alimentar 3V3, flashear y retirar el puente. El punto crítico lo enuncia literalmente RevK: *"THESE CAN BE LIVE"*. **La fuente LNK304 no aislada del Shelly referencia el GND del ESP a la fase de red**; conectar un USB-UART al PC con el Shelly enchufado destruye el USB y puede poner al usuario en peligro. El procedimiento seguro es Shelly desconectado de AC, alimentar 3V3 desde el USB-UART (≥350 mA; muchos FTDI no llegan), y retirar el USB antes de reconectar AC. La alternativa con aislador digital **ADUM1201** (2.5 kV) existe pero es logística de ingeniería avanzada.

### 3.3 Decisión para un producto propio

Para desarrollo la elección es obvia: mantener el USB-C. Para **producción** depende del modelo de negocio. Si se quiere producto cerrado y tamper-proof, la ruta es **omitir el USB-C del PCB, dejar únicamente un header UART interno de 6 pines accesible solo abriendo el encapsulado**, quemar eFuses de Secure Boot v2 + Flash Encryption y firmar OTA con clave privada. Esto ahorra ~USD 0.50 de BOM (conector USB-C + TVS), reduce la superficie de ataque y es exactamente lo que hace Shelly Gen3/Gen4 —con la consecuencia conocida de que **varios Shelly Gen3 bloquean el OTA a Tasmota** por su bootloader firmado. Si el modelo es nicho maker / home automation abierta, se deja el USB-C accesible y se permite al usuario flashear ESPHome o Tasmota, siguiendo la estrategia de Shelly Gen2. Para un primer producto comercial en Colombia, una topología intermedia —**USB-C tapado bajo etiqueta removible "garantía"**— funciona tanto para producción como para depuración de devoluciones.

## 4. Shelly Plus 1 frente a la placa genérica ESP32-C6

### 4.1 Tabla comparativa multidimensional

| Aspecto | Shelly Plus 1 | Placa ESP32-C6_Relay_X1 |
|---|---|---|
| SoC | ESP32-U4WDH (Xtensa LX6, 160 MHz single o 240 MHz dual según PCB) | ESP32-C6 (RISC-V HP 160 MHz + LP 20 MHz) |
| Wi-Fi | 802.11 b/g/n (Wi-Fi 4) | 802.11 b/g/n/**ax** (Wi-Fi 6, TWT, OFDMA) |
| Bluetooth | BLE 4.2 | BLE 5 + Direction Finding |
| 802.15.4 | No | Zigbee 3.0 / Thread 1.3 / Matter nativo |
| Seguridad HW | Secure Boot v1, Flash Encryption | Secure Boot v2 RSA-3072, Flash Enc. XTS-AES, TEE |
| Fuente | Offline no aislada LNK304 110–240 VAC + 24–48 VDC + 12 VDC | Externa: 7–60 VDC buck XL7015 + AMS1117, o 5 V USB |
| Aislamiento lógica↔red | Solo el relé; resto del PCB al potencial de red | Completo si la fuente externa es aislada |
| Relé | Panasonic AJVN5241F SPST-NO 16 A, AgSnO₂, 2500 Vrms, UL/CSA/VDE | Songle SRD-05 SPDT 10 A, AgCdO, 1500 Vrms, sin certificación del componente |
| Entrada switch externo | Sí, terminal SW nativo (GPIO4) | No nativa |
| Protección térmica | NTC interno sobre GPIO32 | Ninguna |
| Protecciones red | Fusible + MOV + filtro LC | Ninguna en la placa |
| Flasheo | Header UART 6-pin 1.27 mm (riesgo eléctrico alto) | USB-C con USB-JTAG nativo (plug-and-play, aislado) |
| Certificaciones | CE, UKCA, RED, LVD, EMC, RoHS, PSTI UK; UL en variante US | Ninguna |
| Dimensiones | 37×42×16 mm, 26 g, cabe tras interruptor | ~75×50×20 mm, no cabe en caja mural |
| Firmware | Shelly Gen2 cerrado (RPC, MQTT, HA nativo) | Libre: ESPHome 2025.6+, Tasmota 13.4+, ESP-Matter |
| Matter | Plus 1 no (Gen3/4 sí) | **Wi-Fi y Thread nativos** |
| Precio retail abril 2026 | ~USD 20–25 (Plus 1 descontinuado) | ~USD 7–11 |

### 4.2 Cuándo Shelly Plus 1 gana

Para **iluminación LED residencial** (1–16 A, 110–240 VAC) el Shelly Plus 1 es la decisión correcta en vivienda existente: su relé Panasonic caracterizado para TV-8, sus protecciones integradas, su NTC térmico, sus 37×42×16 mm detrás del interruptor y su entrada SW nativa para mantener el interruptor físico funcionando son imposibles de replicar por BOM. Su certificación CE/UKCA/RED (y UL en la variante US) cubre responsabilidad legal. La integración Home Assistant nativa no requiere compilar nada. A un precio de USD 20–25, **el coste marginal de fabricarse uno propio sin certificar ya es más caro** una vez se incluye el encapsulado y los componentes de potencia equivalentes.

### 4.3 Cuándo la placa C6 gana (o justifica un diseño propio basado en ella)

La ventaja real del C6 no es el precio —cuando se le añade fuente aislada, relé Hongfa, MOV, fusible, filtro EMI y caja V-0, el BOM iguala o supera al Shelly— sino **features que el Plus 1 no ofrece**:

- **Matter over Thread**: hub casero compatible con Apple Home, Google Home, SmartThings, Alexa sin depender de Wi-Fi. Un Thread Border Router ya instalado en el hogar (HomePod mini 2ª gen, Nest Hub 2ª gen, eero) descubre el dispositivo al momento.
- **Wi-Fi 6 con TWT**: consumo reducido en redes domésticas saturadas (100+ dispositivos), latencia menor.
- **Zigbee 3.0 coordinator** si se apunta a hub propietario.
- **LP-core para sensores always-on** con consumo ultra-bajo.
- **Cargas DC**: sistemas solares 48 V, automotriz 12/24 V, industrial —donde Shelly no tiene producto equivalente en ese rango de tensión.
- **Secure Boot v2 + TEE** si se apunta a producto comercial con firma y anti-rollback.

### 4.4 Iluminación y AC en el caso del usuario

Sintetizando las consideraciones anteriores para Bogotá con tensión nominal 110 V y cargas reales: para **iluminación LED estándar** (hasta ~5 A por circuito) el Shelly Plus 1 resuelve el problema directamente. Para **AC mini-split 12 000 BTU inverter** el Shelly se queda en el límite (aceptable con snubber RC, recomendable contactor externo); **la placa C6 con Songle no debe usarse directamente en ningún caso**. Para **AC no-inverter o AC >12 000 BTU o calefactores resistivos >10 A continuos**, **ambas opciones requieren contactor externo obligatoriamente**, y en ese escenario la ventaja de Shelly se diluye porque lo que manda la carga es el contactor Schneider LC1D09/LC1D12, no el relé interno. Ahí sí tiene sentido una placa C6 con Matter + contactor externo + snubber, porque el diferencial de UX (HomeKit/Matter directo) justifica el desarrollo.

### 4.5 Economía del desarrollo propio

El BOM unitario en volumen 500 unidades de un rediseño del PCB (ESP32-C6-MINI-1 + Hongfa HF115F-I + HLK-5M05 + MOV + fusible + NTC + snubber + PCB 4 capas + ensamblaje SMT + caja V-0) cae en **USD 10–15**. La certificación **RETIE esquema 5 por familia en CIDET o ICONTEC** cuesta entre **COP 15–40 millones (~USD 4–10 k)** más unas COP 5–10 M de pre-compliance EMC; FCC SDoC añade USD 5–12 k si se exporta a EE. UU. y UL listing USD 15–30 k. Con Shelly Plus 1 a ~USD 15 mayorista, el **punto de equilibrio está en 1 000–3 000 unidades** si se apunta al mismo caso de uso. Por debajo de eso, **comprar Shelly e integrar** es el negocio correcto; por encima, y especialmente si el diferencial es Matter-over-Thread o hub Zigbee/Thread, el diseño propio sobre C6 se paga.

## 5. Recomendación final para desarrollar un producto residencial en Colombia

La ruta de ingeniería y regulatoria recomendada es una síntesis directa de lo anterior.

En **hardware**, rediseñar la placa manteniendo el ESP32-C6-WROOM-1 (módulo pre-certificado FCC/CE/IC por Espressif —esto ahorra 2–3 meses y USD 8–15 k de ensayos de radio), **sustituir el Songle por un Hongfa HF115F-I/005-1HS3A** con inrush de 120 A / 20 ms caracterizado, o —para AC y cargas >10 A— implementar **topología piloto con contactor Schneider LC1D09** externo. Añadir un **front-end AC con fusible 1 A, MOV 14D471K, filtro EMI X2 + CMC, fusible térmico 130 °C y módulo aislado HLK-5M05 (prototipo) o IRM-05-05 (producción certificada)**, todo con 6 mm de creepage y slot fresado bajo la fuente. Incluir **snubber RC 100 Ω + 100 nF X2 630 VAC** en paralelo con los contactos del relé de salida, **diodo flyback 1N4148 en la bobina**, **NTC 10 kΩ B=3350** en GPIO ADC para protección térmica (réplica del diseño Shelly), y **entrada SW externa opto-aislada** para mantener el interruptor físico en la pared funcionando en paralelo al control Wi-Fi. Encapsulado **ABS/PC UL 94 V-0 IP54**.

En **firmware**, **ESPHome 2025.6.0 o superior** sobre framework ESP-IDF (Arduino no soportado en C6 dentro de ESPHome) es el camino de menor fricción para integrar con Home Assistant vía API nativa. **Tasmota 13.4+** es la alternativa con binarios `tasmota32c6-*` ya maduros. Para Matter —el diferencial clave sobre Shelly— usar **ESP-Matter SDK 1.4+** de Espressif, decidiendo entre Matter-over-Wi-Fi (más simple, commissioning BLE) o Matter-over-Thread (requiere Thread Border Router en la red del usuario); ambos no pueden operar simultáneamente sobre el C6 por limitación RF. En producción, **quemar Secure Boot v2 + Flash Encryption** antes de salir de fábrica y mantener la clave privada en HSM.

En **validación**, antes de lanzar someter prototipos a **10 000 ciclos de encendido-apagado** con carga real (6 bombillas LED de tres marcas distintas + un mini-split 12 000 BTU con cable testigo en NO/COM) midiendo resistencia de contacto al inicio, a 1 000, a 5 000 y a 10 000 ciclos; cualquier degradación >20% es fracaso de diseño del relé escogido. Medir EMC pre-compliance conducido 150 kHz–30 MHz en LISN casero antes del laboratorio formal.

En **regulación**, iniciar **RETIE Resolución 40117/2024 esquema 5 "marca continua"** con **CIDET** o **ICONTEC** como organismos acreditados ONAC —la certificación de Shelly internacional **no sustituye** RETIE, hay que emitir certificado local. El producto cae inequívocamente bajo el alcance de RETIE como "interruptor electrónico" y se evalúa contra **NTC 1337 / IEC 60669-2-1 / IEC 60730-1**. Presupuesto realista Colombia-only: **COP 35–85 millones** entre ensayos, certificación, inscripción SIC y asesoría legal. **CRC no aplica** para Wi-Fi/BLE (Resolución CRC 4507/2014). **RETILAP no aplica** salvo que se integre con luminaria. Comercializar sin certificar expone a **multas hasta 2 000 SMLMV (~COP 2 848 millones en 2026)**, responsabilidad objetiva por producto defectuoso bajo Ley 1480/2011 con subrogación por aseguradoras, y posible responsabilidad penal por homicidio/lesiones culposos en caso de incendio —inaceptable para un producto que conmuta red en el hogar. Para exportación, el módulo Espressif pre-certificado permite declarar **FCC Part 15 B SDoC + CE self-declaration (LVD + EMC + RED + RoHS)** sin organismo notificado; UL Listing es voluntario pero crítico para ventas residenciales serias en EE. UU.

### Conclusión

La ESP32-C6_Relay_X1 es **exactamente el vehículo correcto para aprender y prototipar** un producto residencial Matter-ready sobre silicio 2024–2026, pero **no es el producto final**. Su SoC es superior al del Shelly Plus 1 en cinco ejes técnicos simultáneos —Wi-Fi 6, BLE 5 DF, 802.15.4, LP-core, Secure Boot v2— y su arquitectura de fuente DC externa permite algo que Shelly no puede hacer sin romper costos: **aislamiento galvánico total del lado lógico**. Pero su relé Songle SRD-05VDC-SL-C es un componente de gama de iniciación sin rating de inrush, su cadena buck+LDO es ineficiente para always-on, y carece por completo de protecciones AC, certificaciones y el cumplimiento mecánico para caja mural. El camino rentable para el caso de uso del usuario —iluminación y aires acondicionados residenciales en Bogotá— es reconocer que **el valor no está en ahorrar USD 3 del relé ni USD 2 de la fuente, sino en el firmware Matter que Shelly no permite y en el modelo de negocio que solo un producto propio habilita**. Diseñar el hardware con el rigor de Shelly, pero usar el C6 para vender una feature que Shelly no tiene, es la única ecuación que cierra.