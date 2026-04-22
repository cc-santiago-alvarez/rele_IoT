# Módulo 6: Interfaz USB-C de Flasheo y Programación — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-20
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 6 es la **puerta de entrada al microcontrolador**: un conector USB-C de 16 pines con la electrónica mínima necesaria para poder flashear, depurar y obtener logs seriales del ESP32-C3 sin abrir la carcasa ni conectar programadores externos. Como el lado DC del producto está **galvánicamente aislado** de la red AC (aislamiento 3000 VAC provisto por el Módulo 2, HLK-5M05), un técnico puede conectar un cable USB con el equipo energizado desde la línea sin riesgo de shock ni daño al PC anfitrión.

Sus funciones son:

1. **Proveer 5 V DC** desde VBUS del cable USB hacia el Módulo 7 (OR-diode selector AC/USB), permitiendo programar la placa **sin necesidad de energizarla por la red AC**. Esto es crítico en fábrica (flasheo masivo) y en laboratorio (bring-up, ingeniería).
2. **Enrutar el par diferencial D+/D−** desde el conector USB-C hacia los pines **GPIO18 (USB D−)** y **GPIO19 (USB D+)** del ESP32-C3, que están *hardwired* al controlador USB Serial/JTAG integrado en silicio (no es USB software ni bit-banging).
3. **Proteger contra ESD** (IEC 61000-4-2 nivel 4: ±8 kV contacto / ±15 kV aire) con un TVS array **USBLC6-2SC6** ubicado físicamente **cerca del conector**, antes de cualquier traza larga.
4. **Limitar la corriente de VBUS** a 500 mA mediante un fusible resettable PTC, evitando que un cortocircuito en la placa dañe al PC anfitrión o al cable USB.
5. **Cumplir el requisito USB-C de pull-downs CC1/CC2 de 5.1 kΩ**, sin los cuales los cables USB-C a USB-C modernos no enumeran el dispositivo (bug documentado en el diseño chino ESP32-C6_Relay_X1 — ver [Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md](Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md)).
6. **Habilitar el auto-reset en silicio** del ESP32-C3 — `esptool.py` envía comandos USB que el controlador USB Serial/JTAG integrado interpreta para entrar en download mode, **sin necesidad** del circuito clásico de dos transistores NPN (DTR/RTS → EN/GPIO0) que usaba el ESP32 original.

**Entradas del módulo:**
- `VBUS` — 5 V DC provistos por el PC anfitrión a través del cable USB (nominal 5.0 V, rango 4.75–5.25 V USB 2.0).
- `D+ / D−` — par diferencial USB 2.0 Full-Speed (12 Mbps) hacia el controlador USB Serial/JTAG del ESP32-C3.
- `GND` — referencia común con el PC anfitrión; unida al plano **GND_DC** del secundario aislado.

**Salidas del módulo:**
- `+5V_USB` — rail filtrado de 5 V hacia el **Módulo 7** (OR-diode selector AC/USB) después del PTC y del filtro VBUS.
- `USB_DN` / `USB_DP` — nets diferenciales hacia **GPIO18 / GPIO19** del **Módulo 5** (ESP32-C3-MINI-1-N4), después del USBLC6 y de los resistores serie de 22 Ω. **Estos son los nets D−/D+ del sistema** (nomenclatura coherente con [Modulo_5_ESP32_C3_Core.md §8](Modulo_5_ESP32_C3_Core.md)). Ver §3 y §4 para el segmento intermedio `USB_DN_CONN` / `USB_DP_CONN` entre el conector y el USBLC6.
- `GND_DC` — plano común compartido con M2/M4/M5/M7/M8/M9.

> **Nota sobre el chip:** aunque el proyecto lleva "C6" en su nombre histórico, el diseño actual usa **ESP32-C3-MINI-1-N4** (ver [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) §2.2). En el ESP32-C3 los pines USB están *hardwired* a **GPIO18/GPIO19**. En un hipotético rediseño con ESP32-C6 los pines serían GPIO12/GPIO13, y este documento debería actualizarse en consecuencia.

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Anatomía del Conector USB-C de 16 Pines

El conector USB-C tiene **24 pines eléctricos** en la especificación completa (12 por fila, reversible), pero el encapsulado **mid-mount SMD de 16 pines** que usa este diseño sólo expone los pines necesarios para USB 2.0 (VBUS, GND, D+/D−, CC1, CC2) más el SHELL metálico. Los pines SBU1/SBU2 y los D+/D− "alternos" de la fila B se dejan **sin conectar internamente** — es una variante común y económica del conector.

Distribución física (fila A y fila B, vistas desde el lado del inserto):

```
     A1  A2  A3  A4  A5  A6  A7  A8  A9  A10 A11 A12
    GND  -   -  VBUS CC1 D+  D−  -   -   -   -  GND
     ────────────────────────────────────────────
     B12 B11 B10 B9  B8  B7  B6  B5  B4  B3  B2  B1
    GND  -   -   -   -   D−  D+  CC2 VBUS -   -  GND
```

**Por qué pines duplicados:** el conector USB-C es **reversible** (se puede insertar en cualquier orientación). Los pines VBUS, GND, D+ y D− están duplicados en ambas filas para que, independientemente de la orientación, siempre haya conexión eléctrica. En este diseño unimos `A6+B6` (D+) y `A7+B7` (D−) internamente — es la práctica estándar en dispositivos USB 2.0 Full-Speed que no implementan los modos alternos (DisplayPort, Thunderbolt, etc.).

### 2.2 Por qué R_CC1 y R_CC2 de 5.1 kΩ son Obligatorios

En USB-C, los pines **CC1 y CC2** son los canales de configuración (*Configuration Channels*). Su función es negociar la orientación del cable, el rol (host vs device) y el perfil de potencia. Un dispositivo USB-C funcional como **device** (lo que somos: esclavo que recibe energía) **debe** presentar un pull-down de **5.1 kΩ ± 20 %** (típicamente ±1 %) desde cada CC a GND. Este pull-down se llama **Rd** en la especificación USB-IF Type-C Rev 2.1 §4.5.

**¿Qué pasa si se omiten?**

| Cable utilizado | Con R_CC1/R_CC2 (5.1 kΩ) | Sin R_CC1/R_CC2 |
|---|---|---|
| USB-A (host PC) → USB-C (device) | Funciona | Funciona (el cable tiene Rp interno) |
| USB-C → USB-C (host moderno como MacBook, Pixel) | Funciona | **No enumera** — host ve CC flotante, no detecta device |
| USB-C → USB-C con PD (cable activo) | Funciona | **No enumera** + potencial daño al cable |

**Evidencia del campo:** el diseño chino **ESP32-C6_Relay_X1** omitió estos dos resistores para ahorrar ~$0.01 por placa. Resultado: la placa sólo se puede flashear con cables USB-A→C (los viejos), y los usuarios que usan cables modernos USB-C→C reportan que el PC "no detecta nada". Este bug está documentado en nuestro propio [análisis de ingeniería inversa](Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md) y fue una de las razones para diferenciarnos con un diseño profesionalmente compliant.

