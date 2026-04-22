# Módulo 8: Detección de Interruptor Físico con Optoacoplador PC817C — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-21
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 8 es la **interfaz sensora de presencia de tensión AC en el interruptor de pared** existente en la instalación eléctrica del usuario. Permite que el Smart Relay detecte si el ocupante acciona el interruptor físico (tipo toggle de pared o pulsador momentáneo) y reaccione en consecuencia (conmutar el relé, reportar estado a Home Assistant vía ESPHome, etc.) **manteniendo aislamiento galvánico completo de 5000 Vrms** entre la zona AC caliente y la zona DC del microcontrolador.

Sus funciones son:

1. **Sensar presencia de red AC** (110 V o 220 V RMS) en el terminal `J_SW` conectado al interruptor de pared.
2. **Rectificar y limitar la corriente AC** a valores seguros (~0.36 mA promedio a 110 V, ~0.72 mA a 220 V) antes de entregarla al LED del optoacoplador.
3. **Convertir la señal AC en una señal lógica DC** mediante el fototransistor del PC817C, con salida `SW_IN` hacia **GPIO5** del ESP32-C3.
4. **Mantener aislamiento galvánico 5000 Vrms** entre la red eléctrica y el lado DC — requisito de seguridad UL/CE y condición para conectar USB mientras la placa está energizada (ver [Modulo_6_USB_C_Flasheo.md](Modulo_6_USB_C_Flasheo.md)).
5. **Filtrar el ripple de 50/60 Hz** resultante de la rectificación de media onda y el rebote mecánico del interruptor mediante un filtro RC hardware (τ = 1 ms) complementado por debounce firmware en ESPHome.
6. **Soportar operación dual 110 V / 220 V** sin cambio de componentes — el mismo circuito opera correctamente en toda la gama residencial y hasta el peor caso high-line de 265 V RMS.

**Entradas del módulo:**
- `SW_HOT` — terminal vivo proveniente del interruptor de pared. Puede estar a +155 V pico (110 V RMS), +311 V pico (220 V RMS) o 0 V según el estado del interruptor.
- `AC_N` — neutro de la red eléctrica (retorno del camino AC), común al Módulo 1.
- `+3.3V` — rail regulado desde el Módulo 4 (ME6211) para el pull-up del fototransistor.
- `GND_DC` — plano común del secundario aislado compartido con M2/M4/M5/M6/M7/M9.

**Salidas del módulo:**
- `SW_IN` — señal lógica activa-baja hacia **GPIO5** del Módulo 5 (ESP32-C3-MINI-1-N4, pin 19 / IO5). LOW cuando el interruptor de pared está cerrado (AC presente), HIGH cuando está abierto.

> **Nota sobre polaridad:** la lógica del driver es **invertida** (interruptor cerrado → GPIO LOW). En ESPHome se corrige con `inverted: true` en la configuración del `binary_sensor` de GPIO5, exponiendo al usuario la semántica natural "cerrado = ON".

### 1.1 Por qué optoacoplador y no un divisor resistivo directo (como Shelly)

Shelly y otros diseños comerciales de bajo costo usan un divisor resistivo directo de la red al GPIO porque su rail DC **ya está al potencial de red** (emplean una fuente buck no-aislada tipo capacitor-dropper o HLW8032). En ese caso el GND del MCU literalmente flota al neutro o a la fase, y poner un divisor resistivo entre L y GPIO no rompe nada — ya estaba "roto" por diseño.

Este diseño es diferente: el Módulo 2 (HLK-5M05) provee **aislamiento galvánico 3000 VAC entre primario AC y secundario DC**. Usar un divisor resistivo desde la red hasta un GPIO del ESP32-C3 **anularía por completo esa barrera**: un transitorio como un rayo acoplado a la red llegaría directo al SoC, al USB-C, y eventualmente al PC del usuario que esté flasheando. Todo el costo del HLK-5M05 y del layout de aislamiento quedaría desperdiciado.

El **PC817C** resuelve esto con una barrera adicional de **5000 Vrms entre su LED (zona AC) y su fototransistor (zona DC)**, cumpliendo IEC 60747-5-5 (reinforced insulation). Es la forma canónica de "cruzar" la frontera AC↔DC con una señal lógica sin romper el aislamiento del módulo de potencia.

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Aislamiento Galvánico con el PC817C

El PC817 es un optoacoplador de 4 pines (disponible en DIP-4 THT o SMD-4P — este diseño usa **SMD-4P** del LCSC C3025164 por disponibilidad de stock y compatibilidad con SMT assembly en JLCPCB) compuesto internamente por un **diodo LED infrarrojo de GaAs** (pines 1-2, zona primaria) y un **fototransistor NPN de silicio** (pines 3-4, zona secundaria). Los dos componentes están encapsulados en el mismo package pero separados por una resina epoxi aislante con rigidez dieléctrica de **5000 Vrms por 1 minuto** (IEC 60747-5-5).

No hay ninguna conexión eléctrica entre ambos lados — toda la transferencia de información ocurre **por luz infrarroja** (~950 nm) del LED al fototransistor. Eso significa:

- Un transitorio de hasta 5000 V pico entre el lado AC y el lado DC **no llega** al fototransistor ni al GPIO del ESP32-C3.
- La frontera del optoacoplador es la **barrera reinforced** que separa la zona caliente (SELV failed + mains) de la zona fría (SELV compliant + USB).
- En el PCB, el cuerpo del PC817 **cruza físicamente un slot fresado de 2 mm** entre las dos zonas — el slot elimina el camino de creepage sobre FR-4, dejando solo el camino a través del cuerpo del optoacoplador (5000 Vrms por diseño).

#### 2.1.1 Grado "C" del PC817

El sufijo "C" (PC817**C**) indica un CTR (Current Transfer Ratio, relación de transferencia de corriente) **mínimo de 200 %** a 5 mA de corriente de LED. Esto es crítico porque en este diseño **el LED opera muy por debajo de 5 mA** — a 110 V operamos con I_LED promedio de solo **0.36 mA**, y a 220 V con **0.72 mA**. A esas corrientes tan bajas, el CTR real baja a ~100 % (110 V) y ~150 % (220 V) según las curvas del datasheet Sharp.

Con un grado "A" (CTR mínimo 80 %) o "B" (CTR 130–260 %) no habría margen suficiente para garantizar saturación del fototransistor a 110 V RMS. El grado "C" con CTR ≥ 200 % garantiza que incluso en el peor caso de dispersión de fábrica y baja corriente, el fototransistor recibe suficiente fotocorriente para llevar el colector cerca de 0 V (saturación).

**Reemplazos válidos:**
- **LTV-817S** (Lite-On, LCSC C7063) — pin-compatible, CTR 200–400 %.
- **EL817(C)** (Everlight, LCSC C2835224) — pin-compatible, CTR 200–400 %.

No se debe sustituir por un PC817A o PC817B — la probabilidad de lecturas incorrectas a 110 V aumenta significativamente.

### 2.2 Rectificación de Media Onda con D2 (1N4007)

El LED del PC817 conduce **solo en el semiciclo positivo** de la AC. Sin rectificación, durante el semiciclo negativo el LED se polarizaría inversamente con ~155 V (a 110 V RMS) y se **destruiría instantáneamente** — el LED infrarrojo del PC817 tiene un V_R (reverse voltage) máximo de solo **6 V** (datasheet Sharp §Absolute Maximum Ratings).

El diodo **D2 (1N4007)** en serie con el LED bloquea el semiciclo negativo:

- **Semiciclo positivo:** D2 conduce, corriente fluye por R3 → R4 → D2 → LED PC817 → neutro. El LED emite luz durante ~8.3 ms (50 Hz) o ~10 ms (60 Hz) por ciclo.
- **Semiciclo negativo:** D2 está polarizado inversamente, bloqueando los 311 V pico (peor caso 220 V) que estresarían al LED. La caída en el LED durante esta fase es 0 V (sin corriente).

#### 2.2.1 Por qué 1N4007 y no 1N4148 o 1N4001

El 1N4007 tiene **V_R = 1000 V**, holgadamente por encima del pico de 311 V (220 V RMS) y incluso del 375 V pico de 265 V RMS (peor caso high-line). Opciones como:
- **1N4148** (V_R = 100 V) — se destruiría a 220 V.
- **1N4001** (V_R = 50 V) — se destruiría incluso a 110 V pico.
- **1N4004** (V_R = 400 V) — marginal a 220 V, sin margen para surges.
- **1N4007** — **elección correcta**, margen de seguridad suficiente para residencial y exportación.

El encapsulado elegido en este diseño es **SMA (DO-214AC)** — versión SMD del 1N4007, para reducir esfuerzo de ensamblaje manual. El MPN `1N4007W` (YUNSUNENERGY, LCSC C727110) es la variante SMA común.

### 2.3 Divisor de Corriente con R3 + R4 (2× 68 kΩ)

