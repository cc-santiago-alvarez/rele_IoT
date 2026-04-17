# Módulo 2: Fuente AC-DC Aislada HLK-5M05 — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0  
**Fecha:** 2026-04-16  
**Autor:** Equipo Domotica  
**Herramienta EDA:** EasyEDA Pro Edition  

---

## 1. Propósito del Módulo

El Módulo 2 es la pieza central de seguridad del Smart Relay. Su función es:

1. **Convertir** 85–265 VAC (50/60 Hz) a 5 VDC aislado con una sola etapa de potencia
2. **Proporcionar** aislamiento galvánico de 3000 VAC — la frontera SELV (Safety Extra-Low Voltage) que separa la zona peligrosa AC de la zona segura DC
3. **Suministrar** hasta 1 A (5 W) de potencia continua para alimentar todo el subsistema DC (ESP32, relé, sensores, LEDs)
4. **Habilitar** la conexión USB segura mientras el circuito está energizado con AC — sin el aislamiento, el USB sería letal
5. **Alimentar** el rail `+5V_AC` hacia el Módulo 7 (selector OR-diode AC/USB)

Las entradas del módulo (`AC_L_OUT` y `AC_N_OUT`) provienen del **Módulo 1** (entrada AC y protecciones). La salida `+5V_AC` alimenta el **Módulo 7**, y `GND_DC` se convierte en la referencia de tierra para todos los módulos DC (M3–M9).

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Topología Flyback del HLK-5M05

El HLK-5M05 es un convertidor AC-DC encapsulado que implementa internamente una **topología flyback** (convertidor de retorno). Esta es la topología estándar para fuentes de baja potencia (1–50 W) que requieren aislamiento galvánico.

**Arquitectura interna (dentro del módulo sellado):**

```
AC IN ──► [Puente rectificador] ──► [C_bulk interno] ──► [MOSFET switch]
  (pins 1,2)   (4 diodos)          (electrolítico HV)      │
                                                              │
                                                    ┌────────┴────────┐
                                                    │   TRANSFORMADOR  │
                                                    │   FLYBACK        │
                                                    │                  │
                                                    │  Primario    Secundario
                                                    │  (zona AC)   (zona DC)
                                                    │                  │
                                                    │  ════════════════│═══ BARRERA GALVÁNICA
                                                    │                  │
                                                    └────────┬────────┘
                                                              │
                                           [Diodo rectificador] ──► [Filtro LC] ──► DC OUT
                                                                                   (pins 3,4)
                                                              │
                                           [Optoacoplador] ◄──┤
                                           (retroalimentación)  │
                                                              [TL431 shunt]
```

**Secuencia de operación ciclo a ciclo (frecuencia ~65–100 kHz):**

1. **Fase de almacenamiento (MOSFET ON):**
   - El controlador PWM interno cierra el MOSFET de potencia
   - Corriente fluye a través del bobinado primario del transformador
   - La energía se almacena en el campo magnético del núcleo de ferrita (gap de aire)
   - El diodo de salida está en reversa → NO hay transferencia al secundario durante esta fase
   - Duración: t_ON (controlado por duty cycle del PWM)

2. **Fase de transferencia (MOSFET OFF):**
   - El controlador PWM abre el MOSFET
   - El campo magnético en el núcleo colapsa
   - La polaridad del voltaje en el secundario se invierte (de ahí "flyback" = retroceso)
   - El diodo de salida se polariza en directa → la energía almacenada se transfiere al secundario
   - La corriente carga los capacitores de salida y alimenta la carga
   - Duración: t_OFF (hasta que el núcleo se descarga completamente en modo DCM)

3. **Modo de conducción discontinua (DCM):**
   - En fuentes de baja potencia como el HLK-5M05, el núcleo se descarga completamente antes de que comience el siguiente ciclo
   - Esto se llama DCM (Discontinuous Conduction Mode)
   - Ventaja del DCM: el diodo de salida se apaga con corriente cero → menos ruido de conmutación y pérdidas de recovery

**¿Por qué flyback y no buck/boost?**
- Un convertidor buck o boost NO proporciona aislamiento galvánico — el primario y secundario comparten una referencia eléctrica
- El flyback usa un transformador con bobinados físicamente separados — la energía se transfiere magnéticamente, no eléctricamente
- Esta separación física es lo que permite los 3000 VAC de aislamiento

### 2.2 Aislamiento Galvánico y el Concepto SELV

**¿Qué es SELV?**

SELV (Safety Extra-Low Voltage) es un concepto de seguridad definido en **IEC 62368-1** (estándar para equipos de audio/video e ICT). Un circuito es SELV cuando:

- Opera a **≤ 60 VDC** (o ≤ 42.4 Vpico AC)
- Está **aislado galvánicamente** de cualquier voltaje peligroso mediante aislamiento reforzado o doble
- En caso de falla simple, el voltaje NO puede exceder los límites SELV

