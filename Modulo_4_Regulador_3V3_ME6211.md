# Módulo 4: Regulador LDO 3.3 V (ME6211C33M5G-N) — Documento Técnico

## Proyecto: Smart Relay ESP32-C3 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-17
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 4 es el **regulador lineal de baja caída (LDO)** que convierte el rail `+5V_COMBINED` (salida del Módulo 7, selector OR-diode AC/USB) en el rail `+3.3V` estable que alimenta toda la electrónica digital del Smart Relay.

Sus funciones son:

1. **Regular** el rail +3.3 V con tolerancia ≤ ±2% bajo toda condición de carga (desde 15 mA de idle hasta 350 mA pico durante transmisión WiFi).
2. **Atenuar** el ruido de conmutación del convertidor flyback del HLK-5M05 (65–100 kHz) y del rizado de la bobina del relé (50/60 Hz) antes de que llegue al microcontrolador y al ADC.
3. **Tolerar** caídas transitorias del rail +5 V durante los bursts de transmisión WiFi (cuando el HLK-5M05 puede deprimirse hasta ~4.5 V) gracias a su **dropout voltage de sólo 100 mV**.
4. **Minimizar** el consumo quiescente (40 µA) para permitir que el ESP32-C3 aproveche su modo deep-sleep sin que el LDO sea la carga dominante.
5. **Proporcionar** una referencia limpia (ruido ≤ 36 µVrms) para la medición del ADC del NTC térmico del Módulo 9.

Las **entradas** del módulo (`+5V_COMBINED`) provienen del **Módulo 7** (combinador OR-diode AC/USB). Las **salidas** (`+3.3V`, `GND_DC`) alimentan:

- **Módulo 5** — ESP32-C3-MINI-1-N4 (consumidor principal, LCSC C2838502).
- **Módulo 8** — optoacoplador secundario (fototransistor del PC817 → GPIO5, con pull-up a +3.3V).
- **Módulo 9** — divisor NTC para el ADC de temperatura y LED de estado GPIO6.
- **Módulo 6** — opcionalmente, el USBLC6-2SC6 (referencia de +3.3V cuando se alimenta vía USB-C).

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Principio del LDO — Arquitectura Interna

El ME6211C33M5G-N es un LDO (Low-Dropout Regulator) lineal, no un convertidor conmutado. Su operación se basa en **cuatro bloques funcionales** integrados en el SOT-23-5:

```
                    ME6211C33 — ARQUITECTURA INTERNA
                    ════════════════════════════════

   VIN ────┬──────────────────────────┬─────────────[Pass PMOS]────── VOUT
           │                          │                    ▲            │
           │                          │                    │            │
           │              [Compensación frecuencial]        │            │
           │                          │                    │            │
           │                          ▼                    │            │
           │              ┌───────────────────┐            │            │
           │              │ Amplificador de   │            │            │
           │              │ error (OpAmp)     │◄───────────┼───[R_fb1]──┤
           │              │                   │            │            │
           │              └──┬────────────────┘            │            │
           │                 │                             │       [R_fb2]
           │                 │                             │            │
   [Enable] CE ──────────────►                             │            │
           │                 │                             │            │
           │                 ▼                             │            │
           │     ┌───────────────────┐                    │            │
           │     │ Referencia de     │                    │            │
           │     │ banda prohibida   │                    │            │
           │     │ (Vref ≈ 0.6 V)    │                    │            │
           │     └───────────────────┘                    │            │
           │                                               │            │
   VSS ────┴───────────────────────────────────────────────┴────────────┘
   (GND)                                                              GND
```

**Ciclo de regulación (loop cerrado):**

1. La **referencia de banda prohibida** (bandgap reference) genera un voltaje interno de ~0.6 V que es insensible a temperatura y a VIN.
2. El **divisor de realimentación R_fb1/R_fb2** (interno, ajustado en fábrica) escala VOUT para que en estado nominal `V_fb = 3.3 V × R_fb2 / (R_fb1 + R_fb2) = 0.6 V`.
3. El **amplificador de error** compara V_fb con Vref. Si VOUT cae, V_fb cae, y el amplificador incrementa la señal de compuerta del pass PMOS → más corriente → VOUT sube.
4. El **transistor pass PMOS** actúa como resistencia variable en serie entre VIN y VOUT. Disipa la diferencia de voltaje × corriente como calor.

**El término "low dropout" se refiere específicamente al pass PMOS**: un LDO convencional con pass NPN necesita VIN – VOUT ≥ ~1.5 V (Vbe + saturation); un LDO con pass PMOS puede operar hasta que VIN – VOUT = R_DS(on) × I_load ≈ 100 mV.

### 2.2 Por Qué LDO Lineal y No Convertidor Buck Conmutado

Para este subsistema (3.3 V @ 350 mA pico, 150 mA promedio), un LDO lineal es superior a un buck por cinco razones concretas:

| Criterio | LDO ME6211C33 | Buck típico (MP2359, TPS62130) |
|---|---|---|
| **Ruido de salida a la carga** | 36 µVrms (integrado 10 Hz – 100 kHz) | 10–50 mVpp de rizado de conmutación + armónicos EMI |
| **Componentes externos** | 3 capacitores (1 µF + 10 µF + 100 nF) | Inductor (4.7 µH), diodo Schottky o low-side MOSFET, 2–4 capacitores, feedback divider, compensación |
| **Área de PCB** | ~15 mm² (SOT-23-5 + 3 MLCC) | ~60 mm² (con inductor 3×3 mm) |
| **Costo BOM** | ~$0.16 total | ~$0.80 total |
| **EMI radiada** | Despreciable (no hay conmutación) | Requiere blindaje, plano GND cuidadoso, y loops cortos |

**Justificación cuantitativa de la eficiencia:**
- En buck: η ≈ 88–92% → disipación ≈ 0.10 W.
- En LDO: η = Vout / Vin = 3.3 / 5.0 = 66% → disipación = 0.255 W @ carga promedio.
- Diferencia de disipación: ~0.15 W. En una PCB que ya está diseñada para disipar 2.5 W (bobina del relé + LDO + HLK), este delta es irrelevante térmicamente.

