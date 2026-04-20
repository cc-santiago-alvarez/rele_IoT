# Módulo 7: Selector de Fuente AC/USB (OR-Diode) — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-20
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 7 es el **árbitro eléctrico** entre las dos posibles fuentes de alimentación de 5 V DC del producto: la fuente AC aislada del **Módulo 2** (HLK-5M05, operación instalada en pared) y la fuente USB del **Módulo 6** (conector USB-C, operación de flasheo/debug en mesa). Su función es **combinar ambas fuentes en un único rail `+5V_COMBINED`** que alimenta al **Módulo 4** (LDO ME6211 → +3.3 V para el SoC) y al **Módulo 3** (bobina del relé K1), **sin permitir que la corriente fluya entre las dos fuentes** (back-feed).

La topología elegida es el clásico **OR-Diode de dos diodos Schottky SS14**, también llamada *diode auctioneering circuit* o *ideal-diode OR* (en su versión simple, no MOSFET). Es el circuito más robusto, más barato y más tolerante a fallas que existe para resolver el problema de "dos fuentes de voltaje similar que no se deben ver la una a la otra".

Sus funciones son:

1. **Entregar corriente a `+5V_COMBINED`** desde la fuente que tenga mayor voltaje instantáneo. Si solo hay AC, D_AC conduce y D_USB bloquea. Si solo hay USB, D_USB conduce y D_AC bloquea. Si ambas están presentes, conducen parcialmente las dos (reparto dependiente de V_F y carga).
2. **Prevenir back-feed AC → USB:** impedir que los 5 V del HLK-5M05 fluyan hacia el puerto USB del PC anfitrión cuando el técnico conecta un cable USB con la placa energizada desde la red. Sin esto, el PC vería 5 V inyectados en su VBUS, lo cual es ilegal por la especificación USB (solo el host puede proveer VBUS) y puede dañar el controlador USB del PC.
3. **Prevenir back-feed USB → AC:** impedir que los 5 V del USB lleguen al secundario del HLK-5M05 cuando la placa no está conectada a la red. Aunque el HLK-5M05 tolera voltaje inverso limitado en su salida, inyectar corriente allí durante el flasheo es desaconsejable y puede descargar los condensadores internos del módulo de forma imprevisible.
4. **Filtrar el rail combinado** con un capacitor electrolítico bulk `C_bulk` de 100 µF que absorbe los transientes de conmutación del relé K1 (pico de ~500 mA durante 20 ms al cierre) y sostiene el rail por algunos ms durante la conmutación entre fuentes AC ↔ USB (hold-up capacitance).
5. **Proveer un punto físico único de medida** del rail de 5 V del sistema: en producción, un pogo-pin sobre `C_bulk` permite verificar salud de la fuente independientemente de cuál esté alimentando.

**Entradas del módulo:**
- `+5V_AC` — rail de 5 V ±0.25 V provisto por el **Módulo 2** (salida del HLK-5M05, pin 4 "+V"), con hasta 1 A de corriente disponible y aislamiento galvánico 3000 VAC frente a la red.
- `+5V_USB` — rail de 5 V ±0.25 V provisto por el **Módulo 6** (salida del filtro VBUS, aguas abajo del PTC1 de 500 mA), disponible solo cuando un cable USB está conectado.

**Salidas del módulo:**
- `+5V_COMBINED` — rail combinado de ~4.55 V nominal (5 V − V_F del Schottky) hacia:
  - **Módulo 4** (ME6211-3.3, pin 1 VIN), que a su vez produce el `+3.3V` del SoC.
  - **Módulo 3** (bobina del relé K1, terminal A2 / pin 2), cátodo del diodo flyback D1 del propio M3.
- `GND_DC` — referencia común compartida con M2, M4, M5, M6, M8, M9 (plano continuo en el secundario aislado).

> **Nota crítica sobre el nombre de la señal:** el documento maestro llama a la salida `+5V_COMBINED`, mientras que los documentos de los demás módulos a veces la llaman simplemente `+5V` o `+5V_DC`. En este documento y en el esquemático EasyEDA Pro usaremos **`+5V_COMBINED`** exclusivamente para evitar ambigüedad con los rails intermedios `+5V_AC` y `+5V_USB`.

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 La Topología OR-Diode: por qué funciona

El circuito OR-Diode resuelve un problema aparentemente trivial pero que tiene trampas: **¿cómo combino dos fuentes de voltaje similar sin que una alimente a la otra?** Si simplemente uniera `+5V_AC` con `+5V_USB` directamente con un cable, pasarían dos cosas malas:

1. **Back-feed:** la fuente con voltaje ligeramente mayor inyectaría corriente en la salida de la otra. En el mejor caso, solo hay pérdidas térmicas. En el peor, el HLK-5M05 ve una corriente inversa no documentada en su salida, o el host USB del PC ve 5 V en su VBUS (que solo él debería proveer) y decide apagar el puerto por sobreprotección — o se daña.
2. **Competencia de reguladores:** ambos reguladores internos (HLK-5M05 y el regulador del PC) intentarían fijar el voltaje a su propio set-point. Cualquier diferencia entre ellos se resuelve con corriente circulando entre ambos, caldeando sus etapas de salida. En régimen estable podría haber oscilaciones o corrientes de corto-circuito cíclicas.

El diodo resuelve esto porque **solo permite paso de corriente en una dirección** (de ánodo a cátodo). Si coloco un diodo con el ánodo en cada fuente y uno los cátodos, la corriente solo puede fluir **desde cada fuente hacia el nodo común**, nunca al revés. El nodo común tomará automáticamente el voltaje de la fuente más alta (menos V_F del diodo), porque el diodo de la fuente más baja queda polarizado en inverso y no conduce.

```
    +5V_AC ────►│ D_AC  ──┐
                          ├── +5V_COMBINED  (= max(V_AC, V_USB) − V_F)
    +5V_USB ───►│ D_USB ──┘
```

La magia es que el circuito se auto-reconfigura: si desconecto AC, D_AC deja de conducir automáticamente y D_USB toma el relevo sin intervención activa. Es un conmutador **sin lógica digital, sin MCU, sin firmware** — pura física de unión PN. Esto es exactamente lo que se necesita en un producto que debe funcionar *antes* de que el MCU arranque (bring-up del ME6211 y del ESP32-C3 depende de que haya 5 V disponibles *ya*).

### 2.2 Análisis de V_F: por qué Schottky y no Silicio

La caída directa del diodo `V_F` es el costo energético de la topología OR. Cada miliamperio que pasa por el rail `+5V_COMBINED` paga un peaje de `V_F × I`. Para el peor caso de nuestro sistema (relé conmutando + TX WiFi pico), la corriente puede alcanzar ~400 mA, y esa caída se resta directamente del rail disponible para el ME6211.

Comparación real entre dos candidatos:

| Parámetro | 1N4007 (silicio) | **SS14 (Schottky)** |
|---|---|---|
| V_F @ 250 mA | ~0.85 V | **~0.40 V** |
| V_F @ 1 A | ~1.10 V | **~0.55 V** |
| Encapsulado | SMA DO-214AC | SMA DO-214AC |
| V_R (reverse) | 1000 V | 40 V |
| Leakage @ V_R max | ~5 µA | ~500 µA @ 40 V, ~50 µA @ 5 V |
| I_rr (recovery) | ~2 µs | ~5 ns |
| Precio (JLCPCB) | ~$0.02 | ~$0.03 |

**Impacto en el rail `+5V_COMBINED`:**

- Con **1N4007**: `V_COMBINED = 5.0 V − 1.0 V = 4.0 V`. Con ME6211 LDO (dropout 100 mV @ 500 mA), el margen para generar 3.3 V es solo **0.6 V**. Cualquier caída adicional (tolerancia de HLK-5M05 −5 %, caída en el PTC con 250 mA, caída en trazas, tolerancia de V_F del propio 1N4007) comprime ese margen hasta hacerlo marginal o negativo en el peor caso combinado. **No es robusto.**

- Con **SS14**: `V_COMBINED = 5.0 V − 0.45 V = 4.55 V`. El ME6211 tiene ahora **1.25 V de margen bruto** (1.15 V real tras el dropout del propio LDO). Incluso con tolerancias acumuladas peor-caso (HLK a 4.75 V, V_F de SS14 a 0.55 V @ 1 A), el rail queda en **4.20 V**, todavía con 900 mV de margen. **Robusto con margen de ingeniería.**

**La pérdida térmica también favorece al Schottky:** `P_diss = V_F × I`. Con 400 mA promedio, un 1N4007 disipa **400 mW** (cerca del límite de 500 mW para SMA sin thermal pad), mientras que el SS14 disipa solo **180 mW**. Menos calor = menos derating = más vida útil del componente adyacente (relé, electrolítico).

**La corriente de fuga inversa del Schottky (I_R)** es el único parámetro donde silicio gana: con 5 V de reverse, un SS14 puede tener 20–50 µA de leakage mientras un 1N4007 tiene <1 µA. En este circuito, el peor caso es cuando una fuente está encendida y la otra no — el diodo "apagado" vería ~5 V en inverso y pasaría ~50 µA al rail que ya está apagado. Este leakage es **completamente despreciable** comparado con los cientos de mA del circuito activo, y en cualquier caso cae en la resistencia de salida de la fuente apagada (el HLK-5M05 tiene un divisor de realimentación interno que absorbe µA sin problema; el regulador del PC ignora µA en su VBUS).

**Conclusión:** SS14 es la elección correcta por margen de 2–3× en todos los parámetros relevantes, con penalización insignificante en leakage.

### 2.3 Prevención de Back-Feed: análisis eléctrico formal

Considere el peor caso: la placa está instalada en pared, alimentada desde AC. El HLK-5M05 entrega 5 V al nodo `+5V_AC`. El técnico conecta un cable USB con la placa energizada. El PC anfitrión también entrega 5 V a VBUS, que llega a `+5V_USB` tras el PTC1 del Módulo 6. **¿Qué pasa?**

- Voltaje en ánodo de D_AC: 5.0 V
- Voltaje en ánodo de D_USB: ~4.95 V (5.0 V − caída en PTC)
- Voltaje en el nodo `+5V_COMBINED`: `max(5.0 − 0.45, 4.95 − 0.45) = 4.55 V` (dominado por D_AC, porque el AC está ligeramente más alto)

Ahora la pregunta clave: **¿cuál es el voltaje en el cátodo de D_USB?** Es 4.55 V. **¿Cuál es el voltaje en el ánodo de D_USB?** Es 4.95 V. Entonces V_AK = 4.95 − 4.55 = **+0.40 V forward**. Parece que D_USB conduce también... ¡y efectivamente conduce un poquito! Pero con V_F = 0.40 V está operando cerca del codo de su curva I-V, donde la corriente es del orden de 1–10 mA. Prácticamente no aporta al rail (dominio absoluto de D_AC), pero tampoco está bloqueando perfectamente. Esto está bien — no es back-feed, es co-conducción benigna.

**El caso que importa de verdad es el opuesto:** la fuente AC se apaga (corte eléctrico, o la placa nunca se conectó a la red, solo a USB de un programador). Entonces:

- Voltaje en ánodo de D_AC: ~0 V (la salida del HLK-5M05 queda flotante o cae rápido por los C_out del M2)
- Voltaje en ánodo de D_USB: 4.95 V
- Voltaje en `+5V_COMBINED`: 4.95 − 0.45 = 4.50 V
- Voltaje V_AK de D_AC: 0 − 4.50 = **−4.50 V reverse**. D_AC polarizado en inverso, no conduce.
- Corriente de leakage de D_AC hacia atrás: I_R(5 V) ≈ 30 µA → carga despreciable para el HLK apagado.

**Resultado:** el rail `+5V_AC` permanece en ~0 V, ningún voltaje peligroso llega al HLK-5M05 ni a la red. El PC entrega solo la corriente que consume el DC-side completo (~50–250 mA según estado del SoC).

**Test práctico para verificar back-feed en bancada:**

1. Alimente la placa solo por USB (sin red AC conectada al J_AC).
2. Mida con multímetro de alta impedancia (≥ 10 MΩ) el voltaje en `+5V_AC` (pin 4 del HLK-5M05, o directamente en el ánodo de D_AC).
3. **Criterio:** debe estar < 0.1 V. Si está > 1 V, hay un problema: o bien D_AC está invertido, o bien hay un cortocircuito en el footprint.

### 2.4 Dimensionamiento de C_bulk: ripple, ESR y hold-up

El capacitor C_bulk de 100 µF cumple **tres funciones simultáneas** en el rail `+5V_COMBINED`:

**Función 1 — Filtro de ripple residual del HLK-5M05.** La salida del HLK-5M05 no es un DC perfecto: tiene ripple de conmutación del flyback interno a ~100 kHz, típicamente 50–100 mV pico a pico. Los capacitores del Módulo 2 (`C_out1` 100 µF + `C_out2` 10 µF + `C_out3` 100 nF) ya filtran la mayoría. Pero el SS14 añade una pequeña componente de ripple adicional por su capacitancia de transición. C_bulk local en el Módulo 7 ayuda a aislar cualquier ripple residual del ME6211 del Módulo 4.

**Función 2 — Reserva de corriente para pulsos de conmutación del relé.** Al cerrar el relé K1, la bobina demanda un pulso de ~500 mA durante ~20 ms (tiempo de establecimiento del campo magnético). Este pulso viene directamente del rail `+5V_COMBINED`. Sin C_bulk, el pulso viajaría por los diodos SS14 y eventualmente por la fuente (HLK o USB), causando una caída temporal de varias centenas de mV que podría resetear al ESP32-C3. Con C_bulk = 100 µF:

```
ΔV = I × Δt / C = 0.5 A × 0.020 s / 100e-6 F = 100 V
```

¡100 V! Esto parece catastrófico, pero no es correcto porque la fuente **sí** suministra corriente durante esos 20 ms. El cálculo correcto es: **si la fuente puede suministrar 1 A (HLK-5M05 rated)**, y la bobina pide 500 mA, entonces C_bulk solo tiene que suplir la diferencia entre la demanda pico y la capacidad de la fuente durante el tiempo que tarde la fuente en reaccionar (lazo de control del HLK, ~1 ms típico). Con eso:

```
ΔV = (0.5 A − 0 A "supply lag") × 0.001 s / 100e-6 F = 5 V
```

Que tampoco es aceptable. El cálculo realista es más complicado porque el HLK responde gradualmente, pero el orden de magnitud de la demanda de C_bulk es: **cuanto más grande, mejor**. 100 µF es el mínimo práctico; 220 µF sería mejor pero aumenta costo y footprint. La práctica industrial (Shelly, Sonoff, HLK reference designs) coincide en 100 µF como sweet spot.

**Función 3 — Hold-up durante transición AC ↔ USB.** Si el técnico desconecta el cable USB con la placa sin red AC, el rail cae a 0 en ~10 ms (determinado por el consumo y C_bulk). Si la red AC está conectada, la transición de D_USB a D_AC es prácticamente instantánea (nanosegundos, limitada por la dinámica del Schottky). C_bulk garantiza que durante esa transición no haya un cruce por cero que resetee al MCU.

**ESR requerido:** para que C_bulk filtre eficientemente los pulsos del relé, su ESR debe ser baja. Un electrolítico estándar de 100 µF / 16 V tiene ESR típica de 100–300 mΩ. Con 500 mA de pulso, la caída por ESR sería 50–150 mV, aceptable. Un low-ESR (50–100 mΩ) mejora esto. El part number elegido (Chengx KS107M016D07RR0VH2FP0, LCSC C43803) está en el rango aceptable pero no es premium — para un producto que debe durar 10 años en pared, considerar el upgrade a Nichicon UWT (LCSC C319547, ESR ~80 mΩ) en revisiones futuras.

**Voltaje nominal:** el rail opera en ~4.55 V. Un capacitor de 10 V tiene derating de 0.45× (opera al 45 % del rated), lo que da vida útil decente. Un capacitor de **16 V** (el que ya está en el BOM actual) opera al 28 %, lo que duplica la vida útil esperada a la misma temperatura. Recomendación firme: **usar 16 V, no 10 V**, aunque cueste 0.5 ¢ más.

### 2.5 Modos de Operación Detallados

| Modo | Condición externa | Estado D_AC | Estado D_USB | V(+5V_COMBINED) | Fuente efectiva | Aplicación típica |
|---|---|---|---|---|---|---|
| **A — Solo AC** | Placa instalada en pared, sin USB | Forward, V_F ≈ 0.45 V | Reverse bias ~ −4.55 V, corta | 4.50–4.65 V | HLK-5M05 100 % | Operación normal de usuario |
| **B — Solo USB** | Flasheo en mesa, sin red AC | Reverse bias ~ −4.50 V, corta | Forward, V_F ≈ 0.45 V | 4.40–4.60 V | PC anfitrión 100 % | Producción, bring-up, debug |
| **C — Ambas fuentes, AC > USB** | Placa en pared, técnico conecta USB | Forward, V_F ≈ 0.45 V (dominante) | Forward leve, V_F ≈ 0.35–0.40 V (marginal) | 4.55 V ≈ igual a modo A | HLK ~90 %, USB ~10 % | Debug in-situ sobre placa energizada |
| **C' — Ambas fuentes, USB > AC** | Raro: HLK produce menos que USB | Forward leve (marginal) | Forward, V_F ≈ 0.45 V (dominante) | 4.55 V ≈ igual a modo B | USB ~90 %, HLK ~10 % | Síntoma de HLK débil o sobrecargado |
| **D — Ninguna fuente** | Placa desconectada | Cortadas | Cortadas | 0 V | — | Almacenamiento |

**Transiciones:**

- **A → B (conectar USB sobre placa en pared):** sin interrupción perceptible. El rail sube ligeramente si el USB llega a estar marginalmente por encima, o se mantiene igual si el HLK domina. No hay back-feed.
- **B → A (energizar red AC mientras se está flasheando vía USB):** sin interrupción. El HLK comienza a alimentar gradualmente (soft-start ~10 ms), el rail sube ligeramente. Puede haber un glitch de <50 mV. esptool no debería perder sincronía.
- **A → D (corte eléctrico):** el HLK-5M05 deja de entregar corriente. El rail decae con τ = C_bulk × R_load. Para una carga típica de 100 mA (ESP32 + LED), τ = 100 µF × 50 Ω = **5 ms**. El MCU tiene 5 ms para detectar undervoltage y hacer un graceful shutdown (flush NVS, etc.). En la práctica no se hace — el MCU simplemente pierde alimentación. Los watchdog timers y la lógica idempotente del firmware son responsables de la recuperación.
- **B → D (desconectar USB sin red AC):** mismo comportamiento que A → D. Decay ~5 ms.
- **C → A (desconectar USB con red AC presente):** sin interrupción. D_USB se apaga, D_AC sigue conduciendo. Imperceptible.

### 2.6 Análisis Térmico del SS14

El SS14 en encapsulado SMA DO-214AC tiene resistencia térmica de unión a ambiente:
- **R_θJA = 88 °C/W** (dato del datasheet MDD para SMA sobre pad estándar de 5×5 mm de cobre).
- **R_θJL = 25 °C/W** (junction to lead, si hay thermal spreading).

Disipación típica:
- Modo A (solo AC, 200 mA promedio): `P_D_AC = 0.42 V × 0.2 A = 84 mW`. `ΔT_junction = 0.084 × 88 = 7.4 °C`. Operando en ambiente de 40 °C, Tj = 47 °C. Muy seguro.
- Modo A, pico de conmutación relé (500 mA durante 20 ms): `P_D_AC_pico = 0.52 V × 0.5 A = 260 mW`. Pulso térmicamente transitorio, no lo lleva al steady state. Tj sube quizá 2–3 °C transitoriamente.
- Modo A + TX WiFi (300 mA sostenidos): `P_D_AC = 0.48 V × 0.3 A = 144 mW`. `ΔT = 12.7 °C`. Tj = 53 °C. Seguro.
- Peor caso teórico (1 A continuo, no ocurre en este diseño): `P = 0.55 × 1 = 550 mW`. `ΔT = 48 °C`. Tj = 88 °C en ambiente de 40 °C. Todavía dentro del rating de Tj_max = 125 °C, pero comprimido — por eso el diseño está dimensionado para 400 mA máximo.

**Recomendación de layout:** pads de 7×7 mm mínimo por diodo (en lugar del 5×5 mm estándar) duplican la disipación al ambiente. Vías stitching al plano GND en el pad del cátodo (el lado con la banda blanca) mejoran más. Para M7, dado que las disipaciones nominales son 84–144 mW, los pads estándar de EasyEDA Pro para huella SMA son suficientes, pero **los pads ampliados de cobre son una mejora trivial y gratuita** que se debe aplicar.

### 2.7 Comportamiento en Transitorios

**Encendido inicial (cold start desde AC):**