La corriente promedio del LED debe estar típicamente entre **0.3 mA y 5 mA**. Con V_pico = 155 V (110 V RMS), V_LED = 1.2 V, V_D2 = 1.0 V, necesitamos una resistencia serie de:

```
R_total = (V_pico - V_LED - V_D2) / I_pico_deseada
       = (155 - 1.2 - 1.0) / 1.1 mA
       = 138 kΩ
```

Se implementa como **dos resistores de 68 kΩ en serie** (R_total = 136 kΩ, suficientemente cercano) en lugar de un único resistor de 136 kΩ por **dos razones de seguridad**:

#### 2.3.1 Reparto de la disipación de potencia

A 220 V RMS, la potencia disipada total es `V_rms² / R_total = 220² / 136k = 0.356 W`. Un único resistor de 136 kΩ 1/2 W estaría al 70 % de su nominal — marginal y con derating térmico severo en una carcasa cerrada. Con **dos resistores de 68 kΩ 1 W cada uno**:

- Potencia por resistor a 110 V: `110² / (2 × 68k) = 0.089 W` → **9 %** del nominal.
- Potencia por resistor a 220 V: `220² / (2 × 68k) = 0.356 W` → **36 %** del nominal.
- Potencia por resistor a 265 V (peor caso): `265² / (2 × 68k) = 0.517 W` → **52 %** del nominal.

Amplio margen térmico incluso en el peor caso. La vida útil del resistor MFR (Metal Film Resistor) a 50 % del nominal es típicamente **> 20 años** según AEC-Q200.

#### 2.3.2 Reparto de la tensión (fiabilidad de aislamiento)

Los resistores THT axiales 1 W típicos tienen una **máxima tensión de trabajo continua de 350–500 V** (limitada por el grabado en espiral de la película metálica). A 220 V RMS, V_pico = 311 V — cerca del límite. Con **dos resistores en serie** la tensión se reparte: cada resistor ve solo ~155 V pico, con margen de 2.3× hasta el límite.

Un único resistor de 136 kΩ en un transitorio de red (surge IEC 61000-4-5, 4 kV / 2 Ω, 100 A) vería más de 2 kV entre sus terminales durante los ~50 µs del pulso — suficiente para arquear el cuerpo del resistor (breakdown en espiral) y cortocircuitar al LED. Con dos resistores el pico se reparte, y cada uno ve solo ~1 kV, por debajo del threshold de arqueo típico de MFR 1 W.

#### 2.3.3 Tipo MFR (Metal Film Resistor) — no wirewound ni carbón

- **Wirewound (bobinados):** añadirían inductancia parásita significativa (varios µH) que formaría un LC tank con capacitancias parásitas y produciría oscilaciones en el transitorio del interruptor. Malo para EMC.
- **Carbón (Carbon Composition):** baja estabilidad térmica (TCR > ±500 ppm/°C), deriva del valor con los ciclos AC, potencia nominal no sostenible a largo plazo.
- **MFR (Metal Film):** TCR ±100 ppm/°C, estabilidad a largo plazo excelente, comportamiento lineal hasta cientos de MHz, sin inductancia parásita significativa. **Elección canónica** para aplicaciones de entrada AC en productos certificables.

El MPN elegido, **YAGEO MFR1WSJT-52-68K** (LCSC C176393), tiene:
- Potencia: 1 W @ 70 °C
- Tolerancia: ±5 %
- TCR: ±100 ppm/°C
- Tensión máx. de trabajo continuo: 500 V
- AEC-Q200 cualificado (automotive-grade)
- Encapsulado: axial D3.3 × L9 mm (pitch estándar 10 mm en PCB)

### 2.4 Pull-up del Fototransistor y Lógica Invertida

Del lado DC, el fototransistor del PC817 actúa como un **interruptor controlado por luz** conectado entre el GPIO5 y GND:

```
+3.3V ──[R5: 10 kΩ pull-up]──┬── GPIO5 (nodo SW_IN)
                              │
                         PC817 Pin 4 (Colector)
                              │
                         PC817 Pin 3 (Emisor)
                              │
                             GND_DC
```

**Estados posibles:**

1. **LED del PC817 apagado** (interruptor de pared abierto, sin AC):
   - Fototransistor en **corte** (alta impedancia, >1 MΩ colector-emisor).
   - R5 (10 kΩ) a +3.3V domina → GPIO5 = **3.3 V (lógica HIGH)**.

2. **LED del PC817 encendido** (interruptor de pared cerrado, AC presente):
   - Fototransistor en **saturación** — la I_colector alcanza el valor limitado por R5 (≈ 330 µA a 3.3 V).
   - V_CE(sat) típico < 0.2 V a esta corriente → GPIO5 ≈ **0 V (lógica LOW)**.

**Verificación de saturación a 110 V (peor caso de baja luz):**
- I_LED_avg = 0.358 mA, CTR ~100 % → I_fototransistor disponible = 0.358 mA.
- I_requerida por R5 para saturar = 3.3 V / 10 kΩ = 0.33 mA.
- **0.358 mA > 0.33 mA** → saturación garantizada con margen de 8 %. ✓

A 220 V el margen sube a 3× (I_disponible ~1.08 mA vs requerida 0.33 mA).

#### 2.4.1 Por qué 10 kΩ para R5 (no 1 kΩ, no 100 kΩ)

- **1 kΩ:** I_requerida subiría a 3.3 mA → a 110 V el fototransistor no saturaría (solo 0.358 mA disponibles). Fallo a bajo voltaje.
- **100 kΩ:** I_requerida bajaría a 33 µA (sobra margen), **pero** la impedancia del nodo GPIO5 sube a 100 kΩ. Esto lo hace susceptible a captación EMI (cable del switch puede medir 5–10 m y actuar como antena), y además alarga la constante de tiempo con C_deb (100 nF) a τ = 10 ms — demasiado lento para debounce efectivo.
- **10 kΩ:** equilibrio óptimo. Saturación garantizada a 110 V, impedancia moderada para inmunidad EMI, τ = 1 ms adecuado para filtrar el ripple de 50/60 Hz sin introducir latencia perceptible.

El MPN **UNI-ROYAL 0402WGF1002TCE** (LCSC C25744) es el mismo resistor 10 kΩ 0402 que se reutiliza en 4 lugares del diseño (R2, R5, R_NTC_PU, R_EN, R_STRAP) — reduce el conteo de SKUs en el BOM de fabricación.

### 2.5 Filtro RC Hardware de Debounce — C_deb (100 nF)

Aunque el LED del PC817 pulsa a **50/60 Hz** (media onda rectificada → pulsos de 50/60 Hz con ancho de ~8.3 ms a 60 Hz y hueco de 8.3 ms), no queremos que GPIO5 oscile a esa frecuencia — el firmware ESPHome vería "apagado-encendido-apagado-encendido..." a 60 Hz en vez de un LOW sólido.

El capacitor **C_deb (100 nF X7R 0402)** en paralelo con R5 forma un filtro pasa-bajos:

```
τ = R × C = 10 kΩ × 100 nF = 1 ms
```

Con τ = 1 ms, la respuesta del nodo GPIO5 a una entrada cuadrada de 60 Hz (16.6 ms de periodo) es que el voltaje alcanza ~99 % del steady-state en ~5τ = 5 ms. Durante cada semiciclo positivo (8.3 ms de LED ON) hay tiempo de sobra para que C_deb se descargue a ~0 V vía el fototransistor saturado; durante el semiciclo negativo (8.3 ms de LED OFF) el pull-up R5 recarga C_deb, pero solo hasta ~30 % del camino a 3.3 V (e^(−8.3/1) ≈ 0). Resultado: el nodo GPIO5 permanece **sólidamente LOW** (~0 V + ~30 mV de ripple) durante toda la presencia de AC.

**Cuando el interruptor se abre:** el LED se apaga instantáneamente, el fototransistor pasa a alta impedancia, R5 empieza a cargar C_deb hacia +3.3V. El nodo llega a 2.3 V (threshold HIGH del ESP32-C3) en ~1.1 ms — latencia imperceptible.

**Debounce complementario en firmware:** además del filtro hardware, la configuración ESPHome usa `delayed_on_off: 50ms` sobre el binary_sensor de GPIO5. Los 50 ms son un estándar de industria para absorber rebote mecánico de interruptores tipo toggle (que pueden rebotar 5–15 ms).

#### 2.5.1 X7R vs otras dielectricidades

El capacitor debe ser **X7R** (o X5R) — NO C0G/NP0 ni Y5V:
- **C0G/NP0:** tolerancia ±5 %, estabilidad térmica excelente, pero cápsula 100 nF 0402 no existe o es prohibitivamente cara.
- **Y5V:** deriva térmica y de voltaje catastrófica (hasta −80 % a +85 °C). Inaceptable.
- **X7R:** deriva térmica moderada (±15 % en −55 a +125 °C), deriva de voltaje ~−15 % a 16 V. Adecuado para filtros de debounce donde ±20 % del capacitor cambia τ de 0.8 ms a 1.2 ms — ambos aceptables.