El Módulo 2 (HLK-5M05) crea la frontera SELV: todo lo que está conectado a sus pins 3 y 4 (lado DC) es SELV, mientras que sus pins 1 y 2 (lado AC) están al potencial peligroso de la red.

**¿Por qué es crítico el aislamiento en este diseño?**

```
                    CON AISLAMIENTO (este diseño)
                    ═════════════════════════════
    RED AC 110V                              USB al PC
        │                                        │
   [HLK-5M05]──3000 VAC──►[ESP32]◄──────────[USB-C]
        │         aislamiento        │            │
      AC GND                      DC GND ═══ USB GND
   (peligroso)                    (seguro, SELV)

   → El usuario puede tocar el USB y el ESP32 con seguridad
   → El PC está protegido contra la red eléctrica
   → Cumple IEC 62368-1 / RETIE


                    SIN AISLAMIENTO (Shelly Plus 1, LNK304)
                    ════════════════════════════════════════
    RED AC 110V                              USB al PC
        │                                        │
   [LNK304 buck]──SIN aislamiento──►[ESP8685]◄──[USB]
        │              │                │         │
      AC GND ═══════ "DC" GND ═══════ GND ═══ USB GND
   (peligroso)     (¡PELIGROSO!)   (¡PELIGROSO!)

   → GND del ESP está al potencial de la red AC
   → Tocar el USB mientras AC está conectado = DESCARGA ELÉCTRICA
   → Shelly mitiga esto con una carcasa sellada sin USB accesible
   → Nuestro diseño NO puede aceptar esto: necesitamos USB accesible para flasheo
```

**¿Cómo logra el aislamiento el HLK-5M05?**

El aislamiento se logra mediante **tres mecanismos internos independientes**:

1. **Transformador con bobinados separados:** los bobinados primario y secundario están enrollados en bobinas físicamente distintas sobre el núcleo de ferrita, con aislamiento de triple capa (triple-insulated wire) entre ellos. No hay conexión eléctrica — la energía se transfiere únicamente por el campo magnético.

2. **Optoacoplador de retroalimentación:** la señal de control que regula el voltaje de salida cruza la barrera ópticamente (fotón LED → fotodiodo/fototransistor). No hay conexión eléctrica en el lazo de retroalimentación.

3. **Separación física en el PCB del módulo:** dentro del encapsulado del HLK-5M05, las trazas del primario y secundario mantienen creepage >6 mm sobre el sustrato cerámico/FR4 interno.

### 2.3 Qué Significa 3000 VAC de Rigidez Dieléctrica

La especificación "3000 VAC de rigidez dieléctrica" (dielectric withstand voltage) se refiere al **test de hi-pot** (high potential):

- Se aplican **3000 VAC RMS** entre los pins de entrada (1, 2) y los pins de salida (3, 4)
- El voltaje se mantiene durante **1 minuto** continuo
- El módulo debe **NO presentar** breakdown (arco eléctrico ni corriente de fuga >1 mA)

**¿Qué significa en la práctica?**

- 3000 VAC RMS equivale a **~4243 Vpico** (3000 × √2)
- Un rayo que induce un surge de 2000 V en la línea AC quedaría contenido en el lado primario — los 3000 VAC de la barrera son más que suficientes
- El peor caso normal en la red colombiana 110 V es el pico de voltaje: 110 × √2 = **155 Vpico**
- El margen es: 4243 / 155 = **27× sobre el pico normal de operación**

**Cumplimiento normativo:**

| Norma | Requisito | HLK-5M05 | Margen |
|---|---|---|---|
| IEC 62368-1 (aislamiento reforzado, 300 VAC trabajo) | 3000 VAC hi-pot | 3000 VAC | Cumple exacto |
| RETIE (Colombia, instalación residencial) | Basado en IEC 62368-1 | 3000 VAC | Cumple |
| Altitud Bogotá 2640 m (factor corrección ×1.14) | Clearance ×1.14, creepage sin cambio | Clearance interno suficiente | Cumple con margen |
| UL 62368-1 (para producción certificada) | Requiere componente UL-listed | HLK-5M05 **NO** es UL-listed | Usar MeanWell IRM-05-5 |

**Nota para producción certificada UL:** el HLK-5M05 NO tiene certificación UL propia. Para obtener UL listing del producto final, sustituir por **MeanWell IRM-05-5** (mismo footprint DIP-4, 5W/1A, certificado UL/TÜV/CB, ~$5-9 USD).

### 2.4 Lazo de Retroalimentación por Optoacoplador

El HLK-5M05 mantiene su salida regulada a 5V ±2% mediante un lazo de control cerrado que cruza la barrera de aislamiento. Aunque los componentes internos no son accesibles, la topología estándar de Hi-Link utiliza:

```
                LADO PRIMARIO (AC)          │ BARRERA │     LADO SECUNDARIO (DC)
                                            │ 3000VAC │
                                            │         │
    ┌──────────────┐                        │         │     +5V_AC (pin 4)
    │  Controlador │◄── señal óptica ◄──────│─────────│──── Optoacoplador LED
    │  PWM (ej.    │    del optoacoplador   │         │          │
    │  THX208)     │                        │         │     ┌────┴────┐
    │              │                        │         │     │  TL431  │
    │  duty cycle  │                        │         │     │  shunt  │
    │      │       │                        │         │     │  reg.   │
    │      ▼       │                        │         │     └────┬────┘
    │   MOSFET     │                        │         │          │
    │   switch     │                        │         │     Divisor resistivo
    └──────────────┘                        │         │     (sensa Vout)
                                            │         │          │
                                            │         │     GND_DC (pin 3)
```

**Secuencia de regulación:**

1. Un **divisor resistivo** en el secundario monitorea el voltaje de salida (+5V_AC)
2. El **TL431** (referencia de voltaje de precisión + amplificador de error) compara Vout con su referencia interna de 2.5 V
3. Si Vout > 5V → el TL431 aumenta la corriente a través del **LED del optoacoplador**
4. La señal luminosa del LED **cruza la barrera ópticamente** hasta el fototransistor en el lado primario
5. El controlador PWM (ej. THX208 o similar) en el primario **reduce el duty cycle** del MOSFET
6. Menos energía se almacena por ciclo → el voltaje de salida baja
7. El lazo se estabiliza cuando Vout = 5.00 V (±2%)

**Características del lazo:**
- Regulación de línea: Vout varía < ±2% cuando Vin cambia de 85 a 265 VAC
- Regulación de carga: Vout varía < ±3% cuando la carga cambia de 0 a 1 A
- Respuesta transitoria: recuperación en < 5 ms ante cambios de carga del 50%

### 2.5 Especificaciones Eléctricas del HLK-5M05

| Parámetro | Valor | Notas |
|---|---|---|
| Voltaje de entrada | 85–265 VAC | Universal input, 50/60 Hz |
| Voltaje de salida | 5.0 VDC ±2% | Regulado por lazo óptico |
| Corriente máxima de salida | 1.0 A continuo | Protección OCP interna |
| Potencia máxima | 5.0 W | Derating sobre 60°C ambiente |
| Aislamiento entrada↔salida | 3000 VAC / 1 min | Hi-pot test per IEC 62368-1 |
| Eficiencia | ~80% típica @ carga completa | ~78% @ 50% carga |
| Ripple de salida | < 50 mVp-p típico | Medido con C_out1 + C_out2 + C_out3 externos |
| Consumo en vacío (no-load) | < 0.1 W | Sin carga conectada |
| Protección sobrecarga | OCP (Over-Current Protection) | Foldback, auto-recovery |
| Protección cortocircuito | SCP (Short-Circuit Protection) | Hiccup mode, auto-recovery |
| Temperatura de operación | -40 °C a +70 °C | Ambiente, sin derating |
| Dimensiones | 34 × 20 × 15 mm | Componente más alto del PCB |
| Encapsulado | DIP-4 (4 pines, pitch 27.4 mm × 7.6 mm) | Pins THT de 1 mm diámetro |

---

## 3. Diagrama de Conexión Pin a Pin

### 3.1 Diagrama General del Módulo

El diagrama muestra el flujo de energía de izquierda a derecha: AC entra por los pins 1/2, cruza la barrera de aislamiento galvánico dentro del HLK-5M05, sale como DC por los pins 3/4, pasa por tres capacitores de filtrado en paralelo, y alimenta al Módulo 7.