1. T = 0: usuario cierra el interruptor mural, aplica 110 V AC a J_AC.
2. T ≈ 10 ms: NTC_inrush termina la fase de soft-start, MOV/fusible OK.
3. T ≈ 50 ms: HLK-5M05 completa su soft-start interno, su salida pin 4 sube de 0 a 5 V (rampa de ~20 ms).
4. T ≈ 70 ms: el ánodo de D_AC supera 0.3 V sobre el cátodo (que está a 0 V porque C_bulk estaba descargado), D_AC empieza a conducir.
5. T ≈ 70–90 ms: corriente inrush a C_bulk: `I = C × dV/dt = 100 µF × 4.55 V / 20 ms = 22.7 mA` promedio, pico típico ~100 mA durante 5 ms. SS14 soporta 30 A no repetitivo → margen infinito.
6. T ≈ 100 ms: rail establecido en 4.55 V. El ME6211 del M4 arranca y produce 3.3 V. El ESP32-C3 hace brown-out release y comienza el boot.

**Hot-plug de USB sobre placa en pared (modo A → C):**

1. Cable USB se inserta. El PC detecta el device (tras R_CC1/R_CC2 del M6) y aplica 5 V a VBUS.
2. VBUS sube a 5 V en ~50 µs (rise time típico de un host USB 2.0 reenumerando).
3. `+5V_USB` (después del PTC y filtros M6) sube a ~4.95 V en ~100 µs.
4. Ánodo de D_USB: 4.95 V. Cátodo (en `+5V_COMBINED`): 4.55 V. V_AK = +0.40 V → conducción leve (2–5 mA).
5. No hay glitch perceptible en el rail. El MCU sigue funcionando. esptool puede conectarse inmediatamente.

**Hot-unplug de USB:**

1. Cable USB se retira. VBUS cae a 0 V en ~1 ms (descarga de C_USB1 vía PTC + corriente del host si lo mantiene).
2. Si AC está presente: D_USB entra en reverse bias progresivo, deja de conducir en ~10 µs. D_AC sigue conduciendo. Sin interrupción.
3. Si AC no está presente: el rail `+5V_COMBINED` decae con τ = C_bulk × R_load ≈ 5 ms (caso típico 100 mA). El MCU pierde alimentación en ~10 ms. Brown-out reset sin gracia.

---

## 3. Asignación de Señales (entradas y salidas)

El Módulo 7 **no tiene pines de conector físico propio**: está compuesto por 3 componentes SMT/THT que se colocan entre los rails de M2/M6 y las entradas de M4/M3. Por tanto, esta sección no lista "pines del conector" como M5 o M6, sino los **terminales de los componentes** y sus nets asociados.

### 3.1 Terminales del diodo D_AC (SS14, SMA DO-214AC)

| Terminal | Nombre físico | Net | Conexión | Observación |
|---|---|---|---|---|
| 1 (ánodo) | A / Anode | `+5V_AC` | Pin 4 "+V" del HLK-5M05 (M2), vía nodo compartido con C_out1, C_out2, C_out3 del M2 | Lado sin banda blanca del encapsulado SMA |
| 2 (cátodo) | K / Cathode | `+5V_COMBINED` | Nodo común con D_USB cátodo, C_bulk pad (+), pin 1 VIN del ME6211 (M4), bobina K1 A2 (M3) | **Lado con la banda blanca**; orientación crítica |

### 3.2 Terminales del diodo D_USB (SS14, SMA DO-214AC)

| Terminal | Nombre físico | Net | Conexión | Observación |
|---|---|---|---|---|
| 1 (ánodo) | A / Anode | `+5V_USB` | Salida del filtro VBUS del M6 (aguas abajo de PTC1, C_USB1, C_USB2) | Lado sin banda blanca |
| 2 (cátodo) | K / Cathode | `+5V_COMBINED` | Nodo común con D_AC cátodo, C_bulk pad (+), pin 1 VIN del ME6211 (M4), bobina K1 A2 (M3) | **Lado con la banda blanca**; orientación crítica |

### 3.3 Terminales del capacitor C_bulk (electrolítico radial 5×7 mm)

| Terminal | Nombre físico | Net | Conexión | Observación |
|---|---|---|---|---|
| 1 (+) | Positive / Anode | `+5V_COMBINED` | Nodo común con cátodos de D_AC y D_USB, pin 1 VIN del ME6211 (M4), bobina K1 A2 (M3) | Terminal más largo al montarlo THT; marcado con muesca en la lata |
| 2 (−) | Negative / Cathode | `GND_DC` | Plano GND_DC del secundario aislado | Terminal corto; marcado con banda negativa en el cuerpo |

**⚠ Orientación crítica del electrolítico:** invertir la polaridad de un capacitor electrolítico bajo 4.55 V de forma sostenida causa **explosión** del capacitor en segundos o minutos (gas interno, ruptura del sello). En producción, verificar la orientación en la inspección AOI antes de primer encendido. En el esquemático EasyEDA Pro, el pin (+) debe estar marcado claramente con el símbolo "+".

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama ASCII Completo del Módulo 7

```
     MÓDULO 7 — SELECTOR DE FUENTE AC/USB (OR-Diode)
     ═══════════════════════════════════════════════════════════

       De Módulo 2 (HLK-5M05)                De Módulo 6 (USB-C, tras PTC1)
              │                                        │
              │ +5V_AC (~5.0 V)                        │ +5V_USB (~4.95 V)
              ▼                                        ▼
     ┌────────────────────┐                   ┌────────────────────┐
     │   D_AC  (SS14)     │                   │   D_USB  (SS14)    │
     │   SMA DO-214AC     │                   │   SMA DO-214AC     │
     │   LCSC C2480       │                   │   LCSC C2480       │
     │                    │                   │                    │
     │   Ánodo ──►├── K   │                   │   Ánodo ──►├── K   │
     │            │       │                   │            │       │
     │           V_F=0.45 V                   │           V_F=0.45 V
     │            │       │                   │            │       │
     └────────────┼───────┘                   └────────────┼───────┘
                  │                                        │
                  │                                        │
                  └────────────────┬───────────────────────┘
                                   │
                                   │ +5V_COMBINED (~4.55 V)
                                   │
                        ┌──────────┼──────────┐
                        │                     │
                        ▼                     │
                 ┌──────────────┐             │
                 │   C_bulk     │             │
                 │   100 µF / 16 V            │
                 │   Radial THT 5×7 mm        │
                 │   LCSC C43803              │
                 │                            │
                 │   (+) ──┬── (−)            │
                 │         │                  │
                 └─────────┼──────────────────┘
                           │                  │
                           ▼                  ▼
                        GND_DC          Hacia Módulo 4 (ME6211 pin 1 VIN)
                                             y Módulo 3 (K1 bobina A2, cátodo D1)

```

### 4.2 Detalle del Nodo D_AC