**Conclusión:** el LDO gana porque el ADC del ESP32 necesita un rail limpio para medir el NTC térmico con 10 bits útiles (1 LSB = 3.3 mV), y la simplicidad de layout reduce riesgo de EMI que comprometa certificaciones (FCC Part 15B, EN 55032).

### 2.3 PSRR a 1 kHz y Ruido — Impacto en la Medición ADC del NTC

El **PSRR (Power Supply Rejection Ratio)** mide cuánto ruido del rail de entrada se propaga al rail de salida. El ME6211 tiene **PSRR = 75 dB @ 1 kHz**.

**¿De dónde viene ruido a 1 kHz en el rail +5V?**

1. Rizado de la bobina del relé conmutando a 50/60 Hz y armónicos (1ᵉʳ–10ᵒ armónico: 50–600 Hz).
2. Rizado del flyback del HLK-5M05 reflejado en el primario y colado por la red AC.
3. Corrientes inductivas de carga (LEDs, cargadores de teléfono) en el mismo circuito de 110 V.

**Cálculo del ruido transferido al rail +3.3V:**

- Si asumimos 50 mVpp de rizado en +5V a 1 kHz:
  - Factor de atenuación PSRR: 75 dB = 5620× (V_in / V_out).
  - Ruido transferido: 50 mV / 5620 = **8.9 µV pp** → ≈ 3.1 µVrms.
- El ruido intrínseco del ME6211 a la salida: **36 µVrms** integrado en 10 Hz – 100 kHz.
- Ruido total en el rail +3.3V: √(3.1² + 36²) ≈ **36.1 µVrms**.

**¿Por qué importa este número?**

El ADC del ESP32-C3 tiene 12 bits (2 canales SAR ADC de 12 bits) con referencia Vref = 3.3 V:
- 1 LSB = 3.3 V / 4096 = **805 µV**.
- El ruido del rail (36 µVrms = ~240 µV pico-pico) representa **<1/3 de un LSB** → no degrada la resolución efectiva del ADC.
- Para comparación, un LDO con PSRR de 40 dB (ej: AMS1117) daría ruido transferido de ~500 µVrms → 0.6 LSB de ruido constante → resolución efectiva degradada a 11 bits.

**Conclusión:** el PSRR alto del ME6211 es lo que permite medir el NTC del Módulo 9 con los 12 bits completos del ADC sin promediado excesivo.

### 2.4 Dropout Voltage de 100 mV — Margen Durante Bursts WiFi

Durante la transmisión WiFi del ESP32-C3, la corriente pico típica es **335 mA** (datasheet WiFi 11b @ +20 dBm TX) durante ventanas de 1–2 ms. Para sizing del LDO usamos **350 mA como pico de diseño** (margen del 4% sobre el peor caso del datasheet). Esto crea dos efectos en cadena:

1. **Caída en el HLK-5M05:** el regulador flyback tiene respuesta transitoria limitada (loop de compensación ~1 kHz). Durante el burst, el rail +5V puede caer momentáneamente a **4.5 V**.
2. **Caída adicional en C_out1 (100 µF electrolítico del M2):** la ESR del electrolítico (~0.2 Ω) × 350 mA = 70 mV adicionales → +5V efectivo = 4.43 V.

**Verificación del dropout para el ME6211:**
- V_in mínimo = 4.43 V.
- V_out nominal = 3.30 V.
- Dropout disponible = V_in – V_out = 1.13 V.
- Dropout requerido por ME6211 a 350 mA ≈ 100 mV × (350/500) = **70 mV**.
- **Margen: 1.13 V – 0.07 V = 1.06 V** → regulación robusta incluso en worst case.

**Comparación con AMS1117-3.3 (rechazado):**
- Dropout del AMS1117: **1.3 V** a plena carga.
- V_in disponible: 4.43 V → V_out = 4.43 – 1.3 = **3.13 V**.
- El ESP32-C3 opera a partir de 3.0 V (rango VDD: 3.0–3.6 V nominal), pero el ADC pierde exactitud por debajo de 3.2 V, y la radio WiFi puede auto-resetearse por brownout configurado a 2.93 V (valor por defecto del BOD).
- **Conclusión:** el AMS1117 causaría resets aleatorios durante la operación normal → rechazo técnico válido.

### 2.5 Estabilidad con MLCC Cerámicos — Elección de C4, C5, C6

Los LDO son notoriamente sensibles a la **ESR (Equivalent Series Resistance)** del capacitor de salida. La estabilidad del loop depende de que el cero del capacitor (f_zero = 1 / (2π·ESR·C)) caiga dentro de una ventana específica.

- **LDO antiguos (ej: AMS1117):** necesitan ESR > 30 mΩ → requieren capacitores de **tantalio** (ESR 100–500 mΩ). No son estables con MLCC cerámicos (ESR ≈ 2–5 mΩ) — oscilan.
- **LDO modernos con compensación interna (ME6211, AP2112, TPS7A):** diseñados específicamente para MLCC → estables con ESR ≤ 10 mΩ y C_out ≥ 1 µF.

**Estrategia de decoupling de 3 capacitores en paralelo:**

| Ref | Valor | Banda de frecuencia que cubre | Razón |
|---|---|---|---|
| **C4** (entrada, 1 µF/25V X7R 0603) | 1 µF | DC – 1 MHz | Reservorio local para el pass PMOS; absorbe transitorios cuando el LDO demanda corriente súbita del rail +5V |
| **C5** (salida bulk, 10 µF/25V X5R 0805) | 10 µF | DC – 500 kHz | Estabilización del loop de realimentación + reserva para transitorios de carga del ESP32 (WiFi TX) |
| **C6** (salida HF, 100 nF/16V X7R 0402) | 100 nF | 1 MHz – 100 MHz | Baja inductancia parasítica (ESL ~1 nH), suprime spikes y resonancias VHF del pass MOSFET |

**Nota sobre C5 — 25V X5R:**
El datasheet del ME6211 recomienda **1 µF mínimo X7R en la salida**. Especificamos 10 µF / 25 V / X5R (LCSC **C15850**) por tres razones:
1. **Margen de derating de voltaje:** a 3.3 V DC, un MLCC de 25V conserva >90% de su capacitancia nominal; uno de 10V conservaría ~75% bajo DC bias.
2. **Stock y commonality:** C15850 ya se usa en los módulos M2, M5, M6, M7 → ahorra ranura de feeder en JLCPCB SMT assembly.
3. **X5R vs X7R:** X5R tiene rango térmico –55 °C a +85 °C (vs X7R hasta +125 °C). Para un rail de bulk output dentro de la caja de interruptor (ambiente 35–55 °C), X5R es más que suficiente.

