# Módulo 3: Relé de Potencia + Driver BJT + Snubber de Contactos — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-17
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 3 es el **actuador de potencia** del Smart Relay. Su función es conmutar una carga AC de 110/220V (hasta 16 A resistivos) bajo el control lógico del ESP32 (3.3V), manteniendo al mismo tiempo un aislamiento galvánico completo entre el lado lógico y el lado de potencia. Sus responsabilidades específicas son:

1. **Conmutar** la línea AC (L) hacia el terminal de carga `J_LOAD` mediante un relé electromecánico Hongfa HF115F-I con contactos de AgSnO₂ y rating de inrush de 120 A / 20 ms.
2. **Traducir** la señal lógica de 3.3V del ESP32 (GPIO4) a una corriente de bobina de 80 mA @ 5V mediante un driver BJT SS8050 en configuración emisor común saturado.
3. **Proteger** el transistor driver contra el pico inductivo de apagado de la bobina mediante un diodo flyback Schottky SS14.
4. **Garantizar seguridad** durante el arranque del ESP32 (GPIO en alta impedancia) mediante una resistencia de pull-down que mantiene el relé desenergizado hasta que el firmware toma control explícito.
5. **Suprimir arco eléctrico** en los contactos del relé al abrir cargas con pequeña componente inductiva mediante una red snubber RC (100 Ω + 10 nF X2).
6. **Mantener** el aislamiento galvánico de 5000 Vrms bobina↔contactos provisto por el relé, cruzando físicamente la barrera de aislamiento del PCB entre la zona DC (pins 1, 2 del relé) y la zona AC (pins 5, 6, 7, 8).

### Entradas del módulo

| Señal | Origen | Descripción |
|---|---|---|
| `+5V_COMBINED` | M7 (OR-diode AC/USB) | Rail de 5V que alimenta la bobina del relé |
| `GPIO4` | M5 (ESP32-C3/C6) | Señal lógica 3.3V de control (HIGH = relé ON) |
| `AC_L_OUT` | M1 (Entrada AC y protecciones) | Línea AC filtrada, entra al contacto COM del relé |
| `AC_N_OUT` | M1 (Entrada AC y protecciones) | Neutro AC, pasa directo al terminal `J_LOAD` pin 2 (no se conmuta) |
| `GND_DC` | M2 (HLK-5M05) | Referencia de tierra del lado DC (retorno del emisor de Q1) |

### Salidas del módulo

| Señal | Destino | Descripción |
|---|---|---|
| `J_LOAD` pin 1 | Carga externa (bombillo, tomacorriente, etc.) | Línea AC conmutada (L cuando relé ON, flotante cuando relé OFF) |
| `J_LOAD` pin 2 | Carga externa | Neutro AC directo (sin conmutación) |

El Módulo 3 es, junto con el Módulo 2 (HLK-5M05) y el Módulo 8 (PC817), uno de los tres componentes del diseño que **cruzan físicamente la barrera de aislamiento** del PCB.

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Principio del Relé Electromecánico

Un relé electromecánico es un interruptor accionado magnéticamente. Su operación se basa en tres etapas físicas:

1. **Conversión eléctrica → magnética:** al aplicar una tensión a la bobina (5 VDC nominal en el HF115F-I), fluye una corriente de 80 mA a través de sus ~62.5 Ω. Esta corriente genera un campo magnético H en el núcleo de hierro dulce de la bobina, proporcional al producto N×I (número de vueltas × corriente).

2. **Conversión magnética → mecánica (fuerza):** el flujo magnético Φ del núcleo atrae una **armadura móvil** de hierro dulce. La fuerza de atracción es F = (Φ² × A) / (2 × μ₀ × g²), donde A es el área de polo y g es el gap de aire. Esta fuerza vence la oposición del resorte de retorno.

3. **Actuación mecánica de contactos:** la armadura, al moverse, cierra físicamente los contactos NO (Normally Open). El HF115F-I tiene **contactos bifurcados** — dos pares de contactos en paralelo accionados por la misma armadura: el nodo COM (pines 6+8 unidos externamente) toca el nodo NO (pines 5+7 unidos externamente), estableciendo continuidad eléctrica entre ellos. Los contactos son pastillas de **plata-óxido de estaño (AgSnO₂)** con resistencia de contacto típica < 100 mΩ (datasheet).

**Al desenergizar la bobina:**
- El campo magnético colapsa
- La energía almacenada en la inductancia de la bobina (~50 mH) debe disiparse — si no hay camino, genera un pico de tensión de hasta varios cientos de voltios (V = −L × dI/dt)
- El resorte de retorno abre los contactos, separando COM y NO físicamente
- Durante la separación, el aire entre los contactos se ioniza momentáneamente → **arco eléctrico**

**Parámetros clave del HF115F-I/005-1HS3A:**

| Parámetro | Valor | Implicación |
|---|---|---|
| Voltaje de operación (pickup) | 3.50 V (70% de 5V) | La bobina debe recibir ≥ 3.5 V para cerrar (datasheet) |
| Voltaje de desconexión (dropout) | 0.5 V (10% de 5V) | Se abre al caer debajo de 0.5 V |
| Resistencia de bobina | 62 Ω ±10% @ 23 °C | Determina corriente de bobina |
| Potencia de bobina | ~400 mW | 80 mA × 5 V = 0.4 W (datasheet) |
| Inductancia de bobina | ~50 mH (estimada) | Determina energía de flyback |
| Configuración de contactos | 1 Form A (SPST-NO) bifurcado | Dos contactos NO en paralelo, 6 pines totales |
| Rating de contactos | 16 A / 250 VAC resistiva @ 85 °C | Carga máxima continua (UL/CUL/VDE) |
| Inrush rating | 120 A / 20 ms | Tolerancia a LED drivers capacitivos |
| Vida mecánica | 10⁷ ciclos | Sin carga — limitado por el resorte |
| Vida eléctrica | 7.5 × 10⁴ ciclos @ 16 A 250 VAC resistiva | Datasheet (1s on / 9s off, 23 °C) |
| Aislamiento bobina↔contactos | **5000 VAC / 1 min** (hi-pot) | Cruce seguro de barrera SELV (datasheet) |
| Aislamiento contactos abiertos | 1000 VAC / 1 min (hi-pot) | Entre COM y NO cuando están abiertos |
| Surge voltage (coil↔contacts) | 10 kV (1.2/50 µs) | Tolerancia a picos de red y rayos |
| Creepage interno (coil↔contacts) | 10 mm | IEC 60335-1 (reinforced insulation) |
| Tiempo de operación | ≤ 15 ms (datasheet) | Retardo desde energizar hasta cierre |
| Tiempo de liberación | ≤ 8 ms (datasheet) | Retardo desde desenergizar hasta apertura |

### 2.2 Por Qué un BJT NPN en Configuración Emisor Común Saturado

El ESP32 NO puede alimentar directamente la bobina del relé por dos razones:
- **GPIO del ESP32 entrega máximo 40 mA @ 3.3V** — la bobina requiere 80 mA @ 5V (factor 2x en corriente y 1.5x en voltaje)
- **Conectar 5V directamente al GPIO dañaría el ESP32** — sus pines no toleran más de 3.3V

La solución estándar es un **transistor como interruptor**. Elegimos el **SS8050 NPN BJT** en configuración emisor común saturado. El principio:

- **Emisor a GND_DC** (pin 3 del SS8050 en SOT-23)
- **Colector al lado frío de la bobina** (pin 1 del relé) — el lado caliente (pin 2) está permanentemente al +5V. Como la bobina es no polarizada, esta asignación es convencional — podrían invertirse.
- **Base controlada por el GPIO4 del ESP32** a través de R1

Cuando GPIO4 = HIGH (3.3V):
- Corriente fluye por R1 hacia la base del SS8050
- El transistor entra en **saturación** — V_CE(sat) ≈ 0.2 V
- El circuito equivalente colector-emisor es prácticamente un corto a GND
- La bobina recibe V_rail − V_CE(sat) = 5.0 − 0.2 = **4.8 V**
- 4.8 V > 3.5 V (pickup) ✓ — el relé cierra con margen (37% sobre pickup)

Cuando GPIO4 = LOW (0V) o Hi-Z (con R2 tirando a GND):
- No hay corriente de base
- El transistor está en **corte** — I_C ≈ 0
- La bobina queda efectivamente desconectada → el relé abre

**¿Por qué NPN y no PNP?**
El NPN en low-side (emisor a GND, colector a la carga) es la configuración más simple y económica para conmutar cargas referenciadas a +V. No requiere level-shifting para controlar la base desde un GPIO que maneja 0-3.3V. El PNP requeriría un segundo transistor de level-shift para poder apagarlo con 3.3V cuando la fuente es 5V.

**¿Por qué BJT y no MOSFET?**
- A 80 mA la diferencia de eficiencia entre BJT saturado (V_CE=0.2V) y MOSFET (R_DS(on)×I = típ 0.1Ω × 0.08A = 8 mV) es irrelevante — 16 mW vs 0.6 mW en una aplicación de bajo duty cycle
- El BJT SS8050 cuesta ~$0.01; un MOSFET equivalente cuesta ~$0.05
- El BJT no requiere protección ESD especial en la base; los MOSFETs con V_GS_max=20V son sensibles a ESD
- **La única ventaja del MOSFET sería si el GPIO fuese 3.3V directo al gate sin pull-down**, pero el diseño ya requiere R1+R2 por seguridad de boot — no hay ahorro real de componentes