```
(a) Nodo D_AC — interfaz M2 → M7

   HLK-5M05 (M2)                     D_AC (SS14, SMA)
   ─────────────                     ────────────────
   Pin 4 "+V" ──── +5V_AC ────►──┤├──── +5V_COMBINED
                                 │
                                 │  cátodo (lado con banda blanca)
                                 ▼
                              Orientación:
                              Ánodo del SMA ───► izquierda (hacia HLK-5M05)
                              Cátodo del SMA ──► derecha (hacia C_bulk)

   Capacitores del M2 ya presentes en el nodo +5V_AC:
     C_out1 (100 µF/16V, LCSC C43803) — bulk
     C_out2 (10 µF/25V 0805, LCSC C15850) — MF
     C_out3 (100 nF/50V 0805, LCSC C49678) — HF
   (estos pertenecen al BOM del M2, no al M7)
```

### 4.3 Detalle del Nodo D_USB

```
(b) Nodo D_USB — interfaz M6 → M7

   Filtro VBUS del M6                D_USB (SS14, SMA)
   ──────────────────                ─────────────────
   +5V_USB ───────────────────►──┤├──── +5V_COMBINED
   (aguas abajo de PTC1,         │
    C_USB1 10 µF, C_USB2 100 nF) │
                                 ▼
                              cátodo (lado con banda blanca)

   Corriente máxima limitada por PTC1 del M6: I_hold = 500 mA.
   Si el flasheo demanda > 500 mA, el PTC se dispara y corta el USB.
```

### 4.4 Detalle del Nodo +5V_COMBINED con C_bulk

```
(c) Nodo de salida +5V_COMBINED — interfaz M7 → M4/M3

                                     ┌───► Módulo 4, ME6211 pin 1 (VIN)
                                     │      (carga principal del SoC)
                                     │
                                     │
   ┌──────┤├───────┐                 │
   │  D_AC  (K)    ├─────┬───────────┤
   │  D_USB (K)    │     │           │
   └──────┤├───────┘     │           │
                         │           │
                        ─┴─          └───► Módulo 3, K1 bobina A2 / pin 2
                        ┬ C_bulk            (cátodo del diodo flyback D1 del M3)
                        │ 100 µF/16V
                        │
                        │
                       GND_DC (plano)
```

### 4.5 Tabla Pin a Pin Exhaustiva

| Componente | Terminal | Net de entrada | Net de salida | Componente aguas arriba | Componente aguas abajo |
|---|---|---|---|---|---|
| **D_AC** | Ánodo | `+5V_AC` | — | HLK-5M05 pin 4 (M2) | — |
| **D_AC** | Cátodo | — | `+5V_COMBINED` | — | C_bulk (+), ME6211 VIN (M4), K1 A2 (M3) |
| **D_USB** | Ánodo | `+5V_USB` | — | Filtro VBUS M6 (aguas abajo de PTC1) | — |
| **D_USB** | Cátodo | — | `+5V_COMBINED` | — | C_bulk (+), ME6211 VIN (M4), K1 A2 (M3) |
| **C_bulk** | (+) | `+5V_COMBINED` | — | Cátodos de D_AC, D_USB | ME6211 VIN (M4), K1 A2 (M3) |
| **C_bulk** | (−) | `GND_DC` | — | Plano GND_DC | — |

**Nets totales del módulo:** 4 (`+5V_AC`, `+5V_USB`, `+5V_COMBINED`, `GND_DC`).
**Nets que cruzan la frontera del módulo:** todos 4 (`+5V_AC` viene de M2, `+5V_USB` viene de M6, `+5V_COMBINED` va a M4 y M3, `GND_DC` es el plano común).

---

## 5. Reglas de Layout PCB

### 5.1 Reglas Críticas

| # | Regla | Motivo | Criterio de verificación |
|---|---|---|---|
| L1 | **D_AC y D_USB colocados físicamente entre el HLK-5M05 (M2) y el ME6211 (M4)**, formando una cadena espacial natural M2 → M7 → M4 | Minimiza longitudes de trazas en el rail más crítico | Inspección visual del placement |
| L2 | **Pads de cobre expandidos** bajo los diodos SMA: 7×7 mm mínimo (vs. 5×5 mm estándar), con vías stitching al plano GND_DC | Disipación térmica 2–3× mejor, márgenes amplios en picos | Verificar en huella editada de EasyEDA Pro |
| L3 | **Traza `+5V_COMBINED` ≥ 0.8 mm** (30 mil), desde cátodos de diodos hasta VIN del ME6211 y bobina K1 | Capacidad de corriente ≥ 1 A con ΔT < 10 °C @ 1 oz Cu (IPC-2152) | Calculadora de ampacidad |
| L4 | **C_bulk físicamente adyacente a los cátodos de los diodos** (≤ 5 mm de separación) | ESR y ESL efectiva mínimas en la ruta de pulso de relé | Inspección visual |
| L5 | **Retorno GND de C_bulk al plano GND_DC mediante ≥ 2 vías Ø 0.3 mm** | Reducir inductancia del camino de retorno para los pulsos de conmutación del relé | Inspección visual |
| L6 | **Orientación de D_AC y D_USB consistente en silkscreen**: ambos con la banda blanca apuntando hacia el mismo lado (hacia `+5V_COMBINED`) | Prevenir errores de ensamblaje visual | Inspección AOI |
| L7 | **Orientación de C_bulk marcada claramente**: silkscreen con "+" muy visible en el pad positivo | Prevenir explosión por polaridad invertida | Inspección AOI, pre-producción |
| L8 | **No rutear trazas de señal (GPIO, D+/D−, UART) por debajo de C_bulk** | Evitar acople parásito del ripple del electrolítico a señales sensibles | Inspección del layout en EasyEDA Pro |
| L9 | **D_AC y D_USB ambos en la cara TOP** (no una en TOP y otra en BOTTOM) | Disipación térmica simétrica; ensamblaje más simple | Inspección visual |
| L10 | **Separación mínima 3 mm entre D_AC y el edge del HLK-5M05** | Evitar calentamiento mutuo; el HLK ya disipa ~500 mW en operación | Inspección mecánica |

### 5.2 Checklist de Layout

- [ ] D_AC colocado a ≤ 15 mm del pin 4 "+V" del HLK-5M05.
- [ ] D_USB colocado a ≤ 15 mm del nodo +5V_USB del Módulo 6 (después del PTC1).
- [ ] C_bulk colocado a ≤ 5 mm de los cátodos de ambos diodos.
- [ ] Pads de cobre bajo D_AC y D_USB expandidos a 7×7 mm con vías stitching al plano GND.
- [ ] Traza `+5V_COMBINED` con ancho ≥ 0.8 mm (30 mil) hasta M4 y M3.
- [ ] Retorno GND de C_bulk con al menos 2 vías de Ø 0.3 mm al plano GND_DC.
- [ ] Banda blanca de D_AC apunta hacia `+5V_COMBINED` (cátodo = salida).
- [ ] Banda blanca de D_USB apunta hacia `+5V_COMBINED` (cátodo = salida).
- [ ] Silkscreen "+" de C_bulk visible y colocado en el pad correcto.
- [ ] Sin trazas de señales bajo C_bulk (plano GND sólido debajo).
- [ ] Ambos diodos en la cara TOP del PCB.
- [ ] Separación ≥ 3 mm entre D_AC y el HLK-5M05.
- [ ] Las orientaciones y huellas de los 3 componentes verificadas contra EasyEDA Pro + datasheets (pin 1 de la huella coincide con pin 1 del componente).
- [ ] Designadores (D_AC, D_USB, C_bulk) legibles en silkscreen y no tapados por los componentes.