```
       ZONA AC (peligrosa)       │  BARRERA  │        ZONA DC (SELV, segura)
       ──────────────────        │ 3000 VAC  │        ──────────────────────
                                  │ AISLAM.  │
  (desde Módulo 1)                │           │                      (hacia Módulo 7)
                                  │           │
              ┌──────────────────────────────────────────┐
  AC_L_OUT ─►│ Pin 1 (AC)                   Pin 4 (+Vo) │─► +5V_AC ──────┬───────┬───────┬──► D_AC (M7)
              │                                           │                 │       │       │     ánodo SS14
              │           HLK-5M05 (U5)                  │                ═╪═     ═╪═     ═╪═
              │      AC-DC Isolated Supply               │              C_out1  C_out2  C_out3
              │          3000 VAC / 5W                   │              100µF   10µF    100nF
              │                                           │              16V     25V     50V
              │                                           │              electr. MLCC    MLCC
              │                                           │              (THT)   X5R     X7R
  AC_N_OUT ─►│ Pin 2 (AC)                   Pin 3 (-Vo) │─► GND_DC ──────┴───────┴───────┴──► GND sistema DC
              └──────────────────────────────────────────┘                                      (M3, M4, M5,
                                  │           │                                                   M6, M7, M8, M9)
                                  │  ┌─────┐  │
                    ···············│··│SLOT │··│·············· (slot fresado 2 mm en el PCB,
                                  │  │ 2mm │  │                 pasa debajo del cuerpo del módulo)
                                  │  └─────┘  │
                                  │           │

  Leyenda de flujo de corriente:
    ─► : dirección de flujo de energía / señal
    ═╪═: capacitor conectado a GND (símbolo esquemático)
    ···: slot fresado del PCB (barrera física de aislamiento)

  Los tres capacitores en paralelo están en orden de montaje recomendado:
    C_out3 (100nF) → más cerca de pins 3/4 (intercepta HF primero)
    C_out2 (10µF)  → en el medio (filtra rizado MF del flyback)
    C_out1 (100µF) → más lejos (reserva bulk, transitorios de carga)
```

### 3.2 Tabla de Nets (Conexiones Eléctricas)

| Net Name | Nodos conectados | Descripción |
|---|---|---|
| `AC_L_OUT` | L1 pin 2 (M1), HLK-5M05 pin 1 (U5) | Línea AC filtrada desde Módulo 1 hacia la fuente |
| `AC_N_OUT` | L1 pin 3 (M1), HLK-5M05 pin 2 (U5) | Neutro AC filtrado desde Módulo 1 hacia la fuente |
| `+5V_AC` | HLK-5M05 pin 4 (U5), C_out1 (+), C_out2 pad 1, C_out3 pad 1, D_AC ánodo (M7) | Rail 5V DC aislado, salida de la fuente AC |
| `GND_DC` | HLK-5M05 pin 3 (U5), C_out1 (−), C_out2 pad 2, C_out3 pad 2, GND de M3-M9 | Referencia de tierra DC aislada galvánicamente de AC |

### 3.3 Diagrama de Conexión Pin a Pin Detallado

```
U5 — HLK-5M05 (Módulo DIP-4, 34 × 20 × 15 mm)
  Pin 1 (AC)  ◄──── AC_L_OUT desde Módulo 1 (L1 pin 2, o NODO_L si L1 bypassed)
  Pin 2 (AC)  ◄──── AC_N_OUT desde Módulo 1 (L1 pin 3, o NODO_N si L1 bypassed)
  Pin 3 (−Vo) ────► GND_DC rail (referencia tierra DC aislada)
  Pin 4 (+Vo) ────► +5V_AC rail (5 VDC aislado)
  [No polarizado: pins 1 y 2 son intercambiables (L/N)]

C_out1 — Electrolítico 100 µF / 16 V (radial THT 5×7 mm)
  Pin (+) ◄──── +5V_AC (HLK-5M05 pin 4)
  Pin (−) ◄──── GND_DC (HLK-5M05 pin 3)
  [Reserva de energía bulk. Absorbe transitorios de carga lenta: activación
   de relé (80 mA step), ráfagas WiFi TX (530 mA, 2 ms). Colocar < 5 mm de pins 3,4]

C_out2 — Cerámico MLCC 10 µF / 25 V X5R (0805)
  Pad 1 ◄──── +5V_AC (en paralelo con C_out1)
  Pad 2 ◄──── GND_DC (en paralelo con C_out1)
  [Filtra rizado de media frecuencia (1 kHz – 100 kHz). Su ESR ultra-bajo
   (~5 mΩ) absorbe el rizado de conmutación del flyback a 65-100 kHz que
   el electrolítico (ESR ~0.5-2 Ω) no puede filtrar eficientemente]

C_out3 — Cerámico MLCC 100 nF / 50 V X7R (0805)
  Pad 1 ◄──── +5V_AC (en paralelo con C_out1 y C_out2)
  Pad 2 ◄──── GND_DC (en paralelo con C_out1 y C_out2)
  [Filtra ruido de muy alta frecuencia (>100 kHz – MHz). Suprime spikes
   de conmutación y resonancias de alta frecuencia]

Salida hacia Módulo 7:
  +5V_AC ─────────────────────────────────────────► D_AC ánodo (SS14 Schottky, M7)
  GND_DC ─────────────────────────────────────────► GND rail de toda la zona DC
```

### 3.4 Barrera de Aislamiento en PCB

El HLK-5M05 es uno de los tres componentes que **cruzan físicamente** la barrera de aislamiento del PCB. El slot fresado de 2 mm pasa debajo del cuerpo del módulo:

```
  VISTA LATERAL (corte transversal del PCB)
  ══════════════════════════════════════════

                    HLK-5M05 (34 × 20 × 15 mm)
                ┌───────────────────────────────┐
                │         Cuerpo sellado         │
                │    (transformador + control)    │
  Pin 1 (AC)───┤                                 ├───Pin 4 (+5V DC)
  Pin 2 (AC)───┤                                 ├───Pin 3 (GND DC)
                └───────┬───────────────┬─────────┘
                        │               │
  ══════════════════════╪═══════════════╪══════════════════════
  │  ZONA AC  │         │  SLOT 2mm    │         │  ZONA DC  │
  │  (trazas  │  ┌──────┘              └──────┐  │  (trazas  │
  │  AC_L_OUT,│  │◄─── 2 mm fresado ────────►│  │  +5V_AC,  │
  │  AC_N_OUT)│  │      (sin cobre)           │  │  GND_DC)  │
  │           │  └────────────────────────────┘  │           │
  ══════════════════════════════════════════════════════════════
               ◄── creepage ≥ 6.5 mm ──►
               ◄── clearance ≥ 5.0 mm ──►


  VISTA SUPERIOR (PCB layout simplificado)
  ═══════════════════════════════════════

  ┌─────────────────────┬───────────────────────────────────────┐
  │                     │                                       │
  │    ZONA AC          │ S          ZONA DC                    │
  │                     │ L                                     │
  │  ┌─AC_L_OUT─pin1┐   │ O    ┌pin4─+5V_AC──►D_AC(M7)       │
  │  │              │   │ T    │                                │
  │  │   HLK-5M05  │   │      │   HLK-5M05                    │
  │  │              │   │ F    │                                │
  │  └─AC_N_OUT─pin2┘   │ R    └pin3─GND_DC──►Sistema DC      │
  │                     │ E        [C_out1][C_out2]             │
  │                     │ S                                     │
  │                     │ A                                     │
  │                     │ D                                     │
  │                     │ O                                     │
  └─────────────────────┴───────────────────────────────────────┘
                        ▲
                    2 mm slot
               (sin cobre, sin vías)
```

**Tres barreras de aislamiento independientes en el PCB:**

| Componente | Aislamiento | Función | Método de cruce |
|---|---|---|---|
| **HLK-5M05** (U5) | 3000 VAC | Potencia: AC → 5V DC | Transformador + optoacoplador interno |
| **PC817C** (U3, M8) | 5000 Vrms | Señal: estado switch AC → GPIO5 DC | Acoplamiento óptico (LED → fototransistor) |
| **HF115F-I** (K1, M3) | 4000 Vrms | Potencia: bobina DC → contactos AC | Separación mecánica + magnética |
| **Slot PCB** | ≥6.5 mm creepage | Barrera física entre trazas | Gap fresado de 2 mm, sin cobre |

---

## 4. Presupuesto de Potencia

### 4.1 Tabla de Consumidores

El HLK-5M05 alimenta (directa o indirectamente) todos los consumidores DC del Smart Relay:

| Consumidor | Condición | Corriente @ 5V | Potencia | Módulo |
|---|---|---|---|---|
| ESP32-C3 TX WiFi (pico 2 ms) | Via LDO 66% eff: 350 mA @ 3.3V ÷ 0.66 | 530 mA | 2.65 W | M5 via M4 |
| ESP32-C3 modem-sleep (promedio) | Via LDO 66% eff: 30 mA @ 3.3V ÷ 0.66 | 45 mA | 0.23 W | M5 via M4 |
| Relé HF115F-I bobina (activado) | Directo 5V / 62.5 Ω | 80 mA | 0.40 W | M3 |
| LED de estado | Via LDO: 2.3 mA @ 3.3V | ~4 mA | 0.01 W | M9 |
| PC817 fototransistor + NTC | Despreciable | ~1 mA | < 0.01 W | M8 + M9 |
| **Total pico** (TX WiFi + relé ON) | Peor caso transitorio (~2 ms) | **~615 mA** | **~3.07 W** | — |
| **Total promedio** (modem-sleep + relé ON) | Operación sostenida normal | **~130 mA** | **~0.65 W** | — |
| **Capacidad HLK-5M05** | Rating nominal continuo | **1000 mA** | **5.00 W** | — |

### 4.2 Análisis de Márgenes

| Métrica | Valor | Evaluación |
|---|---|---|
| Carga pico / Capacidad | 615 mA / 1000 mA = **61.5%** | Margen de 38.5% — excelente |
| Carga promedio / Capacidad | 130 mA / 1000 mA = **13%** | Opera muy holgado |
| Margen sobre pico | (1000 − 615) / 615 = **+63%** | Suficiente para variaciones de componente |
| Duración del pico | ~2 ms (duración de TX WiFi burst) | Transitorio, no sostenido |

**Comparación con alternativas:**