### 2.3 Cálculos del Driver — Saturación Garantizada

**Corriente de colector:**
- I_C = I_bobina = V_rail / R_coil = 5.0 V / 62.5 Ω = **80 mA**

**Corriente de base mínima para saturación:**
- Del datasheet del SS8050, hFE mínimo @ 80 mA = 100
- Por lo tanto I_B(min) = I_C / hFE = 80 / 100 = 0.8 mA
- Pero esto NO es suficiente — en saturación profunda se requiere **overdrive**: la base debe recibir 2–3× la corriente mínima para garantizar V_CE(sat) bajo en todo el rango de temperatura, tolerancias de fabricación y envejecimiento
- **Factor de overdrive elegido: 3** → I_B = 2.4 mA

**Cálculo de R1 (resistencia de base):**
- Con R2 presente (10 kΩ a GND), el voltaje en la base NO es 3.3V directo — hay un divisor
- V_base_efectivo = V_GPIO × R2 / (R1 + R2) = 3.3 × 10k / (1k + 10k) = **3.0 V**
- V_BE(on) del SS8050 @ 80 mA ≈ 0.7 V
- Caída en R1 = V_GPIO − V_base = 3.3 − 3.0 = 0.3 V — **NO**, este cálculo es incorrecto para el divisor cuando la base conduce

**Análisis correcto con R1/R2 y base conduciendo:**
- Cuando Q1 está en saturación, V_base = V_BE(on) ≈ 0.7 V (clamp por la unión base-emisor)
- Corriente por R1: I_R1 = (V_GPIO − V_BE) / R1 = (3.3 − 0.7) / 1.0 kΩ = **2.6 mA**
- Corriente por R2 (desde base a GND): I_R2 = V_BE / R2 = 0.7 / 10 kΩ = **0.07 mA**
- Corriente de base real: I_B = I_R1 − I_R2 = 2.6 − 0.07 = **2.53 mA**
- Factor de saturación = I_B × hFE / I_C = 2.53 × 100 / 80 = **3.16x** ✓ (supera el factor 3 objetivo)

**R1 = 1.0 kΩ (E24 estándar):** valor óptimo. Un valor menor (ej. 680 Ω) sobre-satura sin beneficio y aumenta el consumo del GPIO; un valor mayor (ej. 1.5 kΩ) reduce el factor de saturación a ~2x, arriesgando V_CE(sat) más alto y calentamiento del Q1.

### 2.4 Divisor R1/R2 — Operación Normal vs. Estado de Boot

La resistencia R2 de pull-down entre base y GND es **crítica para la seguridad** durante el arranque del ESP32.

**Fase 1: Pre-arranque y boot del ESP32 (GPIO4 = Hi-Z):**

Durante los primeros ~50-200 ms tras aplicar alimentación, los pines GPIO del ESP32 están en **alta impedancia** (tri-state) antes de que el firmware configure su dirección. En este estado:

```
  GPIO4 = Hi-Z (desconectado efectivamente)
     │
  [R1 = 1 kΩ]
     │
     ●── V_base
     │
  [R2 = 10 kΩ]
     │
    GND
```

- Sin R2: V_base podría flotar a cualquier valor por ruido capacitivo o fugas → riesgo de activación espuria del relé
- Con R2: V_base = 0 V (R1 ve un nodo flotante en un extremo y R2 a GND en el otro → la corriente que podría inducir ruido se drena a través de R2)
- Resultado: **Q1 en corte, bobina desenergizada, relé abierto** ✓

Esto es esencial: **el relé nunca debe cerrarse espontáneamente al encender el equipo**, especialmente en aplicaciones residenciales donde una activación involuntaria podría encender un aparato sin consentimiento del usuario.

**Fase 2: Operación normal GPIO4 = HIGH (3.3V):**

```
  GPIO4 = 3.3V
     │
  [R1 = 1 kΩ]  →  2.53 mA
     │
     ●── V_base = 0.7 V (clamped por V_BE)
     │
  [R2 = 10 kΩ]  ←  0.07 mA (fuga a GND)
     │
    GND
```

- Q1 en saturación profunda → relé cerrado ✓

**Fase 3: Operación normal GPIO4 = LOW (0V):**

```
  GPIO4 = 0V (drive LOW del ESP32)
     │
  [R1 = 1 kΩ]
     │
     ●── V_base = 0 V (GPIO activo tira a GND)
     │
  [R2 = 10 kΩ]
     │
    GND
```

- I_B = 0, Q1 en corte, relé abierto ✓

**¿Por qué R2 = 10 kΩ y no, por ejemplo, 4.7 kΩ?**
- Valor demasiado bajo: más consumo durante operación normal (V_BE/R2 = 0.7/4.7k = 0.15 mA vs 0.07 mA con 10k)
- Valor demasiado alto (>22 kΩ): menor inmunidad a ruido durante boot — corrientes de fuga en el orden de µA podrían generar décimas de voltio en V_base
- **10 kΩ es el punto óptimo**: drena corrientes de fuga del orden de µA a mV (ruido despreciable) sin añadir consumo sensible

### 2.5 Diodo Flyback D1 (SS14 Schottky)

**El problema:** cuando Q1 se apaga, la corriente por la bobina (80 mA) NO puede interrumpirse instantáneamente — una inductancia almacena energía en su campo magnético (E = ½LI²) y se opone a cambios de corriente:

V_L = −L × dI/dt

Si dI/dt se hace muy grande (interrupción abrupta), V_L crece sin límite hasta que algo cede. Sin protección, este pico puede alcanzar **cientos de voltios** — suficiente para perforar la unión colector-emisor del SS8050 (V_CEO(max) = 25 V).

**La solución — diodo flyback:**

```
                   +5V
                    │
           ┌────────┤
           │        │
       [K1 Bobina]  │
           │        │
           ├────────┤ ◄── D1 (SS14) cátodo a +5V, ánodo al colector
           │        │      Durante OFF: camino de circulación para la corriente de bobina
           │        │
      Q1 Colector   │
```

Cuando Q1 está ON: D1 está polarizado en reversa (ánodo a 0.2V = V_CE(sat), cátodo a 5V) → NO conduce.

Cuando Q1 se apaga: la corriente de la bobina busca un camino para seguir fluyendo. D1 se polariza en directa y la corriente circula en el lazo **bobina → D1 → +5V rail → bobina**, disipando la energía almacenada como calor en la resistencia óhmica de la bobina misma.

**Voltaje clampado:** V_clamp = V_rail + V_F(D1) = 5.0 + 0.5 = **5.5 V** en el colector de Q1. Muy por debajo del V_CEO de 25 V del SS8050 ✓.

**¿Por qué Schottky SS14 y no 1N4148/1N4007?**

| Parámetro | SS14 Schottky | 1N4148 Si | 1N4007 Si |
|---|---|---|---|
| V_F @ 0.1 A | **0.4–0.5 V** | 0.7–0.9 V | 0.9–1.0 V |
| Trr (recovery) | **Muy rápido (ns)** | 4 ns | 30 µs |
| V_R_max | 40 V | 75 V | 1000 V |
| I_F rating | 1 A | 0.3 A | 1 A |
| Encapsulado | SMA (compacto) | SOD-123 | SMA |

El SS14 da un **clamp más limpio** (menor V_clamp = menor estrés en Q1) y tiempo de recuperación despreciable. El 1N4148 también funciona pero deja el colector en 5.7V en vez de 5.5V. El 1N4007 también funciona pero es overkill (1000V de V_R) — aunque en la práctica se usa comúnmente por bajo costo.

**Energía disipada por ciclo:**
- E = ½ × L × I² = ½ × 50 mH × (80 mA)² = 0.16 mJ
- En forma de pulso de corriente: pico de 80 mA decayendo exponencialmente con τ = L/R_coil = 50 mH / 62.5 Ω = **0.8 ms**
- Disipación en el SS14: P = V_F × I_avg × duty ≈ despreciable (< 0.01 mW promedio a 1 ciclo/seg)

### 2.6 Snubber RC en Contactos — Supresión de Arco y Anti-Flicker

**El problema del arco eléctrico al abrir contactos bajo carga:**

Cuando los contactos del relé comienzan a separarse con corriente circulando, el aire entre ellos se ioniza por el campo eléctrico y por efecto termoiónico — se forma un **arco de plasma** que sostiene la corriente a pesar de la separación mecánica. Este arco:
- Quema el material de contactos (erosión progresiva)
- Genera EMI (interferencia electromagnética radiada)
- Reduce la vida útil del relé (cargas inductivas pueden reducir de 10⁵ a 10³ ciclos)

**La solución — red snubber RC en paralelo con los contactos:**

```
           K1 COM (pines 6+8)          K1 NO (pines 5+7)
                 │                              │
        AC_L ────┤                              ├──── LOAD_OUT (a J_LOAD)
                 │                              │
                 ├── [R_snub 100Ω 1W] ─────────┤
                 │                              │
                 └── [C_snub 10nF X2 275VAC] ──┘
```