---

## 6. BOM Completo — Módulo 7 con Códigos LCSC

Todos los componentes están verificados para importación directa desde **LCSC / JLCPCB Standard Parts Library** y son compatibles con **EasyEDA Pro Edition**. Los 3 componentes primarios ya están presentes en el CSV maestro [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) (filas 11, 12, 39) y están marcados con ✓.

### 6.1 Componentes Primarios

| # | Ref | Componente | Valor / Specs | Encaps. | LCSC | Fabricante | MPN | Qty | Precio aprox. | CSV |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **D_AC** | Diodo Schottky OR-diode fuente AC | 40 V / 1 A, V_F=0.45 V @ 1 A, I_R=500 µA max @ 40 V | SMA DO-214AC | **C2480** | MDD (Microdiode Electronics) | SS14 | 1 | $0.03 | ✓ |
| 2 | **D_USB** | Diodo Schottky OR-diode fuente USB | 40 V / 1 A, V_F=0.45 V @ 1 A | SMA DO-214AC | **C2480** | MDD (Microdiode Electronics) | SS14 | 1 | $0.03 | ✓ |
| 3 | **C_bulk** | Capacitor electrolítico bulk +5V rail | 100 µF / 16 V, ±20 %, 105 °C, low-ESR | Radial THT 5×7 mm, P=2.5 mm | **C43803** | Chengx | KS107M016D07RR0VH2FP0 | 1 | $0.02 | ✓ |

**Total componentes:** 3 referencias (2 part numbers únicos — D_AC y D_USB son idénticos SS14). **Subtotal BOM Módulo 7: ≈ $0.08 USD** por placa (sin ensamblaje). El módulo más barato del proyecto.

> **Nota importante sobre C_bulk voltaje:** el documento maestro [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md §9](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) especifica `100 µF / 10 V`, pero el BOM actual y este documento usan **16 V** para mayor derating (operación al 28 % del rated vs. 45 %, duplica la vida útil esperada a temperatura ambiente típica). Mismo footprint, mismo precio. Esta actualización debe reflejarse en v1.1 del documento maestro.

### 6.2 Componentes Alternativos / Segundas Fuentes

#### 6.2.1 Alternativas para D_AC / D_USB (Schottky 40 V / 1 A, SMA DO-214AC)

Son componentes drop-in (misma huella SMA, mismas V_R y I_F ratings). La única diferencia es V_F típica y fabricante — cualquiera funciona eléctricamente.

| Opción | LCSC | MPN | Fabricante | V_F @ 1 A | V_R | Encaps. | Notas |
|---|---|---|---|---|---|---|---|
| **Primario** | **C2480** | SS14 | MDD (Microdiode) | 0.55 V | 40 V | SMA DO-214AC | ✓ En stock JLCPCB Standard Parts Library, abril 2026 |
| Alt. 1 | **C8678** | MBRA140T3G | ON Semi / Onsemi | 0.55 V | 40 V | SMA DO-214AC | Drop-in, mayor disponibilidad global, precio similar |
| Alt. 2 | **C22452** | B140-13-F | Diodes Incorporated | 0.50 V | 40 V | SMA DO-214AC | Drop-in, V_F ligeramente mejor |
| Alt. 3 | **C181282** | LSS14 | LRC | 0.50 V | 40 V | SMA DO-214AC | Low-cost alternative, segunda fuente china |
| Alt. 4 | **C964175** | SS14 | HY Electronic (Cayman) | 0.55 V | 40 V | SMA DO-214AC | Mismo MPN SS14, otro fabricante |
| Upgrade premium | **C58848** | STPS1L40A | STMicroelectronics | 0.42 V | 40 V | SMA DO-214AC | V_F más baja (–30 mV @ 1 A), mejor rail — vale la pena para revisión premium del producto |
| ❌ NO usar | — | 1N4007 | (cualquiera) | 1.00 V | 1000 V | SMA | V_F demasiado alta (rail cae a 4.0 V, margen marginal ME6211). Ver §2.2 |
| ❌ NO usar | — | 1N5819 | (cualquiera) | 0.55 V | 40 V | DO-41 axial | Huella incorrecta (axial vs SMD), no drop-in en PCB SMT |

#### 6.2.2 Alternativas para C_bulk (100 µF / 16 V o 10 V, electrolítico radial THT)

| Opción | LCSC | MPN | Fabricante | Specs | Dimensiones | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C43803** | KS107M016D07RR0VH2FP0 | Chengx | 100 µF / 16 V / 105 °C | 5×7 mm, P=2.5 mm | ✓ En stock JLCPCB, ya en BOM |
| Alt. 1 | **C45783** | REA101M1CBK-0511P | Lelon | 100 µF / 16 V / 105 °C | 5×11 mm, P=2.0 mm | Mayor altura, mejor vida útil térmica |
| Alt. 2 | **C319547** | UWT1C101MCL1GS | Nichicon | 100 µF / 16 V / 105 °C, low-ESR ~80 mΩ | 6.3×5.8 mm, P=2.5 mm | **Premium**: ESR más baja, 5000 h a 105 °C, fabricante japonés referencia |
| Alt. 3 | **C396167** | EEE-FT1V101AR | Panasonic | 100 µF / 35 V / 105 °C | 6.3×5.4 mm SMD | SMD en vez de THT (footprint diferente), over-spec voltaje |
| Alt. 4 | **C16133** | RVT1V101M0511 | Lelon | 100 µF / 35 V / 85 °C | 5×11 mm, P=2.0 mm | Over-spec voltaje pero 85 °C (no 105 °C) — evitar para producto residencial certificable |
| Alt. 5 (10 V) | **TBD — verificar en lcsc.com** | (buscar "100µF 10V 5x7 radial THT") | varios | 100 µF / 10 V / 105 °C | 5×7 mm | Solo si se quiere adherir estrictamente al maestro v1.0. Recomendación: **no usar, preferir 16 V** |
| ❌ NO usar | — | (cualquier cerámico MLCC) | — | — | — | Un MLCC 100 µF tiene de-rating severo bajo DC bias (50–80 % de pérdida) — no reemplaza al electrolítico para bulk de rail |

**Footprint recomendado en EasyEDA Pro:** `CAP-TH_BD5.0-P2.5` (radial D5×H7 mm, pitch 2.5 mm). El part `C43803` en la librería LCSC ya trae la huella correcta; importar directamente por número LCSC en EasyEDA Pro → *Place Component by LCSC*.

#### 6.2.3 Nota sobre el uso de diodo ideal / MOSFET P-channel

Una alternativa más sofisticada sería reemplazar los diodos Schottky por un **diodo ideal con controlador MOSFET P-channel** (e.g., LTC4412 o TPS2115A). Esto reduciría V_F de 0.45 V a ~0.02 V (la Rds_on del MOSFET multiplicada por la corriente), mejorando el rail a ~4.98 V.