### 2.6 Corriente Quiescente de 40 µA — Impacto en Deep-Sleep

El ESP32-C3 tiene un modo **deep-sleep** con consumo típico de 5 µA (RTC memory retention + RTC timer). Cuando el dispositivo despierta cada 30 min para reportar estado (uso típico de automatización residencial), el consumo promedio depende fuertemente del hardware de soporte:

**Presupuesto de corriente en deep-sleep (solo sistemas siempre-ON):**

| Elemento | Corriente | Comentario |
|---|---|---|
| ESP32-C3 en deep-sleep | 5 µA | RTC on, wake on timer |
| **ME6211C33 (Iq)** | **40 µA** | Siempre activo mientras haya +5V |
| LED de estado (apagado) | 0 µA | GPIO driven LOW = LED off |
| Divisor NTC (10 kΩ pull-up) | 0 µA | Pull-up sólo drena cuando ADC mide (ciclado) |
| Pull-up del fototransistor | 0 µA | PC817 off (sin AC switch presionado) |
| **Total DC-side quiescent** | **~45 µA** | |

**Si hubiésemos elegido AMS1117 (Iq = 5–11 mA):**
- Consumo en "deep-sleep" = 5000 + 40 = **~5 mA** (100× más).
- La fuente AC-DC HLK-5M05 opera a 25% de su eficiencia mínima en esa región → pierde ~0.15 W continuos como calor.
- El **AMS1117 sólo derrota el propósito del deep-sleep** — nunca se pensaron para aplicaciones de bajo consumo.

**En el contexto de producto comercial:** 45 µA × 8766 h/año = 0.39 A·h/año de consumo standby. A 110V × 0.39 A·h × 365 = ~16 Wh/año de energía "tragada" por el regulador lineal → irrelevante en factura de luz, pero relevante para certificaciones como **ErP Lot 6** (Europa — límite 0.5 W en standby, que NO incluye LDO quiescent, pero sí incluye la pérdida de la AC-DC asociada).

---

## 3. Análisis Térmico y Zona Segura de Operación (SOA)

### 3.1 Cálculo de Disipación

```
P_diss = (V_in - V_out) × I_out
```

| Condición | V_in | I_out | P_diss | Duty cycle |
|---|---|---|---|---|
| Idle (deep-sleep) | 5.00 V | 0.045 mA | 0.077 mW | 100% |
| Reposo activo (conectado WiFi) | 5.00 V | 120 mA | 0.204 W | 100% |
| **Pico TX WiFi** | 4.50 V | **350 mA** | **0.420 W** | <2% (ráfagas de 1–2 ms) |
| **Promedio ponderado 24h** | 5.00 V | **150 mA** | **0.255 W** | 100% |

### 3.2 Resistencia Térmica del SOT-23-5

El encapsulado SOT-23-5 sin disipador tiene θ_JA ≈ **220–260 °C/W** (hoja ME6211 típica sin PCB pad).

**Con las 4 vías térmicas y el plano de cobre inferior**, la resistencia térmica efectiva se reduce a:
- θ_JA efectivo ≈ **150 °C/W** (con 4 vías de 0.3 mm + plano GND ≥ 100 mm²).

### 3.3 Incremento de Temperatura de Juntura

**Condición nominal (150 mA promedio):**
- ΔT_JA = P_diss × θ_JA = 0.255 × 150 = **38.3 °C**.
- Ambiente en caja de interruptor residencial (Bogotá, 2640 m, interior de muro): **35 °C**.
- T_j estacionario = 35 + 38.3 = **73.3 °C**.
- Shutdown térmico del ME6211 = **150 °C** → margen de **77 °C** (muy cómodo).

**Condición pico (350 mA, bursts WiFi):**
- ΔT_JA instantáneo teórico = 0.420 × 150 = 63 °C.
- **Pero el burst dura <2 ms** — la masa térmica del encapsulado + cobre absorbe el transitorio.
- Constante térmica típica del SOT-23-5 + 4 vías ≈ 5 s → en 2 ms, la temperatura de juntura sube <0.04% del ΔT estacionario.
- T_j real durante burst ≈ 73 °C + 0.025 °C ≈ **73.3 °C** (impercibible).

### 3.4 Derating a Alta Temperatura Ambiente

Si el Smart Relay se instala en un lugar con temperatura ambiente más alta (ej: caja de interruptor expuesta al sol en techo, 55 °C):
- T_j = 55 + 38.3 = **93.3 °C** → todavía dentro del rango de operación (–40 a +125 °C).
- Margen a shutdown térmico: 57 °C.

**Conclusión:** el diseño térmico es robusto. No se requiere disipador adicional ni limitación de corriente.

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama General del Módulo

El diagrama refleja la disposición **del símbolo esquemático en EasyEDA Pro** (vista del esquemático, no del footprint físico). En el símbolo, los pines 1/2/3 están en el lado izquierdo y los pines 5/4 en el lado derecho.

**Concepto clave:** los tres capacitores son de desacople. Cada uno tiene una terminal en un rail (`+5V_COMBINED` o `+3.3V`) y la otra terminal en `GND_DC`. **No van en serie entre sí** — van todos en paralelo contra tierra.

- **C4** = capacitor de **entrada** → entre `+5V_COMBINED` y `GND_DC`, lo más cerca posible del Pin 1 (VIN).
- **C5** = capacitor de **salida bulk** → entre `+3.3V` y `GND_DC`, cerca del Pin 5 (VOUT).
- **C6** = capacitor de **salida HF** → entre `+3.3V` y `GND_DC`, todavía **más cerca** del Pin 5 que C5.

C5 y C6 están en paralelo entre sí (ambos entre +3.3V y GND). C4 está solo en la entrada.