El MPN **YAGEO CC0402KRX7R9BB104** (LCSC C14663) es el mismo capacitor 100 nF que se reutiliza en 4 lugares (C6, C_vdd2, C_EN, C_deb).

### 2.6 Compatibilidad 110 V / 220 V — Tabla Consolidada

| Parámetro | 110 V RMS | 220 V RMS | 265 V RMS (peor caso) | Unidad | Límite |
|---|---|---|---|---|---|
| V_pico (AC) | 155 | 311 | 375 | V | — |
| I_LED_pico | 1.13 | 2.27 | 2.74 | mA | < 50 mA (PC817 max) ✓ |
| I_LED_promedio (½ onda) | 0.358 | 0.722 | 0.871 | mA | — |
| P(R3 = R4 = 68 kΩ) | 0.089 | 0.356 | 0.517 | W | < 1 W nominal ✓ |
| P_total (R3 + R4) | 0.18 | 0.71 | 1.03 | W | — |
| V_R2 (sobre cada resistor) | 77 | 155 | 187 | V pico | < 500 V nominal ✓ |
| CTR efectivo @ I_LED_avg | ~100 % | ~150 % | ~180 % | % | ≥ 200 % @ 5 mA (spec C) |
| I_colector resultante | 0.36 | 1.08 | 1.57 | mA | — |
| I_requerida por R5 (sat) | 0.33 | 0.33 | 0.33 | mA | — |
| Margen de saturación | 1.08× | 3.27× | 4.75× | — | > 1.0× requerido ✓ |
| V_GPIO5 (estado cerrado) | ~0.15 | ~0.10 | ~0.08 | V | < 0.5 V threshold LOW ✓ |

**Conclusión:** el circuito opera con margen adecuado en toda la gama 90–265 V RMS sin cambios de componentes.

---

## 3. Asignación de Pines — PC817C (SMD-4P) y Posición en PCB

### 3.1 Pinout del PC817C (vista desde arriba, con el notch a la izquierda)

```
            ┌───────────────┐
            │  ●            │   ← notch indicador
  Pin 1 ────┤ Anode    NC   ├──── Pin 4 (Collector)
            │  (LED)        │
  Pin 2 ────┤ Cathode       ├──── Pin 3 (Emitter)
            │  (LED)        │
            └───────────────┘

       Lado AC  ←──────────────→  Lado DC
       (primario)                 (secundario)
```

El pinout lógico es **idéntico en DIP-4 y SMD-4P** — solo cambia el método de soldadura (THT vs SMT gull-wing).

### 3.2 Correspondencia con el símbolo EasyEDA Pro

Huella LCSC **C3025164** — GOODWORK `PC817C`, encapsulado **SMD-4P** (pitch entre filas 7.62 mm, pitch entre pines de una fila 2.54 mm, pads gull-wing superficiales). El símbolo en EasyEDA Pro expone los 4 pines directamente sin bondeo interno.

**Alternativa THT (si se prefiere ensamblaje manual):** huella LCSC C115500 (Wuxi China Resources Huajing) en encapsulado **DIP-4** pasante. Mismas dimensiones del cuerpo, pines atravesando el PCB. La huella debe cambiarse explícitamente en EasyEDA Pro — no son intercambiables en el PCB aunque compartan el símbolo esquemático.

| Pin símbolo | Función | Zona | Net |
|---|---|---|---|
| 1 | Anode (LED +) | **AC primario** | `OPTO_LED_A` |
| 2 | Cathode (LED −) | **AC primario** | `AC_N` (neutro) |
| 3 | Emitter (fototransistor) | **DC secundario** | `GND_DC` |
| 4 | Collector (fototransistor) | **DC secundario** | `SW_IN` |

### 3.3 Posición Física en el PCB

El cuerpo del PC817 debe colocarse **cruzando el slot fresado** de 2 mm que separa las zonas AC y DC:

```
   ────────────────────────────────────────────
   ZONA AC (hot)              ZONA DC (cool)
   
   [R3]   [R4]                                
     │      │                                 
     └───[D2]──┐                              
              │                                
          ┌───┴──────╔═══════╗────────┐       
          │          ║       ║        │       
          │  ●Pin 1  ║PC817C ║  Pin 4─┼── [R5]───── +3.3V
          │          ║       ║        │       ┐
          │   Pin 2  ║       ║  Pin 3─┼── GND │
          │          ╚═══════╝        │       │ [C_deb]
          └─────┬────────────────────┬┘       │     
                │  ← SLOT 2 mm →     │        └── GND
                ▼                    ▼
           Creepage AC/DC: ≥ 8 mm medido sobre el cuerpo del PC817
   
   ────────────────────────────────────────────
```

**Criterios críticos:**
- El **slot fresado** debe tener al menos **2 mm de ancho** a lo largo de toda la frontera AC/DC bajo el PC817. Esto elimina el camino de creepage sobre FR-4 (que normalmente requeriría 6.4 mm según IEC 60664-1 @ 300 V RMS Pollution Degree 2).
- La separación entre pines 1-2 (lado AC) y pines 3-4 (lado DC) del propio PC817 es **7.62 mm** (pitch entre filas, idéntico en SMD-4P y DIP-4), garantizando el creepage funcional por encima del cuerpo del componente.
- **Ningún otro componente, via o traza de señal** debe cruzar el slot entre las zonas AC y DC. El PC817 es el único "puente" permitido.
- La huella LCSC C66463 en EasyEDA Pro incluye el outline correcto — verificar en DRC que no hay vías dentro del keepout del cuerpo.

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama ASCII Completo del Módulo 8

```
   MÓDULO 8 — DETECCIÓN DE INTERRUPTOR FÍSICO (PC817C)
   ════════════════════════════════════════════════════

   ┌──────────────────────  ZONA AC (primario, hot) ──────────────────────┐
   │                                                                      │
   │   J_SW pin 1 (SW_HOT, hacia interruptor de pared)                    │
   │          │                                                           │
   │          ▼                                                           │
   │   ┌──────────┐                                                       │
   │   │ R3 68 kΩ │  1 W MFR axial THT                                    │
   │   │  ±5%     │                                                       │
   │   └────┬─────┘                                                       │
   │        │                                                             │
   │        ▼                                                             │
   │   ┌──────────┐                                                       │
   │   │ R4 68 kΩ │  1 W MFR axial THT                                    │
   │   │  ±5%     │                                                       │
   │   └────┬─────┘                                                       │
   │        │                                                             │
   │        ▼                                                             │
   │   ┌──────────┐                                                       │
   │   │ D2 1N4007│  Ánodo arriba (hacia R4), cátodo abajo               │
   │   │  SMA     │   (hacia LED del PC817)                               │
   │   └────┬─────┘                                                       │
   │        │                                                             │
   │        ▼                                                             │
   │     OPTO_LED_A  ───────────► Pin 1 (Anode) del PC817                 │
   │                                       │                              │
   │                                       │   ┌─────────────┐            │
   │                                       │   │             │            │
   │                                       ▼   │   PC817C    │            │
   │                                  [LED IR ~950 nm]       │            │
   │                                       │   │             │            │
   │                                       │   └─────────────┘            │
   │                                       ▼                              │
   │     AC_N  ◄──────────────────── Pin 2 (Cathode) del PC817            │
   │       │                                                              │
   │       ▼                                                              │
   │   J_SW pin 2 (AC_N, neutro, común con Módulo 1)                      │
   │                                                                      │
   └──────────────────────────────────────────────────────────────────────┘
                                        │
                    ═══════════════════ SLOT FRESADO 2 mm ═══════════════
                          (aislamiento galvánico 5000 Vrms PC817C)
                                        │
   ┌──────────────────────  ZONA DC (secundario, cool) ───────────────────┐
   │                                                                      │
   │                              ┌─────────────┐                         │
   │                              │             │                         │
   │                              │   PC817C    │                         │
   │                              │             │                         │
   │                              └─────────────┘                         │
   │                                  │     │                             │
   │                             Pin 4 │     │ Pin 3                      │
   │                          (Coll.)  │     │ (Emit.)                    │
   │                                  │     │                             │
   │                                  ▼     ▼                             │
   │         +3.3V                   SW_IN  GND_DC                        │
   │           │                      │     │                             │
   │           │                      │     │                             │
   │      ┌────┴───┐                  │     │                             │
   │      │ R5 10k │ 0402 SMD         │     │                             │
   │      │ ±5%    │                  │     │                             │
   │      └────┬───┘                  │     │                             │
   │           │                      │     │                             │
   │           └──────────────────────┤     │                             │
   │                                  │     │                             │
   │                         ┌────────┤     │                             │
   │                         │        │     │                             │
   │                         │        ▼     ▼                             │
   │                         │     SW_IN ─────► GPIO5 (Módulo 5, pin 19)  │
   │                         │                                            │
   │                    ┌────┴───┐                                        │
   │                    │ C_deb  │ 100 nF X7R 0402                        │
   │                    │ 100 nF │                                        │
   │                    └────┬───┘                                        │
   │                         │                                            │
   │                         ▼                                            │
   │                       GND_DC                                         │
   │                                                                      │
   └──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Diagrama ASCII Lado AC Detallado (primario)

```
   J_SW  — Terminal Block Screw KF128-5.0-2P (LCSC C474950)
   ══════════════════════════════════════════════════════════

        ┌─────────────────────────────────┐
        │                                 │
        │   ○ Pin 1 (SW_HOT)              │◄── Cable hacia interruptor de pared
        │                                 │
        │   ○ Pin 2 (AC_N / neutro)       │◄── Cable hacia neutro residencial
        │                                 │
        └─────────────────────────────────┘
                │                    │
                │ Net: SW_HOT        │ Net: AC_N
                │ (0 V o 155/311 V   │ (0 V, referencia AC)
                │  pico según estado │
                │  del interruptor)  │
                │                    │
                ▼                    │
           ┌────────────┐            │
           │  R3  68kΩ  │  1W MFR    │
           │  THT axial │  C176393   │
           └─────┬──────┘            │
                 │                   │
                 │  (V_pico/2 máx    │
                 │   en este nodo)   │
                 ▼                   │
           ┌────────────┐            │
           │  R4  68kΩ  │  1W MFR    │
           │  THT axial │  C176393   │
           └─────┬──────┘            │
                 │                   │
                 ▼                   │
            ╔═══════╗                │
            ║  D2   ║  1N4007W SMA   │
            ║  ──▷│ ║  (DO-214AC)    │
            ║  ánodo  → cátodo       │
            ╚═══╤═══╝                │
                │                    │
                │ Net: OPTO_LED_A    │
                ▼                    │
          ┌─────────────┐            │
          │   PC817C    │            │
          │ Pin 1 ●     │ LED anode  │
          │   (Anode)   │            │
          │   │         │            │
          │   ▼         │            │
          │  [LED GaAs] │            │
          │   │         │            │
          │   ▼         │            │
          │ Pin 2       │ LED cathode│
          │ (Cathode)   │◄───────────┘
          └─────────────┘