**¿Vale la pena?** No en este producto:
- Añade 2 IC (~$0.80–1.50) y 2 MOSFET P-ch (~$0.30 c/u) → **+$1.40 al BOM** vs. $0.06 de dos SS14.
- Complejidad de layout +30 % (necesita sensing de voltaje).
- El rail actual (4.55 V) ya tiene margen suficiente para ME6211 (1.15 V de headroom). No hay problema real que resolver.
- En un producto de alto volumen (>100k unidades/año) donde el cost de $1.40 escala, podría reconsiderarse. Para este proyecto residencial de bajo volumen, **Schottky SS14 es la elección óptima**.

---

## 7. Checklist de Verificación del Módulo

Antes de enviar el diseño a fabricación JLCPCB:

### 7.1 Verificación de Esquemático

- [ ] **D_AC presente** con ánodo en `+5V_AC` y cátodo en `+5V_COMBINED`.
- [ ] **D_USB presente** con ánodo en `+5V_USB` y cátodo en `+5V_COMBINED`.
- [ ] Ambos diodos con **mismo part number SS14** y símbolo de diodo Schottky (banda en el cátodo).
- [ ] **C_bulk presente** con pad (+) en `+5V_COMBINED` y pad (−) en `GND_DC`.
- [ ] Valor de C_bulk: **100 µF / 16 V**, electrolítico (NO cerámico MLCC).
- [ ] Net `+5V_COMBINED` conectado a:
  - VIN del ME6211 (pin 1 del M4)
  - Bobina A2 del K1 / pin 2 (M3) — coincide con el cátodo del flyback D1 del M3.
- [ ] Net `+5V_AC` conectado a:
  - Pin 4 "+V" del HLK-5M05 (M2)
  - Nodo del que cuelgan C_out1, C_out2, C_out3 (M2)
- [ ] Net `+5V_USB` conectado al nodo de salida del filtro VBUS del M6 (aguas abajo del PTC1).
- [ ] `GND_DC` confirmado como plano común entre M2, M4, M5, M6, M7, M8, M9.
- [ ] Símbolos de diodo con orientación gráfica consistente (flecha apuntando del ánodo al cátodo) — no invertidos en la captura.
- [ ] Designadores D_AC, D_USB, C_bulk asignados y únicos en todo el esquemático.

### 7.2 Verificación de BOM

- [ ] **C2480 (SS14)** verificado en stock en JLCPCB Standard Parts Library en la fecha de orden.
- [ ] **C43803 (C_bulk Chengx)** verificado en stock. Si no hay, activar Alt. 1 (C45783 Lelon) o Alt. 2 (C319547 Nichicon).
- [ ] Huellas SMA DO-214AC y radial 5×7 mm importadas correctamente desde LCSC en EasyEDA Pro (sin warnings de "footprint mismatch" en el ERC).
- [ ] Campo `LCSC_Part` correctamente poblado en el CSV para las 3 filas (M7 zone).
- [ ] Campo `Module` = `M7` en las 3 filas del CSV maestro.
- [ ] Verificado que C2480 NO tiene conflicto de stock con las otras 2 referencias del mismo MPN en el BOM (D1 del M3, D_AC del M7, D_USB del M7 suman 3 unidades totales).

### 7.3 Pruebas Post-Ensamblaje (en laboratorio)

| # | Prueba | Procedimiento | Criterio de éxito |
|---|---|---|---|
| T1 | **Medida de rail en modo Solo AC** | Desconectar USB. Conectar 110 V AC al J_AC. Medir `+5V_COMBINED` con multímetro. | 4.40–4.70 V DC |
| T2 | **Medida de rail en modo Solo USB** | Desconectar AC. Conectar cable USB al PC. Medir `+5V_COMBINED`. | 4.35–4.65 V DC |
| T3 | **Medida de rail en modo Ambas fuentes** | AC + USB conectados simultáneamente. Medir `+5V_COMBINED`. | ≈ igual a T1 (dominado por HLK) |
| T4 | **Test de back-feed AC → USB** | Solo AC conectado, sin USB. Medir voltaje en ánodo de D_USB (o directamente en el pad VBUS del conector USB-C). | < 0.1 V DC (rail USB apagado) |
| T5 | **Test de back-feed USB → AC** | Solo USB conectado, sin AC. Medir voltaje en ánodo de D_AC (o en pin 4 del HLK-5M05). | < 0.1 V DC (rail AC apagado) |
| T6 | **Test de ripple sobre C_bulk** | Osciloscopio en `+5V_COMBINED`, 10 mV/div AC-coupled. Activar relé cada 500 ms. | Ripple pico a pico < 200 mV durante conmutación |
| T7 | **Test térmico de D_AC** | Termografía infrarroja o termopar en cuerpo de D_AC. Operación con relé activo y TX WiFi sostenido (>5 min). Ambiente 25 °C. | T_case < 60 °C |
| T8 | **Test térmico de D_USB** | Idem T7 pero con alimentación USB (y carga representativa, e.g., MCU en TX WiFi). | T_case < 55 °C (menor corriente típica por PTC del M6) |
| T9 | **Test de transición AC → USB en caliente** | Placa alimentada por AC. Conectar USB. Ver si MCU resetea (monitor serial). | MCU no resetea, no hay glitch perceptible en rail |
| T10 | **Test de transición USB → AC en caliente** | Placa alimentada por USB, flasheando. Energizar AC. Ver si esptool pierde sync. | esptool completa write_flash sin errores |
| T11 | **Test de corte de AC** | Placa en operación normal. Desconectar AC. Medir tiempo que el MCU mantiene alimentación. | 4–8 ms (consistente con C_bulk × I_load) |
| T12 | **Inspección visual de polaridad C_bulk** | Lupa 10× o AOI: verificar silkscreen "+" coincide con pad positivo del cap ensamblado. | Sin inversión de polaridad |

---

## 8. Conexión con Otros Módulos

Tabla resumen de las señales que cruzan la frontera del Módulo 7. En el esquemático jerárquico de EasyEDA Pro, estos son los *hierarchical sheet pins* del bloque M7.

| Señal / Net | Dirección desde M7 | Origen / Destino | Componente en el camino | Zona |
|---|---|---|---|---|
| `+5V_AC` | Entrada | **Módulo 2** (HLK-5M05 pin 4 "+V") | C_out1 ∥ C_out2 ∥ C_out3 del M2 ya filtran | DC |
| `+5V_USB` | Entrada | **Módulo 6** (salida del filtro VBUS) | Aguas abajo de PTC1, C_USB1, C_USB2 del M6 | DC |
| `+5V_COMBINED` | Salida | **Módulo 4** (ME6211 pin 1 VIN) + **Módulo 3** (K1 A2 / cátodo D1) | D_AC ∥ D_USB → C_bulk → carga | DC |
| `GND_DC` | Plano común | M2, M4, M5, M6, M8, M9 | Plano continuo | DC |

**Frontera de aislamiento:** todas las señales del Módulo 7 están en zona **DC** del secundario aislado. El rail `+5V_AC` entra *después* del aislamiento galvánico 3000 VAC del HLK-5M05 — o sea, `+5V_AC` es una señal DC aislada, no una señal de red viva. El Módulo 7 no cruza la barrera de aislamiento.