**Diseño:** R_CC1 y R_CC2 son resistores de precisión **5.1 kΩ ±1 %** (0402), uno por cada CC. Nunca se comparten con un solo resistor común — cada CC debe tener su propio Rd porque el host usa el voltaje diferencial entre CC1 y CC2 para detectar la orientación del cable.

### 2.3 Protección ESD con USBLC6-2SC6

Los pines D+/D− del ESP32-C3 están conectados directamente al controlador USB Serial/JTAG en silicio. Una descarga ESD de ±8 kV (lo que ocurre rutinariamente al tocar un dispositivo con un cuerpo cargado) puede **dañar permanentemente** estas entradas. La especificación UL/CE EMC y el IEC 61000-4-2 exigen que cualquier puerto externo sobreviva a **nivel 4: ±8 kV contacto / ±15 kV aire**.

El **USBLC6-2SC6** de STMicroelectronics es un **TVS array** de 4 líneas (2 I/O pairs) en SOT-23-6 diseñado específicamente para USB 1.1/2.0. Su topología interna — confirmada por la huella EasyEDA del LCSC C7519 — está compuesta de **4 diodos rectificadores de steering + 1 diodo Zener central**:

```
                           +3.3V (desde M4)
                                │
                         Pin 5 (VBUS rail interno)
                                │
                   ┌────────────┼────────────┐
                   │            │            │
                 ─▷│─         ─▷│─         ─▷│─ ──┐   (diodos steering al positivo)
              I/O1 │       Zener│        I/O2│    │
        (pin 1↔6)  │            ▼            │(pin 3↔4)
                   │          [TVS]           │
                   │            ▼            │
                 ─◁│─         ─◁│─         ─◁│─ ──┤   (diodos steering al negativo)
                   │            │            │
                   └────────────┼────────────┘
                                │
                          Pin 2 (GND rail interno)
                                │
                              GND_DC
```

**Cómo funciona el clamp:**
- **Pulso ESD positivo** en I/O1 o I/O2 → el diodo steering superior conduce y lleva la sobretensión al rail VBUS interno → el Zener central se abre cuando VBUS supera el umbral (~6 V) y shunta la energía a GND.
- **Pulso ESD negativo** → el diodo steering inferior conduce y lleva la sobretensión directamente a GND.

El clamp ocurre en **< 1 ns**, mucho antes de que el transitorio llegue al ESP32-C3. La capacitancia agregada a D+/D− es sólo **~3 pF** por línea, compatible con USB Full-Speed (el límite USB-IF es 50 pF total sobre el par).

#### 2.3.1 Por qué alimentamos VCC del USBLC6 a +3.3 V (no +5 V)

El datasheet del USBLC6-2SC6 permite VCC en un rango 3.3–5.5 V. Sin embargo, el nivel de clamp del diodo Zener positivo sigue a VCC + ~0.7 V. Alimentando a **+3.3 V**, el clamp queda en ~4 V — más cercano al rail del ESP32-C3 (3.3 V) y por tanto **más seguro** para las entradas digitales. Si alimentáramos a +5 V, el clamp subiría a ~5.7 V, dejando expuestos los pines del SoC a tensiones ~2.4 V por encima de su VDD durante el pulso ESD.

### 2.4 Resistores Serie de 22 Ω en D+/D−

Los resistores **R_DP y R_DM (22 Ω ±5 %, 0402)** en serie con D+ y D− cumplen dos funciones:

1. **Terminación de impedancia** — la especificación USB 2.0 §7.1.1 requiere que el par diferencial tenga **90 Ω ±15 %** de impedancia característica. Un driver USB (tanto el del PC como el del ESP32-C3) tiene impedancia de salida de ~45 Ω *single-ended*. Los 22 Ω en serie elevan la impedancia de salida efectiva a ~67 Ω — cuando se combina con la impedancia del cable (90 Ω), se minimizan las reflexiones en el par diferencial.
2. **Limitación de corriente en caso de ESD** — si un transitorio supera el clamp del USBLC6, los 22 Ω en serie limitan la corriente que puede llegar al GPIO del SoC a ~V_clamp / 22 Ω ≈ 150 mA durante la fracción de nanosegundo que dura el pulso residual.

**Valor crítico:** 22 Ω. No 10 Ω (reflexiones excesivas), no 33 Ω (eye-diagram marginal a 12 Mbps), no 0 Ω (reflexiones severas). 22 Ω es el valor de referencia de Espressif en todos sus diseños oficiales y en el **ESP32-C3 Hardware Design Guidelines v1.3** §"USB Serial/JTAG".

### 2.5 Fusible Resettable PTC (500 mA, V_max 24 V)

El **PTC1** es un fusible *polymer positive temperature coefficient* — Jinrui `JK-nSMD050-24` (LCSC C369160) o equivalente. Su comportamiento:

| Parámetro | Valor | Comentario |
|---|---|---|
| I_hold (corriente sostenida sin disparo) | 500 mA | Superior al consumo máximo esperado (250 mA TX WiFi + margen) |
| I_trip (corriente a la que dispara rápido) | 1.1 A | Cortocircuito franco |
| V_max | 24 V | V_hold nominal 6 V — apto para VBUS de 5 V |
| Tiempo de disparo @ 1 A | ~2 s | No explota un fusible de vidrio; enfría y se recupera |
| Resistencia en conducción | ~0.25 Ω típ. | Caída < 125 mV a 500 mA → despreciable |

**Ubicación:** en serie con VBUS, **inmediatamente después del conector**, antes del filtro VBUS (C_USB1/C_USB2). Así, si un cortocircuito ocurre en cualquier punto de la placa (incluido el filtro de entrada), el PTC lo ve y actúa antes de que el cable o el PC se vean afectados.

### 2.6 Filtrado VBUS — C_USB1 y C_USB2

VBUS desde un PC anfitrión no es DC puro: tiene ripple del convertidor del PC (típ. 50–100 mVpp a 100–400 kHz), transitorios de conmutación del hub y caídas durante transitorios de carga. El par de capacitores hace doble decoupling:

- **C_USB1** (10 µF / 25 V X5R 0805) — *bulk*. Almacena energía para cubrir transitorios de consumo del ESP32-C3 durante ráfagas TX WiFi (hasta 350 mA pico por 1–2 ms). Su ESR alta (~30 mΩ) no importa porque actúa en baja frecuencia.
- **C_USB2** (100 nF / 16 V X7R 0402) — *high-frequency*. Desacopla el ruido de alta frecuencia (1–100 MHz). Su pequeño tamaño y ESL baja (~0.5 nH) lo hacen efectivo hasta ~50 MHz.

**Orden físico en la placa:** `VBUS → PTC1 → [C_USB1 en paralelo con C_USB2] → salida +5V_USB → M7`. C_USB2 va **más cerca del destino** (entrada de M7) que C_USB1.

### 2.7 Drenaje del SHELL — R_shell y C_shell

El SHELL metálico del conector USB-C es la **jaula de Faraday** del cable. Tiene dos requisitos contradictorios:

1. Debe estar **conectado a GND** para blindar contra EMI (de lo contrario la placa falla CE Clase B).
2. **No debe estar directamente atado a GND** — si el PC anfitrión y la placa tienen GND con potenciales ligeramente distintos (muy común en instalaciones con múltiples fuentes), se forma un *ground loop* que puede generar corrientes de ~100 mA por el SHELL, causando ruido audible y riesgo de daño al cable.

La solución estándar es un **paralelo RC**:

```
  SHELL ──┬──[R_shell: 1 MΩ]── GND   (drenaje DC: descarga estática; bloquea ground loops DC)
          │
          └──[C_shell: 4.7 nF]── GND  (acople AC: los transitorios EMI encuentran camino de baja Z a GND)
```

En DC, R_shell es prácticamente un aislamiento (1 MΩ × típ. 1 mV de diferencia de potencial = 1 nA). En AC (ruido EMI > 10 kHz), C_shell domina con impedancia de ~3.4 kΩ a 10 kHz bajando a 34 Ω a 1 MHz — camino sólido para los transitorios.

**Este circuito es obligatorio** para certificación CE EMC (EN 55032 Clase B) y ayuda a pasar inmunidad (EN 61000-4-6).

### 2.8 Auto-Reset en Silicio del ESP32-C3

Los ESP32 originales (ESP32, ESP32-S2) requerían un circuito de **dos transistores NPN** conectados a DTR/RTS del USB-UART bridge para manejar la secuencia de entrada a download mode:

```
ESP32 clásico (NO aplica aquí):
  DTR ──[T1 NPN]── EN (reset)
  RTS ──[T2 NPN]── GPIO0 (boot mode)
```

El **ESP32-C3** elimina esa complejidad: su **controlador USB Serial/JTAG integrado** expone comandos propietarios vía USB (`RESET`, `BOOTLOADER`) que `esptool.py` envía como paquetes USB Vendor Requests. El controlador interno los traduce a pulsos en las señales internas EN y GPIO9.

**Flujo de flasheo típico (`esptool.py write_flash ...`):**
1. `esptool.py` abre el dispositivo USB que enumera como `10c4:ea60` (ESP32-C3 USB Serial).
2. Envía comando USB Vendor Request `BOOTLOADER` → el SoC internamente baja GPIO9 y pulsa EN.
3. El bootloader de ROM arranca en modo download y responde al handshake de `esptool.py` vía el mismo USB.
4. `esptool.py` escribe el firmware a flash por USB a ~460800 baud efectivos.
5. Envía `RESET` → el SoC se reinicia normalmente ejecutando el firmware nuevo.

**Fallback manual:** si el firmware anterior *crashea* durante boot y nunca llega a inicializar el USB Serial/JTAG, el auto-reset falla. En ese caso:
1. Mantener presionado **SW1** (BOOT, conectado a GPIO9 → GND, ver [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) §2.3).
2. Pulsar **SW2** (RESET, conectado a EN → GND).
3. Soltar SW1. El SoC arranca en modo download esperando comandos por USB.
4. Ejecutar `esptool.py ... --before no_reset --after hard_reset`.

### 2.9 Flujo de Entrega de Potencia

```
  Cable USB (VBUS 5 V del PC)
       │
       ▼
  Pines A4 + B4 del J_USB
       │
       ▼
  PTC1 (500 mA resettable, 1206)  ── límite de corriente
       │
       ▼
  [C_USB1 10 µF 0805] + [C_USB2 100 nF 0402]  ── filtrado
       │
       ▼
  Net +5V_USB  ──────────────────► Módulo 7 (OR-diode selector AC/USB)
                                          │
                                          ▼
                                  Módulo 4 (LDO ME6211 3.3V)
                                          │
                                          ▼
                                  Módulo 5 (ESP32-C3)
```

La conmutación AC ↔ USB la maneja el **Módulo 7** (OR-diode selector), fuera del alcance de este documento — ver sección 8 del [documento maestro](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md).

---

## 3. Asignación de Pines — Símbolo J_USB vs Conector USB-C Físico

El conector USB-C del estándar tiene **24 pines eléctricos** (12 por fila, reversible). La huella **EasyEDA Pro del LCSC C165948** (Korean Hroparts `TYPE-C-31-M-12`, mid-mount SMD con tabs) simplifica ese pinout a **16 pines de símbolo**: 12 pines lógicos (lado izquierdo) + 4 pines EH del shield (lado derecho). El fabricante bondea internamente las parejas de pines redundantes por reversibilidad — el diseñador no necesita unirlos externamente en el esquemático.

### 3.1 Pines del Símbolo (EasyEDA Pro, lo que ves al colocar el componente)

| Pin símbolo | Función | Pines físicos del USB-C que bondea |
|---|---|---|
| `A1B12` | GND | A1 + B12 |
| `A4B9` | VBUS | A4 + B9 |
| `B8` | SBU2 | B8 (sin bondear; NC en USB 2.0) |
| `A5` | CC1 | A5 |
| `B7` | DN2 (D− fila B) | B7 |
| `A6` | DP1 (D+ fila A) | A6 |
| `A7` | DN1 (D− fila A) | A7 |
| `B6` | DP2 (D+ fila B) | B6 |
| `A8` | SBU1 | A8 (sin bondear; NC en USB 2.0) |
| `B5` | CC2 | B5 |
| `B4A9` | VBUS | B4 + A9 |
| `B1A12` | GND | B1 + A12 |
| `EH 1..4` | Shield (housing) | 4 tabs mecánicos del shell metálico |

### 3.2 Pines del conector físico USB-C no cubiertos por este símbolo

Los siguientes pines del estándar USB-C 3.x están **omitidos** en la huella C165948 (no tienen pad en el PCB): **A2, A3, A10, A11** (par SuperSpeed TX+/TX−/RX−/RX+ de la fila A) y **B2, B3, B10, B11** (par SuperSpeed de la fila B). Esto es normal en conectores USB 2.0 Full-Speed de costo moderado — el producto no usa USB 3.x.

### 3.3 Notas clave para el dibujo del esquemático

- **DP1/DP2 y DN1/DN2 NO están bondeados internamente** en el símbolo. Debes dibujar explícitamente la unión (wire) entre `A6`–`B6` y entre `A7`–`B7` para que la reversibilidad del USB-C funcione. Es un error común omitir esta unión y que la placa solo enumere en una orientación del cable.
- **A1B12 y B1A12 son GND** — ambos pines del símbolo deben ir al mismo plano `GND_DC`.
- **A4B9 y B4A9 son VBUS** — ambos pines del símbolo deben ir al mismo net `VBUS_IN` (antes del PTC1).
- **SBU1 (A8) y SBU2 (B8)** — marcar con flag "No Connect (NC)" en ERC; el símbolo los expone pero en USB 2.0 no se usan.
- **EH 1..4** — los 4 pines del shield deben ir al mismo net `SHIELD`, que se cierra a `GND_DC` a través del paralelo R_shell ∥ C_shell (ver §2.7).

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama ASCII Completo del Módulo 6