```

### 4.3 Diagrama ASCII Lado DC Detallado (secundario)

Hay solo **3 componentes** del lado DC (R5, PC817 pines 3-4, C_deb) más la conexión a GPIO5. La topología es un clásico **pull-up con colector abierto + filtro RC**:

```
   PC817C — Lado DC (fototransistor) y filtro de salida
   ═════════════════════════════════════════════════════

                 +3.3V  (rail del Módulo 4 / ME6211)
                   │
                   │
              ┌────┴────┐
              │   R5    │   ← pull-up (10 kΩ, 0402, C25744)
              │  10 kΩ  │
              └────┬────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
        │          │          │
        │          ▼          │
        │      ═══════        │
        │      Net SW_IN ─────┼──────► GPIO5 del ESP32-C3
        │                     │        (pin 19 del módulo MINI-1,
        │                     │         Módulo 5)
        │                     │
   ┌────┴─────┐         ┌─────┴────┐
   │  PC817   │         │  C_deb   │   ← filtro debounce
   │  Pin 4   │         │  100 nF  │     (0402 X7R, C131394)
   │(Colector)│         │  50 V    │
   │          │         │  X7R     │
   │     │    │         │          │
   │     ▼    │         │          │
   │  [foto-  │         │          │
   │   trans] │         │          │
   │     │    │         │          │
   │     ▼    │         │          │
   │  Pin 3   │         │          │
   │(Emisor)  │         │          │
   │    │     │         │          │
   └────┼─────┘         └────┬─────┘
        │                    │
        │                    │
        └──────────┬─────────┘
                   │
                   ▼
                 GND_DC   (plano común con M2/M4/M5/M6/M7/M9)
```

**Lectura del diagrama (4 conexiones físicas en el PCB):**

1. **R5** — pad 1 a **+3.3V**, pad 2 al **pin 4 del PC817** (colector). Net `SW_IN` nace aquí.
2. **C_deb** — pad 1 al mismo **pin 4 del PC817** (en paralelo con el colector), pad 2 a **GND_DC**.
3. **Pin 3 del PC817** (emisor) — directo a **GND_DC**.
4. **Net `SW_IN`** (que es eléctricamente idéntico a "pin 4 del PC817") — sale como traza hacia **GPIO5** del ESP32-C3.

En el esquemático de EasyEDA Pro se ve como **un solo nodo con 4 conexiones** (una junction point):
- R5 viene de arriba (+3.3V)
- C_deb baja a GND
- Pin 4 del PC817 entra por un lado
- Traza `SW_IN` sale hacia GPIO5

### 4.4 Cómo se comporta el circuito (analogía mecánica)

Piensa el fototransistor del PC817 como un **interruptor mecánico** que conecta `SW_IN` a `GND_DC` cuando "ve" luz del LED interno:

| Estado del interruptor de pared | LED del PC817 | Fototransistor (Pin 4 ↔ Pin 3) | Nodo `SW_IN` |
|---|---|---|---|
| **Abierto** (no hay AC en J_SW) | Apagado | **Abierto** (alta impedancia) | R5 tira hacia +3.3V → **HIGH (3.3 V)** |
| **Cerrado** (AC presente en J_SW) | Pulsando a 50/60 Hz | **Cerrado** (baja impedancia, ≈V_CE_sat = 0.2 V) | R5 pierde contra el "corto" del transistor → **LOW (~0 V)** |

El **C_deb** absorbe el ripple de 50/60 Hz del LED (rectificación de media onda) para que `SW_IN` quede plano DC y no oscile a frecuencia de red.

### 4.5 Tabla Pin a Pin Exhaustiva — PC817C (U3)

Huella **SMD-4P** del LCSC C3025164 (primario recomendado) o **DIP-4 through-hole** del LCSC C115500 (alternativa THT). Pin 1 identificado por el punto/notch en el encapsulado.

| Pin | Nombre | Zona | Net | Conexión aguas arriba | Conexión aguas abajo |
|---|---|---|---|---|---|
| 1 | Anode (LED) | **AC primario** | `OPTO_LED_A` | D2 cátodo (salida del rectificador de media onda) | — (interno al optoacoplador) |
| 2 | Cathode (LED) | **AC primario** | `AC_N` | — (interno al optoacoplador) | J_SW pin 2 (neutro de red) |
| 3 | Emitter (fototransistor) | **DC secundario** | `GND_DC` | — (interno) | Plano GND_DC común |
| 4 | Collector (fototransistor) | **DC secundario** | `SW_IN` | — (interno) | Nodo común con R5 + C_deb → GPIO5 |

> **Nota de orientación en EasyEDA Pro:** el símbolo del C66463 debe colocarse con pines 1-2 hacia la zona AC del esquemático (izquierda por convención) y pines 3-4 hacia la zona DC (derecha). La huella en el PCB debe alinearse de forma que el cuerpo cruce el slot fresado de forma consistente con esta orientación.

### 4.6 Tabla Pin a Pin — J_SW (Terminal Block Screw)

Huella **THT paso 5.0 mm** del LCSC C474950 (Cixi Kefa `KF128-5.0-2P`). Rating 300 V / 10 A — suficiente para 110/220 V residencial.

| Pin | Función | Zona | Net | Conexión externa (campo) | Conexión interna PCB |
|---|---|---|---|---|---|
| 1 | SW_HOT | **AC primario** | `SW_HOT` | Cable hacia interruptor de pared (otro extremo a fase L) | R3 pad 1 |
| 2 | AC_N | **AC primario** | `AC_N` | Cable hacia neutro residencial | Junction con PC817 pin 2 (cathode LED) |

> **Instalación de campo:** el usuario cablea **L → interruptor → J_SW pin 1 (SW_HOT)** y **N → J_SW pin 2 (AC_N)**. Cuando el interruptor está cerrado, hay diferencia de potencial AC entre los dos terminales y el LED del PC817 conduce en semiciclos positivos. Cuando el interruptor está abierto, ambos terminales quedan al mismo potencial (neutro flotante o ambos a N), el LED no conduce.

### 4.7 Subcircuitos Detallados

```
(a) Limitador de corriente AC + rectificador   (b) Pull-up + filtro debounce DC

   SW_HOT                                         +3.3V
     │                                              │
   [R3: 68 kΩ 1 W MFR]                           [R5: 10 kΩ 0402]
     │                                              │
   [R4: 68 kΩ 1 W MFR]                              ├──► SW_IN ──► GPIO5
     │                                              │
   [D2: 1N4007 SMA]  ──▷│─                       [C_deb: 100 nF X7R 0402]
     │                                              │
     ▼                                              ▼
   Pin 1 PC817 (Anode LED)                       GND_DC
                                                    ▲
                                                    │
                                              Pin 3 PC817 (Emitter)
                                                    ▲
                                              Pin 4 PC817 (Collector) ◄── SW_IN