```
         ENTRADA (+5V_COMBINED)                  SALIDA (+3.3V)
              de M7                        hacia M5, M8, M9
                │                                   ▲
                │                                   │
                │  ┌─────── U2 ME6211C33 ────────┐  │
                │  │                              │  │
                ├──┤ 1 ● VIN             VOUT ● 5 ├──┤
                │  │                              │  │
                │  │ 2 ● VSS               NC ● 4 ├──✕ (flotante)
                │  │                              │
                └──┤ 3 ● CE                       │
                   │                              │
                   │  SOT-23-5  —  vista símbolo  │
                   └──────────────────────────────┘
                │         (pin 2 VSS)              │
                │              │                   │
                │              │                   │
      ┌─────────┤              │         ┌─────────┼─────────┐
      │         │              │         │         │         │
      │        ═╪═             │        ═╪═       ═╪═        │
      │   C4   ─┴─            │   C5   ─┴─   C6  ─┴─        │
      │  1µF/25V               │  10µF/25V   100nF/16V       │
      │  X7R 0603              │  X5R 0805   X7R 0402        │
      │    │                   │    │         │              │
      │    │                   │    │         │              │
      └────┴───────────────────┴────┴─────────┴──────────────┘
                              GND_DC
                     (plano común de tierra DC)
```

**Leyenda:**
- `═╪═` / `─┴─` = símbolo de capacitor MLCC no polarizado (los dos pads son intercambiables).
- Las líneas horizontales superior e inferior son los rails comunes (`+5V_COMBINED` arriba-izquierda, `+3.3V` arriba-derecha, `GND_DC` abajo que atraviesa todo).

**Lectura rápida en EasyEDA (donde va cada pin de cada capacitor):**

| Componente | Pad 1 → conecta a | Pad 2 → conecta a |
|---|---|---|
| **C4** (1µF 0603) | `+5V_COMBINED` (mismo net que U2 pin 1) | `GND_DC` |
| **C5** (10µF 0805) | `+3.3V` (mismo net que U2 pin 5) | `GND_DC` |
| **C6** (100nF 0402) | `+3.3V` (mismo net que U2 pin 5) | `GND_DC` |

**Puente Pin 1 ↔ Pin 3 (CE siempre ON):**
- En EasyEDA Pro, simplemente colocas el wire-label `+5V_COMBINED` en el cable que sale del Pin 1 **y** en el cable que sale del Pin 3. Quedan conectados por el mismo net-name — no necesitas dibujar una traza diagonal.

**Pin 4 (NC):**
- Colocas el símbolo "No Connect" (✕) sobre el pin 4 para que el ERC de EasyEDA no dé warning.

**Notas del diagrama:**
- El pin 3 (CE) se puentea directamente al pin 1 (VIN) mediante un wire-label `+5V_COMBINED` común en EasyEDA — no se requiere traza física diagonal en el PCB, sólo net-label compartido.
- El pin 4 (NC) **se deja flotante** en el esquemático (sin net, sin DNP). EasyEDA Pro marcará un warning de ERC que se puede resolver colocando un "no-connect flag" (X) sobre el pin.
- Los tres capacitores (C4, C5, C6) van entre sus respectivos rails (+5V_COMBINED o +3.3V) y GND_DC — todos comparten la misma referencia de tierra.
- El círculo `●` en el pin 1 (VIN) del símbolo EasyEDA indica el pin 1 (convención estándar), no es polaridad.

### 4.2 Tabla de Nets (Conexiones Eléctricas)

| Net name | Nodos conectados | Corriente pico | Ancho de traza mín. |
|---|---|---|---|
| `+5V_COMBINED` | M7 salida → U2 pin 1 → U2 pin 3 → C4 pad 1 | 350 mA | 0.5 mm |
| `GND_DC` | U2 pin 2 → C4 pad 2 → C5 pad 2 → C6 pad 2 → plano inferior | 350 mA retorno | Plano ≥ 100 mm² |
| `+3.3V` | U2 pin 5 → C5 pad 1 → C6 pad 1 → M5/M8/M9 inputs | 350 mA | 0.5 mm |
| `CE` (interno) | U2 pin 1 puenteado a U2 pin 3 | <1 µA | 0.15 mm (cualquiera) |

**Todas las nets son de zona DC — no hay cruce de la barrera de aislamiento galvánica en este módulo.**

### 4.3 Pinout del ME6211C33M5G-N (vista superior según datasheet Microne)

```
          VISTA SUPERIOR (top view) — marcado hacia arriba
          ═════════════════════════════════════════════════

                    ┌──────────────┐
                    │              │
          Pin 5  ●──┤  ME6211      ├──●  Pin 4
         (VOUT)     │  C33M5G-N    │     (NC)
                    │              │
                    │   [marking]  │
                    │              │
          Pin 4  ●──┤              ├──●  Pin 3
         (NC)       │              │     (CE)
                    │              │
          Pin 1  ●──┤              ├──●  Pin 2
         (VIN)      │              │     (VSS/GND)
                    └──────────────┘

          Dimensiones: 2.9 × 1.6 × 1.1 mm (L × W × H)
          Pitch: 0.95 mm típico entre pines
          Encapsulado: SOT-23-5 (también llamado SOT-353, SC-74A)
```

**Asignación de pines (según datasheet Microne ME6211C33):**

| Pin | Nombre | Función | Conexión en este diseño |
|---|---|---|---|
| 1 | VIN | Entrada de alimentación (2.5 V – 6.5 V) | `+5V_COMBINED` |
| 2 | VSS | Tierra | `GND_DC` (con 4 vías térmicas al plano) |
| 3 | CE | Chip Enable, activo alto (≥ 0.9 V = ON) | Puenteado a VIN (siempre ON) |
| 4 | NC | No connect (internamente no conectado) | Flotante en PCB |
| 5 | VOUT | Salida regulada 3.3 V | `+3.3V` |

**Nota crítica sobre el pin 3 (CE):**
- El pin CE permite apagar el LDO en modo "ship mode" (consumo <1 µA).
- En este diseño, **NO usamos esta función** — el LDO debe estar siempre ON para que el ESP32 pueda despertar de deep-sleep por timer RTC.
- Conexión: **puentear CE a VIN con una traza corta** (NO dejar flotante — podría acoplar ruido y apagar el LDO aleatoriamente).
- Alternativa: usar una resistencia pull-up de 100 kΩ entre CE y VIN. En este diseño optamos por el puente directo para simplificar (ahorra 1 componente).