Numeración y etiquetas de pines tomadas directamente de la **huella oficial EasyEDA Pro del LCSC C165948** (Korean Hroparts `TYPE-C-31-M-12`). El símbolo tiene **12 pines lógicos en el lado izquierdo** (agrupando físicamente los pares GND y VBUS redundantes por reversibilidad del USB-C) más **4 pines EH** (Earth/Housing, el shield mecánico) en el lado derecho:

- `A1B12`, `B1A12` — **GND** (cada uno bondea 2 pines físicos del conector internamente en el fabricante).
- `A4B9`, `B4A9` — **VBUS** (cada uno bondea 2 pines físicos, total 4 pines VBUS para 3 A nominal).
- `A5` — **CC1**, `B5` — **CC2** (pines individuales, no bondeados).
- `A6` — **DP1** (D+ fila A), `B6` — **DP2** (D+ fila B). Pines **separados** en el símbolo — deben unirse externamente por traza a `USB_DP_CONN`.
- `A7` — **DN1** (D− fila A), `B7` — **DN2** (D− fila B). Igual: unir externamente a `USB_DN_CONN`.
- `A8` — **SBU1**, `B8` — **SBU2** (no usados en USB 2.0, dejar NC).
- `EH 1..4` — los 4 pines del shield / housing del conector, todos al mismo net `SHIELD`.

```
    MÓDULO 6 — USB-C FLASHEO (J_USB: TYPE-C-31-M-12, LCSC C165948)
    ═══════════════════════════════════════════════════════════════

                                                              Hacia Módulo 7
                                                              (OR-diode AC/USB)
                                                                      ▲
                                                                      │ +5V_USB
                                                                      │
                                                                      │
  J_USB — símbolo EasyEDA Pro (12 pines izq. + 4 EH der.)             │
  ┌──────────────────────────────────┐                                │
  │                                  │                                │
  │  A1B12 ── GND ───────────────────┼──► GND_DC (plano)              │
  │                                  │                                │
  │  A4B9 ─── VBUS ──────────┐       │                                │
  │                           ├── VBUS_IN ──[PTC1]──┬─► +5V_USB ──────┘
  │  B4A9 ─── VBUS ──────────┘       │   (500 mA)   │
  │                                  │              ├──[C_USB1 10µF]── GND_DC
  │                                  │              │
  │                                  │              └──[C_USB2 100nF]── GND_DC
  │                                  │
  │  B8 ───── SBU2 ── NC             │
  │  A8 ───── SBU1 ── NC             │
  │                                  │
  │  A5 ───── CC1 ──[R_CC1 5.1kΩ]────┼──► GND_DC
  │  B5 ───── CC2 ──[R_CC2 5.1kΩ]────┼──► GND_DC
  │                                  │
  │  A6 ───── DP1 ───┐               │
  │                  ├── USB_DP_CONN ┼──► USBLC6 pin 3 (I/O2)
  │  B6 ───── DP2 ───┘               │       │
  │                                  │       └──► pin 4 ──[R_DP 22Ω]──► GPIO19 (USB_DP → M5)
  │                                  │
  │  A7 ───── DN1 ───┐               │
  │                  ├── USB_DN_CONN ┼──► USBLC6 pin 1 (I/O1)
  │  B7 ───── DN2 ───┘               │       │
  │                                  │       └──► pin 6 ──[R_DM 22Ω]──► GPIO18 (USB_DN → M5)
  │                                  │
  │  B1A12 ── GND ───────────────────┼──► GND_DC (plano)
  │                                  │
  │  EH 1 ──┐                        │
  │  EH 2 ──┤                        │
  │  EH 3 ──┼── SHIELD ──┬──[R_shell 1MΩ]── GND_DC
  │  EH 4 ──┘            │
  │                      └──[C_shell 4.7nF]── GND_DC
  │                                  │
  └──────────────────────────────────┘

         USBLC6-2SC6 (U_TVS) — SOT-23-6 (huella LCSC C7519)
         ══════════════════════════════════════════════════

                       ┌─────────────────────┐
  USB_DN_CONN ► Pin 1 ─┤                     ├─ Pin 6 ► R_DM 22Ω ► GPIO18 (USB_DN)
                       │                     │
       GND_DC ──────── Pin 2 ───  USBLC6  ── Pin 5 ◄──── +3.3V (desde M4)
                       │      (TVS array)    │
  USB_DP_CONN ► Pin 3 ─┤                     ├─ Pin 4 ► R_DP 22Ω ► GPIO19 (USB_DP)
                       └─────────────────────┘

  Nota: pin 1 ↔ pin 6 (ambos I/O1) y pin 3 ↔ pin 4 (ambos I/O2) están
  internamente conectados por el símbolo — una sola señal eléctrica entra
  por un lado y sale por el otro, con el clamp tapando desde el medio.
```

### 4.2 Tabla Pin a Pin Exhaustiva — Símbolo J_USB (huella C165948)

Correspondencia **1:1 con los 16 pines del símbolo EasyEDA Pro** del LCSC C165948. Los pines `A1B12`, `B1A12`, `A4B9` y `B4A9` son agrupaciones bondeadas internamente por el fabricante (conveniencia del símbolo; eléctricamente equivalentes a unir los pines físicos correspondientes). Los 4 pines EH son el shield metálico del conector expuestos en 4 puntos de soldadura mecánica.

| # | Pin símbolo | Pines físicos | Función | Net | Conexión aguas abajo |
|---|---|---|---|---|---|
| 1 | `A1B12` | A1 + B12 | GND | `GND_DC` | Plano GND |
| 2 | `A4B9` | A4 + B9 | VBUS | `VBUS_IN` | Junction con `B4A9` → PTC1 pad 1 |
| 3 | `B8` | B8 | SBU2 | — | **NC** (marcar "No Connect" en ERC) |
| 4 | `A5` | A5 | CC1 | `CC1` | R_CC1 (pad 1) → GND |
| 5 | `B7` | B7 | DN2 (D− fila B) | `USB_DN_CONN` | Junction con `A7` (DN1) → USBLC6 pin 1 |
| 6 | `A6` | A6 | DP1 (D+ fila A) | `USB_DP_CONN` | Junction con `B6` (DP2) → USBLC6 pin 3 |
| 7 | `A7` | A7 | DN1 (D− fila A) | `USB_DN_CONN` | Junction con `B7` (DN2) → USBLC6 pin 1 |
| 8 | `B6` | B6 | DP2 (D+ fila B) | `USB_DP_CONN` | Junction con `A6` (DP1) → USBLC6 pin 3 |
| 9 | `A8` | A8 | SBU1 | — | **NC** (marcar "No Connect" en ERC) |
| 10 | `B5` | B5 | CC2 | `CC2` | R_CC2 (pad 1) → GND |
| 11 | `B4A9` | B4 + A9 | VBUS | `VBUS_IN` | Junction con `A4B9` → PTC1 pad 1 |
| 12 | `B1A12` | B1 + A12 | GND | `GND_DC` | Plano GND |
| 13 | `EH 1` | Shell tab 1 | Shield mecánico | `SHIELD` | R_shell ∥ C_shell → GND_DC |
| 14 | `EH 2` | Shell tab 2 | Shield mecánico | `SHIELD` | R_shell ∥ C_shell → GND_DC |
| 15 | `EH 3` | Shell tab 3 | Shield mecánico | `SHIELD` | R_shell ∥ C_shell → GND_DC |
| 16 | `EH 4` | Shell tab 4 | Shield mecánico | `SHIELD` | R_shell ∥ C_shell → GND_DC |