**Cómo funciona:**
1. Con contactos cerrados: la corriente fluye por los contactos (baja resistencia ~30 mΩ); el snubber ve tensión ≈ 0 → corriente despreciable por el snubber.
2. Al momento de abrir: la corriente de la carga (si es inductiva) genera un dV/dt alto entre contactos. Esta tensión carga rápidamente C_snub a través de R_snub, desviando energía del arco.
3. El arco se extingue antes de que llegue a sostenerse → vida útil extendida.

**Análisis del valor RC elegido — por qué 10 nF y no 100 nF:**

Con los contactos **abiertos**, el snubber actúa como una impedancia en serie entre L y LOAD_OUT. Esto causa una **corriente de fuga** permanente hacia la carga, incluso con el relé OFF:

- Con C_snub = 100 nF:
  - Z_C @ 60 Hz = 1 / (2π × 60 × 100 nF) = **26.5 kΩ**
  - I_fuga @ 110 V = 110 / 26.5 kΩ = **4.15 mA**

- Con C_snub = 10 nF:
  - Z_C @ 60 Hz = 1 / (2π × 60 × 10 nF) = **265 kΩ**
  - I_fuga @ 110 V = 110 / 265 kΩ = **0.41 mA**

**Por qué importa la corriente de fuga: ghost flickering en LEDs**

Los drivers LED económicos (bombillos de bajo costo) tienen un umbral de **corriente mínima de sostenimiento** típicamente entre 1–3 mA. Si fluye más corriente de fuga que ese umbral, el driver se "enciende parcialmente" y el LED **parpadea débilmente** incluso con el interruptor apagado — el "ghost flicker" tan común en instalaciones residenciales modernas.

- 4.15 mA (con 100 nF) → ghost flicker **garantizado** en LEDs baratos
- 0.41 mA (con 10 nF) → por debajo del umbral, sin parpadeo

**Por qué X2 275 VAC y no un cerámico común:**

El C_snub está directamente conectado a la línea AC. Si falla en cortocircuito, causa un cortocircuito fase-neutro potencialmente incendiario. Los capacitores **clase X2** están diseñados para fallar en **circuito abierto** (film plástico auto-sanante) — una falla segura. Su rating de 275 VAC contiene el peor caso de sobretensión sostenida en la red (110V × 2.5 = 275V conforme IEC 60384-14). El uso de capacitores sin clasificación de seguridad en esta posición es una **violación de código eléctrico**.

**Disipación en R_snub:**
- Tensión RMS sobre el snubber con contactos abiertos: V_snub ≈ V_AC × Z_C/(Z_C+R) ≈ 110V (R despreciable vs Z_C)
- I_RMS por el snubber: ~0.41 mA
- P en R_snub = I² × R = (0.41 mA)² × 100 Ω = **17 µW** — ridículamente baja
- Se elige R_snub **1W** no por disipación continua sino por **capacidad de pulso** durante los eventos de apertura de contacto (pulsos de decenas de amperios durante microsegundos)

### 2.7 Rating de Inrush — Compatibilidad con LED Drivers Capacitivos

Los bombillos LED modernos contienen un driver AC-DC con un **capacitor de bulk de entrada** (típ. 4.7–10 µF electrolítico). Al cerrar el relé en el instante del pico de AC (~155 V), este capacitor descargado se carga en microsegundos a través de una impedancia de línea muy baja:

I_inrush_pico = V_pico / Z_línea ≈ 155 V / 1.3 Ω ≈ **120 A durante ~200 µs**

Esta corriente de inrush **fuunde los contactos** de un relé mal elegido — literalmente los funde y los suelda cerrados. Es la causa #1 de falla en relés residenciales sin rating adecuado.

**El HF115F-I tiene rating explícito de 120 A / 20 ms** — diseñado específicamente para esta aplicación. Otros relés (como el Songle SRD-05 usado en muchos productos de bajo costo) **NO publican este rating** porque sus contactos AgCdO/AgNi no lo toleran. Ver la ingeniería inversa de la ESP32-C6_Relay_X1 donde se documentó la soldadura de contactos tras ~50 ciclos con una sola bombilla LED.

### 2.8 Cruce de la Frontera de Aislamiento

El relé K1 **cruza físicamente la barrera de aislamiento** del PCB. Sus pines están divididos en dos zonas eléctricas completamente separadas:

```
              VISTA SUPERIOR DEL HF115F-I EN PCB
              ═════════════════════════════════

  ┌──────────────────┬─────────────────────────────┐
  │   ZONA DC        │ S       ZONA AC             │
  │  (SELV, segura)  │ L      (110/220 V peligrosa)│
  │                  │ O                           │
  │  pin 1 ───────── ┤ T      ┌── pin 5 (NO1)      │
  │   (Coil A1)      │        │                    │
  │                  │ F      ├── pin 6 (COM1)     │
  │   [cuerpo K1]    │ R      │                    │
  │                  │ E      ├── pin 7 (NO2)      │
  │  pin 2 ───────── ┤ S      │                    │
  │   (Coil A2)      │ A      └── pin 8 (COM2)     │
  │                  │ D                           │
  │                  │ O                           │
  └──────────────────┴─────────────────────────────┘
                     ▲
             Slot fresado 2 mm
           (SIN cobre, SIN vías)
                     │
                     ◄── 5000 Vrms aislamiento interno
                         (creepage 10 mm) del propio
                         cuerpo del relé — datasheet Hongfa
```

El aislamiento de **5000 Vrms** entre bobina (pines 1, 2) y contactos (pines 5, 6, 7, 8) está **incorporado en el relé mismo** — se logra mediante la separación mecánica entre la bobina enrollada y los contactos, más un encapsulado plástico que impone 10 mm de creepage interno (datasheet Hongfa). El diseño del PCB solo debe **respetar** esta frontera, no crearla.

**Reglas de diseño derivadas:**

1. Los pines 1 y 2 están en la zona DC; los pines 5, 6, 7, 8 están en la zona AC
2. **Ninguna traza puede cruzar el slot fresado** que separa estas zonas (el slot pasa bajo el cuerpo del relé, entre los 20.16 mm que separan los pines de bobina de los de contactos)
3. El creepage entre cobre de zona DC y cobre de zona AC debe ser ≥ 6.5 mm (IEC 62368-1 aislamiento reforzado + factor altitud Bogotá ×1.14)
4. No debe haber vías en la zona del slot
5. Los puentes pin 5 ═══ pin 7 y pin 6 ═══ pin 8 quedan enteramente en la zona AC (no cruzan la barrera)

El relé K1, junto con el HLK-5M05 (M2) y el PC817 (M8), son los **tres y solo tres componentes** del diseño que pueden cruzar la barrera. Cualquier otra conexión entre zonas es una violación crítica de seguridad.

---

## 3. Diagrama de Conexión Pin a Pin

### 3.1 Diagrama General del Módulo

El módulo se entiende mejor en **tres vistas**: primero un diagrama de bloques de alto nivel (qué entra, qué sale, qué hay dentro), luego el detalle del lado DC (el driver que mueve la bobina) y finalmente el detalle del lado AC (los contactos y el snubber). La barrera de aislamiento de **5000 Vrms** está **dentro del relé mismo** — separa eléctricamente el lado DC (bobina) del lado AC (contactos).

**Pinout oficial del HF115F-I/005-1HS3A según datasheet Hongfa (6 pines totales):**

| Pin | Función | Zona | Notas |
|---|---|---|---|
| **1** | Bobina (Coil A1) | DC | No polarizada — intercambiable con pin 2 |
| **2** | Bobina (Coil A2) | DC | No polarizada — intercambiable con pin 1 |
| **5** | Contacto bifurcado | AC | Se **une externamente con pin 7** (lado NO) |
| **6** | Contacto bifurcado | AC | Se **une externamente con pin 8** (lado COM) |
| **7** | Contacto bifurcado | AC | Se **une externamente con pin 5** (lado NO) |
| **8** | Contacto bifurcado | AC | Se **une externamente con pin 6** (lado COM) |

Los pines 3 y 4 **no existen** en este encapsulado — la numeración salta de 2 a 5. Los contactos son **bifurcados** (reforzados): dos pares de contactos mecánicamente paralelos accionados por la misma armadura. Para aprovechar el rating de 16 A deben unirse externamente pin 5 con pin 7, y pin 6 con pin 8. Si solo se usa un par (ej. 5-6), la capacidad de corriente se reduce a ~8 A.

#### Vista 1 — Qué se conecta a cada pin del K1 (orientación del símbolo en EasyEDA)

Este diagrama usa la **misma orientación del símbolo** que ves en EasyEDA Pro: pines **2, 6, 8 arriba** del componente y pines **1, 5, 7 abajo**. Las flechas muestran qué se conecta a cada pin, saliendo hacia arriba o hacia abajo según corresponda.