(c) Camino completo de la señal

   Interruptor de pared cerrado
         │
         ▼
   AC pico 155 V (110 V RMS) o 311 V (220 V RMS) entre SW_HOT y AC_N
         │
         ▼
   R3 + R4 (136 kΩ) limitan corriente a ≤ 2.3 mA pico
         │
         ▼
   D2 (1N4007) rectifica — solo pasa semiciclos positivos
         │
         ▼
   LED del PC817 emite pulsos de luz infrarroja a 50/60 Hz
         │
         ▼ (atravesando barrera de 5000 Vrms por el aire + epoxy)
         │
   Fototransistor satura — colector baja a ~0 V
         │
         ▼
   C_deb absorbe el ripple de 50/60 Hz (τ = 1 ms)
         │
         ▼
   GPIO5 = LOW sólido
         │
         ▼
   ESPHome lee binary_sensor con inverted:true → "interruptor ON"
```

---

## 5. Cálculos Eléctricos Consolidados

### 5.1 Parámetros de entrada

| Parámetro | Valor | Origen |
|---|---|---|
| R_total (R3 + R4) | 136 kΩ | BOM, Cixi MFR1WSJT |
| R5 (pull-up DC) | 10 kΩ | BOM, UNI-ROYAL 0402WGF1002TCE |
| C_deb | 100 nF | BOM, YAGEO CC0402KRX7R9BB104 |
| V_LED (PC817 forward, típ.) | 1.2 V @ 1 mA | Datasheet Sharp §Electrical Characteristics |
| V_D2 (1N4007 forward) | 1.0 V @ 1 mA | Datasheet YUNSUNENERGY |
| V_CE(sat) PC817C | 0.2 V @ I_c = 1 mA | Datasheet Sharp |
| CTR PC817C @ 5 mA | 200 % mínimo | Datasheet Sharp (grado C) |
| CTR PC817C @ 0.5 mA | ~100 % típico | Curva de datasheet (degradación a baja I_F) |
| Vpp ripple aceptable en GPIO5 | < 0.5 V | Margen respecto a VIL = 0.25 × V_DD = 0.825 V |

### 5.2 Cálculos para 110 V RMS (operación Colombia doméstico)

```
V_pico = 110 × √2 = 155.6 V
V_drop_fijo = V_LED + V_D2 = 1.2 + 1.0 = 2.2 V
V_útil = V_pico − V_drop = 153.4 V

I_LED_pico = V_útil / R_total = 153.4 / 136,000 = 1.128 mA
I_LED_promedio (½ onda, factor 0.318) = 1.128 × 0.318 = 0.359 mA
I_LED_RMS (½ onda, factor 0.5) = 1.128 × 0.5 = 0.564 mA

Potencia en cada resistor (R3 = R4 = 68 kΩ):
  P = V_rms² / (2 × R) = 110² / (2 × 68,000) = 0.089 W
  → 8.9 % de la potencia nominal (1 W). Margen excelente.

Potencia total disipada por el módulo:
  P_total = 2 × 0.089 + (V_D2 + V_LED) × I_LED_avg
         = 0.178 + 2.2 × 0.000359
         = 0.179 W ≈ 0.18 W

CTR efectivo @ I_LED_avg = 0.359 mA:
  Según curva de datasheet Sharp, CTR cae a ~0.5× del valor a 5 mA
  CTR ≈ 200 % × 0.5 = 100 %
  → I_colector disponible = 0.359 × 1.0 = 0.359 mA

I_requerida por R5 para saturar GPIO5 a LOW:
  I_R5 = (3.3 − 0.2) / 10,000 = 0.310 mA

Margen de saturación: 0.359 / 0.310 = 1.16× ≥ 1.0 ✓
V_GPIO5 (estado "interruptor cerrado") ≈ V_CE(sat) = 0.2 V → LOW sólido ✓
```

### 5.3 Cálculos para 220 V RMS (exportación / algunas zonas industriales)

```
V_pico = 220 × √2 = 311.1 V
I_LED_pico = (311.1 − 2.2) / 136,000 = 2.271 mA
I_LED_promedio = 2.271 × 0.318 = 0.722 mA

Potencia en cada resistor:
  P = 220² / (2 × 68,000) = 0.356 W
  → 35.6 % de la potencia nominal. Margen adecuado.

CTR efectivo @ 0.722 mA ≈ 150 % (curva datasheet)
I_colector disponible = 0.722 × 1.5 = 1.083 mA

Margen de saturación: 1.083 / 0.310 = 3.5× ≥ 1.0 ✓
V_GPIO5 ≈ 0.1 V → LOW sólido con mucho margen ✓
```

### 5.4 Cálculos para 265 V RMS (peor caso high-line, margen hasta surge)

```
V_pico = 265 × √2 = 374.8 V
I_LED_pico = (374.8 − 2.2) / 136,000 = 2.74 mA   ← < 50 mA PC817 max ✓
I_LED_promedio = 2.74 × 0.318 = 0.871 mA

Potencia en cada resistor:
  P = 265² / (2 × 68,000) = 0.517 W
  → 51.7 % de la potencia nominal. Derating térmico válido en carcasa cerrada.

Todos los márgenes satisfactorios. El circuito es seguro hasta 265 V RMS sostenido.
```

### 5.5 Constantes de tiempo relevantes

```
Filtro RC debounce:
  τ = R5 × C_deb = 10 kΩ × 100 nF = 1.0 ms

Tiempo para que GPIO5 suba de 0 V a 2.3 V (threshold HIGH del ESP32-C3):
  t = −τ × ln((3.3 − 2.3) / 3.3) = −τ × ln(0.303) = 1.19 × τ = 1.19 ms

Tiempo para que GPIO5 baje de 3.3 V a 0.5 V (threshold LOW):
  t = −τ × ln(0.5 / 3.3) = 1.89 × τ = 1.89 ms
  (asumiendo fototransistor con I_sat limitada por R5)

Latencia total de detección del cambio de estado del interruptor:
  t_hardware = 1.2–2 ms (filtro RC)
  t_firmware = 50 ms (ESPHome delayed_on_off)
  t_total = ~52 ms → imperceptible para el usuario