> **Nota sobre D+ y D−:** en este símbolo los pines `DP1`/`DP2` (y `DN1`/`DN2`) son **pines separados** que EasyEDA Pro NO bondea internamente. El diseñador debe dibujar explícitamente la unión (wire/junction) entre `A6`–`B6` (ambos a `USB_DP_CONN`) y entre `A7`–`B7` (ambos a `USB_DN_CONN`). Si se omite la unión, sólo una fila del conector USB-C conducirá datos y la reversibilidad se pierde.

**Resumen de señales activas que salen del conector (6 nets):**

| Señal | Pines del símbolo | Destino |
|---|---|---|
| `VBUS_IN` | `A4B9`, `B4A9` | PTC1 → filtro → M7 |
| `CC1` | `A5` | R_CC1 → GND |
| `CC2` | `B5` | R_CC2 → GND |
| `USB_DP_CONN` | `A6` + `B6` | USBLC6 pin 3 |
| `USB_DN_CONN` | `A7` + `B7` | USBLC6 pin 1 |
| `SHIELD` | `EH 1..4` | R_shell ∥ C_shell → GND_DC |
| `GND_DC` | `A1B12`, `B1A12` | Plano GND |

**Pines NC (no conectar, aplicar flag "No Connect" en ERC):** `SBU1` (A8), `SBU2` (B8).

### 4.3 Tabla Pin a Pin — USBLC6-2SC6 (U_TVS)

Huella **SOT-23-6** del LCSC C7519. Numeración estándar STMicro (vista desde arriba, pin 1 marcado con el punto).

| Pin | Nombre | Dirección | Net | Conexión |
|---|---|---|---|---|
| 1 | I/O1 | Bidireccional | `USB_DN_CONN` | Desde junction A7+B7 del conector USB-C |
| 2 | GND | Power | `GND_DC` | Plano GND |
| 3 | I/O2 | Bidireccional | `USB_DP_CONN` | Desde junction A6+B6 del conector USB-C |
| 4 | I/O2 | Bidireccional | `USB_DP` | Hacia R_DP (22 Ω) → GPIO19 del ESP32-C3 |
| 5 | VBUS | Power input | `+3.3V` | Desde M4 (LDO ME6211) — **no** a VBUS 5 V del USB |
| 6 | I/O1 | Bidireccional | `USB_DN` | Hacia R_DM (22 Ω) → GPIO18 del ESP32-C3 |

**Topología interna del USBLC6:** los pines 1↔6 y 3↔4 están internamente conectados (son las dos salidas de un mismo par de clamps). La señal "entra" por un lado (pin 1 o 3, el lado del conector) y "sale" por el otro (pin 6 o 4, el lado del SoC). El clamp a VBUS (pin 5) protege contra sobrevoltajes positivos y el clamp a GND (pin 2) protege contra sobrevoltajes negativos. Capacitancia añadida por línea: ~3 pF (dato del datasheet ST).

### 4.4 Subcircuitos Detallados

```
(a) Filtrado VBUS                       (b) Pull-downs CC                    (c) Drenaje SHELL
─────────────────────                   ─────────────────────                ─────────────────────

  VBUS_IN                                 CC1                                   SHELL
    │                                      │                                     │
  [PTC1: 500 mA]                         [R_CC1: 5.1 kΩ ±1 %]                  [R_shell: 1 MΩ]──┐
    │                                      │                                     │               │
    ├──► +5V_USB (a M7)                   GND                                   GND              │
    │                                                                                            │
  [C_USB1: 10 µF 0805]                    CC2                                                    │
    │                                      │                                   [C_shell: 4.7 nF]─┤
   GND                                   [R_CC2: 5.1 kΩ ±1 %]                                    │
    │                                      │                                                     │
  [C_USB2: 100 nF 0402]                   GND                                                   GND
    │
   GND
```

```
(d) Junction D+ y D−

  A6 (D+) ──┐
            ├── USB_DP_CONN ────► USBLC6 pin 3
  B6 (D+) ──┘                    │
                                 │  (dentro del USBLC6)
                                 ▼
                              USBLC6 pin 4 ── R_DP (22 Ω) ──► GPIO19 (USB_DP)


  A7 (D−) ──┐
            ├── USB_DN_CONN ───► USBLC6 pin 1
  B7 (D−) ──┘                    │
                                 │  (dentro del USBLC6)
                                 ▼
                              USBLC6 pin 6 ── R_DM (22 Ω) ──► GPIO18 (USB_DN)
```

---

## 5. Reglas de Layout PCB

### 5.1 Reglas Críticas

| # | Regla | Motivo | Criterio de verificación |
|---|---|---|---|
| L1 | Impedancia diferencial **90 Ω ±15 %** en el par D+/D− | USB 2.0 §7.1.1 | Calculadora del stackup JLCPCB 4 capas: traza 0.2 mm, separación 0.15 mm, referencia a GND plane continuo |
| L2 | Longitudes iguales D+ y D−, **skew < 150 mil (3.8 mm)** | Mantener integridad del par diferencial | Regla de `matched length` en EasyEDA Pro |
| L3 | **Ground plane sólido** bajo todo el par D+/D−, sin cortes por trazas ni vías mal ubicadas | Referencia de retorno | Inspección visual + ODB++ en el fabricante |
| L4 | Los resistores R_DP y R_DM se colocan **antes** del USBLC6 sólo si el driver está del lado del SoC. En nuestro caso, el driver ES el ESP32-C3 → los 22 Ω van **después** del USBLC6 pero **antes** del GPIO del SoC | Posición de terminación correcta para eye-diagram limpio | Revisar: conector → USBLC6 → R_DP/R_DM → GPIO |
| L5 | **PTC1 y USBLC6 a ≤ 5 mm del conector USB-C** | Minimizar el área de loop ESD antes del clamp | Regla de placement en EasyEDA Pro |
| L6 | **Vías stitching en el SHELL** (mínimo 4 vías Ø 0.3 mm del pad SHELL al plano GND_DC) para que las corrientes ESD tengan camino de baja inductancia | EMC + ESD | Inspección manual |
| L7 | **Keepout mecánico** del conector: 1.5 mm de margen hacia bordes del PCB y tornillos de la carcasa | El conector soporta fuerzas de inserción/extracción (hasta 10 N) | Modelo 3D del gabinete |
| L8 | Net `VBUS_IN` con ancho ≥ **0.4 mm** (≥ 500 mA @ 1 oz Cu, ΔT < 10 °C) | Capacidad de corriente del PTC sin caída excesiva | Calculadora de ampacidad IPC-2152 |
| L9 | R_CC1 y R_CC2 a ≤ 5 mm de los pines CC del conector | La norma USB-C exige 5.1 kΩ "at the device's USB-C connector" | Placement visual |
| L10 | C_USB2 (100 nF) más cerca que C_USB1 (10 µF) del punto de uso (salida +5V_USB hacia M7) | Cascada de decoupling: HF primero, bulk después | Placement visual |