### 4.4 Diagrama de Conexión Pin a Pin Detallado

```
U2 — ME6211C33M5G-N (SOT-23-5)
  Pin 1 (VIN)   ◄──── +5V_COMBINED (desde M7) + C4 pad 1 + pin 3 (puente)  [ZONA DC]
  Pin 2 (VSS)   ◄──── GND_DC (con 4 vías térmicas 0.3 mm al plano inferior)[ZONA DC]
  Pin 3 (CE)    ◄──── PUENTE A PIN 1 (enable permanente — nunca flotante)  [ZONA DC]
  Pin 4 (NC)    ──── NO CONECTAR (dejar flotante en el esquemático)         [ZONA DC]
  Pin 5 (VOUT)  ────► +3.3V (hacia M5/M8/M9) + C5 pad 1 + C6 pad 1          [ZONA DC]

C4 — Capacitor entrada MLCC (1 µF / 25 V X7R 0603, Samsung CL10B105KA8NNNC)
  Pad 1 ◄──── +5V_COMBINED (mismo nodo que U2 pin 1)                       [ZONA DC]
  Pad 2 ◄──── GND_DC                                                       [ZONA DC]
  [COLOCAR A ≤ 2 mm DEL PIN 1 DE U2 — loop de corriente debe ser mínimo]

C5 — Capacitor salida bulk MLCC (10 µF / 25 V X5R 0805, Samsung CL21A106KAYNNNE)
  Pad 1 ◄──── +3.3V (mismo nodo que U2 pin 5)                              [ZONA DC]
  Pad 2 ◄──── GND_DC                                                       [ZONA DC]
  [COLOCAR A ≤ 3 mm DEL PIN 5 DE U2 — estabilidad del loop depende de esto]

C6 — Capacitor salida HF MLCC (100 nF / 16 V X7R 0402, YAGEO CC0402KRX7R9BB104)
  Pad 1 ◄──── +3.3V (mismo nodo que U2 pin 5)                              [ZONA DC]
  Pad 2 ◄──── GND_DC                                                       [ZONA DC]
  [COLOCAR A ≤ 1.5 mm DEL PIN 5 DE U2 — MÁS CERCA QUE C5]
  [ORDEN DESDE U2: primero C6 (HF), luego C5 (bulk) — intercepta transitorios
   antes de que se propaguen por la inductancia parasítica de la traza]
```

### 4.5 Resumen Visual de Colocación Física

```
                      PLANO DE LAYOUT DEL MÓDULO 4 (vista superior)
                      ══════════════════════════════════════════════

                            ◄── 8 mm ──►

                      ┌─────────────────────────┐
                      │                         │  ▲
                      │  [C4]                   │  │
                      │  1µF                    │  │
            +5V_COMB  ├───■ Pin1      Pin5 ■────┤  │ 6 mm
            ────────► │                         │  │
                      │     [ U2 ME6211 ]       │  │
                      │                         │  │
            GND_DC    ├───■ Pin2      Pin3 ■────┤  │
            ◄─────────│                         │  │
                      │  Pin4 (NC)              │  │
                      │                         │  │
                      │  [C6]  [C5]             │  │
                      │  100nF 10µF      +3.3V  │  ▼
                      │                  ──────►│
                      └─────────────────────────┘

     Vías térmicas: ●●●● bajo Pin 2 (VSS) → plano GND inferior
     Área total del módulo: ~48 mm² (8 × 6 mm)
```

---

## 5. Reglas de Layout PCB

### 5.1 Reglas Críticas de Colocación

| Regla | Valor | Razón |
|---|---|---|
| Distancia C4 ↔ U2 pin 1 (VIN) | ≤ 2 mm | Minimizar inductancia parasítica de la traza; loop de entrada más pequeño = mejor rechazo de ruido |
| Distancia C6 ↔ U2 pin 5 (VOUT) | ≤ 1.5 mm | El capacitor HF debe ver la menor inductancia posible (ESL domina > 1 MHz) |
| Distancia C5 ↔ U2 pin 5 (VOUT) | ≤ 3 mm | Estabilidad del loop de realimentación depende de C_out cercano con ESR baja |
| Orden desde pin 5: C6 → C5 | Obligatorio | HF primero intercepta transitorios antes de que lleguen al bulk |
| Vías térmicas bajo pin 2 (VSS) | 4 × Ø 0.3 mm | θ_JA de 220 °C/W → 150 °C/W. Sin vías, T_j supera 100 °C en condiciones extremas |
| Ancho de traza +5V_COMBINED | ≥ 0.5 mm | 350 mA × 10 °C rise en cobre 1 oz |
| Ancho de traza +3.3V | ≥ 0.5 mm | Mismo argumento de corriente |
| Plano GND_DC bajo U2 | ≥ 100 mm² continuo | Disipador térmico + retorno de corriente de baja impedancia |
| Net CE (pin 3) | Traza ≤ 1 mm al pin 1 | Puente físico directo — nunca rutear lejos |
| Pin 4 (NC) | Dejar flotante en PCB | Pad puede existir pero SIN traza conectada |

### 5.2 Checklist de Layout

- [ ] C4 colocado a ≤ 2 mm del pin 1 del U2, en la misma cara del PCB.
- [ ] C6 colocado **antes** que C5 en la traza de salida (C6 más cerca del pin 5).
- [ ] C5 colocado a ≤ 3 mm del pin 5 del U2.
- [ ] 4 vías térmicas (0.3 mm diámetro) bajo el pad de pin 2 (VSS) conectando al plano GND_DC en la capa inferior.
- [ ] Traza de puente CE → VIN corta (<1 mm), no rutear por vías.
- [ ] Pin 4 (NC) sin traza conectada (verificar en ERC).
- [ ] Plano GND continuo bajo el módulo completo (U2 + C4 + C5 + C6).
- [ ] Sin vías atravesando el footprint del U2 (interferirían con disipación).
- [ ] Thermal relief **deshabilitado** en pin 2 (VSS) — queremos conducción máxima al plano.
- [ ] Thermal relief **habilitado** en C4/C5/C6 (facilita soldadura de reflow).