```

---

## 6. BOM Completo del Módulo 8 — Códigos LCSC

> ⚠️ **IMPORTANTE — Auditoría de códigos LCSC (verificación directa en lcsc.com, abril 2026):** los códigos LCSC del [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) maestro para este módulo tienen **errores sistemáticos**. Esta sección contiene los códigos **verificados directamente con el catálogo LCSC**. Los códigos del CSV deben corregirse antes del envío a fabricación — ver §6.3 "Auditoría de errores del CSV maestro".

### 6.1 Componentes Primarios (LCSC verificados)

| # | Ref | Componente | Valor / Specs | Encapsulado | **LCSC** | Fabricante | MPN | Qty | Verificado |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **U3** (U_OPTO) | Optoacoplador PC817C | CTR ≥ 200 % grado C, V_CE = 35 V, 5000 Vrms, 950 nm | **SMD-4P** (cruza slot) | **C3025164** | GOODWORK | PC817C | 1 | ✓ lcsc.com — 612k stock |
| 2 | **D2** | Diodo rectificador 1N4007 | V_R = 1000 V, I_F = 1 A, V_F ≤ 1.1 V | SMA (DO-214AC) | **C727081** | TWGMC | 1N4007 | 1 | ✓ lcsc.com |
| 3 | **R3** | Resistencia limitadora AC | 68 kΩ ±1 %, 1 W, TCR ±50 ppm/°C, MFR, V_max 350 V | Axial THT D3.3 × L9.2 mm | **C385419** | TyoHM | RN1WS68KΩFT/BA1 | 1 | ✓ lcsc.com (⚠ verificar stock al pedir) |
| 4 | **R4** | Resistencia limitadora AC | 68 kΩ ±1 %, 1 W, TCR ±50 ppm/°C, MFR, V_max 350 V | Axial THT D3.3 × L9.2 mm | **C385419** | TyoHM | RN1WS68KΩFT/BA1 | 1 | ✓ lcsc.com (⚠ verificar stock al pedir) |
| 5 | **R5** | Pull-up fototransistor | 10 kΩ ±1 %, 1/16 W, TCR ±100 ppm/°C | 0402 SMD | **C25744** | UNI-ROYAL | 0402WGF1002TCE | 1 | ✓ lcsc.com |
| 6 | **C_deb** | Capacitor debounce HW | 100 nF / 50 V, X7R, ±10 % | 0402 SMD | **C131394** | YAGEO | CC0402KRX7R9BB104 | 1 | ✓ lcsc.com |
| 7 | **J_SW** | Terminal block screw 2P | KF128-5.0-2P, 300 V / 10 A | THT pitch 5.0 mm | **C474950** | Cixi Kefa Elec | KF128-5.0-2P | 1 | ✓ lcsc.com |

**Total componentes:** 7 referencias (6 part numbers únicos — R3/R4 comparten MPN).
**Subtotal BOM Módulo 8 (con códigos correctos):** ≈ $0.37 USD por placa (sin ensamblaje, precios LCSC abril 2026).

### 6.2 Componentes Alternativos / Segundas Fuentes

Todos los primarios están vigentes en LCSC a abril 2026. Se listan alternativas verificadas como respaldo ante out-of-stock temporales o para diseños derivados.

#### 6.2.1 Alternativas para U3 (PC817C — optoacoplador 5000 Vrms)

| Opción | LCSC | MPN | Package | Fabricante | Notas |
|---|---|---|---|---|---|
| **Primario (recomendado)** | **C3025164** | PC817C | **SMD-4P** | GOODWORK | ✓ 612k stock abril 2026, $0.039/ud, ideal para SMT assembly JLCPCB |
| Alt. 1 (THT) | **C115500** | PC817C | DIP-4 | Wuxi China Resources Huajing | ⚠ Sin stock abril 2026. Mantener como referencia THT si se prefiere soldadura manual. Huella diferente al primario |
| Alt. 2 | **C181889** | EL817(C) | DIP-4 | Everlight | CTR 200–400 %, verificar stock — paquete THT |
| Alt. 3 | **C181890** | EL817(C)-F | DIP-4 | Everlight | Variante con marking "F", THT |
| Alt. 4 | **C115459** | LTV-817-C-IN | DIP-4 | Lite-On | CTR 200–400 %, THT |
| ❌ NO usar | — | PC817A o PC817B | — | — | CTR insuficiente a 110 V RMS |

> ⚠️ **Atención al dibujar el esquemático:** la huella SMD-4P del primario **C3025164** tiene el mismo pitch entre filas (7.62 mm) que el DIP-4 clásico, pero los pads son superficiales (gull-wing) en vez de pines THT atravesando la placa. Confirma en EasyEDA Pro que la huella importada desde LCSC corresponde a **SMD-4P** y no a DIP-4; si se cambia entre ambas variantes, la huella del PCB debe regenerarse.

#### 6.2.2 Alternativas para D2 (1N4007 — rectificador 1000 V / 1 A, SMA)

| Opción | LCSC | MPN | Fabricante | Diferencia vs primario | Drop-in? |
|---|---|---|---|---|---|
| **Primario** | **C727081** | 1N4007 | TWGMC | ✓ Verificado en lcsc.com, SMA (DO-214AC) 1 kV / 1 A | — |
| Alt. 1 | **C2892673** | 1N4007 | YONGYUTAI | ✓ Verificado, SMA 1 kV / 1 A | **Sí** |
| Alt. 2 | **C2988638** | 1N4007(M7) | UMW (Youtai Semiconductor) | ✓ Verificado, SMA con marking "M7" | **Sí** |
| Alt. 3 | **C126233** | 1N4007-M7 | Shikues | SMA DO-214AC, compatible | **Sí** |
| Alt. 4 | **C559085** | S-FM407 | LRC | 1 A / 1000 V, SMA, equivalente genérico | **Sí** |
| ❌ NO usar | **C727110** | 1N4148W | TWGMC | **V_R = 100 V, SOD-123** — es el código equivocado del CSV maestro, NO usar | **Falla inmediata** |
| ❌ NO usar | — | 1N4001, 1N4002 | — | V_R = 50 V o 100 V — insuficiente | **No** |
| ❌ NO usar | — | 1N4004 | — | V_R = 400 V, sin margen para surges a 220 V | **No recomendado** |

#### 6.2.3 Alternativas para R3 / R4 (68 kΩ, 1 W, MFR axial THT)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C385419** | RN1WS68KΩFT/BA1 | TyoHM | ✓ Verificado en lcsc.com — 68 kΩ ±1 % 1 W MFR axial D3.3×L9.2 mm, TCR ±50 ppm/°C, V_max 350 V. ⚠ Stock fluctuante (al momento de redactar estaba en 0 con notificación activable); confirmar al pedir |
| Alt. 1 | *buscar por MPN* | **MFR1WSJT-52-68K** | YAGEO | 1 W axial D3.3×L9 mm ±5 % TCR ±100 ppm/°C. Familia verificada en LCSC (C176411 = 1K, C176399 = 100K, C176405 = 150K, C176415 = 200K). Buscar el C-number del valor 68K en [lcsc.com](https://www.lcsc.com/search?q=MFR1WSJT-52-68K) |
| Alt. 2 | *buscar por MPN* | **MFR-1W-68KR-J** | UNI-ROYAL | 1 W MFR axial D3.3 mm, tolerancia ±5 %, buscar por MPN en lcsc.com |
| Alt. 3 | *buscar por MPN* | **MF1/68KJT-52** | FH (Guangdong Fenghua) | 1 W MFR equivalente, ±5 % |
| ❌ NO usar | C385422 | RN1WS**7.68K**ΩFT/BA1 | TyoHM | **7.68 kΩ (10× menor)** — familia correcta pero valor incorrecto. Quemaría resistores a 220 V (P = 3.15 W > 1 W nominal) |
| ❌ NO usar | C20502029 | NFR0300006809JAC00 | VISHAY | **68 Ω, 3 W** — valor 1000× menor. Destruiría el LED PC817 con 1.14 A y disiparía ~89 W → incendio |
| ❌ NO usar | C5140126 | SCR0805F68K | VO | **0805 SMD, 125 mW** — valor correcto pero 8× menor potencia y paquete incompatible con la huella PCB |
| ❌ NO usar | **C176393** | MFR-12JT-52-510R | YAGEO | **510 Ω, 167 mW** — código equivocado del CSV maestro. Destrucción instantánea |
| ❌ NO usar | — | cualquier 1/4 W 68 kΩ | — | Se quemaría a 220 V (P/resistor = 0.356 W > 0.25 W nominal) |
| ❌ NO usar | — | carbón composition | — | TCR alto, deriva AC, poco fiable a largo plazo |

> **Consejo de compra:** si al hacer el pedido C385419 sigue sin stock, usa la Alt. 1 (YAGEO `MFR1WSJT-52-68K`, típicamente con stock alto) o cualquier otro 68 kΩ / 1 W / MFR / axial D3.3×L9 mm con stock inmediato — las especificaciones eléctricas son equivalentes. Mantener el MPN en el CSV actualizado.

#### 6.2.4 Alternativas para R5 (10 kΩ, 0402)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C25744** | 0402WGF1002TCE | UNI-ROYAL | ±1 %, 1/16 W |
| Alt. 1 | **C25755** | RC0402JR-0710KL | YAGEO | ±5 %, misma especificación eléctrica |
| Alt. 2 | **C98737** | ERJ-2GEJ103X | Panasonic | ±5 %, 1/16 W, stock masivo |
| Alt. 3 | **C17414** | 0402WGF103JTCE | UNI-ROYAL | ±5 %, mismo fabricante |

#### 6.2.5 Alternativas para C_deb (100 nF, 0402, X7R)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C131394** | CC0402KRX7R9BB104 | YAGEO | ✓ Verificado en lcsc.com — 100 nF 50 V X7R 0402 ±10 % |
| Alt. 1 | **C60474** | CC0402KRX7R7BB104 | YAGEO | ✓ Verificado, variante de proceso (7BB104), 0402 X7R |
| Alt. 2 | **C83056** | 0402B104K160CT | Walsin Tech | X7R 16 V 100 nF 0402 ±10 % |
| Alt. 3 | **C387940** | 0402B104J160CT | Walsin Tech | X7R 16 V 100 nF 0402 ±5 % (tolerancia mejor) |
| Alt. 4 | **C1850488** | VJ0402Y104KXJCW1BC | Vishay Intertech | X7R 16 V 100 nF 0402 ±10 % |
| ❌ NO usar | **C14663** | CC0603KRX7R9BB104 | YAGEO | **Es 0603, NO 0402** — es el código equivocado del CSV maestro. Footprint del PCB no coincide |
| ❌ NO usar | — | Y5V 100 nF | — | Deriva −80 % a 85 °C → pierde debounce a temperatura de operación |

#### 6.2.6 Alternativas para J_SW (terminal block screw 2P, pitch 5.0 mm)

| Opción | LCSC | MPN | Fabricante | Notas |
|---|---|---|---|---|
| **Primario** | **C474950** | KF128-5.0-2P | Cixi Kefa Elec | 300 V / 10 A, reutilizado patrón con J_LOAD/J_AC del BOM (diferente pitch) |
| Alt. 1 | **C474954** | KF128-7.5-2P | Cixi Kefa Elec | **Pitch 7.5 mm** — requiere ajustar huella, pero mismo fabricante |
| Alt. 2 | **C2909370** | KF301-5.0-2P | Cixi Kefa Elec | Variante con palanca de ajuste (mismo pitch 5.0 mm) |
| Alt. 3 | **C395872** | DB128V-5.0-2P | DEGSON | Equivalente genérico 300 V / 10 A pitch 5.0 mm | 

> **Nota sobre conductor:** el cable del interruptor de pared típicamente es AWG 18 (1 mm²) en instalaciones colombianas. Tanto el KF128-5.0-2P como alternativas acomodan 12–24 AWG según datasheet.

### 6.3 Auditoría de errores del CSV maestro [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv)

La verificación directa en **lcsc.com** (abril 2026) detectó que los códigos LCSC del CSV maestro para los 7 componentes del M8 tienen los siguientes errores. **El CSV debe actualizarse antes de enviar a fabricación.**

| Ref | Código en CSV | Componente real en LCSC | Código CORRECTO | Estado |
|---|---|---|---|---|
| U3 (PC817C) | `C66463` | **No existe** — página 404 en lcsc.com | **C3025164** (GOODWORK PC817C **SMD-4P**, 612k stock abril 2026) — alternativa THT: C115500 Wuxi China Resources Huajing PC817C DIP-4 (actualmente sin stock) | ❌ CSV equivocado |
| D2 (1N4007) | `C727110` | **TWGMC 1N4148W** (100 V / 300 mA, SOD-123) — ¡diodo de señal, no rectificador de red! | **C727081** (TWGMC 1N4007 SMA DO-214AC 1 kV / 1 A) | ❌ CSV equivocado (peligroso — se destruiría en el primer pico de 220 V) |
| R3 / R4 (68 k 1 W) | `C176393` | **YAGEO MFR-12JT-52-510R** (510 Ω, 167 mW) — resistor SMD pequeño | **C385419** (TyoHM RN1WS68KΩFT/BA1, 68 kΩ 1 W ±1 % MFR axial D3.3×L9.2 mm, 350 V, ±50 ppm/°C — verificar stock al pedir) | ❌ CSV equivocado (se quemaría instantáneamente a 220 V) |
| R5 (10 k 0402) | `C25744` | UNI-ROYAL 0402WGF1002TCE, 10 k 0402 | **C25744** (mismo) | ✓ CSV correcto |
| C_deb (100 nF 0402) | `C14663` | **YAGEO CC0603KRX7R9BB104** (paquete **0603**, no 0402) — huella incorrecta | **C131394** (YAGEO CC0402KRX7R9BB104, 0402 50 V X7R) | ❌ CSV equivocado (huella PCB no coincidirá) |
| J_SW (KF128-5.0-2P) | `C474950` | Cixi Kefa KF128-5.0-2P, 2P pitch 5.0 mm | **C474950** (mismo) | ✓ CSV correcto |

**Implicaciones de seguridad críticas:**
- **D2 con C727110 (1N4148W)** en vez del 1N4007: el 1N4148W tiene V_R = 100 V. A 110 V RMS (155 V pico) la primera vez que entre un semiciclo negativo, el diodo se destruye y desde ese momento el LED del PC817 queda expuesto al semiciclo inverso completo → **LED quemado en el primer ciclo**. A 220 V la falla sería aún más rápida.
- **R3/R4 con C176393 (510 Ω, 167 mW)** en vez del 68 k 1 W: la corriente a través del LED del PC817 sería `155 V / (510 + 510) = 152 mA` — **3× el máximo absoluto del PC817** (50 mA). **El optoacoplador se destruye antes del primer semiciclo completo**, y los resistores SMD de 167 mW disiparían `155² / 1020 = 23.5 W` → incendio.

**Acción requerida:** actualizar las filas 4, 10, 15, 16, 35 del CSV con los códigos correctos antes de fabricar. Las filas 17 (R5) y 53 (J_SW) están correctas.

---

## 7. Reglas de Layout PCB

### 7.1 Reglas Críticas de Aislamiento y Creepage

| # | Regla | Motivo | Criterio de verificación |
|---|---|---|---|
| L1 | **Slot fresado de 2 mm** a lo largo de la frontera AC/DC, bajo el cuerpo del PC817 | Elimina el camino de creepage sobre FR-4; el aislamiento queda definido por el PC817 mismo (5000 Vrms) | Gerber visual (mechanical layer) + DRC específica "slot width ≥ 2 mm" |
| L2 | **Creepage AC/DC ≥ 8 mm** medido sobre cualquier camino conductor | IEC 60664-1 Pollution Degree 2 @ 300 V RMS reinforced = 6.4 mm; 8 mm añade margen | Rule check manual con calibrador digital sobre PCB fabricado |
| L3 | **Ninguna via, traza o componente** puede cruzar el slot fresado entre zonas AC y DC (excepto el cuerpo del PC817) | Cualquier cruce crea una ruta conductiva paralela al optoacoplador, anulando el aislamiento | DRC visual: seleccionar slot y verificar que no hay objetos dentro |
| L4 | **Clearance AC/DC ≥ 5 mm** (distancia por aire) medido entre cualquier pad AC y cualquier pad DC | Resiste pulsos de descarga sin arqueo | IEC 60664-1 BIL = 2.5 kV → 5 mm recomendado |
| L5 | Zona AC identificada visualmente en silkscreen con trama de "peligro AC" o label `AC ZONE` | Evita que un técnico toque con probes la zona caliente | Silkscreen review pre-fabricación |
| L6 | **R3 y R4** espaciados al menos **2 mm entre sí** (axial) — no superpuestos ni en contacto térmico | Aislamiento entre resistores con 155 V pico diferencial + dispersión de calor | Placement visual |
| L7 | **D2 (SMA)** orientado con ánodo hacia R4 y cátodo hacia PC817 pin 1 — **verificar la banda del diodo en la huella** | Orientación invertida destruye el LED en el primer semiciclo negativo | Visual check del silkscreen + datasheet SMA (cátodo = banda) |
| L8 | Net `SW_HOT` ancho de traza ≥ **0.5 mm**, clearance a otras trazas AC ≥ **0.8 mm** | Aguanta corriente y arqueo a 311 V pico | IPC-2221 tabla de clearance |
| L9 | **Pads de R3 y R4 ampliados** (teardrop) en las conexiones axiales | Mejora la resistencia mecánica y térmica con componentes THT de 1 W | Rule "teardrops on through-hole pads" en EasyEDA Pro |
| L10 | Pad del PC817 pin 1 (Anode LED) con **relief pad** si va a plano, o conexión directa si es traza | La corriente del LED es solo mA, relief suficiente | Placement standard |
| L11 | **C_deb (100 nF) colocado físicamente junto a GPIO5** del ESP32-C3 (distancia < 5 mm) | Minimiza loop de AC pick-up entre el cable del switch (que puede ser largo) y el pin del SoC | Placement visual |
| L12 | R5 colocado entre el pin 4 del PC817 y el nodo SW_IN, físicamente **cerca del PC817** | Minimiza impedancia parásita del pull-up | Placement visual |

### 7.2 Checklist de Layout

- [ ] Slot fresado de 2 mm presente en el plano mecánico bajo el cuerpo del PC817.
- [ ] Silkscreen identifica claramente la zona AC (label `AC ZONE` o trama).
- [ ] Clearance medido entre pads AC (R3, R4, D2, J_SW) y pads DC (R5, C_deb) ≥ 5 mm.
- [ ] Creepage AC↔DC ≥ 8 mm sobre cualquier camino conductor (medido con herramienta de clearance de EasyEDA Pro).
- [ ] PC817 (U3) cruza el slot correctamente, con pines 1-2 en AC y 3-4 en DC.
- [ ] R3 y R4 montados axialmente con distancia ≥ 2 mm entre cuerpos.
- [ ] D2 con orientación correcta (banda/cátodo hacia PC817 pin 1).
- [ ] Net SW_HOT con ancho ≥ 0.5 mm, sin componentes pequeños compartiendo el pad.
- [ ] C_deb a < 5 mm del pin GPIO5 del Módulo 5 (conexión corta al SoC).
- [ ] R5 a < 5 mm del pin 4 del PC817.
- [ ] Sin trazas de señal DC cruzando bajo R3, R4 o D2 (posible acople capacitivo a AC).
- [ ] Plano GND_DC **no** se extiende bajo R3, R4, D2 ni bajo el cuerpo AC del PC817.
- [ ] Verificación final con DRC de EasyEDA Pro: sin errores de clearance/creepage.

---

## 8. Checklist de Verificación del Módulo

### 8.1 Verificación de Esquemático

- [ ] **PC817C (U3) grado C** seleccionado (no PC817A ni PC817B).
- [ ] **D2 orientación correcta**: ánodo hacia R4, cátodo hacia PC817 pin 1.
- [ ] R3 y R4 **en serie** entre SW_HOT y el ánodo de D2 (no en paralelo).
- [ ] **R5 (10 kΩ)** entre +3.3V y pin 4 del PC817 (Collector).
- [ ] **C_deb (100 nF)** entre pin 4 del PC817 y GND_DC (paralelo al colector-emisor).
- [ ] **Emitter (pin 3)** del PC817 directamente a GND_DC.
- [ ] **Cathode LED (pin 2)** directamente a J_SW pin 2 (AC_N).
- [ ] Net label `SW_IN` conectado al pin 19 (IO5) del ESP32-C3-MINI-1-N4 en el Módulo 5.
- [ ] Net `AC_N` es el mismo net que el neutro del Módulo 1.
- [ ] **No hay conexión eléctrica** entre la zona AC (R3, R4, D2, pines 1-2 del PC817) y la zona DC (pines 3-4 del PC817, R5, C_deb, GPIO5, +3.3V, GND_DC) excepto a través del cuerpo del PC817.
- [ ] ERC sin warnings sobre "No Connect" ni "Unconnected pin" en el PC817.

### 8.2 Verificación de BOM

- [ ] Los **7 LCSC primarios** están en stock en JLCPCB Standard Parts Library.
- [ ] El **PC817X2NSZ0F** es la variante **grado C** (verificar marking en el encapsulado).
- [ ] Huellas importadas correctamente desde LCSC en EasyEDA Pro (sin warnings de "footprint mismatch" en ERC).
- [ ] El encapsulado del 1N4007W coincide con la huella SMA (DO-214AC) en el PCB.
- [ ] Los resistores MFR 1 W axiales tienen el pitch correcto (10 mm) para las huellas del PCB.

### 8.3 Pruebas Post-Ensamblaje (en laboratorio)

| # | Prueba | Criterio de éxito |
|---|---|---|
| T1 | Continuidad R3 a R4 (en serie) | 136 kΩ ±10 % con multímetro |
| T2 | Continuidad R5 de +3.3V a pin 4 PC817 | 10 kΩ ±5 % |
| T3 | Orientación D2 | Banda visible orientada hacia el PC817 (lado del slot) |
| T4 | Aislamiento AC/DC con megger @ 500 VDC (sin PC817 instalado previamente) | R > 1 GΩ entre zona AC y GND_DC |
| T5 | Energizar placa con 110 V RMS, interruptor **abierto** | V(GPIO5) = 3.3 V ±0.1 V (HIGH sólido) |
| T6 | Energizar placa con 110 V RMS, interruptor **cerrado** | V(GPIO5) < 0.3 V (LOW sólido, con ripple < 0.3 V pp) |
| T7 | Repetir T5 y T6 con 220 V RMS (si aplica) | Mismos criterios |
| T8 | Medir con osciloscopio en GPIO5 al cerrar interruptor | Transición de 3.3 V a 0 V en < 2 ms |
| T9 | Medir con osciloscopio en GPIO5 al abrir interruptor | Transición de 0 V a 3.3 V en < 2 ms |
| T10 | Test de aislamiento dieléctrico (hipot) 3000 VAC entre primario AC y secundario DC por 60 s | Sin arqueo, corriente de fuga < 1 mA |
| T11 | Firmware ESPHome: binary_sensor GPIO5 con `inverted: true` | Home Assistant muestra "ON" con interruptor cerrado |
| T12 | Test de debounce: 100 pulsaciones rápidas del interruptor | Sin dobles triggers en el log de ESPHome (`delayed_on_off: 50ms`) |
| T13 | Operación sostenida 24h a 220 V | Sin derivas, sin falsos disparos, temperatura R3/R4 < 90 °C |

---

## 9. Conexión con Otros Módulos

Tabla resumen de las señales que cruzan la frontera del Módulo 8. En el esquemático jerárquico de EasyEDA Pro, estos son los *hierarchical sheet pins* del bloque M8.

| Señal / Net | Dirección desde M8 | Destino | Componente en el camino | Zona |
|---|---|---|---|---|
| `SW_HOT` | Entrada externa | **Interruptor de pared** (instalación del usuario) | J_SW pin 1 | AC (primario) |
| `AC_N` | Entrada externa + común | **Módulo 1** (neutro de la red) + PC817 pin 2 | J_SW pin 2 | AC (primario) |
| `SW_IN` | Salida | **Módulo 5** (ESP32-C3-MINI-1-N4, pin 19 / IO5 / GPIO5) | R5 pull-up + C_deb filter | DC (secundario) |
| `+3.3V` | Entrada | **Módulo 4** (ME6211 VOUT) | — | DC |
| `GND_DC` | Plano común | M2, M4, M5, M6, M7, M9 | Plano continuo | DC |

**Frontera de aislamiento:** el Módulo 8 es **uno de los dos únicos módulos que cruzan la barrera AC/DC** (el otro es el Módulo 2, HLK-5M05). Lo hace mediante el PC817C (5000 Vrms reinforced insulation) y el slot fresado de 2 mm que lo acompaña en el PCB.

**Coherencia con Módulo 5:** el net `SW_IN` de este documento corresponde al mismo net definido en [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) líneas 32, 234, 297, 391, 420, 603. Allí se confirma:
- Pin físico: **19** del módulo ESP32-C3-MINI-1-N4.
- Función: **IO5 / GPIO5** configurable como `INPUT` con pull-up interno (aunque usamos el externo R5 por fiabilidad y para definir la constante de tiempo del debounce).
- No es strapping pin, no interfiere con boot mode, no conflicta con USB Serial/JTAG.

**Configuración ESPHome recomendada:**

```yaml
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO5
      mode:
        input: true
        pullup: false   # usamos pull-up externo R5 10kΩ
      inverted: true    # hardware es activo-bajo (LED PC817 → colector LOW)
    name: "Interruptor de Pared"
    filters:
      - delayed_on_off: 50ms   # debounce firmware complementario al RC hardware
    on_press:
      - switch.toggle: relay_principal