### 5.2 Checklist de Layout

- [ ] D+ y D− enrutados como **par diferencial** (función *diff pair* de EasyEDA Pro).
- [ ] Impedancia 90 Ω configurada en el stackup (calculadora JLCPCB).
- [ ] Longitudes igualadas (skew < 150 mil).
- [ ] Plano GND continuo bajo todo el par diferencial, desde el conector hasta el ESP32-C3.
- [ ] USBLC6 a < 5 mm del conector USB-C.
- [ ] PTC1 a < 5 mm del conector USB-C, en serie con VBUS antes del filtro.
- [ ] R_DP y R_DM colocados entre el USBLC6 y el ESP32-C3 (no entre el conector y el USBLC6).
- [ ] R_CC1 y R_CC2 a < 5 mm de los pines CC1 (A5) y CC2 (B5).
- [ ] C_USB2 (100 nF) físicamente más cerca de la salida +5V_USB hacia M7 que C_USB1.
- [ ] Al menos 4 vías de Ø 0.3 mm en los pads SHELL conectando al plano GND_DC.
- [ ] VBUS trace ancho ≥ 0.4 mm.
- [ ] Tabs de anclaje mecánico (TAB1, TAB2) soldados al plano GND con footprint reforzado.
- [ ] Sin trazas de señal pasando bajo el conector USB-C (sólo GND plane debajo).
- [ ] Conector alineado al borde del PCB con 1.5 mm de margen mecánico.
- [ ] Orientación del USBLC6 verificada contra el datasheet (pin 1 marcado correctamente en la huella).

---

## 6. BOM Completo — Módulo 6 con Códigos LCSC

Todos los componentes están verificados para importación directa desde **LCSC / JLCPCB Standard Parts Library** y son compatibles con **EasyEDA Pro Edition**. Los códigos marcados con ✓ están confirmados en [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv); los marcados con ⚠ son propuestos y deben validarse en LCSC antes de compra.

### 6.1 Componentes Primarios

| # | Ref | Componente | Valor / Specs | Encaps. | LCSC | Fabricante | MPN | Qty | Precio aprox. | CSV |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **J_USB** | Conector USB-C 16-pin mid-mount SMD con tabs | Tipo-C 2.0, reversible, VBUS 3 A nominal | SMD mid-mount | **C165948** | Korean Hroparts Elec | TYPE-C-31-M-12 | 1 | $0.25 | ✓ |
| 2 | **U_TVS** | USB ESD TVS array 4 líneas | IEC 61000-4-2 Nivel 4, 3 pF/línea, clamp ~6 V | SOT-23-6 | **C7519** | STMicroelectronics | USBLC6-2SC6 | 1 | $0.12 | ✓ |
| 3 | **PTC1** | Fusible resettable PPTC | 500 mA I_hold, 1.1 A I_trip, V_max 24 V | 1206 SMD | **C369160** | Jinrui | JK-nSMD050-24 | 1 | $0.06 | ✓ |
| 4 | **R_CC1** | Resistor pull-down CC1 USB-C | 5.1 kΩ ±1 %, 1/16 W | 0402 | **C25905** | UNI-ROYAL | 0402WGF5101TCE | 1 | $0.005 | ✓ |
| 5 | **R_CC2** | Resistor pull-down CC2 USB-C | 5.1 kΩ ±1 %, 1/16 W | 0402 | **C25905** | UNI-ROYAL | 0402WGF5101TCE | 1 | $0.005 | ✓ |
| 6 | **R_DP** | Resistor terminación serie D+ | 22 Ω ±5 %, 1/16 W | 0402 | **C25092** | UNI-ROYAL | 0402WGF220JTCE | 1 | $0.005 | ✓ |
| 7 | **R_DM** | Resistor terminación serie D− | 22 Ω ±5 %, 1/16 W | 0402 | **C25092** | UNI-ROYAL | 0402WGF220JTCE | 1 | $0.005 | ✓ |
| 8 | **C_USB1** | Capacitor MLCC bulk VBUS | 10 µF / 25 V, ±10 %, X5R | 0805 | **C15850** | Samsung Electro-Mechanics | CL21A106KAYNNNE | 1 | $0.02 | ✓ |
| 9 | **C_USB2** | Capacitor MLCC HF decouple VBUS | 100 nF / 16 V, ±10 %, X7R | 0402 | **C14663** | YAGEO | CC0402KRX7R9BB104 | 1 | $0.01 | ✓ |
| 10 | **R_shell** | Resistor drenaje SHELL USB | 1 MΩ ±1 %, 1/16 W | 0402 | **C26083** | UNI-ROYAL | 0402WGF1004TCE | 1 | $0.005 | ⚠ |
| 11 | **C_shell** | Capacitor acople AC SHELL | 4.7 nF / 50 V, ±10 %, X7R | 0402 | **C106208** | YAGEO | CC0402KRX7R9BB472 | 1 | $0.008 | ⚠ |

**Total componentes:** 11 referencias (9 part numbers únicos — R_CC1/R_CC2 comparten, R_DP/R_DM comparten). **Subtotal BOM Módulo 6: ≈ $0.48 USD** por placa (sin ensamblaje).

> **Acción de seguimiento crítica:**
>
> 1. **R_shell y C_shell no están en el CSV maestro** [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv). Agregarlos antes del envío a fabricación (filas nuevas con zona `M6`).
>
> 2. **R_shell — LCSC confirmado:** `C26083` = UNI-ROYAL `0402WGF1004TCE`, 1 MΩ 0402 ±1 %, verificado en lcsc.com (607 K unidades en stock, abril 2026). El código `C25741` propuesto inicialmente era incorrecto (resultó ser 100 kΩ).
>
> 3. **C_shell — LCSC confirmado:** `C106208` = YAGEO `CC0402KRX7R9BB472`, 4.7 nF 50 V X7R 0402 ±10 %, verificado en lcsc.com (1.33 M unidades en stock, abril 2026). El código `C1546` propuesto inicialmente era incorrecto (resultó ser 100 pF C0G).

### 6.2 Componentes Alternativos / Segundas Fuentes

Todos los primarios tienen LCSC vigente a abril 2026. Se listan alternativas verificadas como respaldo ante out-of-stock temporales.

#### 6.2.1 Alternativas para J_USB (conector USB-C 16-pin)