```
     +5V_COMBINED                          AC_L_OUT            AC_L_OUT
     (desde M7)                            (desde M1)          (desde M1)
         +                                     │                   │
     D1 cátodo                                 └─────── ┬ ─────────┘
         │                                              │
         │                                        puente externo
         │                                         (pin 6 ═══ pin 8)
         ▲                                              ▲
         │                                        ┌─────┴─────┐
         │                                        ▲           ▲
         │                                        │           │
       pin 2                                    pin 6       pin 8
      (Coil A2)                                 (COM1)      (COM2)
         ●                                        ●           ●
    ┌────┼────────────────────────────────────────┼───────────┼────┐
    │    │                                        │           │    │
    │    │                                        │           │    │
    │    │             K1                         └─── ┬ ─────┘    │
    │    │    HF115F-I/005-1HS3A                      │            │
    │    │    (5V, 16A, 1 Form A bifurcado)    contactos bifurcados│
    │    │                                                         │
    │    │                                                         │
    │    │     ════ AISLAMIENTO 5000 Vrms (interno) ════           │
    │    │                                                         │
    │    │                                        ┌─── ┬ ─────┐    │
    │    │                                        │           │    │
    └────┼────────────────────────────────────────┼───────────┼────┘
         ●                                        ●           ●
       pin 1                                    pin 5       pin 7
      (Coil A1)                                 (NO1)       (NO2)
         │                                        │           │
         │                                        ▼           ▼
         │                                        └─────┬─────┘
         │                                              │
         ▼                                        puente externo
                                                   (pin 5 ═══ pin 7)
    Q1 Colector                                         │
    (SS8050 pin 2)                                      ▼
         +                                        ┌─── LOAD_OUT ───┐
     D1 ánodo                                     │                │
                                                  ▼                ▼
                                            J_LOAD pin 1     snubber
                                            (carga AC)       R_snub + C_snub
                                                                    │
                                                                    ▼
                                                              (otro extremo
                                                               del snubber al
                                                               nodo AC_L_OUT
                                                               = pin 6+8)


    ┌──────── ZONA DC (segura) ─────────┐  ┌──────── ZONA AC (peligrosa) ────────┐
           pines 1 y 2 (bobina)                pines 5, 6, 7, 8 (contactos)
           corriente pequeña (80 mA)           corriente grande (hasta 16 A)
           lógica 3.3 V / 5 V                  110 / 220 VAC


    Neutro AC (AC_N_OUT desde M1) ───────────────────► J_LOAD pin 2
    (no pasa por el K1 — va directo a la carga)
```

**Resumen de 6 conexiones — una por pin:**

| Pin K1 | Se conecta a | Net | Zona |
|:---:|:---|:---:|:---:|
| **1** | Colector de Q1 + ánodo de D1 | `K1_COIL_NEG` | DC |
| **2** | Rail `+5V_COMBINED` (desde M7) + cátodo de D1 | `+5V_COMBINED` | DC |
| **5** | Puente a pin 7 + `LOAD_OUT` (a J_LOAD pin 1 + snubber) | `LOAD_OUT` | AC |
| **6** | Puente a pin 8 + `AC_L_OUT` (desde M1) + snubber | `AC_L_OUT` | AC |
| **7** | Puente a pin 5 (misma red que pin 5) | `LOAD_OUT` | AC |
| **8** | Puente a pin 6 (misma red que pin 6) | `AC_L_OUT` | AC |

**Componentes fuera del K1 pero dentro del Módulo 3:**

- **Lado DC (control):** Q1 (SS8050), D1 (SS14), R1 (1 kΩ), R2 (10 kΩ) — entre GPIO4 del ESP32 y los pines 1-2 del K1.
- **Lado AC (potencia):** R_snub (100Ω 1W), C_snub (10nF X2) en serie, en paralelo con los contactos del K1 (entre los dos nodos bifurcados).
- **Terminal de salida:** J_LOAD (KF128-7.5-2P) — pin 1 recibe `LOAD_OUT`, pin 2 recibe `AC_N_OUT` directo.
- **El neutro NO pasa por el K1** — va directo desde M1 a J_LOAD pin 2.

**Resumen de zonas:**
- **Lado DC (pines 1, 2):** corriente pequeña (80 mA), lógica 3.3V/5V, lado seguro al tacto.
- **Lado AC (pines 5, 6, 7, 8):** corriente grande (hasta 16A), 110/220V peligrosa, aislada por el cuerpo del relé (5000 Vrms).

#### Vista 2 — Detalle del lado DC (driver de la bobina)

Este es el circuito que convierte la señal lógica de 3.3V del ESP32 en 80 mA de corriente de bobina. Todo queda en la **zona DC** del PCB. La bobina del HF115F-I es **no polarizada** — cualquiera de los dos pines (1 o 2) puede ir al +5V; por convención tomamos pin 2 = lado alto (+5V) y pin 1 = lado conmutado por Q1.

```
                          +5V_COMBINED (desde M7)
                                   │
                                   ├───────────────────┐
                                   │                   │
                                   ▼                   │
                            K1 pin 2 (Coil A2)         │
                                   │                   │
                           ┌───────┴───────┐           │
                           │               │           │
                           │  BOBINA K1    │   D1 SS14 (flyback)
                           │  62 Ω         │           ▲
                           │  ~50 mH       │           │ cátodo
                           │  400 mW       │           │ (banda del SMA
                           └───────┬───────┘           │  hacia +5V)
                                   │                   │
                            K1 pin 1 (Coil A1)         │ ánodo
                                   │                   │
                                   ├───────────────────┘
                                   │
                                   ▼
                           Q1 Colector (SS8050 pin 2)
                                   │
                                   │
   GPIO4 ──►[R1:1kΩ]──► Q1 Base (SS8050 pin 1)
   (3.3V desde M5)             │
                        [R2:10kΩ] (pull-down anti-boot)
                               │
                              GND_DC
                               │
                     Q1 Emitter (SS8050 pin 3) ──► GND_DC


   Funcionamiento:
     • GPIO4 = 3.3V  → Q1 satura → K1 pin 1 cae a ~0.2V → bobina
                       recibe 4.8V > V_pickup 3.5V → relé CIERRA en < 15 ms
     • GPIO4 = 0V    → Q1 en corte → bobina desenergizada → relé ABRE en < 8 ms
     • GPIO4 = Hi-Z  → R2 tira la base a 0V → Q1 en corte → relé ABRE (seguro en boot)
     • D1: cuando Q1 se apaga, la bobina descarga su energía por D1 al +5V,
           protegiendo al colector de Q1 del pico inductivo

   Nota: pines 1 y 2 del K1 son intercambiables (bobina no polarizada);
         la convención aquí es pin 2 = +5V, pin 1 = colector. Si se invierte
         la bobina funciona igual, pero D1 debe mantener la misma polaridad
         relativa (cátodo siempre al lado que va al +5V).
```

#### Vista 3 — Detalle del lado AC (contactos bifurcados + snubber + salida)

Este es el circuito de potencia que conmuta la carga. Todo queda en la **zona AC** del PCB. El HF115F-I tiene **dos contactos SPST-NO en paralelo dentro del mismo encapsulado** (bifurcación): cuando la bobina energiza, ambos contactos cierran simultáneamente. Para aprovechar el rating de 16 A deben unirse externamente los pares (5+7) y (6+8).

```
                             ╔═══════════════════════════════╗
                             ║      Contactos K1             ║
                             ║   (bifurcados, dentro del relé)║
                             ╚═══════════════════════════════╝

                             pin 6           pin 8
                               │               │
          ┌────────────────────┴───────────────┴─────────┐
          │                                              │
   AC_L_OUT ──►  ●────── 6 ───┐    ┌─── 8 ──────●        │
   (desde M1)    (COM1)   ─ ─ │    │ ─ ─    (COM2)      │
                  armadura    │    │   armadura           │
                  contacto    │    │   contacto         (dos contactos
                   móvil      │    │    móvil            en paralelo,
                              ▼    ▼                     cierran juntos
                            [cuando bobina ON]           cuando bobina ON)
                              │    │
                  (NO1)   ─ ─ │    │ ─ ─    (NO2)
                    ●────── 5 ───┘    └─── 7 ──────●     │
                      │                               │   │
                      └───────────┬───────────────────┘   │
                                  │                       │
                             pin 5                    pin 7
                                  │                       │
          └───────────────────────┴───────────────────────┘
                                  │
                               LOAD_OUT  ──────► J_LOAD pin 1 (carga AC)


   Conexiones externas obligatorias (ponen los contactos en paralelo):
     • pin 6 ═══ pin 8   → AC_L_OUT entra a este nodo (COM bifurcado)
     • pin 5 ═══ pin 7   → LOAD_OUT sale de este nodo (NO bifurcado)


   Snubber en paralelo con los contactos (entre los dos nodos bifurcados):
                   nodo (6+8, COM)                  nodo (5+7, NO)
                         │                                │
                         ├──── [R_snub 100Ω 1W] ─────────┤
                         │                                │
                         ├──── [C_snub 10nF X2] ─────────┤
                         │                                │
                         (R_snub y C_snub en serie entre ambos nodos)


   AC_N_OUT ──────────────────────────────────────────────► J_LOAD pin 2
   (neutro desde M1: pasa directo, NO se conmuta — la carga queda en circuito
    cuando los contactos COM↔NO están cerrados; queda desconectada de L
    cuando están abiertos)


   Leyenda:
     ─ ─ ─  : contacto abierto (relé OFF)
     ────   : conexión eléctrica cerrada (relé ON)
     ═══    : conexión externa obligatoria entre pines bifurcados
     ╔═╗    : cuerpo físico del relé (encapsulado plástico sellado)
     ●      : pin del relé en el PCB
```