| Fuente | Capacidad | Carga pico 615 mA | Estado |
|---|---|---|---|
| **HLK-5M05** (elegida) | 1000 mA | 61.5% de carga | Holgado, sin estrés térmico |
| HLK-PM01 (3W / 600 mA) | 600 mA | **102.5% de carga** | Sobrecargado en picos → inaceptable |
| MeanWell IRM-05-5 | 1000 mA | 61.5% de carga | Idéntico, pero UL certified ($5-9) |

### 4.3 Perfil Térmico

El HLK-5M05 tiene una eficiencia de ~80%, lo que significa que disipa ~20% de la potencia de entrada como calor.

| Condición | Potencia de salida | Potencia disipada (calor) | Temp. carcasa estimada |
|---|---|---|---|
| Pico TX WiFi (2 ms) | 3.07 W | ~0.77 W (pero solo 2 ms) | Irrelevante (transitorio) |
| Promedio modem-sleep + relé | 0.65 W | ~0.16 W | Ambiente + ~5 °C |
| Peor caso sostenido (TX continuo + relé) | ~3.07 W | ~0.77 W | Ambiente + ~15-20 °C |
| **Límite de operación** | 5.0 W | ~1.25 W | **70 °C ambiente máximo** |

**Caso peor realista:**
- Temperatura ambiente en caja de conexión empotrada en pared en Bogotá: ~35 °C
- Elevación de temperatura por disipación a carga promedio: ~5 °C
- Temperatura de carcasa: ~40 °C — **muy por debajo del límite de 70 °C**
- TF1 (fusible térmico de 115 °C) solo se dispararía ante una falla catastrófica interna

---

## 5. BOM Completo — Módulo 2 con Códigos LCSC

Todos los componentes verificados para importación desde LCSC/JLCPCB y compatibles con EasyEDA Pro.

### 5.1 Componente Principal

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 1 | U5 | HLK-5M05 AC-DC aislada | 5V/1A, 3000 VAC isol., 85-265 VAC in, 5W | Module DIP-4, 34×20×15 mm | **C209907** | HI-LINK | HLK-5M05 | $2.50 |

### 5.2 Filtrado de Salida (Zona DC) — Esquema de 3 Capacitores

El filtrado de salida usa **tres capacitores en paralelo** que cubren todo el espectro de frecuencia:

| Banda de frecuencia | Responsable | Por qué ese componente |
|---|---|---|
| Baja frecuencia (DC – 1 kHz) | C_out1 (100 µF electrolítico) | Alta capacitancia para reserva de energía bulk y absorción de transitorios lentos (relé ON/OFF, WiFi TX bursts) |
| Media frecuencia (1 kHz – 100 kHz) | C_out2 (10 µF MLCC X7R) | ESR ultra-bajo (~5 mΩ) filtra el rizado de conmutación del flyback (65-100 kHz) y transitorios rápidos de la bobina del relé |
| Alta frecuencia (>100 kHz – MHz) | C_out3 (100 nF MLCC X7R) | Baja inductancia parasítica, suprime spikes de conmutación y resonancias VHF |

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 2 | C_out1 | Electrolítico bulk output | 100 µF / 16 V, ±20%, 2000 hrs @ 105°C | Radial THT 5×7 mm (pitch 2 mm) | **C43803** | Chengx (Dongguan Chengxing) | KS107M016D07RR0VH2FP0 | $0.01 |
| 3 | C_out2 | MLCC cerámico MF | 10 µF / 25 V, ±10%, X5R | 0805 SMD | **C15850** | Samsung Electro-Mechanics | CL21A106KAYNNNE | $0.02 |
| 4 | C_out3 | MLCC cerámico HF | 100 nF / 50 V, ±10%, X7R | 0805 SMD | **C49678** | YAGEO | CC0805KRX7R9BB104 | $0.01 |

### 5.3 Componentes Alternativos / Equivalentes

Para cada componente, si el código LCSC primario no está disponible al momento del pedido:

| Ref | Primario | Alternativa 1 | Alternativa 2 | Notas |
|---|---|---|---|---|
| U5 | C209907 (HI-LINK HLK-5M05) | **MeanWell IRM-05-5** (para producción UL/TÜV/CB) | HLK-PM05 (pin-compatible, 5W variante) | IRM-05-5 tiene mismo footprint DIP-4 y certificación UL. Buscar en Mouser/Digikey |
| C_out1 | C43803 (Chengx 100µF/16V, 5×7 mm) | C43805 (Chengx 100µF/16V, 6.3×7 mm KS107M016E07RR0VH2FP0) | C43348 (Chengx 100µF/16V, 5×11 mm KM107M016D11RR0VH2FP0) | Cualquier electrolítico 100µF ≥10V radial, footprint similar |
| C_out2 | C15850 (Samsung CL21A106KAYNNNE, 10µF/25V X5R 0805) | C19702 (Murata GRM21 series 10µF/25V X5R 0805) | C96446 (YAGEO CC0805KKX7R8BB106, 10µF/10V X7R 0805) | Cualquier MLCC 10µF ≥10V, 0805, ESR bajo. X5R/X7R aceptable |
| C_out3 | C49678 (YAGEO CC0805KRX7R9BB104, 100nF/50V X7R 0805) | C17903 (Samsung CL21B104KBCNNNC, 100nF/50V X7R 0805) | C14663 (YAGEO CC0402KRX7R9BB104, 100nF/50V X7R 0402) | Cualquier MLCC 100nF ≥16V X7R, 0805 preferido |