| Opción | LCSC | MPN | Fabricante | Diferencia vs primario | Drop-in? |
|---|---|---|---|---|---|
| **Primario** | **C165948** | TYPE-C-31-M-12 | Korean Hroparts | — | — |
| Alt. 1 | **C2765186** | USB4085-GF-A | GCT (Gold Connector Technology) | Preferencia del diseño de referencia; mejor calidad mecánica | **Sí** (footprint 16-pin compatible — verificar 3D) |
| Alt. 2 | **C320463** | DX07S016JA1R1500 | JAE | Industrial-grade, mayor costo | **Sí** (mismo estándar 16-pin mid-mount) |

#### 6.2.2 Alternativas para U_TVS (USBLC6-2SC6 ESD array)

| Opción | LCSC | MPN | Fabricante | Diferencia vs primario | Drop-in? |
|---|---|---|---|---|---|
| **Primario** | **C7519** | USBLC6-2SC6 | STMicroelectronics | — | — |
| Alt. 1 | **C7484** | IP4220CZ6 | Nexperia (ex NXP) | Equivalente funcional, mismo SOT-23-6 | **Sí** — pinout idéntico |
| Alt. 2 | **C84500** | ESDA6V1-5SC6 | STMicroelectronics | TVS diferencial 6 V, capacitancia ligeramente menor (1.5 pF) | **Sí** — pinout compatible |

#### 6.2.3 Alternativas para PTC1 (fusible resettable 500 mA / 1206)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C369160** | JK-nSMD050-24 | Jinrui | ✓ Verificado abril 2026 — 73 K unidades en stock, I_hold 500 mA / V_max 24 V |
| Alt. 1 | **TBD** | nanoSMDC050F-2 | Littelfuse | 1206, I_hold 500 mA, V_max 6 V — buscar MPN en lcsc.com |
| Alt. 2 | **TBD** | MF-MSMF050-2 | Bourns | ⚠ Footprint 1812 (no 1206); requiere ajustar huella |
| ❌ NO usar | C369153 | JK-nSMD005 | Jinrui | **50 mA** (10× demasiado pequeño); dispara en cualquier ráfaga TX WiFi |

#### 6.2.4 Alternativas para R_CC1 / R_CC2 (5.1 kΩ / 0402 / ±1 %)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C25905** | 0402WGF5101TCE | UNI-ROYAL | ±1 %, 1/16 W |
| Alt. 1 | **C25915** | RC0402FR-075K1L | YAGEO | ±1 %, mejor marca |
| Alt. 2 | **C98732** | ERJ-2RKF5101X | Panasonic | ±1 %, 1/16 W, stock masivo |

#### 6.2.5 Alternativas para R_DP / R_DM (22 Ω / 0402 / ±5 %)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C25092** | 0402WGF220JTCE | UNI-ROYAL | ±5 %, 1/16 W |
| Alt. 1 | **C25103** | RC0402JR-0722RL | YAGEO | ±5 %, tolerancia igual, mejor marca |
| Alt. 2 | **C25104** | RC0402FR-0722RL | YAGEO | ±1 % (mejor tolerancia; innecesaria pero aceptable) |

#### 6.2.6 Alternativas para C_USB1 (10 µF / 25 V / 0805 / X5R)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C15850** | CL21A106KAYNNNE | Samsung | X5R 25 V |
| Alt. 1 | **C96446** | GRM21BR61E106KA73L | Murata | X5R 25 V, equivalente funcional |
| Alt. 2 | **C45783** | CL21A106KPFNNNE | Samsung | Misma línea, variante de lote |

#### 6.2.7 Alternativas para C_USB2 (100 nF / 16 V / 0402 / X7R)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C14663** | CC0402KRX7R9BB104 | YAGEO | X7R 16 V |
| Alt. 1 | **C1525** | 0402B104K160CT | Walsin | X7R 16 V, compatible |
| Alt. 2 | **C52923** | CL05B104KO5NNNC | Samsung | X7R 16 V, alternativa de marca |

#### 6.2.8 Alternativas para R_shell (1 MΩ / 0402 / ±1 %)

| Opción | LCSC | MPN | Fabricante | Paquete | Tol. | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C26083** | 0402WGF1004TCE | UNI-ROYAL | 0402 | ±1 % | ✓ Verificado abril 2026 — 607 K unidades en stock |
| Alt. 1 | **C22935** | 0603WAF1004T5E | UNI-ROYAL | 0603 | ±1 % | 1.28 M unidades en stock; usar si se prefiere 0603 o C26083 se agota |
| Alt. 2 | **TBD** | RC0402FR-071ML | YAGEO | 0402 | ±1 % | Código LCSC a verificar por MPN en lcsc.com |
| ❌ NO usar | C25741 | 0402WGF1003TCE | UNI-ROYAL | 0402 | ±1 % | **100 kΩ** — propuesto erróneamente en v1.0 draft; excluir |

> **Nota:** R_shell sólo conduce corrientes µA DC (drenaje estático), así que cualquiera de las dos opciones primario/Alt.1 funciona eléctricamente. El 0402 es preferible por consistencia con el resto del M6.

#### 6.2.9 Alternativas para C_shell (4.7 nF / 50 V / X7R / 0402)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C106208** | CC0402KRX7R9BB472 | YAGEO | ✓ Verificado abril 2026 — 1.33 M unidades en stock |
| Alt. 1 | **TBD** | CL05B472KB5NNNC | Samsung | Buscar por MPN en lcsc.com si C106208 no hay stock |
| Alt. 2 | **TBD** | GRM155R71H472KA01D | Murata | Buscar por MPN en lcsc.com si C106208 no hay stock |
| ❌ NO usar | C1546 | 0402CG101J500NT | FH | **100 pF C0G** — propuesto erróneamente en v1.0 draft; excluir |

---

## 7. Checklist de Verificación del Módulo

Antes de enviar el diseño a fabricación JLCPCB:

### 7.1 Verificación de Esquemático

- [ ] **R_CC1 (5.1 kΩ) presente** entre CC1 (pin A5) y GND.
- [ ] **R_CC2 (5.1 kΩ) presente** entre CC2 (pin B5) y GND.
- [ ] Valor exacto **5.1 kΩ ±1 %** (no 4.7 kΩ ni 5.6 kΩ — la norma USB-C es estricta).
- [ ] **PTC1 en serie con VBUS**, nunca en paralelo ni en el camino de retorno GND.
- [ ] **Filtro VBUS** (C_USB1 10 µF + C_USB2 100 nF) **después** del PTC, no antes.
- [ ] **USBLC6 VCC (pin 5) conectado a +3.3V** (no a VBUS 5 V).
- [ ] **R_DP y R_DM** de 22 Ω ubicados entre el USBLC6 y el ESP32-C3 (no entre el conector y el USBLC6).
- [ ] **Pines A6 + B6 unidos** antes del USBLC6 (junction D+).
- [ ] **Pines A7 + B7 unidos** antes del USBLC6 (junction D−).
- [ ] **Pines A4 + B4 unidos** antes del PTC (junction VBUS).
- [ ] **Pines A1, A12, B1, B12** conectados al plano GND_DC.
- [ ] **SHELL** conectado vía R_shell (1 MΩ) y C_shell (4.7 nF) a GND_DC, **no directamente**.
- [ ] **SBU1, SBU2, SS pairs**: sin conexión (NC) en el esquemático; flag de "No Connect" aplicado para suprimir warnings de ERC.