**En resumen:**
- El lado DC (Vista 2) le "pide" al relé que cierre mediante 80 mA por la bobina (pines 1 y 2).
- El lado AC (Vista 3) deja pasar la corriente a la carga cuando los contactos bifurcados (pares 5+7 y 6+8) se cierran.
- Los dos lados están completamente aislados eléctricamente — solo se comunican por la fuerza magnética del imán del relé (aislamiento de **5000 Vrms** según datasheet Hongfa).
- **Crítico:** los pines bifurcados (5+7 y 6+8) deben unirse externamente en el PCB. Si solo se usa un par de contactos, la capacidad de corriente se reduce a la mitad y el rating de inrush de 120 A/20 ms ya no aplica.

### 3.2 Tabla de Nets (Conexiones Eléctricas)

| Net Name | Nodos conectados | Zona | Descripción |
|---|---|---|---|
| `+5V_COMBINED` | M7 output, K1 pin 2 (Coil A2), D1 cátodo | DC | Rail 5V que alimenta el lado alto de la bobina del relé |
| `GPIO4` | ESP32 GPIO4 (M5), R1 pad 1 | DC | Señal lógica 3.3V de control |
| `Q1_BASE` | R1 pad 2, R2 pad 1, Q1 pin 1 (B) | DC | Nodo interno del divisor, V_base ≈ 0.7V cuando ON |
| `K1_COIL_NEG` | K1 pin 1 (Coil A1), D1 ánodo, Q1 pin 2 (C) | DC | Lado frío de la bobina, conmutado por Q1 |
| `GND_DC` | R2 pad 2, Q1 pin 3 (E), M2 GND | DC | Referencia de tierra DC |
| `AC_L_OUT` | M1 (salida post-protecciones), K1 pin 6, K1 pin 8, R_snub pad 1, C_snub pad 1 | AC | Línea AC de entrada al nodo COM bifurcado (pines 6+8 unidos) |
| `LOAD_OUT` | K1 pin 5, K1 pin 7, R_snub pad 2, C_snub pad 2, J_LOAD pin 1 | AC | Salida AC del nodo NO bifurcado (pines 5+7 unidos) hacia la carga |
| `AC_N_OUT` | M1 (neutro post-protecciones), J_LOAD pin 2 | AC | Neutro directo hacia la carga (no conmutado) |

### 3.3 Pinout del HF115F-I (vista inferior según datasheet Hongfa)

El HF115F-I/005-1HS3A tiene **6 pines totales** — la numeración salta de 2 a 5 (no existen pines 3 ni 4). Los pines 1 y 2 están en un extremo del cuerpo (bobina), y los pines 5, 6, 7, 8 están en el extremo opuesto (contactos bifurcados).

```
          VISTA INFERIOR (bottom view) - lado soldadura del PCB
          ═════════════════════════════════════════════════════

          29.0 mm (longitud del cuerpo)
     ◄──────────────────────────────────────────►

     ┌───────────────────────────────────────────────┐
     │                                               │
     │    ●                                 ●    ●   │ ─┐
     │   (2)                               (6)  (8)  │  │
     │                                               │  │ 12.7 mm
     │              HF115F-I/005-1HS3A               │  │ (ancho)
     │              5V 16A 1 Form A                  │  │
     │                                               │  │
     │    ●                                 ●    ●   │  │
     │   (1)                               (5)  (7)  │ ─┘
     │                                               │
     └───────────────────────────────────────────────┘
          ◄────── 20.16 mm ──────►  ◄─5.04mm─►
                                    ◄2.6mm►

     ┌─── BOBINA (zona DC) ───┐     ┌─── CONTACTOS BIFURCADOS (zona AC) ───┐
     │                        │     │                                      │
     │  Pin 1: Coil A1        │     │  Pin 5: NO1  (se une con pin 7)     │
     │  Pin 2: Coil A2        │     │  Pin 6: COM1 (se une con pin 8)     │
     │  (no polarizada,       │     │  Pin 7: NO2  (se une con pin 5)     │
     │   intercambiables)     │     │  Pin 8: COM2 (se une con pin 6)     │
     └────────────────────────┘     └──────────────────────────────────────┘

  Dimensiones: 29.0 × 12.7 × 15.7 mm (L × W × H) — datasheet Hongfa
  Pitch bobina↔contactos: 20.16 mm (cruza la barrera de aislamiento)
  Diámetro de pines: 1.3 mm THT
```

**Configuración de contactos — "1 Form A bifurcado":**

El símbolo EIA "1 Form A" indica SPST-NO (Single Pole Single Throw, Normally Open). La versión del HF115F-I tiene la particularidad de estar **bifurcada**: dos contactos independientes accionados por la misma armadura. La ventaja:

1. **Redundancia:** si un contacto acumula óxido o carbón, el otro mantiene conducción.
2. **Menor resistencia de contacto:** dos caminos en paralelo reducen R_contacto efectiva.
3. **Mayor capacidad de corriente:** los 16 A nominales se distribuyen entre ambos contactos (≈ 8 A cada uno).

**Para que el rating de 16 A (y el inrush de 120 A) sea válido es OBLIGATORIO unir externamente pin 5 con pin 7, y pin 6 con pin 8.** Si solo se cablea un par, la capacidad se reduce a la mitad.

### 3.4 Diagrama de Conexión Pin a Pin Detallado

```
K1 — HF115F-I/005-1HS3A (Relé THT 6 pines, 29×12.7×15.7 mm)
  Pin 1 (Coil A1) ◄──── K1_COIL_NEG (Q1 Collector + D1 ánodo)           [ZONA DC]
  Pin 2 (Coil A2) ◄──── +5V_COMBINED (desde M7) + D1 cátodo             [ZONA DC]
                         ─── (bobina no polarizada: 1 y 2 son intercambiables)
  Pin 5 (NO1)     ────► LOAD_OUT (unido con pin 7)                      [ZONA AC]
  Pin 6 (COM1)    ◄──── AC_L_OUT (unido con pin 8, desde M1)          [ZONA AC]
  Pin 7 (NO2)     ────► LOAD_OUT (unido con pin 5) → J_LOAD pin 1       [ZONA AC]
  Pin 8 (COM2)    ◄──── AC_L_OUT (unido con pin 6, desde M1)          [ZONA AC]

  [CRÍTICO: pines 5 y 7 deben unirse en el PCB (mismo net LOAD_OUT).
   Pines 6 y 8 deben unirse en el PCB (mismo net AC_L_OUT).
   Estos puentes externos activan los contactos bifurcados en paralelo,
   duplicando la capacidad de corriente y sostenen el inrush de 120 A/20 ms.]

Q1 — SS8050 NPN BJT (SOT-23)
  Pin 1 (Base)     ◄──── Q1_BASE (nodo común R1 + R2)                   [ZONA DC]
  Pin 2 (Collector)◄──── K1_COIL_NEG (K1 pin 1 + D1 ánodo)              [ZONA DC]
  Pin 3 (Emitter)  ────► GND_DC                                         [ZONA DC]
  [CRÍTICO: verificar pinout exacto en datasheet del fabricante
   LCSC C164886 — Changjiang Electronics usa convención estándar SOT-23]

D1 — SS14 Schottky (SMA / DO-214AC, cátodo marcado con banda)
  Ánodo   ◄──── K1_COIL_NEG (K1 pin 1, Q1 pin 2)                        [ZONA DC]
  Cátodo  ◄──── +5V_COMBINED (mismo nodo que K1 pin 2)                  [ZONA DC]
  [POLARIDAD CRÍTICA: invertir el D1 provoca cortocircuito al +5V
   tan pronto se energize el circuito — falla catastrófica del Q1]

R1 — Resistor base driver (1.0 kΩ ±5% 0402)
  Pad 1 ◄──── GPIO4 (ESP32 M5)                                          [ZONA DC]
  Pad 2 ◄──── Q1_BASE (base de Q1 + R2 pad 1)                           [ZONA DC]

R2 — Resistor pull-down anti-boot (10 kΩ ±5% 0402)
  Pad 1 ◄──── Q1_BASE (base de Q1 + R1 pad 2)                           [ZONA DC]
  Pad 2 ◄──── GND_DC                                                    [ZONA DC]

R_snub — Resistor snubber (100 Ω ±5% 1 W MFR axial, D3.3×L9 mm)
  Pad 1 ◄──── AC_L_OUT (nodo K1 pin 6 + pin 8)                        [ZONA AC]
  Pad 2 ◄──── LOAD_OUT (nodo K1 pin 5 + pin 7)                          [ZONA AC]
  [EN SERIE CON C_snub — orden indistinto entre R y C]

C_snub — Capacitor snubber (10 nF X2 310 VAC, film P=7.5 mm, KYET)
  Pad 1 ◄──── AC_L_OUT (nodo K1 pin 6 + pin 8, a través de R_snub)    [ZONA AC]
  Pad 2 ◄──── LOAD_OUT (nodo K1 pin 5 + pin 7)                          [ZONA AC]
  [CAPACITOR X2 OBLIGATORIO POR SEGURIDAD — fallas en cortocircuito
   de cerámicos convencionales en red AC causan incendios]

J_LOAD — Terminal de salida (KF128-7.5-2P, THT pitch 7.5 mm)
  Pin 1 ◄──── LOAD_OUT (K1 pines 5+7 unidos, snubber)                   [ZONA AC]
  Pin 2 ◄──── AC_N_OUT (desde M1, neutro directo)                     [ZONA AC]
  [Etiquetar silkscreen: "LOAD OUT — MAX 16A 250V AC"]
```