### 5.3 Consideraciones de EMI

- El módulo NO radia EMI significativa (es lineal, no conmutado).
- Sin embargo, puede **actuar como antena receptora** del ruido del flyback del HLK-5M05 o del driver del relé → por eso PSRR alto es crítico.
- No requiere jaula de Faraday ni plano de blindaje adicional.

---

## 6. Selección de Componentes — Comparativa y Justificación

### 6.1 Tabla Comparativa (AMS1117 vs AP2112K vs ME6211)

| Parámetro | AMS1117-3.3 ❌ | AP2112K-3.3 ⚠️ | **ME6211C33 ✅** |
|---|---|---|---|
| Tipo | LDO NPN | LDO PMOS | **LDO PMOS** |
| Dropout voltage @ 500 mA | 1.3 V | 250 mV | **100 mV** |
| Corriente máx. salida | 1 A | 600 mA | **500 mA** |
| Corriente quiescente (Iq) | 5–11 mA | 55 µA | **40 µA** |
| PSRR @ 1 kHz | 65 dB | 70 dB | **75 dB** |
| Ruido de salida (10 Hz–100 kHz) | ~250 µVrms | ~55 µVrms | **~36 µVrms** |
| Estable con MLCC cerámicos | **No** (requiere tantalio) | Sí | **Sí** |
| Pin de Enable (CE) | No | Sí | **Sí** |
| Encapsulado | SOT-223 (grande) | SOT-23-5 | **SOT-23-5** |
| Precio LCSC (qty ≥ 10) | ~$0.08 | ~$0.16 | **~$0.12** |

### 6.2 Justificación Final

1. **Dropout 100 mV** — tolera caídas del +5V rail hasta 3.4 V antes de perder regulación; AMS1117 fallaría a 4.6 V.
2. **Iq 40 µA** — compatible con modo deep-sleep del ESP32-C3; AMS1117 consume 100× más.
3. **PSRR 75 dB** — mantiene el ADC del ESP32 con 12 bits útiles para medir NTC térmico.
4. **Estable con MLCC** — no requiere tantalio (caros, propensos a fallas en cortocircuito cuando envejecen, ambientalmente cuestionables).
5. **Precio intermedio** — $0.12 vs $0.16 del AP2112K (sorprendentemente más barato que su competencia directa por volumen de fabricación en Asia).

**El ME6211C33 gana en 4 de 5 categorías y empata en la quinta.**

---

## 7. BOM Completo — Módulo 4 con Códigos LCSC

Todos los componentes verificados para importación directa desde LCSC/JLCPCB y compatibles con EasyEDA Pro Edition.

### 7.1 Componentes Primarios

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 1 | U2 | ME6211C33M5G-N | LDO 3.3 V / 500 mA, dropout 100 mV, Iq 40 µA, PSRR 75 dB @ 1 kHz | SOT-23-5 | **C82942** | Microne (Nanjing Micro One Elec) | ME6211C33M5G-N | $0.12 |
| 2 | C4 | Capacitor MLCC de entrada | 1 µF / 25 V, ±10%, X7R | 0603 SMD | **C15849** | Samsung Electro-Mechanics | CL10B105KA8NNNC | $0.01 |
| 3 | C5 | Capacitor MLCC de salida (bulk) | 10 µF / 25 V, ±10%, X5R | 0805 SMD | **C15850** | Samsung Electro-Mechanics | CL21A106KAYNNNE | $0.02 |
| 4 | C6 | Capacitor MLCC de salida (HF) | 100 nF / 16 V, ±10%, X7R | 0402 SMD | **C14663** | YAGEO | CC0402KRX7R9BB104 | $0.01 |

**Total componentes primarios: 4 part numbers únicos, $0.16 USD en BOM.**

### 7.2 Componentes Alternativos / Equivalentes (Segundas Fuentes)

Si algún código LCSC primario no está disponible al momento del pedido, estas son las alternativas verificadas. Se listan **2 equivalentes por componente** ordenados por proximidad funcional.

#### 7.2.1 Alternativas para U2 (LDO regulador)

| Opción | LCSC | MPN | Fabricante | Dropout | Iq | Drop-in? | Notas |
|---|---|---|---|---|---|---|---|
| **Primario** | **C82942** | ME6211C33M5G-N | Microne | 100 mV | 40 µA | — | Selección base |
| Alternativa 1 | **C6186** | AP2112K-3.3TRG1 | Diodes Inc. | 250 mV | 55 µA | **Sí (pin-compatible SOT-23-5)** | Más ruidoso (~55 µVrms), dropout más alto pero aceptable. Requiere cambio de MPN en BOM, no cambio de footprint |
| Alternativa 2 | **C5446** | XC6206P332MR-G | Torex Semiconductor | 250 mV | 1 µA | **No — SOT-23-3** | **Iq ultra-bajo** (ideal para battery-powered), pero encapsulado de 3 pines (no tiene CE ni NC). Requiere rediseño de footprint. Plan B si el SOT-23-5 desaparece del mercado |

#### 7.2.2 Alternativas para C4 (capacitor entrada 1 µF/25V 0603 X7R)

| Opción | LCSC | MPN | Fabricante | Valor | Dielectric | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C15849** | CL10B105KA8NNNC | Samsung | 1 µF / 25 V | X7R | Selección base |
| Alternativa 1 | **C1588** | GRM188R71E105KA12D | Murata | 1 µF / 25 V | X7R | Drop-in directo, precio similar |
| Alternativa 2 | **C28323** | CC0603KRX7R9BB105 | YAGEO | 1 µF / 16 V | X7R | 16 V en vez de 25 V — suficiente para rail de 5 V (derating 3×), ligeramente más barato |

#### 7.2.3 Alternativas para C5 (capacitor bulk salida 10 µF/25V 0805 X5R)

| Opción | LCSC | MPN | Fabricante | Valor | Dielectric | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C15850** | CL21A106KAYNNNE | Samsung | 10 µF / 25 V | X5R | Selección base. Ya usada en M2, M6, M7 |
| Alternativa 1 | **C96446** | GRM21BR61E106KA73L | Murata | 10 µF / 25 V | X5R | Drop-in directo, calidad Murata (referencia en la industria) |
| Alternativa 2 | **C19702** | CL21A106KOQNNNE | Samsung | 10 µF / 16 V | X5R | 16 V en vez de 25 V; suficiente para rail 3.3 V con derating >4×. Ligeramente más barato |