```

---

## 10. Referencias

- **PC817 Series Datasheet** — Sharp Corporation, Rev Nov 2006. Secciones "Electrical Characteristics" (CTR grado C), "Absolute Maximum Ratings" (V_R LED = 6 V), "Isolation" (5000 Vrms).
- **1N4007 Datasheet** — YUNSUNENERGY (MPN `1N4007W`), Rev 2021. V_R = 1000 V, I_F = 1 A, encapsulado SMA.
- **IEC 60664-1:2020** — Insulation coordination for equipment within low-voltage supply systems (creepage y clearance para reinforced insulation).
- **IEC 60747-5-5:2020** — Semiconductor devices — Discrete devices — Optoelectronic isolators (definición de aislamiento óptico 5000 Vrms).
- **UL 1577** — Optical Isolators (prueba estadounidense equivalente IEC 60747-5-5).
- **CEN EN 55032:2015** — Electromagnetic compatibility of multimedia equipment - Emission requirements (Clase B residencial).
- **ESP32-C3 Technical Reference Manual v1.1** — Espressif Systems, Capítulo "GPIO and IO_MUX" (umbral VIL/VIH, pull-up interno opcional).
- **YAGEO MFR-Series Datasheet** — Metal Film Resistors 1 W axial, AEC-Q200 qualified, TCR ±100 ppm/°C.
- [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) — documento maestro del proyecto, sección "Módulo 8: Detección de interruptor físico (PC817C)" (líneas 713–817).
- [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) — núcleo ESP32-C3; destino del net `SW_IN` en GPIO5 (pin 19).
- [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) — fuente del rail `+3.3V` que alimenta R5.
- [Modulo_1_Entrada_AC_Protecciones.md](Modulo_1_Entrada_AC_Protecciones.md) — origen del net `AC_N` (neutro compartido).
- [Modulo_2_Fuente_AC_DC_HLK5M05.md](Modulo_2_Fuente_AC_DC_HLK5M05.md) — el otro módulo que cruza la barrera AC/DC (complementa el aislamiento).
- [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) — lista maestra de materiales; filas con `Module=M8`: 4 (U3 PC817C), 10 (D2 1N4007), 15 (R3), 16 (R4), 17 (R5), 35 (C_deb), 53 (J_SW).

### Aclaración respecto al documento maestro

La sección del Módulo 8 en [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md) (líneas 713–817) contiene un error tipográfico mezclando referencias a `GPIO5` y `GPIO6`. La fuente canónica es el **BOM** (descripción de R5: "Pull-up phototransistor GPIO5") y el **Módulo 5** (líneas 32, 234, 391 — todas confirman GPIO5 / pin 19 / net `SW_IN`). Este documento del Módulo 8 usa **GPIO5** consistentemente. GPIO6 está reservado para el **Módulo 9** (LED de estado).

---

**Fin del documento — Módulo 8: Detección de Interruptor Físico con PC817C v1.0**