### 3.5 Barrera de Aislamiento — El Tercer Cruce del PCB

El K1 es el **tercer componente** del diseño que cruza físicamente la barrera de aislamiento del PCB, junto con el HLK-5M05 (M2) y el PC817 (M8):

| Componente | Aislamiento | Función | Pins zona DC | Pins zona AC |
|---|---|---|---|---|
| HLK-5M05 (U5, M2) | 3000 VAC | Potencia: AC → 5V DC | 3 (−Vo), 4 (+Vo) | 1, 2 (AC in) |
| PC817 (U3, M8) | 5000 Vrms | Señal: switch AC → GPIO5 | 3, 4 (fototransistor) | 1, 2 (LED) |
| **HF115F-I (K1, M3)** | **5000 Vrms** | **Potencia: 5V DC → AC** | **1, 2 (Coil A1/A2)** | **5+7 (NO), 6+8 (COM)** |

El slot fresado del PCB pasa bajo los tres componentes. El layout debe garantizar que los pins DC y AC de cada uno queden en lados opuestos del slot.

---

## 4. Presupuesto de Potencia / Térmico

### 4.1 Disipación por Componente

| Componente | Condición | Corriente | Tensión | Potencia | Observaciones |
|---|---|---|---|---|---|
| K1 bobina | Relé activado continuo | 80 mA | 5.0 V | **0.40 W** | Consumo dominante del módulo |
| Q1 (SS8050) | Saturado, I_C=80 mA | 80 mA | V_CE(sat) ≈ 0.2 V | 16 mW | Despreciable en SOT-23 (θ_JA ≈ 250 °C/W → ΔT ≈ 4 °C) |
| R1 (1 kΩ) | GPIO HIGH, conduce base | 2.6 mA | 2.6 V | 6.8 mW | Por debajo del 1/16W — margen 10x |
| R2 (10 kΩ) | Operación normal | 0.07 mA | 0.7 V | 0.05 mW | Despreciable |
| D1 (SS14) | OFF durante relé ON | 0 A | 4.8 V (reversa) | 0 W | Solo conduce ~1 ms tras apagar relé |
| R_snub (100 Ω 1W) | Contactos abiertos, 110 VAC | 0.41 mA RMS | Despreciable | ~17 µW | Pulsos de decenas de A durante apertura (transitorio) |
| C_snub (10nF X2) | Contactos abiertos | 0.41 mA RMS | 110 VAC | Reactiva, no disipa | Cap X2, fail-open seguro |
| **Total continuo** | Relé ON | 82.7 mA @ 5V | — | **~0.42 W** | Despreciable vs 5W del HLK-5M05 |

### 4.2 Rating de Carga — Lo Que el Módulo Puede Conmutar

| Tipo de carga | Rating continuo | Rating de inrush | Vida eléctrica estimada |
|---|---|---|---|
| Resistiva (incandescente, calefactor, horno) | **16 A / 250 VAC** | — | 10⁵ ciclos |
| LED driver capacitivo (bombillos) | 10 A | **120 A / 20 ms** ✓ | 10⁵ ciclos (gracias al rating de inrush) |
| Motor (ventilador, bomba) | **8 A / 250 VAC** | ~6× I_nom (~50 A arranque) | 3×10⁴ ciclos (reducción por inductancia) |
| Inductivo (solenoide, balasto magnético) | **5 A / 250 VAC** | Variable | 10⁴ ciclos |

**Limitaciones prácticas del diseño:**
- A 16 A continuos, las trazas de PCB deben tener ≥ 2.5 mm de ancho con cobre de 1 oz (o ≥ 1.5 mm con cobre de 2 oz)
- El terminal J_LOAD (KF128-7.5-2P) tiene rating de **15 A** — ligeramente inferior al relé. Para cargas continuas > 15 A usar terminal de mayor rating (KF301 10 A o terminales industriales)
- Cargas inductivas pesadas (motor > 1 HP) reducen la vida del relé a < 10⁴ ciclos incluso con snubber — para esos casos se recomienda un contactor externo

### 4.3 Cálculo de Vida Útil

**Escenario típico residencial:**
- Carga: bombillo LED 15 W @ 110 VAC → I = 0.14 A continuo
- Inrush: ~80 A durante ~100 µs (driver capacitivo)
- Activaciones: 10 ciclos/día (encendido + apagado nocturno × 5)
- Vida eléctrica @ esta carga: **> 5 × 10⁵ ciclos** (mucho mayor que 10⁵ rating porque la carga es muy inferior a la nominal)

Años estimados de vida útil:
- 5 × 10⁵ ciclos / 10 ciclos/día / 365 = **~137 años** ✓

Incluso para una carga pesada (aire acondicionado de ventana, 8 A inductivo, 10 ciclos/día):
- 10⁴ ciclos / 10 ciclos/día / 365 ≈ **2.7 años** — aceptable para producto residencial

### 4.4 Consideraciones Térmicas

El módulo no tiene disipación significativa — la bobina (0.4 W) se disipa sobre un área de ~29×12.7 = **368 mm²**, resultando en densidad de potencia de ~1.1 mW/mm². Elevación de temperatura del cuerpo del relé sobre ambiente: **< 10 °C**. El rango de operación del HF115F-I es −40 a +85 °C, más que suficiente para una caja empotrada en pared en Bogotá (35 °C ambiente).

---

## 5. BOM Completo — Módulo 3 con Códigos LCSC

Todos los componentes verificados para importación desde LCSC/JLCPCB y compatibles con EasyEDA Pro Edition.

### 5.1 Componente Principal

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 1 | K1 | Hongfa HF115F-I/005-1HS3A | Relé 5V 1 Form A (SPST-NO bifurcado) 16A, AgSnO₂, 120A/20ms inrush, 5000Vrms isol. | THT 6 pines, 29×12.7×15.7 mm | **C2976795** | Hongfa | HF115F-I/005-1HS3A | $2.20 |

### 5.2 Driver BJT y Protección de Bobina

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 2 | Q1 | SS8050 NPN BJT | V_CE=25V, I_C=1.5A, hFE 120-200 | SOT-23 | **C164886** | Changjiang Electronics (JSCJ) | SS8050-G | $0.01 |
| 3 | D1 | SS14 Schottky | 40V / 1A, V_F=0.5V, Trr<10ns | SMA (DO-214AC) | **C2480** | MDD (Microdiode Electronics) | SS14 | $0.02 |
| 4 | R1 | Resistor base | 1.0 kΩ ±5% 1/16W | 0402 SMD | **C11702** | UNI-ROYAL (Uniroyal Elec) | 0402WGF1001TCE | $0.001 |
| 5 | R2 | Resistor pull-down anti-boot | 10 kΩ ±5% 1/16W | 0402 SMD | **C25744** | UNI-ROYAL (Uniroyal Elec) | 0402WGF1002TCE | $0.001 |

### 5.3 Snubber de Contactos

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 6 | R_snub | Resistor snubber | 100 Ω ±1% 1W MFR (±50 ppm/°C) | Axial THT D3.3×L9 mm | **C172915** | YAGEO | MFR1WSFTE52-100R | $0.03 |
| 7 | C_snub | Capacitor snubber X2 | 10 nF X2 ±10% 310 VAC film | Box film THT, P=7.5 mm | **C2693738** | KYET | PX103K2C0702 | $0.05 |

### 5.4 Terminal de Salida

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 8 | J_LOAD | Terminal block salida | KF128-7.5-2P, 300V / 15A, tornillo | THT, pitch 7.5 mm | **C474954** | Cixi Kefa Elec | KF128-7.5-2P | $0.15 |

### 5.5 Componentes Alternativos / Equivalentes

Para cada componente, alternativa viable si el LCSC code primario no está disponible al momento del pedido:

| Ref | Primario | Alternativa 1 (económica) | Alternativa 2 (premium/certificada) | Notas |
|---|---|---|---|---|
| K1 | C2976795 (Hongfa HF115F-I/005-1HS3A) | C88396 (Omron G5LE-1-VD, 5V 10A SPDT, UL/VDE) | Panasonic AHES-1241 (5V 16A, UL, más caro $3-5) | G5LE tiene menor rating (10A vs 16A) pero UL listed. Usar sólo con cargas < 8A. Verificar footprint DIP-5 vs DIP-6 |
| Q1 | C164886 (SS8050) | C8575 (MMBT2222A NPN, 40V/600mA, SOT-23) | C20526 (BC817-40, 45V/500mA Philips/Nexperia) | MMBT2222A y BC817 tienen I_C menor (500-600 mA) pero de sobra para 80 mA. hFE similar. Pinout SOT-23 idéntico |
| D1 | C2480 (SS14 Schottky) | C81598 (1N4148W SOD-123, 100V/200mA, V_F=0.7V) | C169 (SS24, 40V/2A SMA, sobredimensionado) | 1N4148W funciona pero deja V_clamp=5.7V en vez de 5.5V. Si la PCB tiene footprint SMA, usar SS24 como drop-in |
| R1 | C11702 (UNI-ROYAL 1kΩ 0402 ±5%) | C21190 (YAGEO RC0402FR-071KL, 1kΩ ±1%) | Cualquier 0402 1kΩ de marca reconocida | El valor no es crítico ±20%; cualquier 1k sirve |
| R2 | C25744 (UNI-ROYAL 10kΩ 0402 ±5%) | C25803 (YAGEO RC0402FR-0710KL, 10kΩ ±1%) | C60818 (Walsin 10kΩ 0402) | Valor no crítico; cualquier 10k 0402 |
| R_snub | C172915 (YAGEO MFR1WSFTE52-100R, 100 Ω ±1% 1W, 3260+ en stock) | C17897 (YAGEO MFR-25FTE52-100R, 100 Ω ±1% 1/4W — sólo para pulsos pequeños) | C58275 (Vishay MRS25 100R 1W, alternativa premium) | Importante: debe ser de **película metálica (MFR)** con rating de pulso, NO de carbón. Capacidad de pulso: ≥ 100 A × 10 µs. **NO usar C176407 (es de 15 Ω, no 100 Ω — error histórico)** |
| C_snub | C2693738 (KYET PX103K2C0702 10nF X2 310V, P=7.5mm, 43k+ en stock, JLCPCB Extended con footprint EasyEDA) | C2979687 (Faratronic C43Q1103M60C000, 10nF **Y2** 300V, P=15mm — Y2 es safety class superior a X2, footprint más grande) | C3300476 (KEMET R46KF210000N0K 10nF X2 275V, P=10mm — calidad premium pero sin stock habitual y requiere solicitar footprint) | **NO USAR** C434188 (Jimson 100nF/275VAC): causa ghost flicker en LEDs. El valor 10 nF es crítico para mantener fuga < 1 mA |
| J_LOAD | C474954 (KF128-7.5-2P 15A) | C7245 (DG128-7.5-2P, clon idéntico, más barato) | C474953 (KF128-7.62-2P pitch 7.62mm — alternativa dimensional) | Verificar que el pitch en EasyEDA coincida con el que se ordene. El rating 15A es el limitante si el relé lleva 16A |

### 5.6 Resumen de Costo del Módulo 3

| Componente | Costo unitario (aprox.) |
|---|---|
| K1 Hongfa HF115F-I (C2976795) | $2.20 |
| Q1 SS8050 (C164886) | $0.01 |
| D1 SS14 (C2480) | $0.02 |
| R1 1kΩ 0402 (C11702) | $0.001 |
| R2 10kΩ 0402 (C25744) | $0.001 |
| R_snub 100Ω ±1% 1W MFR (C172915) | $0.03 |
| C_snub 10nF X2 310V (C2693738) | $0.05 |
| J_LOAD KF128-7.5-2P (C474954) | $0.15 |
| **TOTAL Módulo 3** | **~$2.53 USD** |

*Precios de LCSC en cantidades ≥ 10 unidades. Sujetos a cambio.*

El costo está dominado por el relé K1 (87% del total). Si se sustituye por un relé genérico no certificado (Songle SRD-05, ~$0.40), el costo baja a ~$0.73 pero **se pierden todos los ratings de seguridad y vida útil** — inaceptable para producto residencial con cargas LED.

---

## 6. Notas de Diseño para EasyEDA Pro

### 6.1 Importación de Componentes

Para importar cada componente en EasyEDA Pro:

1. Abrir **Library** → **LCSC Parts**
2. Buscar por código LCSC (ej: `C2976795` para K1)
3. Verificar que el footprint coincida exactamente con el encapsulado listado
4. Para componentes 3D (especialmente el K1), descargar el modelo STEP desde el fabricante (Hongfa, en este caso) si no está incluido — crítico para verificar altura total del PCB en el enclosure
5. Colocar en el esquemático y asignar la referencia (K1, Q1, D1, R1, R2, R_snub, C_snub, J_LOAD)

**Nota crítica sobre el HF115F-I:** el footprint de LCSC C2976795 debe coincidir con el datasheet Hongfa (1 Form A bifurcado, 6 pines):
- **Pines presentes:** 1, 2 (bobina, un extremo) + 5, 6, 7, 8 (contactos, otro extremo) — **no hay pines 3 ni 4**
- **Pitch bobina↔contactos:** 20.16 mm (esta distancia define dónde va el slot de aislamiento)
- **Pitch entre pines bifurcados:** 2.6 mm (entre 5-6 y entre 7-8), 5.04 mm (entre 6-8 y entre 5-7)
- **Diámetro de pin:** 1.3 mm — usar pads de 1.8 mm con anular ring ≥ 0.25 mm
- La huella (silkscreen) del cuerpo del relé debe tener dimensiones 29×12.7 mm — crítico para el slot de aislamiento

### 6.2 Reglas de Layout Críticas

| Regla | Valor | Razón |
|---|---|---|
| **Posición del K1** | Cruza el slot fresado 2 mm (entre los 20.16 mm bobina↔contactos) | Pines 1, 2 (bobina) en zona DC; pines 5, 6, 7, 8 (contactos) en zona AC |
| **Creepage bobina↔contactos del K1** (internamente en el slot) | ≥ 6.5 mm | IEC 62368-1 aislamiento reforzado + factor altitud Bogotá (×1.14) |
| **Puentes de bifurcación obligatorios** | pin 5 ═══ pin 7 (net LOAD_OUT); pin 6 ═══ pin 8 (net AC_L_OUT) | Activa los contactos bifurcados en paralelo — sin estos puentes solo se usa la mitad del rating de 16 A |
| **Posición de Q1, D1, R1, R2** | A < 5 mm de K1 pin 1 (Coil A1) | Minimizar loop area de la corriente de conmutación de la bobina |
| **Polaridad de D1** | Cátodo (banda) hacia **+5V rail** (K1 pin 2), ánodo hacia K1 pin 1 | Invertir polaridad destruye Q1 al energizar |
| **Orientación de Q1 (SS8050 SOT-23)** | Verificar pinout: B=1, C=2, E=3 | Convención estándar LCSC; confirmar en datasheet C164886 |
| **Posición del snubber** | R_snub + C_snub en serie entre nodo (pin 6+pin 8) y nodo (pin 5+pin 7), en zona AC | Snubber debe estar sobre los contactos, NO sobre la carga |
| **Orientación del snubber** | R y C en serie, orden indistinto | El orden R-C vs C-R no afecta el comportamiento |
| **Ancho traza +5V_COMBINED y bobina** | ≥ 0.3 mm | Lleva 80 mA + margen |
| **Ancho traza GPIO4** | ≥ 0.15 mm (señal lógica) | Solo lleva 2.6 mA |
| **Ancho traza AC_L_OUT, LOAD_OUT** | ≥ 2.5 mm (cobre 1 oz) o ≥ 1.5 mm (cobre 2 oz) | Lleva hasta 16 A — crítico para evitar fusión en overload |
| **Ancho traza AC_N_OUT** | ≥ 2.5 mm | Lleva misma corriente que L |
| **Zona del slot** | Sin cobre ni vías en ancho de 2 mm centrado en el slot | Barrera física de aislamiento |
| **Vías bajo el cuerpo de K1** | Prohibidas en zona del slot | Comprometen el aislamiento |
| **Silkscreen en K1** | Indicador de Pin 1 + ref K1 + etiqueta "RELAY" | Identificación de polaridad |
| **Silkscreen en J_LOAD** | "LOAD OUT MAX 16A 250V AC" + indicador L / N | Seguridad en producción |
| **Thermal relief en pads THT de K1** | Sí, 4 spokes estándar | Facilita soldadura manual/reflow |
| **Altura del K1 (15.7 mm)** | Define altura del enclosure | Junto con HLK-5M05 son los componentes más altos |

### 6.3 Consideraciones Mecánicas

- El **HF115F-I (29 mm de largo)** es el componente con mayor footprint horizontal del PCB, junto al HLK-5M05 (34 mm). Orientar ambos de modo que la barrera de aislamiento atraviese ambos paralelamente.
- Los pines THT del K1 tienen diámetro ~1.0 mm — usar pads de 1.8 mm con anular ring ≥ 0.3 mm
- El J_LOAD (KF128-7.5-2P) debe estar en el **borde del PCB** para acceso con destornillador
- Considerar añadir un **orificio de ventilación** en el enclosure sobre el K1 si la caja está sellada — el relé genera ~10 °C de elevación sostenida al activarse

### 6.4 Consideraciones de Seguridad para PCB

- **Zona AC pintada visualmente:** usar fondo amarillo/ámbar en el silkscreen alrededor del K1 zona AC con texto "AC ZONE ⚡"
- **Zona DC pintada visualmente:** usar fondo verde o azul claro con texto "DC ZONE SELV"
- La línea del slot de aislamiento debe ser claramente visible en el silkscreen de ambas caras
- El Q1 debe estar en el lado DC — bajo ninguna circunstancia debe posicionarse cerca de los pines de contactos (5, 6, 7, 8) del K1
- Verificar con **DRC personalizado** en EasyEDA Pro que no haya trazas que crucen el slot en ninguna capa

---

## 7. Conexión con Otros Módulos