### 7.2 Verificación de BOM

- [ ] Los 9 LCSC primarios del CSV están en stock en JLCPCB Standard Parts Library.
- [ ] Los 2 LCSC propuestos (R_shell C25741, C_shell C1546) verificados manualmente en lcsc.com — si no hay stock, usar las Alt.1 indicadas en §6.2.8 y §6.2.9.
- [ ] Huellas importadas correctamente desde LCSC en EasyEDA Pro (sin warnings de "footprint mismatch" en el ERC).
- [ ] R_shell y C_shell añadidos al CSV maestro antes de generar la BOM final.

### 7.3 Pruebas Post-Ensamblaje (en laboratorio)

| # | Prueba | Criterio de éxito |
|---|---|---|
| T1 | Continuidad R_CC1 a GND | 5.1 kΩ ±1 % con multímetro |
| T2 | Continuidad R_CC2 a GND | 5.1 kΩ ±1 % con multímetro |
| T3 | Medida de VBUS con cable USB-A→C | 5.0 V ±0.25 V en C_USB1 |
| T4 | Medida de VBUS con cable USB-C→C (Pixel, MacBook) | Dispositivo enumera → 5.0 V ±0.25 V en C_USB1 |
| T5 | Enumeración en PC Linux: `lsusb` | Aparece `10c4:ea60` (ESP32-C3 USB Serial/JTAG) |
| T6 | Flasheo con esptool: `esptool.py -p /dev/ttyACM0 flash_id` | Responde con Chip ID, sin errores |
| T7 | Flasheo de firmware real: `esptool.py write_flash 0x0 firmware.bin` | Exit code 0, velocidad > 400 kbps |
| T8 | Botón BOOT + RESET para fallback manual | Entra en download mode sin USB enumeration previo |
| T9 | ESD test (opcional, con gun IEC 61000-4-2) | ±8 kV contacto en el conector, sin daño ni reset |
| T10 | Corto deliberado en +5V_USB | PTC1 dispara en < 3 s; PC no se daña; rearma al quitar el corto |

---

## 8. Conexión con Otros Módulos

Tabla resumen de las señales que cruzan la frontera del Módulo 6. En el esquemático jerárquico de EasyEDA Pro, estos son los *hierarchical sheet pins* del bloque M6.

| Señal / Net | Dirección desde M6 | Destino | Componente en el camino | Zona |
|---|---|---|---|---|
| `+5V_USB` | Salida | **Módulo 7** (OR-diode selector AC/USB) | PTC1 → C_USB1 ∥ C_USB2 | DC |
| `USB_DP` | Salida bidireccional | **Módulo 5** (ESP32-C3 pin 27, GPIO19) | R_DP (22 Ω) | DC |
| `USB_DN` | Salida bidireccional | **Módulo 5** (ESP32-C3 pin 26, GPIO18) | R_DM (22 Ω) | DC |
| `+3.3V` | Entrada | **Módulo 4** (ME6211 VOUT) | — | DC |
| `GND_DC` | Plano común | M2, M4, M5, M7, M8, M9 | Plano continuo | DC |

**Frontera de aislamiento:** todas las señales del Módulo 6 están en zona **DC** del secundario aislado. Ninguna cruza la barrera galvánica 3000 VAC que separa el primario AC (M1/M2) del secundario. Esto es lo que hace seguro conectar un USB mientras la placa está energizada desde la red.

**Coherencia con Módulo 5:** las señales `USB_DN` y `USB_DP` de este documento corresponden a las señales del mismo nombre en [Modulo_5_ESP32_C3_Core.md §8](Modulo_5_ESP32_C3_Core.md), conectadas a los pines **26 (IO18)** y **27 (IO19)** del módulo ESP32-C3-MINI-1-N4 respectivamente.

---

## 9. Estimación de Consumo del Módulo 6

| Estado | Corriente desde VBUS | Comentario |
|---|---|---|
| **Reposo** (ESP32-C3 en light-sleep) | ~5 mA | Consumo vía el path USB → M7 → M4 → M5 |
| **Idle WiFi asociado** | 15–25 mA | Beacon listening cada DTIM |
| **RX WiFi** | ~80 mA | Recepción sostenida |
| **TX WiFi sostenido** | 240 mA | Raro en operación normal |
| **Pico TX WiFi** | **~350 mA** | Pulsos de 1–2 ms |
| **Flasheo activo (esptool write)** | 150–250 mA | Combinación de USB TX/RX + escritura a flash interna |

**Margen sobre el PTC1 (I_hold 500 mA):** 2× en el peor caso sostenido (240 mA TX WiFi sostenido). Los picos de 350 mA son demasiado cortos para disparar el PTC (tiempo térmico ~100 ms).

**Caída de voltaje en el camino VBUS → +5V_USB:**
- PTC1 en conducción: ~0.25 Ω × 250 mA = 62 mV
- Trazas VBUS (0.4 mm ancho × 20 mm largo × 1 oz Cu): ~10 mΩ × 250 mA = 2.5 mV
- **Total:** ~65 mV → +5V_USB típico de 4.935 V con cable ideal → suficiente para el OR-diode + LDO aguas abajo.

---

## 10. Referencias

- **USB 2.0 Specification Rev 2.0** — USB-IF, 2000. §7.1.1 "Electrical and Mechanical Specifications" (impedancia diferencial 90 Ω).
- **USB Type-C Cable and Connector Specification Rev 2.1** — USB-IF, 2021. §4.5 "Connector Wiring" (Rd = 5.1 kΩ pull-downs en device).
- **USBLC6-2SC6 Datasheet** — STMicroelectronics, DS2033 Rev 11, 2019.
- **ESP32-C3 Technical Reference Manual v1.1** — Espressif Systems, Capítulo "USB Serial/JTAG Controller".
- **ESP32-C3 Hardware Design Guidelines v1.3** — Espressif Systems, §"USB Interface" (resistencias serie 22 Ω de referencia).
- **esptool.py documentation** — https://docs.espressif.com/projects/esptool/en/latest/esp32c3/ — auto-reset en silicio con USB Serial/JTAG.
- **IEC 61000-4-2 Edition 2.0** — ESD immunity test (niveles 1–4).
- **EN 55032:2015** — EMC emisión Clase B para equipos residenciales.
- [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) — documento maestro del proyecto, sección "Módulo 6: USB-C (flasheo y programación)" (líneas 607–668).
- [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) — núcleo ESP32-C3, destino de las señales USB_DN/USB_DP.
- [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) — fuente del rail +3.3V que alimenta VCC del USBLC6.
- [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) — lista maestra de materiales; filas 4, 22–25, 39–40, 46, 52 contienen los 9 componentes primarios verificados del Módulo 6.
- [Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md](Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md) — análisis del bug de los CC resistors omitidos en el diseño chino.

---

**Fin del documento — Módulo 6: USB-C Flasheo y Programación v1.0**