#### 7.2.4 Alternativas para C6 (capacitor HF salida 100 nF/16V 0402 X7R)

| Opción | LCSC | MPN | Fabricante | Valor | Dielectric | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C14663** | CC0402KRX7R9BB104 | YAGEO | 100 nF / 16 V | X7R | Selección base. Ya usada en M2, M5, M6, M8 |
| Alternativa 1 | **C1525** | GRM155R71C104KA88D | Murata | 100 nF / 16 V | X7R | Drop-in directo, calidad Murata |
| Alternativa 2 | **C307331** | 0402B104K160CT | Walsin Technology | 100 nF / 16 V | X7R | Segunda fuente asiática, precio competitivo |

**Nota de procurement:** verificar stock actualizado en LCSC.com antes de generar el pedido — precios y disponibilidad cambian semanalmente. Los códigos listados estaban vigentes al 2026-04-17.

### 7.3 Assets EasyEDA Pro (Símbolo + Footprint + 3D)

EasyEDA Pro Edition importa automáticamente símbolo, footprint y modelo 3D de cualquier parte con código LCSC. El proceso es: **Insert → LCSC Parts → pegar código → Place**.

| Ref | LCSC | EasyEDA Pro — Acción | JLCPCB Assembly Tier | Fee adicional |
|---|---|---|---|---|
| U2 | C82942 | Insert → LCSC → `C82942` → confirmar SOT-23-5 | **Extended Part** | ~$3 USD feeder setup |
| C4 | C15849 | Insert → LCSC → `C15849` → confirmar 0603 | Basic Part | Sin fee |
| C5 | C15850 | Insert → LCSC → `C15850` → confirmar 0805 | Basic Part | Sin fee |
| C6 | C14663 | Insert → LCSC → `C14663` → confirmar 0402 | Basic Part | Sin fee |

**Nota de costo de ensamble JLCPCB:**
- Los 3 MLCC son "Basic Parts" (biblioteca siempre cargada en las máquinas SMT de JLCPCB) → **sin sobrecosto**.
- U2 ME6211C33M5G-N es "Extended Part" → requiere setup de feeder (~$3 USD una vez por lote). Para cantidades ≥ 20 piezas, este costo se amortiza a <$0.15 por unidad.

### 7.4 Resumen de Costo del Módulo 4

| Componente | Costo unitario aprox. (LCSC, qty ≥ 10) |
|---|---|
| U2 ME6211C33M5G-N (C82942) | $0.12 |
| C4 1 µF / 25 V X7R 0603 (C15849) | $0.01 |
| C5 10 µF / 25 V X5R 0805 (C15850) | $0.02 |
| C6 100 nF / 16 V X7R 0402 (C14663) | $0.01 |
| **TOTAL componentes** | **~$0.16 USD** |
| Fee JLCPCB (amortizado por 20 unidades) | +$0.15 |
| **TOTAL con ensamble** | **~$0.31 USD por unidad** |

---

## 8. Checklist de Verificación Pre-Fabricación y Pruebas Eléctricas

### 8.1 Verificación del Esquemático (EasyEDA Pro)

- [ ] U2 colocado con las 5 pines del SOT-23-5 correctamente asignados (VIN, VSS, CE, NC, VOUT).
- [ ] Pin 3 (CE) conectado directamente a pin 1 (VIN) — net común `+5V_COMBINED`.
- [ ] Pin 4 (NC) sin conexiones (aparece flotante en ERC).
- [ ] C4 conectado entre net `+5V_COMBINED` y `GND_DC`.
- [ ] C5 y C6 conectados en paralelo entre net `+3.3V` y `GND_DC`.
- [ ] Nets `+5V_COMBINED`, `+3.3V`, `GND_DC` con nombres idénticos a los usados en M5, M7, M8, M9.
- [ ] DRC (Design Rule Check) ejecutado sin errores en este módulo.
- [ ] ERC (Electrical Rule Check) ejecutado — no debe haber pins flotantes ni nets sin conexión.

### 8.2 Verificación del Layout PCB

- [ ] C4 a ≤ 2 mm de pin 1 del U2.
- [ ] C6 a ≤ 1.5 mm de pin 5 del U2 (más cerca que C5).
- [ ] C5 a ≤ 3 mm de pin 5 del U2.
- [ ] 4 vías térmicas de Ø 0.3 mm bajo pad pin 2 (VSS) conectando al plano GND inferior.
- [ ] Plano GND_DC continuo bajo todo el módulo (≥ 100 mm²).
- [ ] Ancho de traza `+5V_COMBINED` ≥ 0.5 mm.
- [ ] Ancho de traza `+3.3V` ≥ 0.5 mm.
- [ ] Silkscreen con referencias `U2`, `C4`, `C5`, `C6` visibles y cerca del componente.
- [ ] Pad 1 de C4/C5/C6 marcado como tal en silkscreen (o polaridad indicada aunque sean MLCC no polarizados — facilita inspección AOI).

### 8.3 Verificación de BOM

- [ ] Los 4 códigos LCSC coinciden con `BOM_Smart_Relay_C6.csv` (columna `LCSC_Part`, filas con módulo M4).
- [ ] Valores de componentes coinciden entre esquemático y BOM.
- [ ] MPNs corresponden a los LCSC codes (sin confusiones de fabricante).
- [ ] Stock verificado en LCSC.com al momento del pedido (cantidad ≥ 100 para prototipos, ≥ 1000 para producción).

### 8.4 Pruebas Eléctricas Post-Fabricación

**Con el Smart Relay alimentado por HLK-5M05 en estado normal (WiFi conectado, sin TX activa):**