### 7.1 Entradas: M5, M7 y M1 → Módulo 3

```
Módulo 5 (ESP32-C3/C6)                           Módulo 3 (Driver BJT)
══════════════════════                           ═════════════════════

  ESP32 GPIO4 ───────────── GPIO4 ─────────────► R1 pad 1 (1 kΩ)
                                                     │
                                                 Q1 base (pin 1)


Módulo 7 (OR-Diode +5V_COMBINED)                 Módulo 3 (Bobina relé)
══════════════════════════════                   ═════════════════════

  C_bulk (+) ──────────── +5V_COMBINED ───────► K1 pin 2 (Coil A2)
                                                  │
                                         D1 cátodo (SS14) ◄── mismo net


Módulo 1 (Entrada AC y protecciones)             Módulo 3 (Contactos COM bifurcados)
═══════════════════════════════════              ═══════════════════════════════════

                                               ┌► K1 pin 6 (COM1)  ┐
  L1 pin 2 (o NODO_L) ───── AC_L_OUT ──────────┤                    ├── mismo net
                                               └► K1 pin 8 (COM2)  ┘   AC_L_OUT
                                                  │
                                         R_snub pad 1 ◄── mismo net
                                         C_snub pad 1 ◄── mismo net


Módulo 1 (Neutro, paso directo al LOAD)          Módulo 3 (J_LOAD pin 2)
═══════════════════════════════════              ══════════════════════

  L1 pin 3 (o NODO_N) ───── AC_N_OUT ─────────► J_LOAD pin 2       [ZONA AC]
  (sin conmutación — neutro directo)
```

### 7.2 Salidas: Módulo 3 → Carga externa

```
Módulo 3 (K1 contactos NO bifurcados)            Terminal externo
═════════════════════════════════════            ════════════════

  K1 pin 5 (NO1) ┐
                 ├──► LOAD_OUT ─┬──► R_snub pad 2 ───────┐
  K1 pin 7 (NO2) ┘              │                        │
  (unidos = un solo net)        │     C_snub pad 2 ─────┤
                                │                        │
                                └──── J_LOAD pin 1 ◄────┴── [AC conmutado]

  AC_N_OUT (desde M1) ─────────────────────────────► J_LOAD pin 2 ◄── [Neutro directo]
```

**Flujo completo de conmutación:**
1. Usuario aplica AC a `J_AC` (M1)
2. AC atraviesa fusible, varistor, NTC (M1) → `AC_L_OUT`, `AC_N_OUT`
3. `AC_L_OUT` entra al nodo COM bifurcado (K1 pines 6+8 unidos)
4. ESP32 ejecuta `gpio_set_level(GPIO4, 1)` → GPIO4 = 3.3V
5. Q1 satura → K1 bobina energizada (80 mA @ 5V, pines 1 y 2)
6. K1 cierra mecánicamente en ≤ 15 ms: los dos contactos bifurcados cierran simultáneamente → COM (6+8) ↔ NO (5+7)
7. `LOAD_OUT` (K1 pines 5+7 unidos) recibe AC L → J_LOAD pin 1 alimenta la carga
8. La carga completa el circuito vía J_LOAD pin 2 (neutro directo)

### 7.3 GND_DC y Aislamiento

El módulo 3 se conecta a `GND_DC` en dos puntos:
- Q1 emisor (SOT-23 pin 3) → retorno de corriente de bobina
- R2 pad 2 → cuerpo del divisor base

Ambos están en zona DC. **Ningún componente del módulo 3 conecta la zona AC con la zona DC por fuera del relé** — esta es la invariante de seguridad fundamental.

---

## 8. Checklist de Verificación Pre-Fabricación

### 8.1 Esquemático

- [ ] Polaridad de D1: cátodo (banda SMA) hacia `+5V_COMBINED` (K1 pin 2), ánodo hacia K1 pin 1
- [ ] Orientación de Q1 (SS8050 SOT-23): Base=pin 1 a R1/R2, Colector=pin 2 a K1 pin 1, Emisor=pin 3 a GND_DC
- [ ] R2 conectado entre **base de Q1 y GND_DC**, NO entre base y colector
- [ ] R1 valor = 1.0 kΩ, R2 valor = 10 kΩ
- [ ] **Puentes de bifurcación en el esquemático:** pin 5 ═══ pin 7 (mismo net LOAD_OUT) y pin 6 ═══ pin 8 (mismo net AC_L_OUT)
- [ ] R_snub + C_snub en **serie** entre nodo (K1 pin 6+8) y nodo (K1 pin 5+7) — paralelo a los contactos, NO a la carga
- [ ] C_snub marcado como **capacitor X2 275 VAC o superior** (símbolo de safety cap) — el KYET PX103K2C0702 es X2 310V
- [ ] K1 pin 2 (Coil A2) al +5V_COMBINED, pin 1 (Coil A1) al colector de Q1 (o invertido — la bobina es no polarizada)
- [ ] K1 pines 6+8 (COM bifurcado) reciben AC_L_OUT; K1 pines 5+7 (NO bifurcado) salen a J_LOAD pin 1
- [ ] J_LOAD pin 2 = AC_N_OUT directo (sin componentes intermedios)

### 8.2 Layout PCB

- [ ] K1 cruza el slot fresado 2 mm con pines 1, 2 en zona DC y pines 5, 6, 7, 8 en zona AC
- [ ] Creepage ≥ 6.5 mm verificado entre cobre zona DC y cobre zona AC en el entorno del K1
- [ ] Clearance ≥ 5.0 mm (gap de aire) a través del slot
- [ ] Q1, D1, R1, R2 ubicados a menos de 5 mm de K1 pin 1 (Coil A1)
- [ ] R_snub y C_snub ubicados a menos de 5 mm de los contactos del K1, completamente en zona AC
- [ ] Ningún cobre, via ni traza cruzando el slot de aislamiento
- [ ] Ancho de traza AC_L_OUT, LOAD_OUT, AC_N_OUT ≥ 2.5 mm (cobre 1 oz) — trazas de potencia
- [ ] Ancho de traza +5V_COMBINED hacia K1 pin 2 ≥ 0.3 mm
- [ ] **Puentes bifurcados verificados en layout:** trazas entre K1 pin 5 y pin 7, y entre pin 6 y pin 8, cortas (< 3 mm) y con ancho ≥ 2.5 mm
- [ ] J_LOAD ubicado en el borde del PCB con acceso de destornillador
- [ ] Silkscreen: "LOAD OUT MAX 16A 250V AC" junto a J_LOAD
- [ ] Silkscreen: identificación de pin 1 del K1 visible
- [ ] Silkscreen: "AC ZONE ⚡" y "DC ZONE SELV" bordeando el slot

### 8.3 Verificación de BOM

- [ ] Disponibilidad en LCSC de todos los códigos antes de ordenar:
  - K1: **C2976795** (Hongfa HF115F-I/005-1HS3A)
  - Q1: **C164886** (SS8050)
  - D1: **C2480** (SS14)
  - R1: **C11702** (1 kΩ 0402)
  - R2: **C25744** (10 kΩ 0402)
  - R_snub: **C172915** (YAGEO MFR1WSFTE52-100R, 100 Ω ±1% 1W MFR axial)
  - C_snub: **C2693738** (KYET PX103K2C0702, 10 nF X2 310V film, P=7.5 mm) — JLCPCB Extended con footprint EasyEDA disponible, >43k en stock
  - J_LOAD: **C474954** (KF128-7.5-2P)
- [ ] Si K1 (C2976795) no está disponible: tener Omron G5LE-1-VD (C88396) como backup con footprint revisado
- [ ] **No sustituir C_snub por capacitor cerámico convencional** — debe ser X2 (film safety)
- [ ] **No sustituir K1 por relé sin rating de inrush** (Songle SRD-05, etc.)

### 8.4 Pruebas Eléctricas Post-Fabricación

- [ ] **Prueba de continuidad:** con relé desenergizado, J_LOAD pin 1 NO tiene continuidad con AC_L_OUT
- [ ] **Prueba de activación:** aplicar 3.3V a GPIO4 → medir K1 pin 1 (colector de Q1) = ~0.2 V (V_CE sat)
- [ ] **Prueba de desactivación:** con GPIO4 flotante (Hi-Z), relé debe permanecer abierto (R2 mantiene base a 0V)
- [ ] **Prueba de aislamiento Hi-Pot:** aplicar **5000 VAC** entre bobina (K1 pin 1 o 2) y contactos (K1 cualquier pin 5, 6, 7 u 8), 1 minuto → sin breakdown (valor del datasheet)
- [ ] **Prueba de continuidad de bifurcación:** verificar con óhmetro que K1 pin 5 ═══ pin 7 tienen 0 Ω (puenteados externamente), e igual K1 pin 6 ═══ pin 8
- [ ] **Prueba de conmutación de carga:** conmutar bombillo LED 15W 100 veces → verificar ausencia de soldadura de contactos
- [ ] **Medición de corriente de fuga:** con relé OFF, medir corriente en J_LOAD pin 1 → debe ser < 1 mA (verificar C_snub correcto)
- [ ] **Verificar que el LED de carga NO parpadea** con relé apagado — anti-ghost flicker

---

**Fin del documento — Módulo 3 listo para esquemático en EasyEDA Pro Edition.**