**Coherencia con el Módulo 4:** el rail `+5V_COMBINED` es la entrada del LDO ME6211 del M4, que a su vez produce el `+3.3V` del sistema. Ver [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) para el diseño del LDO.

**Coherencia con el Módulo 3:** la bobina del relé K1 (HF115F-I/005-1HS3A) se alimenta desde `+5V_COMBINED`. El diodo flyback D1 (también SS14, LCSC C2480, fila 9 del BOM maestro) está en paralelo con la bobina, con cátodo en `+5V_COMBINED` y ánodo en el colector del Q1 (SS8050). Ver [Modulo_3_Rele_Potencia_Snubber.md](Modulo_3_Rele_Potencia_Snubber.md) para el detalle.

**Coherencia con el Módulo 6:** el `+5V_USB` es la salida del filtro VBUS del M6 (no el VBUS crudo del conector USB-C). La corriente está limitada a 500 mA por el PTC1. Ver [Modulo_6_USB_C_Flasheo.md §8](Modulo_6_USB_C_Flasheo.md) para el origen de esta señal.

---

## 9. Estimación de Consumo del Módulo 7

El Módulo 7 no consume corriente por sí mismo en operación — es un circuito pasivo. Lo que sí hace es **introducir pérdidas** en la ruta de potencia: cada miliamperio que pasa por un diodo paga una caída de V_F (~0.45 V) que se disipa como calor.

### 9.1 Pérdidas por Corriente (Modo A — Solo AC)

| Estado del sistema | Corriente a través de D_AC | Disipación en D_AC | ΔT sobre ambiente |
|---|---|---|---|
| MCU en light-sleep | ~10 mA | ~4.5 mW | < 0.5 °C |
| MCU idle WiFi asociado | ~30 mA | ~13.5 mW | < 1.5 °C |
| MCU RX WiFi + relé abierto | ~90 mA | ~40 mW | ~3.5 °C |
| MCU TX WiFi sostenido + relé abierto | ~250 mA | ~110 mW | ~10 °C |
| MCU TX WiFi pico + relé cerrado (con bobina) | ~420 mA | **~200 mW** | ~18 °C |
| Pulso de conmutación relé (20 ms) | ~500 mA pico | ~260 mW transitorio | < 3 °C transitorio |

En el peor caso sostenido, Tj del SS14 en ambiente de 40 °C: 40 + 18 = **58 °C**. Margen sobre Tj_max (125 °C): 67 °C. **Holgado.**

### 9.2 Caída de Voltaje en el Rail (cost del OR-diode)

| Corriente en D_AC (o D_USB activo) | V_F típica | V(`+5V_COMBINED`) resultante (con V_in = 5.0 V) |
|---|---|---|
| 10 mA | 0.35 V | 4.65 V |
| 50 mA | 0.40 V | 4.60 V |
| 100 mA | 0.42 V | 4.58 V |
| 250 mA | 0.45 V | 4.55 V |
| 500 mA | 0.50 V | 4.50 V |
| 1 A (peor caso teórico, no ocurre) | 0.55 V | 4.45 V |

**Impacto en el ME6211 (LDO M4):** el ME6211 necesita mínimo `V_OUT + V_dropout = 3.3 + 0.1 = 3.4 V` en su VIN para regular. Con `+5V_COMBINED = 4.45 V` en el peor caso (y suponiendo −5 % de tolerancia del HLK, o sea 4.75 V en +5V_AC antes del diodo, y 1 A de corriente), el ME6211 ve **4.20 V**, todavía con **800 mV de headroom sobre su mínimo funcional**. Robustez confirmada.

### 9.3 Consumo de Leakage del Diodo Apagado

| Voltaje inverso V_R | I_R típica (SS14) | Notas |
|---|---|---|
| 5 V | ~30 µA | Caso normal cuando una fuente está apagada |
| 40 V | ~500 µA | Caso imposible en este circuito (V_R nunca supera 5 V) |
| Tj = 85 °C, V_R = 5 V | ~150 µA | Leakage se multiplica ~5× con temperatura |

**Impacto práctico:** 30–150 µA de leakage fluyendo del lado activo hacia el lado apagado es **despreciable**. El HLK-5M05 apagado tiene su salida flotante con capacitancia de 100 µF en paralelo — 30 µA cargarían ese capacitor a 5 V en ~17 segundos, pero nunca llega porque el HLK tiene un divisor de realimentación interno que drena más. En la práctica, el rail apagado permanece < 100 mV.

---

## 10. Referencias

- **SS14 Schottky Rectifier Datasheet** — MDD Microdiode Electronics, Rev B, 2018. Parámetros V_F, I_F, I_R, Tj_max, R_θJA.
- **STPS1L40A Low-Drop Power Schottky Rectifier Datasheet** — STMicroelectronics, DS5816 Rev 4, 2020. Alternativa premium con V_F más baja.
- **ME6211 Series 500 mA CMOS LDO Datasheet** — Microne (Nanjing Micro One Electronics), Rev 1.2, 2019. Dropout 100 mV @ 500 mA — define el mínimo `+5V_COMBINED` funcional.
- **HLK-5M05 AC-DC Module Datasheet** — HI-LINK, Rev 2.0, 2020. Salida 5 V ±0.25 V, rating 1 A, aislamiento 3000 VAC.
- **HF115F-I/005-1HS3A Relay Datasheet** — Hongfa, Rev 2021. Coil 5 V, 71 mA nominal, pulso inicial 500 mA @ 20 ms.
- **IPC-2152** — Standard for Determining Current-Carrying Capacity in Printed Board Design. Base del dimensionamiento de ancho de traza.
- **AN-2155 Application Note** — "OR-ing Power Supplies" — Texas Instruments, 2013. Análisis teórico general de OR-diode vs. ideal diode.
- [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) — documento maestro del proyecto, sección "Módulo 7: Selector de fuente AC/USB (OR-Diode)" (líneas 671–709).
- [Modulo_2_Fuente_AC_DC_HLK5M05.md](Modulo_2_Fuente_AC_DC_HLK5M05.md) — fuente AC-DC aislada, origen de la señal `+5V_AC`.
- [Modulo_3_Rele_Potencia_Snubber.md](Modulo_3_Rele_Potencia_Snubber.md) — relé K1 y driver BJT, destino de `+5V_COMBINED` (bobina + flyback).
- [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) — LDO 3.3 V, destino principal de `+5V_COMBINED`.
- [Modulo_6_USB_C_Flasheo.md](Modulo_6_USB_C_Flasheo.md) — interfaz USB-C, origen de la señal `+5V_USB`.
- [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) — lista maestra de materiales; filas 11, 12, 39 contienen los 3 componentes primarios verificados del Módulo 7.
- [Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md](Ingeniería inversa de la ESP32-C6_Relay_X1 y ruta a un producto residencial certificable.md) — el diseño chino NO tiene selector de fuente (solo AC), por lo que no se puede flashear in-situ con la placa sin abrir la carcasa — diferencial de producto nuestro.

---

**Fin del documento — Módulo 7: Selector de Fuente AC/USB (OR-Diode) v1.0**