### 5.4 Resumen de Costo del Módulo 2

| Componente | Costo unitario (aprox.) |
|---|---|
| U5 HLK-5M05 (C209907) | $2.50 |
| C_out1 100µF/16V electrolítico (C43803) | $0.01 |
| C_out2 10µF/25V MLCC X5R (C15850) | $0.02 |
| C_out3 100nF/50V MLCC X7R (C49678) | $0.01 |
| **TOTAL Módulo 2** | **~$2.54 USD** |

*Precios de LCSC en cantidades ≥ 10 unidades. Sujetos a cambio.*

---

## 6. Notas de Diseño para EasyEDA Pro

### 6.1 Importación de Componentes

Para importar cada componente en EasyEDA Pro:

1. Abrir **Library** → **LCSC Parts**
2. Buscar por código LCSC (ej: `C209907`)
3. Verificar que el footprint coincida con el encapsulado listado
4. Colocar en el esquemático y asignar la referencia (U5, C_out1, C_out2, C_out3)

**Nota sobre el HLK-5M05:** el footprint en LCSC puede no incluir el slot de aislamiento. Se recomienda dibujar un **slot mecánico** (board cutout) de 2 mm de ancho debajo del cuerpo del módulo en la capa mecánica del PCB.

### 6.2 Reglas de Layout Críticas

| Regla | Valor | Razón |
|---|---|---|
| Posición HLK-5M05 | Cruza el slot fresado 2 mm | Pins 1,2 en zona AC; pins 3,4 en zona DC |
| Creepage AC↔DC (a través del slot) | ≥ 6.5 mm | IEC 62368-1 aislamiento reforzado + factor altitud Bogotá (×1.14) |
| Clearance AC↔DC (gap de aire vía slot) | ≥ 5.0 mm | Con factor de corrección para 2640 m altitud |
| Proximidad capacitores de salida | C_out1, C_out2 y C_out3 dentro de 5 mm de pins 3,4 | Minimizar loop area de corriente de ripple |
| Orden de capacitores (desde HLK) | C_out3 (100nF) más cerca → C_out2 (10µF) → C_out1 (100µF) | MLCC HF primero para interceptar spikes antes de que se propaguen |
| Prohibición trazas cruzando slot | **Absoluta** — ninguna traza ni vía | Destruiría el aislamiento galvánico del diseño |
| Vías en zona del slot | Clearance ≥ 1 mm desde bordes del slot | Cobre en zona de slot compromete aislamiento |
| Ancho traza +5V_AC | ≥ 0.5 mm (mínimo), recomendado 1.0 mm | Rail 5V lleva hasta 615 mA pico |
| Ancho traza GND_DC | ≥ 0.5 mm o plano de cobre | Retorno de corriente para todo el subsistema DC |
| Planos de tierra | GND_AC y GND_DC **separados** | Se conectan SOLO internamente a través del HLK-5M05 pin 3 |
| Silkscreen en slot | "ISOLATION BARRIER 3000V — DO NOT BRIDGE" | Marcado de seguridad obligatorio para producción |
| Montaje TF1 (M1) | En contacto mecánico con carcasa HLK-5M05 | Fusible térmico debe detectar sobrecalentamiento de la fuente |

### 6.3 Consideraciones de Seguridad para PCB

- **No rutear trazas** ni ubicar componentes dentro de 1 mm del borde del slot de aislamiento
- **No colocar vías** debajo del cuerpo del HLK-5M05 que conecten capas AC con DC
- El HLK-5M05 es el componente más alto (**15 mm**) — define la altura mínima del enclosure
- **Thermal relief** en pads THT del HLK-5M05 para facilitar soldadura manual
- En ambas caras del slot, marcar zonas con **silkscreen**: "AC ZONE ⚡ DANGER" y "DC ZONE SELV"
- Los pines del HLK-5M05 tienen **1 mm de diámetro** — usar pads de 1.8 mm mínimo con anular ring ≥ 0.3 mm

---

## 7. Conexión con Otros Módulos

### 7.1 Entrada: Módulo 1 → Módulo 2