| Medición | Punto de prueba | Valor esperado | Instrumento | Criterio de aceptación |
|---|---|---|---|---|
| Voltaje de entrada | U2 pin 1 vs GND_DC | 4.95 – 5.05 V DC | Multímetro DC | Dentro del rango |
| Voltaje de salida | U2 pin 5 vs GND_DC | 3.27 – 3.33 V DC | Multímetro DC | Tolerancia ±1% |
| Corriente quiescente (idle WiFi) | In-series con +5V_COMBINED | ~100 µA total (40 µA LDO + 60 µA ESP32 modem-sleep) | Multímetro µA | ≤ 200 µA |
| Ripple de salida | U2 pin 5 vs GND_DC | ≤ 5 mVpp | Osciloscopio 20 MHz BW, AC coupled | Sin picos >10 mV |
| Respuesta transitoria | +3.3V rail durante WiFi TX burst | Caída pico ≤ 100 mV, recovery ≤ 10 µs | Osciloscopio con trigger al GPIO TX | Sin undervoltage sostenido |
| Temperatura del U2 | Centro del encapsulado | ≤ 60 °C a 25 °C ambient | Termopar tipo K o cámara térmica | Margen ≥ 90 °C al shutdown |
| Ruido integrado 10 Hz–100 kHz | +3.3V rail | ≤ 50 µVrms | Analizador de espectro / ADC precisión | Confirma PSRR de catálogo |

### 8.5 Pruebas de Estrés y Edge Cases

- **Encendido con carga pesada (cold start):** el LDO debe arrancar limpiamente con el ESP32 ya demandando 150 mA. No debe haber oscilación ni retardo >10 ms en VOUT.
- **Cortocircuito de salida:** el ME6211 tiene limitación de corriente interna (~800 mA típico) y shutdown térmico. Aplicar cortocircuito momentáneo (<1 s) y verificar que el LDO se recupera al retirarlo.
- **Sobretemperatura:** calentar la PCB a 70 °C ambient con pistola de calor, confirmar regulación dentro de ±2% y temperatura del U2 ≤ 110 °C.
- **Brownout del rail +5V:** simular caída del rail de 5 V a 3.5 V con fuente programable. VOUT debe caer proporcionalmente (tracking mode) sin oscilar.

---

## 9. Conexión con Otros Módulos

### 9.1 Entrada: Módulo 7 → Módulo 4

```
Módulo 7 (OR-diode selector)            Módulo 4 (ME6211 LDO)
════════════════════════════            ══════════════════════

+5V_AC (desde M2) ──┐
                    ├──[D_AC]──┐
+5V_USB (desde M6)──┤          ├── +5V_COMBINED ──► U2 pin 1 (VIN)
                    ├──[D_USB]─┘                    U2 pin 3 (CE puenteado)
                    │                                C4 pad 1
                    │                         ┌── GND_DC
                    └── GND_DC ◄──────────────┘
```

El rail `+5V_COMBINED` es el resultado del **OR-diode** del Módulo 7 (sumado entre +5V_AC del HLK-5M05 y +5V_USB del puerto USB-C). Este rail puede oscilar entre 4.3 V y 5.2 V según la fuente activa y la carga — el ME6211 tolera todo ese rango con dropout de 100 mV.

### 9.2 Salidas: Módulo 4 → Módulos 5, 8, 9

```
Módulo 4 (ME6211)                        Consumidores de +3.3V
══════════════════                       ═══════════════════════════

U2 pin 5 (VOUT) ───┬──► +3.3V ───┬──► M5: ESP32-C3-MINI-1-N4 VDD (120 mA nom / 335 mA pk)
                   │              │      + C_vdd1 (10 µF) + C_vdd2 (100 nF)
              C5   │              │
              C6   │              ├──► M8: PC817 pull-up de fototransistor GPIO5 (~0.3 mA)
                   │              │
                   │              ├──► M9: NTC_temp pull-up 10 kΩ para ADC GPIO3 (~0.15 mA)
                   │              │
                   │              └──► M9: LED_status R_LED 470Ω GPIO6 (~3 mA cuando ON)
                   │
                   └──► GND_DC (plano inferior común a todos los módulos DC)
```

**Presupuesto de corriente de salida del LDO:**

| Consumidor | I_idle | I_activo | I_pico |
|---|---|---|---|
| ESP32-C3 (WiFi conectado, sin TX) | 15 mA | 120 mA | — |
| ESP32-C3 TX WiFi burst (pico típico) | — | — | 335 mA |
| PC817 fototransistor pull-up | 0 | 0.3 mA | 0.3 mA |
| NTC pull-up (ciclado ADC) | 0 | 0.15 mA | 0.15 mA |
| LED estado (ON) | 0 | 3 mA | 3 mA |
| **Total típico (reposo activo)** | **~15 mA** | **~125 mA** | **~355 mA** |

Margen al máximo del LDO (500 mA): **29%** → suficiente para variaciones de producción y picos transitorios.

### 9.3 Referencia GND_DC

El plano `GND_DC` del Módulo 4 es **el mismo** plano GND_DC que alimenta:
- Secundario del HLK-5M05 (M2).
- Bobina del relé (M3).
- ESP32-C3 (M5).
- USB-C (M6) — vía referencia común después del fusible PTC.
- Fototransistor del optoacoplador (M8).
- NTC y LED (M9).

**No hay separación de tierra analógica vs digital** en este diseño — el ADC del ESP32 trabaja con ruido bajo gracias al PSRR del LDO, no por aislamiento de tierra. Un plano GND continuo es **mejor** para frecuencias de RF (WiFi 2.4 GHz) que tierras separadas con un único punto de unión.

---

## 10. Referencias Cruzadas

- **Documento maestro del diseño:** `Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md` (sección 6, líneas 440–504).
- **BOM canónica:** `BOM_Smart_Relay_C6.csv` (filas con módulo M4: U2, C4, C5, C6).
- **Datasheet fabricante:** ME6211C33 — Nanjing Micro One Electronics Co., Ltd. (buscar "ME6211 datasheet" en LCSC.com/products/C82942).
- **Estándares aplicables:**
  - IEC 62368-1 — equipos de audio/video e ICT (aplicable al Smart Relay como dispositivo ICT).
  - FCC Part 15B — emisiones radiadas (el LDO no contribuye significativamente, pero el módulo M5 sí).
  - RETIE Colombia — Reglamento Técnico de Instalaciones Eléctricas (aplica al conjunto del Smart Relay).

---

*Fin del documento Módulo 4 — Regulador LDO 3.3 V (ME6211C33M5G-N).*