```
Módulo 1 (Protecciones AC)                    Módulo 2 (HLK-5M05)
════════════════════════                       ══════════════════════

L1 pin 2 (salida bobinado A) ──── AC_L_OUT ──────► HLK-5M05 pin 1 (AC)
L1 pin 3 (salida bobinado B) ──── AC_N_OUT ──────► HLK-5M05 pin 2 (AC)

Si L1 está bypassed (prototipo):
NODO_L (post-NTC1) ──────────── AC_L_OUT ──────► HLK-5M05 pin 1 (AC)
NODO_N (J_AC pin 2) ─────────── AC_N_OUT ──────► HLK-5M05 pin 2 (AC)
```

**Nota:** el HLK-5M05 es **no polarizado** — los pins 1 y 2 son intercambiables (L y N pueden ir en cualquier pin). Esto es porque internamente tiene un puente rectificador de onda completa.

### 7.2 Salida: Módulo 2 → Módulo 7 → Sistema

```
Módulo 2 (HLK-5M05)                Módulo 7 (OR-Diode)              Sistema
══════════════════                  ══════════════════               ════════

HLK-5M05 pin 4 ── +5V_AC ──► D_AC ánodo (SS14) ──► cátodo ──┐
                                                                ├── +5V_COMBINED
                              D_USB ánodo (SS14) ──► cátodo ──┘      │
                              (desde M6, USB VBUS via PTC1)          │
                                                                      ├──► ME6211 VIN (M4) → 3.3V
                                                               [C_bulk 100µF]
                                                                      ├──► K1 Coil+ (M3) → Relé
                                                                      │
                                                                    GND_DC
```

**Flujo de potencia:** HLK-5M05 → +5V_AC → D_AC (V_F ≈ 0.45V) → +5V_COMBINED (~4.55V) → ME6211 LDO → +3.3V para ESP32 y periféricos.

### 7.3 GND_DC — Referencia de Tierra DC

El pin 3 del HLK-5M05 establece el **único punto de referencia de tierra** para toda la zona DC (SELV). Todos los módulos DC se conectan a este rail:

| Módulo | Componentes conectados a GND_DC |
|---|---|
| **M2** (este módulo) | C_out1 (−), C_out2 pad 2, C_out3 pad 2 |
| **M3** (Relé driver) | Q1 emisor, R2 pull-down, D1 ánodo (flyback) |
| **M4** (LDO 3.3V) | U2 VSS (pin 2), C4, C5, C6 |
| **M5** (ESP32-C3) | Múltiples pins GND + pad inferior, C_vdd1, C_vdd2, C_EN |
| **M6** (USB-C) | Shell USB, GND pins, C_USB1, C_USB2 |
| **M7** (OR-Diode) | C_bulk (−), nodo común retorno |
| **M8** (PC817 DC side) | U3 pin 3 (emisor fototransistor), C_deb |
| **M9** (Sensores/LEDs) | NTC GND, LED cátodo, SW1/SW2 retorno |

**Punto crítico:** GND_DC **NO tiene conexión eléctrica** con el neutro AC ni con ningún punto de la zona AC. La única conexión entre zona AC y zona DC es **magnética** (a través del transformador interno del HLK-5M05) y **óptica** (a través del optoacoplador interno y del PC817).

---

## 8. Checklist de Verificación Pre-Fabricación

- [ ] Verificar disponibilidad de cada LCSC code antes de ordenar: C209907, C43803, C15850, C49678
- [ ] Confirmar que las dimensiones del HLK-5M05 (34×20×15 mm) caben en el layout del PCB
- [ ] Verificar que el slot fresado de 2 mm pasa bajo el cuerpo del HLK-5M05 entre pins AC (1,2) y DC (3,4)
- [ ] Medir creepage ≥ 6.5 mm a través del slot en el layout (entre cobre zona AC y cobre zona DC)
- [ ] Medir clearance ≥ 5.0 mm (gap de aire) a través del slot
- [ ] Confirmar que TF1 (Módulo 1) está posicionado en contacto físico con la carcasa del HLK-5M05
- [ ] Verificar que C_out1, C_out2 y C_out3 están dentro de 5 mm de los pins 3 y 4 del HLK-5M05
- [ ] Confirmar orden de colocación: C_out3 (100nF) más cerca de pins → C_out2 (10µF) → C_out1 (100µF)
- [ ] Confirmar que **ninguna traza ni vía** cruza el slot de aislamiento en ninguna capa
- [ ] Verificar que los planos GND_AC y GND_DC están completamente separados (sin puentes)
- [ ] Comprobar conexión correcta: +5V_AC → D_AC ánodo (SS14, Módulo 7)
- [ ] Revisar silkscreen: marcas de zona AC/DC y texto de barrera de aislamiento presentes
- [ ] Para producción UL: evaluar sustitución por MeanWell IRM-05-5 (certificado UL/TÜV/CB)
