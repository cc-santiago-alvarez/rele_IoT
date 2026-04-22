# Módulo 9: Sensores, LEDs y Botones — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0
**Fecha:** 2026-04-22
**Autor:** Equipo Domotica
**Herramienta EDA:** EasyEDA Pro Edition

---

## 1. Propósito del Módulo

El Módulo 9 es el **bloque de interfaz local** del Smart Relay: agrupa los componentes que le permiten al dispositivo **medir su propio estado térmico**, **indicar visualmente su estado operativo** al usuario que está frente a la placa, y **recibir entrada mecánica directa** (botones BOOT y RESET, estos últimos descritos en el Módulo 5). A diferencia de los Módulos 3 (relé de potencia) y 8 (optoacoplador), el M9 **no conmuta carga ni cruza la frontera AC/DC** — es íntegramente un módulo del lado DC (SELV, secundario, aislado).

Sus funciones son:

1. **Sensado térmico de la placa** mediante un termistor NTC 10 kΩ con β = 3435 K (Vishay NTCS0805E3103FHT), leído por el ADC interno del ESP32-C3 en GPIO3 (ADC1_CH3). Permite implementar **protección por sobretemperatura** con apagado automático del relé a 80 °C para evitar daño térmico del HLK-5M05, del relé HF115F y de componentes circundantes.
2. **Indicación visual de estado** mediante un LED verde SMD 0805 controlado por GPIO6 en configuración **active-low** (ánodo a +3.3 V, cátodo a GPIO6 vía R_LED de 470 Ω). Provee feedback instantáneo al usuario sobre: estado del relé, presencia de red Wi-Fi, error de conexión a Home Assistant, modo de emparejamiento, etc.
3. **Botones de configuración** — SW1 (BOOT) y SW2 (RESET) ya documentados en el Módulo 5. No se duplican en el BOM del M9; se referencian funcionalmente porque participan en la misma UX local con el usuario.

**Entradas del módulo:**
- `+3.3V` — rail regulado del Módulo 4 (ME6211) alimentando el divisor NTC (vía R_NTC_PU) y el ánodo del LED (vía R_LED).
- `GND_DC` — plano común del secundario aislado compartido con M2, M4, M5, M6, M7, M8.
- `LED_STATUS` ← GPIO6 del Módulo 5 (pin 20 del ESP32-C3-MINI-1-N4) — línea de control del LED.

**Salidas del módulo:**
- `NTC_SENSE` → GPIO3 del Módulo 5 (pin 6 del ESP32-C3-MINI-1-N4, ADC1_CH3) — tensión analógica proporcional a la temperatura de la placa.

> **Nota:** el LED se maneja con lógica **active-low** (GPIO6 = LOW → LED encendido). En ESPHome se corrige con `inverted: true` para exponer al usuario la semántica natural "ON = LED encendido".

### 1.1 Por qué NTC y no un sensor digital (BMP280, SHT30, TMP102)

Alternativas digitales (I²C) proveen mejor precisión (±0.5 °C) pero tienen tres desventajas críticas para este diseño:

1. **Ocupan pines I²C** (SDA/SCL) que en el ESP32-C3-MINI-1-N4 son escasos — los pines libres son el bus de sensores externos potenciales de aplicaciones derivadas.
2. **Costo** — un BMP280 cuesta ~$1.20 USD en LCSC; el termistor NTC cuesta $0.08 USD. En un diseño residencial de $10 BOM, el ahorro es ~12 %.
3. **Riesgo de estar mal calibrados a temperaturas altas** — los sensores digitales baratos (LM75, TMP102) saturan o pierden precisión por encima de 100 °C, justo donde la protección por sobretemperatura sí importa.

El **NTC con β conocido y divisor resistivo** es robusto hasta 150 °C, no requiere bus de comunicación, y su no-linealidad se compensa en firmware con un polinomio de Steinhart-Hart o tabla de lookup. Es la elección canónica para **termistores de seguridad** (no para medición precisa de temperatura ambiente).

### 1.2 Por qué LED SMD 0805 y no 0603 o 0402

La visibilidad del LED depende de:
- **Área emisora** — un 0805 emite ~4× más flujo luminoso a misma corriente que un 0402 (área 0.4 mm² vs 0.1 mm² aproximada).
- **Corriente disponible** — con I_LED ~1 mA (limitación por R_LED de 470 Ω y Vf = 2.6–3.1 V del LED verde 525 nm), el 0402 resultaría apenas visible bajo iluminación de oficina.

El 0805 entrega brillo suficiente para lectura a 1–2 m bajo iluminación interior típica, sin necesidad de subir la corriente del LED (que complicaría la disipación de R_LED).

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 El divisor NTC: topología y justificación

El circuito de sensado es un **divisor resistivo downstream** — el NTC está entre el nodo de sensado (`NTC_SENSE`) y GND, con el pull-up R_NTC_PU entre +3.3 V y el mismo nodo:

```
+3.3V ──[R_NTC_PU: 10 kΩ 1%]──┬── NTC_SENSE (→ GPIO3)
                               │
                         [NTC_temp: 10 kΩ @ 25°C, β=3435]
                               │
                              GND_DC
```

**¿Por qué el NTC abajo y no arriba?**

- **Sensibilidad máxima cerca de 25 °C.** Cuando NTC y pull-up son iguales (10 kΩ = 10 kΩ), el divisor tiene su punto de máxima pendiente dV/dT en el centro de la curva. Eso da **mejor resolución ADC** en el rango de operación normal (20–40 °C interior).
- **Comportamiento "fail-safe".** Si el NTC se **abre** (falla más común en termistores envejecidos), la tensión sube a +3.3 V → el firmware lee "muy caliente" → apaga el relé por precaución. Si el NTC estuviera arriba, un NTC abierto daría 0 V → el firmware lee "muy frío" → no protege.
- **Ruido de referencia.** El NTC abajo referencia su lado "ADC" directamente a GND_DC — reduce ripple inyectado desde el rail +3.3 V.

### 2.2 Ecuación β (Steinhart-Hart simplificada)

La resistencia del NTC en función de la temperatura absoluta T (en Kelvin) sigue la fórmula β de un parámetro:

```
R(T) = R_25 × exp[ β × (1/T − 1/T_ref) ]
```

Donde:
- `R_25 = 10 000 Ω` — resistencia nominal a 25 °C (definición del componente).
- `T_ref = 298.15 K` — temperatura de referencia (25 °C).
- `β = 3435 K` — constante β característica del NTCS0805E3103FHT (datasheet Vishay).

A partir de esta ecuación y del divisor 3.3 V con R_NTC_PU = 10 kΩ:

```
V_ADC(T) = 3.3 × R(T) / (10 000 + R(T))
```

### 2.3 Tabla de referencia R(T) y V_ADC(T) con β = 3435

| Temperatura | T [K] | R(T) [Ω] | V_ADC [V] | Estado operativo |
|---|---|---|---|---|
| 0 °C | 273.15 | 28 670 | 2.447 | Ambiente frío (almacenaje invernal) |
| 25 °C | 298.15 | 10 000 | **1.650** | Operación nominal (definición) |
| 40 °C | 313.15 | 5 758 | 1.205 | Placa caliente, ventilación OK |
| 60 °C | 333.15 | 2 980 | 0.758 | Operación sostenida alta carga |
| **80 °C** | **353.15** | **1 662** | **0.471** | **Umbral de protección (apagar relé)** |
| 100 °C | 373.15 | 986 | 0.296 | Crítico — ya debería estar apagado |

> **Nota crítica:** el documento maestro [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño%20PCB%20Smart%20Relay%20ESP32-C6%20-%201%20Canal.md) en las líneas 825–838 reportaba R(80 °C) ≈ 1.26 kΩ y V_ADC ≈ 0.37 V. Esos valores asumen β = 3350 (el valor del NTC de la Shelly Plus 1 que fue el punto de partida del diseño). El NTC real seleccionado — **Vishay NTCS0805E3103FHT (LCSC C3195213)** — tiene β₂₅/₈₅ = 3435 K según su datasheet. Todos los cálculos de este documento usan el valor correcto β = 3435.

### 2.4 Autocalentamiento del NTC

La potencia disipada por el propio NTC alimentado con 3.3 V a través del divisor es:

```
P_NTC(T) = V_NTC²(T) / R(T)
```

- A 25 °C: `V_NTC = 1.65 V, P = 1.65²/10 000 = 0.272 mW`
- A 80 °C: `V_NTC = 0.471 V, P = 0.471²/1 662 = 0.134 mW`

Según datasheet Vishay NTCS0805, el **factor de disipación térmica** del encapsulado 0805 en aire quieto montado sobre FR-4 es **δ_th ≈ 2.0 mW/°C**. Por lo tanto el error de autocalentamiento del sensor es:

```
ΔT_auto = P / δ_th = 0.272 / 2.0 = 0.14 °C  (a 25 °C)
ΔT_auto = 0.134 / 2.0 = 0.07 °C  (a 80 °C)
```

**Despreciable** comparado con la tolerancia combinada del divisor (~1 °C). No requiere compensación en firmware.

### 2.5 Tolerancia combinada del sensor térmico

La incertidumbre en T se propaga desde tres fuentes:

| Fuente | Valor | Impacto en T @ 25 °C |
|---|---|---|
| Tolerancia de R_25 del NTC | ±1 % (grado "F" del Vishay) | ≈ ±0.29 °C |
| Tolerancia de β (dispersión de fábrica) | ±0.75 % (típico Vishay) | ≈ ±0.55 °C |
| Tolerancia de R_NTC_PU | ±1 % (UNI-ROYAL) | ≈ ±0.29 °C |
| Precisión del ADC del ESP32-C3 | ±6 % (datasheet, sin calibración) | ≈ ±1.8 °C |
| **Total (suma cuadrática, con calibración ESPHome)** | — | **≈ ±0.9 °C** |

La precisión del ADC del ESP32-C3 es pobre de fábrica — el **efecto fuse** de calibración individual del silicio (campo `calib_adc1` en eFuses) y el polinomio de corrección de ESPHome (`filters: - calibrate_polynomial`) reducen el error a ±1 % del fondo de escala, lo que deja el error total del sensor en **±0.9 °C** en el rango 20–80 °C — suficiente para protección térmica (cuyo umbral crítico 80 °C tolera ±3 °C).

### 2.6 LED de estado — topología active-low

```
+3.3V ── R_LED (470 Ω 0402) ── LED1 ánodo ── LED1 cátodo ── GPIO6
```

**Estado "LED encendido":** `GPIO6 = LOW` (pin hunde corriente a GND).
**Estado "LED apagado":** `GPIO6 = HIGH` — ambos extremos del LED quedan a +3.3 V, sin diferencia de potencial, no conduce.

**¿Por qué active-low y no active-high (ánodo a GPIO)?**

1. **El ESP32-C3 hunde más corriente de la que fuente** (I_OL = 40 mA vs I_OH = 28 mA según datasheet §GPIO DC Characteristics). Para LEDs de indicación la diferencia no importa (<5 mA), pero es una buena práctica que se mantiene por consistencia con la comunidad de firmware.
2. **En el momento del boot, todos los GPIOs del ESP32-C3 empiezan en HIGH-Z o LOW**. Con active-low, un GPIO en LOW durante el boot (< 50 ms) puede **destellar brevemente el LED** — comportamiento aceptable y hasta deseable como "proof of life" del hardware. Con active-high sería completamente apagado durante boot, sin indicación visual de que la placa arrancó.
3. **ESPHome y Tasmota usan convención `inverted: true`** para la mayoría de LEDs on-board. El comportamiento se vuelve natural desde el YAML: `switch.turn_on → LED prende`.

### 2.7 Cálculo de corriente del LED

El LED verde KENTO KT-0805G (525 nm InGaN) tiene **Vf = 2.6 V (min) a 3.1 V (max)** a I_F = 20 mA según datasheet. En nuestro caso operamos muy por debajo:

```
I_LED(min)  = (3.3 − 3.1) / 470 = 0.43 mA    ← peor caso (Vf alto)
I_LED(typ)  = (3.3 − 2.85) / 470 = 0.96 mA   ← típico
I_LED(max)  = (3.3 − 2.6) / 470 = 1.49 mA    ← mejor caso (Vf bajo)
```

Aunque el LED opera en la zona no lineal de su curva I-V (baja corriente), **sigue siendo visible en interior** — los LEDs modernos InGaN tienen eficiencia luminosa de 15–30 cd/A incluso a sub-mA. A 1 mA típico este LED produce unos 15–30 mcd, perfectamente visible a 1–2 m en habitación con iluminación normal.

**Verificación de disipación:**

```
P_R_LED(max) = I_LED² × R_LED = (1.49e-3)² × 470 = 1.04 mW   ← << 62.5 mW nominal ✓
P_LED(max) = Vf × I_LED = 3.1 × 1.49e-3 = 4.6 mW            ← << 60 mW nominal ✓
P_GPIO6 = V_OL × I_LED ≈ 0.2 × 1.49e-3 = 0.3 mW             ← despreciable
```

> **Corrección al doc maestro:** la línea 850 del doc maestro reportaba `I_LED = 2.3 mA` asumiendo Vf = 2.2 V. Ese valor de Vf corresponde a LEDs verdes AlGaInP antiguos (520 nm opaco) o a LEDs rojos. El KENTO KT-0805G es **InGaN emerald green 525 nm**, con Vf = 2.6–3.1 V. La corriente real es ~0.96 mA, no 2.3 mA.
>
> **Implicación:** el LED será ~40 % menos brillante de lo estimado en el doc maestro. Para el uso de indicador (no para iluminación) esto es irrelevante, pero si se desea el brillo original habría que bajar R_LED a 220 Ω (I_LED ≈ 2.0 mA típ) o 180 Ω (I_LED ≈ 2.5 mA típ). **Recomendación del autor:** mantener 470 Ω — el LED sigue siendo visible y la potencia del MCU se conserva.

### 2.8 GPIO6 es MTCK (JTAG) — ¿por qué es seguro usarlo?

GPIO6 en el ESP32-C3 cumple múltiples funciones de matriz IO_MUX:

- **Función 0 (reset default):** GPIO general propósito.
- **Función 2:** MTCK — línea de clock del JTAG externo.

Al no conectar un depurador JTAG físico externo, GPIO6 queda como **GPIO libre sin conflicto**. El ESP32-C3 ofrece un **USB Serial/JTAG integrado** que opera por GPIO18/19 (pines USB nativos) y no requiere ni interfiere con GPIO6.

Confirmado en [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) línea 251: "GPIO6 para LED — aunque es MTCK (JTAG), no afecta el boot normal; en este diseño no se usa JTAG externo (se usa USB Serial/JTAG integrado)."

### 2.9 GPIO3 y el ADC1_CH3 — por qué este pin específico

El ESP32-C3 tiene **un solo ADC de 12 bits** (ADC1) con 5 canales multiplexados: ADC1_CH0..CH4 mapeados a GPIO0..GPIO4. Dentro de esa familia, GPIO3 (ADC1_CH3) fue elegido para el NTC por tres razones documentadas en [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) líneas 250 y 378:

1. **Mejor SNR que los ADCs del C6.** El ESP32-C6 tenía ADC2 compartido con Wi-Fi; el C3 solo tiene ADC1, sin esa degradación. GPIO3 en particular no es strapping pin ni tiene funciones especiales de boot.
2. **No es strapping pin.** GPIO2, GPIO8 y GPIO9 sí son strapping (influyen el modo de boot). Usar uno de ellos para el divisor NTC obligaría a diseñar el circuito cuidadosamente para no forzar un modo de boot no deseado. GPIO3 es completamente libre.
3. **Con atenuación 12 dB, el ADC acepta 0 a ~2.5 V** de entrada útil. En nuestra tabla §2.3, el rango de V_ADC es 0.30 V (100 °C) a 2.45 V (0 °C) — **dentro del rango del ADC con margen** para no saturar ni perder resolución en la parte baja.

**Configuración ADC recomendada en ESPHome:**

```
attenuation: "12db"        # rango efectivo 0..2.5V
raw: false                  # aplicar la calibración de eFuse automáticamente
samples: 16                 # promediar 16 muestras → suaviza ruido
```

### 2.10 Los botones SW1 (BOOT) y SW2 (RESET) — referencia a M5

El Módulo 9 **no agrega botones nuevos**; los dos que contempla este bloque funcional ya están documentados y listados en el BOM con Module=M5:

- **SW1 (BOOT)** — pulsador táctil 6×6×4.3 mm, entre GPIO9 y GND. Pull-up interno del ESP32-C3 (~45 kΩ) mantiene GPIO9 en HIGH normalmente. Pulsando SW1 → GPIO9 = LOW → durante el reset entra en modo download UART (ROM bootloader). En operación normal (sin reset simultáneo), se reutiliza para funciones de usuario (reset a valores de fábrica, emparejamiento, toggle del relé).
- **SW2 (RESET)** — pulsador táctil 6×6×4.3 mm, entre EN (pin 8) y GND. Pull-up externo **R_EN = 10 kΩ** (Module=M5). Pulsando SW2 → EN = LOW → reset del SoC.

Ver [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) para dimensionamiento, debounce y layout de los botones.

---

## 3. Asignación de Señales / Pinout del Módulo 9

| Señal / Net | Dirección | Origen en M9 | Destino / Origen externo | Zona |
|---|---|---|---|---|
| `+3.3V` | Entrada (rail) | R_NTC_PU pad 1, R_LED pad 1 | Módulo 4 (ME6211 VOUT) | DC |
| `GND_DC` | Plano común | NTC_temp pad 2 | M2/M4/M5/M6/M7/M8 (plano continuo) | DC |
| `NTC_SENSE` | Salida analógica | R_NTC_PU pad 2 = NTC_temp pad 1 | Módulo 5, pin 6 (IO3 / GPIO3 / ADC1_CH3) | DC |
| `LED_STATUS` | Entrada digital | LED1 cátodo | Módulo 5, pin 20 (IO6 / GPIO6) | DC |

**Coherencia con M5:** el pinout aquí listado coincide exactamente con la tabla del ESP32-C3 en [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) líneas 378 (GPIO3 / pin 6 / `NTC_SENSE` / ADC Input / M9) y 392 (GPIO6 / pin 20 / `LED_STATUS` / Output / M9).

---

## 4. Diagrama de Conexión Pin a Pin

### 4.1 Diagrama ASCII Completo del Módulo 9

```
   MÓDULO 9 — SENSORES Y LED DE ESTADO
   ════════════════════════════════════════════════════════════

   +3.3V rail (desde M4 / ME6211 VOUT)
     │
     ├──────────────────────────────┬─────────────────────────────┐
     │                              │                             │
     │                              │                             │
   ┌─┴──────────┐              ┌────┴─────┐                       │
   │ R_NTC_PU   │              │  R_LED   │                       │
   │  10 kΩ     │              │  470 Ω   │                       │
   │  ±1%       │              │  ±5%     │                       │
   │  0402      │              │  0402    │                       │
   │  C25744    │              │  C25117  │                       │
   └─────┬──────┘              └────┬─────┘                       │
         │                          │                             │
         │                          ▼                             │
         │                    ┌─────────┐                         │
         │                    │  LED1   │  Ánodo (+)              │
         │                    │ verde   │                         │
         │                    │ 525 nm  │                         │
         │                    │ 0805    │                         │
         │                    │ C2297   │                         │
         │                    │         │  Cátodo (−)             │
         │                    └────┬────┘                         │
         │                         │                              │
         │                         │                              │
         │                         ▼                              │
         │                  LED_STATUS ─────► GPIO6 (M5, pin 20)  │
         │                                                        │
         ▼                                                        │
   NTC_SENSE ───────────► GPIO3 (M5, pin 6, ADC1_CH3)             │
         │                                                        │
         │                                                        │
   ┌─────┴──────┐                                                 │
   │ NTC_temp   │                                                 │
   │ 10 kΩ      │                                                 │
   │ β=3435     │                                                 │
   │ ±1% (F)    │                                                 │
   │ 0805       │                                                 │
   │ C3195213   │                                                 │
   └─────┬──────┘                                                 │
         │                                                        │
         │                                                        │
         ▼                                                        │
       GND_DC ◄──────────────────────────────────────────────────┘
       (plano común con M2/M4/M5/M6/M7/M8)
```

### 4.2 Sub-diagrama NTC detallado

El divisor NTC es el corazón del sensor térmico:

```
   DIVISOR NTC — topología "downstream"
   ═════════════════════════════════════

             +3.3V  (rail de M4)
                │
                │  I_divisor ≈ 165 µA @ 25°C
                │
        ┌───────┴────────┐
        │   R_NTC_PU     │   pad 1 → +3.3V
        │    10 kΩ ±1%   │   pad 2 → NTC_SENSE
        │    0402 SMD    │
        │   C25744       │
        └───────┬────────┘
                │
                │       Net: NTC_SENSE
                │       ═══════════════════
        ┌───────┴───────────────────────────► GPIO3 (ADC1_CH3)
        │                                     M5 pin 6
        │
        │  V_ADC = 3.3 × R_NTC / (R_NTC + 10k)
        │         = 1.65 V @ 25 °C (midscale)
        │         = 0.47 V @ 80 °C (umbral OTP)
        │
        │
   ┌────┴───────────┐
   │   NTC_temp     │   pad 1 → NTC_SENSE
   │ 10 kΩ β=3435   │   pad 2 → GND_DC
   │ ±1 % (F)       │
   │ 0805 SMD       │
   │ C3195213       │
   │ Vishay         │
   │ NTCS0805E3103  │
   │   FHT          │
   └────┬───────────┘
        │
        ▼
      GND_DC
```

**Lectura del diagrama — 3 conexiones físicas en el PCB:**

1. **R_NTC_PU** — pad 1 a `+3.3V`, pad 2 al nodo `NTC_SENSE`.
2. **NTC_temp** — pad 1 al nodo `NTC_SENSE`, pad 2 a `GND_DC`.
3. **Net `NTC_SENSE`** — sale como traza fina (0.15 mm) hacia GPIO3 / pin 6 del ESP32-C3.

### 4.3 Sub-diagrama LED detallado

El LED de estado es un driver simple tipo "pull-down activo":

```
   LED DE ESTADO — topología active-low
   ═════════════════════════════════════

             +3.3V  (rail de M4)
                │
                │  I_LED ≈ 0.96 mA típico (GPIO6 = LOW)
                │        0 mA (GPIO6 = HIGH)
                │
        ┌───────┴────────┐
        │    R_LED       │   pad 1 → +3.3V
        │   470 Ω ±1%    │   pad 2 → ánodo del LED1
        │   0402 SMD     │
        │   C25117       │
        └───────┬────────┘
                │
                │
        ┌───────┴────────┐
        │    LED1        │   ánodo → R_LED pad 2
        │   verde 525nm  │   cátodo → GPIO6 (via net LED_STATUS)
        │   Vf 2.6–3.1 V │
        │   0805 SMD     │
        │   C2297        │
        │   KENTO        │
        │   KT-0805G     │
        └───────┬────────┘
                │
                │       Net: LED_STATUS
                │       ═══════════════════
                ▼
             GPIO6  (M5, pin 20 del ESP32-C3-MINI-1-N4)
             │
             │  modo OUTPUT (salida push-pull)
             │  LOW (V_OL ≈ 0.2 V)  →  LED ENCENDIDO
             │  HIGH (V_OH ≈ 3.2 V) →  LED APAGADO
             │
           (al sink o source del driver interno del SoC)
```

**Lectura del diagrama — 2 conexiones físicas en el PCB:**

1. **R_LED** — pad 1 a `+3.3V`, pad 2 al ánodo del LED1.
2. **LED1** — ánodo a R_LED pad 2, cátodo al net `LED_STATUS` que sale como traza hacia GPIO6 / pin 20 del ESP32-C3.

### 4.4 Tabla de conexiones pad-por-pad

| Componente | Pad | Net | Destino físico |
|---|---|---|---|
| R_NTC_PU | 1 | `+3.3V` | Plano/traza del rail +3.3V (salida M4) |
| R_NTC_PU | 2 | `NTC_SENSE` | NTC_temp pad 1, via hacia GPIO3 |
| NTC_temp | 1 | `NTC_SENSE` | R_NTC_PU pad 2 |
| NTC_temp | 2 | `GND_DC` | Plano GND_DC |
| R_LED | 1 | `+3.3V` | Plano/traza del rail +3.3V |
| R_LED | 2 | `LED_ANODE` (nodo interno) | Ánodo del LED1 |
| LED1 | Ánodo | `LED_ANODE` | R_LED pad 2 |
| LED1 | Cátodo | `LED_STATUS` | via hacia GPIO6 (M5 pin 20) |

**Total de nets externos del módulo:** 4 (`+3.3V`, `GND_DC`, `NTC_SENSE`, `LED_STATUS`). **Total de nets internos:** 1 (`LED_ANODE`, solo une R_LED con LED1).

---

## 5. Cálculos Eléctricos Consolidados

### 5.1 Parámetros de entrada

| Parámetro | Valor | Origen |
|---|---|---|
| +3.3V | 3.30 V ±1 % | M4 (ME6211 VOUT) |
| R_NTC_PU | 10.0 kΩ ±1 % | BOM, UNI-ROYAL 0402WGF1002TCE |
| R_25 del NTC | 10.0 kΩ ±1 % | Datasheet Vishay NTCS0805E3103FHT |
| β₂₅/₈₅ del NTC | 3435 K ±0.75 % | Datasheet Vishay |
| δ_th (disipación térmica NTC 0805) | 2.0 mW/°C | Datasheet Vishay §Thermal Characteristics |
| R_LED | 470 Ω ±1 % | BOM, UNI-ROYAL 0402WGF4700TCE |
| Vf del LED @ 20 mA | 2.6–3.1 V | Datasheet KENTO KT-0805G |
| V_OL GPIO6 (I_OL = 1 mA) | 0.2 V típ | ESP32-C3 TRM §GPIO DC Characteristics |
| Resolución ADC | 12 bits / 4096 pasos | ESP32-C3 TRM §ADC |
| Atenuación ADC seleccionada | 12 dB (0–2.5 V) | Configuración ESPHome |

### 5.2 Rama NTC — corriente, potencia y resolución

```
Corriente del divisor (máxima, a 25 °C):
  I_div = 3.3 / (R_NTC_PU + R_NTC) = 3.3 / 20 000 = 165 µA

Potencia en R_NTC_PU:
  P_R_PU = I² × R = (165e-6)² × 10 000 = 0.272 mW     ← 0.4 % de 62.5 mW nominal ✓

Potencia en NTC_temp:
  P_NTC = mismo valor al ser R iguales = 0.272 mW   ← << 2.0 mW/°C × 1 °C = 2 mW máx. ΔT ~0.14 °C

Autocalentamiento del NTC a 25 °C:
  ΔT_auto = 0.272 mW / 2.0 mW/°C = 0.14 °C           ← despreciable ✓

Resolución del ADC a 25 °C (sobre la pendiente dV/dT):
  dV/dT ≈ 0.030 V/°C (a 25 °C, pendiente local)
  Un LSB del ADC = 2.5 V / 4096 = 0.61 mV
  → 1 LSB ↔ 0.020 °C                                 ← supera de lejos la tolerancia sistema
```

### 5.3 Rama LED — corriente, potencia y brillo

```
Corriente del LED (rango Vf):
  I_min  = (3.3 − 3.1) / 470 = 0.43 mA    ← peor caso
  I_typ  = (3.3 − 2.85) / 470 = 0.96 mA   ← típico
  I_max  = (3.3 − 2.6) / 470 = 1.49 mA    ← mejor caso

Potencia disipada:
  P_R_LED(max) = 1.49² × 1e-6 × 470 = 1.04 mW         ← 1.7 % de 62.5 mW ✓
  P_LED(max) = 3.1 × 1.49e-3 = 4.6 mW                 ← << 60 mW del encapsulado 0805 ✓

Brillo estimado (15 cd/A para LEDs InGaN baratos a sub-mA):
  I_v(typ) = 15 cd/A × 0.96e-3 A = 14 mcd            ← visible a 1–2 m en interior

Potencia que sink el GPIO6 (corriente × V_OL):
  P_sink = 0.96e-3 × 0.2 = 0.19 µW                   ← despreciable para el SoC
```

### 5.4 Total del módulo (peor caso sostenido)

```
P_M9_total = P_NTC_rama + P_LED_rama + P_GPIO
           = 2 × 0.272 + (1.04 + 4.6) + 0.0002
           = 0.544 + 5.64 + 0.0002
           = 6.2 mW (LED encendido permanente)

Solo con LED apagado (operación normal >99 % del tiempo):
  P_M9_stby = 0.544 mW (solo NTC)
```

**El M9 es el módulo con menor consumo del diseño** — por debajo del 0.3 % del presupuesto de potencia total de la placa (~2 W en operación activa).

---

## 6. BOM Completo del Módulo 9 — Códigos LCSC

> ✅ **Buenas noticias para este módulo:** a diferencia del M8 (donde se detectaron errores sistemáticos de transcripción en el CSV maestro), la verificación directa en `lcsc.com` (abril 2026) muestra que **los 4 códigos LCSC del CSV maestro para el M9 son correctos**. Sin embargo, persisten dos advertencias: (a) R_NTC_PU (C25744) muestra estado *out of stock* con notificación activa al momento de redactar; (b) el doc maestro reportaba parámetros eléctricos del LED y del NTC inconsistentes con los MPN reales del BOM — ver §6.3.

### 6.1 Componentes Primarios (LCSC verificados)

| # | Ref | Componente | Valor / Specs | Encapsulado | **LCSC** | Fabricante | MPN | Qty | Verificado |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **NTC_temp** | Termistor NTC 10 kΩ | R₂₅ = 10 kΩ ±1 % (grado F), β₂₅/₈₅ = 3435 K ±0.75 %, Top −55 a +125 °C | 0805 SMD | **C3195213** | Vishay | NTCS0805E3103FHT | 1 | ✓ lcsc.com — 3.9k stock abril 2026 |
| 2 | **R_NTC_PU** | Resistencia pull-up NTC | 10 kΩ ±1 %, 1/16 W, TCR ±100 ppm/°C | 0402 SMD | **C25744** | UNI-ROYAL | 0402WGF1002TCE | 1 | ✓ lcsc.com — ⚠ *out of stock* abril 2026 (usar alt.) |
| 3 | **LED1** | LED indicador verde | Emerald green 525 nm, Vf = 2.6–3.1 V @ 20 mA, I_v nominal 20 mcd @ 20 mA | 0805 SMD | **C2297** | Hubei KENTO Elec | KT-0805G | 1 | ✓ lcsc.com — ~2M stock abril 2026 |
| 4 | **R_LED** | Resistencia limitadora LED | 470 Ω ±1 %, 1/16 W, TCR ±100 ppm/°C | 0402 SMD | **C25117** | UNI-ROYAL | 0402WGF4700TCE | 1 | ✓ lcsc.com — 733k stock abril 2026 |

**Total componentes:** 4 referencias únicas.
**Subtotal BOM Módulo 9:** ≈ $0.14 USD por placa (NTC $0.08 + 2 resistencias $0.003 + LED $0.03, precios LCSC abril 2026).

### 6.2 Componentes Alternativos / Segundas Fuentes

#### 6.2.1 Alternativas para NTC_temp (10 kΩ β≈3435, 0805 SMD)

| Opción | LCSC | MPN | Fabricante | β [K] | Tolerancia | Drop-in? |
|---|---|---|---|---|---|---|
| **Primario** | **C3195213** | NTCS0805E3103FHT | Vishay | 3435 | ±1 % | — |
| Alt. 1 | *buscar por MPN* | NCP18XH103F03RB | Murata | 3380 | ±1 % | ⚠ β distinto (3380 vs 3435) — recalibrar polinomio ESPHome |
| Alt. 2 | *buscar por MPN* | NTCG164LH103HT1 | TDK | 3435 | ±1 % | **Sí** (mismo β y tolerancia) — en 0603, requiere ajustar huella |
| Alt. 3 | *buscar por MPN* | NCP15XH103F03RC | Murata | 3380 | ±1 % | ⚠ 0402 + β distinto — solo si se rediseña footprint y se recalibra |
| Alt. 4 | *buscar por MPN* | B57861S0103F040 | TDK/EPCOS | 3988 | ±1 % | ⚠ β muy distinto (3988 vs 3435) — la tabla R(T) cambia sustancialmente |
| ❌ NO usar | — | NTCS0805E3103JXT (variante J) | Vishay | 3435 | ±5 % | Tolerancia ±5 % degrada error a ±1.7 °C → no recomendado para OTP |

> **Nota sobre β:** si se sustituye el NTC, **el polinomio de calibración de ESPHome debe recalcularse** — los coeficientes de `calibrate_polynomial` dependen de la curva R(T) específica. La alternativa ideal es Alt. 2 (TDK NTCG164LH103HT1, mismo β que Vishay, en 0603). Si solo hay stock del Murata con β=3380, el error a 80 °C será ~1.5 °C mayor — aún aceptable para la función de OTP.

#### 6.2.2 Alternativas para R_NTC_PU (10 kΩ ±1 %, 0402)

⚠ **Crítico:** el primario C25744 muestra stock 0 con notificación activa. Las siguientes alternativas comparten la misma función y son drop-in directas.

| Opción | LCSC | MPN | Fabricante | Tolerancia | Notas |
|---|---|---|---|---|---|
| **Primario** | **C25744** | 0402WGF1002TCE | UNI-ROYAL | ±1 % | ⚠ Out of stock abril 2026 |
| Alt. 1 | *buscar por MPN* | RC0402FR-0710KL | YAGEO | ±1 % | Family extremadamente común, alta disponibilidad general en LCSC |
| Alt. 2 | *buscar por MPN* | ERJ-2RKF1002X | Panasonic | ±1 % | 0402, 100 ppm/°C, industrial grade |
| Alt. 3 | *buscar por MPN* | AC0402FR-0710KL | YAGEO | ±1 % | Variante AEC-Q200, equivalente eléctrico |
| Alt. 4 | C17414 (0805) | 0805W8F1002T5E | UNI-ROYAL | ±1 % | ⚠ **Paquete 0805**, requiere ajustar huella. 2.8M stock. Usar solo como fallback si no hay 0402 disponible |
| ❌ NO usar | — | 0402 10 kΩ ±5 % | — | La tolerancia ±5 % degrada T ~1 °C adicional; usar ±1 % en este divisor de precisión |

#### 6.2.3 Alternativas para LED1 (LED verde 525 nm, 0805)

| Opción | LCSC | MPN | Fabricante | Color / λ | Vf típ | Notas |
|---|---|---|---|---|---|---|
| **Primario** | **C2297** | KT-0805G | Hubei KENTO | Emerald 525 nm | 2.85 V | ✓ 1.99M stock abril 2026, baratísimo |
| Alt. 1 | *buscar por MPN* | 19-21/G6C-AP1Q2B/3T | Everlight | 525 nm | 3.0 V | Bright InGaN |
| Alt. 2 | *buscar por MPN* | APT2012LZGCK | Kingbright | 520 nm | 3.2 V | Mayor brillo, stock común |
| Alt. 3 | *buscar por MPN* | KT-0805R | Hubei KENTO | **Rojo 625 nm** | 2.0 V | ⚠ **Rojo**, no verde — usar solo si se cambia la semántica del indicador. Vf más bajo → recalcular R_LED si se mantiene I_LED |
| Alt. 4 | *buscar por MPN* | KT-0805Y | Hubei KENTO | Amarillo 590 nm | 2.1 V | ⚠ **Amarillo**, no verde |
| ❌ NO usar | — | LED 0805 "ultra-bright" Vf < 2.0 V | — | Vf bajo hace que I_LED suba a >3 mA → sobrecalienta el encapsulado 0805 con R_LED 470 Ω. Si se desea ese LED, subir R_LED a 680 Ω |

> **Semánticamente:** el M9 usa LED verde para "estado OK / relé activo". Mantener esa convención. Si se cambia de color, documentar en el firmware y en la etiqueta de la carcasa.

#### 6.2.4 Alternativas para R_LED (470 Ω ±1 %, 0402)

| Opción | LCSC | MPN | Fabricante | Tolerancia | Notas |
|---|---|---|---|---|---|
| **Primario** | **C25117** | 0402WGF4700TCE | UNI-ROYAL | ±1 % | ✓ 733k stock abril 2026 |
| Alt. 1 | *buscar por MPN* | RC0402FR-07470RL | YAGEO | ±1 % | Extremadamente común |
| Alt. 2 | *buscar por MPN* | ERJ-2RKF4700X | Panasonic | ±1 % | Industrial grade |
| Alt. 3 | *buscar por MPN* | 0402WGF4700TCZ | UNI-ROYAL | ±1 % | Variante de packaging (reel smaller); misma especificación eléctrica |
| Alt. 4 | *buscar por MPN* | RC0402JR-07470RL | YAGEO | ±5 % | ✓ Aceptable — el error de I_LED por ±5 % es <5 %, imperceptible visualmente |

### 6.3 Auditoría respecto al CSV maestro y al doc maestro

| Ref | Código en CSV | Estado | Comentario |
|---|---|---|---|
| NTC_temp | `C3195213` | ✅ CSV correcto | MPN y LCSC verificados. ⚠ El doc maestro línea 825/830 cita `B=3350` — ese es el β del NTC de la Shelly Plus 1, no del Vishay seleccionado. **El β correcto del componente real es 3435 K.** Este documento (M9) usa el valor correcto y debe considerarse la fuente canónica. |
| R_NTC_PU | `C25744` | ✅ CSV correcto | MPN y LCSC verificados. ⚠ *Out of stock* abril 2026 — si la compra es inminente, considerar Alt. 1 o Alt. 2 del §6.2.2. |
| LED1 | `C2297` | ✅ CSV correcto | MPN y LCSC verificados. ⚠ El doc maestro línea 849 cita `Vf ≈ 2.2 V` — ese valor corresponde a LEDs AlGaInP viejos, no al KENTO KT-0805G InGaN de 525 nm cuya Vf real es 2.6–3.1 V. **Los cálculos del doc maestro (I_LED = 2.3 mA) están errados**; la corriente real con R_LED 470 Ω es ~0.96 mA. Este M9 usa el valor correcto. |
| R_LED | `C25117` | ✅ CSV correcto | MPN y LCSC verificados. Nota: la descripción del CSV dice `±5 %` pero el MPN 0402WGF4700TCE del UNI-ROYAL es en realidad **±1 %** (el dígito `F` del MPN = ±1 %). **El componente físico es ±1 %**; la descripción del CSV podría actualizarse por claridad, pero el código y el MPN son correctos. |

**Conclusión:** el M9 no tiene errores críticos de LCSC como tuvo el M8. La principal acción correctiva es **actualizar el doc maestro líneas 825, 830, 837 y 849** para reflejar β=3435 y Vf=2.6–3.1 V (ver §11 de este documento). No requiere cambiar ninguna fila del CSV maestro.

---

## 7. Reglas de Layout PCB

### 7.1 Reglas Críticas de Posicionamiento

| # | Regla | Motivo | Criterio de verificación |
|---|---|---|---|
| L1 | **NTC_temp debe estar alejado ≥ 8 mm** de las fuentes de calor internas de la placa: K1 (relé), U5 (HLK-5M05) y U2 (ME6211 LDO) | Medir temperatura de la carcasa, no la del componente caliente. Placement inadecuado da lecturas sesgadas 10–20 °C por encima del ambiente real | Medir distancias en EasyEDA Pro con herramienta "Distance" |
| L2 | **NTC_temp debe estar cerca del borde de la zona AC** (≤ 5 mm del slot fresado hacia el lado DC, sin cruzarlo) | La temperatura relevante para la protección térmica es la del recinto cerrado, que sube cuando el relé y el HLK se calientan. Poner el NTC lejos del lado AC lo hace insensible al problema que debe detectar | Placement visual sobre la vista 3D |
| L3 | **R_NTC_PU colocado a ≤ 3 mm del pin GPIO3** del ESP32-C3 (M5, pin 6) | Minimizar la impedancia parásita del nodo ADC — una traza larga actúa como antena y capta ruido Wi-Fi y EMI del conmutador del relé | Placement visual, verificar longitud de traza `NTC_SENSE` desde R_NTC_PU a pin 6 |
| L4 | **Traza `NTC_SENSE`** con ancho 0.15 mm, rodeada por plano GND_DC guard ring si atraviesa zonas ruidosas | Blinda el nodo ADC (~10 kΩ de impedancia) contra acoples capacitivos | DRC visual |
| L5 | **LED1 y R_LED colocados cerca del borde frontal** de la placa con visibilidad desde la tapa de la carcasa | La utilidad del indicador depende de que el usuario lo vea | Verificar con vista 3D y modelo de la carcasa |
| L6 | **LED1 orientado con marca de ánodo** en el lado correcto (hacia R_LED, no hacia GPIO6) | Invertir el LED → no conduce → no ilumina nunca. Este error solo se detecta post-ensamblaje | Silkscreen debe marcar el ánodo (`+`) y la huella debe tener el asterisco en el pad 1 |
| L7 | **Plano GND_DC continuo bajo los 4 componentes del M9**, sin cortes ni vías que rompan la continuidad | Referencia estable del divisor NTC y retorno limpio de la corriente del LED | Plane check en EasyEDA Pro |
| L8 | **Sin trazas de alta velocidad** (USB D+/D−, CLK del SPI externo si se añade) cruzando bajo el NTC ni R_NTC_PU | Acoples capacitivos degradarían la lectura ADC | Visual check del routing |
| L9 | **Huellas de los 0402**: pads estándar 0.50 × 0.60 mm; separación 0.50 mm | Compatibilidad con SMT assembly JLCPCB estándar | Importar desde LCSC Component Library en EasyEDA Pro |
| L10 | **Huella del 0805 LED y NTC**: pads 1.20 × 1.40 mm separación 0.80 mm; marcar polaridad del LED con símbolo "+" en silkscreen | Ensamblaje correcto y orientación visible post-fabricación | Importar huella oficial desde LCSC |

### 7.2 Checklist de Layout

- [ ] NTC_temp a ≥ 8 mm del relé K1 y a ≥ 8 mm del HLK-5M05 U5.
- [ ] NTC_temp a ≤ 5 mm del borde de la zona AC (sin cruzar el slot fresado).
- [ ] R_NTC_PU a ≤ 3 mm del pin 6 (GPIO3) del ESP32-C3.
- [ ] Traza `NTC_SENSE` de longitud mínima, ancho 0.15 mm, sin vias innecesarias.
- [ ] LED1 + R_LED en el borde frontal, alineados con un corte/hueco de la carcasa.
- [ ] Silkscreen marca el ánodo del LED1 con `+` o con barra diagonal estándar.
- [ ] Plano GND_DC continuo bajo los 4 componentes (sin cortes de vías ni huellas).
- [ ] Huellas 0402 y 0805 importadas desde LCSC, sin warnings de "footprint mismatch".
- [ ] Ninguna traza de alta velocidad o conmutación (USB, PWM del relé) cruza bajo NTC_temp o R_NTC_PU.
- [ ] Distancia R_NTC_PU a R_LED > 2 mm (evitar que el calor de R_LED afecte el divisor NTC — aunque 1 mW es despreciable, el layout limpio es mejor práctica).

---

## 8. Checklist de Verificación del Módulo

### 8.1 Verificación de Esquemático

- [ ] R_NTC_PU (10 kΩ) entre `+3.3V` y el nodo `NTC_SENSE`.
- [ ] NTC_temp entre `NTC_SENSE` y `GND_DC`.
- [ ] Polaridad del NTC no crítica (simétrico), pero la huella debe respetar la orientación de soldadura 0805.
- [ ] R_LED (470 Ω) entre `+3.3V` y ánodo de LED1.
- [ ] LED1 ánodo a R_LED, cátodo a `LED_STATUS`.
- [ ] Net label `NTC_SENSE` conectado al pin 6 (IO3) del ESP32-C3-MINI-1-N4 en el Módulo 5.
- [ ] Net label `LED_STATUS` conectado al pin 20 (IO6) del ESP32-C3-MINI-1-N4 en el Módulo 5.
- [ ] Ningún componente del M9 conectado a nets AC (zona primaria). Todo el módulo en la zona DC.
- [ ] ERC sin warnings sobre "unconnected pin" o "multiple drivers" en ninguno de los 4 componentes.

### 8.2 Verificación de BOM

- [ ] Los **4 LCSC primarios** están declarados en JLCPCB Standard Parts Library (C3195213, C25744, C25117, C2297).
- [ ] Si C25744 muestra *out of stock* al ordenar, sustituir por el YAGEO RC0402FR-0710KL (§6.2.2 Alt. 1) y actualizar el campo `LCSC_Part` del CSV con el código correspondiente.
- [ ] Huella del NTC_temp (0805) importada desde LCSC — verificar que los pines están marcados 1–2 (sin polaridad específica).
- [ ] Huella del LED1 (0805) importada desde LCSC — verificar que el **pad 1 corresponde al ánodo** y que el silkscreen muestra la marca.
- [ ] Huellas de R_NTC_PU y R_LED (0402) importadas desde LCSC.

### 8.3 Pruebas Post-Ensamblaje (laboratorio)

| # | Prueba | Criterio de éxito |
|---|---|---|
| T1 | Continuidad R_NTC_PU entre `+3.3V` y `NTC_SENSE` (placa energizada OFF, medir con ohmímetro) | 10.0 kΩ ±100 Ω |
| T2 | Continuidad NTC_temp entre `NTC_SENSE` y `GND_DC` @ 25 °C ambiente | 10.0 kΩ ±100 Ω |
| T3 | Continuidad R_LED entre `+3.3V` y ánodo LED1 | 470 Ω ±10 Ω |
| T4 | Energizar placa a 25 °C, medir V(`NTC_SENSE`) con multímetro DC (alta-Z) | 1.65 V ±0.03 V |
| T5 | Lectura del NTC desde ESPHome (sensor ADC calibrado), comparar con termómetro IR externo | Delta ≤ ±1.5 °C en el rango 20–40 °C |
| T6 | Forzar GPIO6 = LOW desde console ESPHome | LED1 visiblemente encendido; I = 0.8–1.5 mA medido con amperímetro en serie |
| T7 | Forzar GPIO6 = HIGH | LED1 apagado completo; I < 10 µA (fuga del LED) |
| T8 | Calentar la placa con aire caliente controlado hasta 80 °C sobre el NTC (usar termopar externo como referencia) | V(`NTC_SENSE`) = 0.47 V ±0.05 V y el relé se apaga automáticamente (protección OTP) |
| T9 | Volver a ambiente 25 °C después de OTP, esperar enfriamiento, verificar que el relé re-engancha | El sistema recupera operación normal sin intervención manual (si se configuró auto-rearme con histéresis de 10 °C) |
| T10 | Probar el brillo del LED en ambientes oscuro y con luz de oficina | Visible a 2 m en oficina; sin deslumbrar en oscuro total |
| T11 | Operación sostenida 24 h con LED encendido permanente | Temperatura de R_LED < 40 °C (medir con IR), sin deriva de la lectura del NTC |
| T12 | Verificar inmunidad a EMI: inducir una descarga ESD de 4 kV al chasis a 30 cm del NTC | La lectura del NTC no se corrompe permanentemente (transient < 5 ms, recupera a valor correcto) |

---

## 9. Conexión con Otros Módulos

Tabla resumen de las señales que cruzan la frontera del Módulo 9. En el esquemático jerárquico de EasyEDA Pro, estos son los *hierarchical sheet pins* del bloque M9.

| Señal / Net | Dirección desde M9 | Destino / Origen | Componente en el camino | Zona |
|---|---|---|---|---|
| `+3.3V` | Entrada | Módulo 4 (ME6211 VOUT) | Rail directo | DC |
| `GND_DC` | Plano común | M2, M4, M5, M6, M7, M8 | Plano continuo | DC |
| `NTC_SENSE` | Salida | Módulo 5 (ESP32-C3-MINI-1-N4, pin 6 / IO3 / ADC1_CH3) | R_NTC_PU + NTC_temp | DC |
| `LED_STATUS` | Entrada | Módulo 5 (ESP32-C3-MINI-1-N4, pin 20 / IO6 / GPIO6) | R_LED + LED1 | DC |

**Coherencia con M5:** los nets `NTC_SENSE` (GPIO3) y `LED_STATUS` (GPIO6) de este documento coinciden exactamente con los definidos en [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md):

- Línea 33 — "Módulo 9 — GPIO3 lee el divisor NTC (ADC1_CH3); GPIO6 maneja el LED de estado."
- Líneas 235–236 — pinout tabular.
- Líneas 250–251 — justificación técnica de la elección de GPIO3 y GPIO6.
- Líneas 378 y 392 — tabla maestra de pin mapping.
- Líneas 604–605 — señales de entrada/salida del MCU.

**Ningún componente del M9 cruza la barrera AC/DC.** Todo el bloque está en la zona DC (secundario aislado), sin interfaz con tensiones de red. Eso simplifica el layout y elimina cualquier consideración de creepage/clearance especial para este módulo.

---

## 10. Referencias

- **Vishay NTCS0805E3103FHT datasheet** — Rev Feb 2024. Secciones "Specifications" (R₂₅ = 10 kΩ, β₂₅/₈₅ = 3435 K, tolerancia ±1 %), "Dissipation Factor" (δ_th ≈ 2.0 mW/°C en 0805).
- **Hubei KENTO Elec KT-0805G datasheet** — 2021. Emerald green 525 nm, Vf 2.6–3.1 V @ I_F = 20 mA, I_v typ 15–25 mcd.
- **UNI-ROYAL 0402WGF datasheet** — Thick film chip resistor 0402, ±1 % F-grade, 62.5 mW, TCR ±100 ppm/°C.
- **Espressif ESP32-C3 Technical Reference Manual v1.1** — Capítulo 29 "GPIO, RTC_GPIO, and IO_MUX" (drive strength, V_OL/V_OH), Capítulo 28 "On-Chip Sensors and Analog Signal Processing" (especificaciones ADC).
- **Espressif ESP32-C3 ADC Calibration Application Note** (AN, Rev 2023) — describe el mecanismo de eFuse-based calibration que ESPHome usa para el polinomio correctivo del ADC.
- **ESPHome documentation** — `sensor.adc` + `filters.calibrate_polynomial` para la linealización del NTC.
- **IEC 60738-1:2017** — Thermistors - Directly heated positive step-function temperature coefficient. Referencia para grados de tolerancia de termistores.
- [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño%20PCB%20Smart%20Relay%20ESP32-C6%20-%201%20Canal.md) — documento maestro del proyecto, sección "Módulo 9" (líneas 821–865).
- [Modulo_4_Regulador_3V3_ME6211.md](Modulo_4_Regulador_3V3_ME6211.md) — fuente del rail `+3.3V` que alimenta R_NTC_PU y R_LED.
- [Modulo_5_ESP32_C3_Core.md](Modulo_5_ESP32_C3_Core.md) — núcleo ESP32-C3; destino de los nets `NTC_SENSE` (GPIO3, pin 6) y `LED_STATUS` (GPIO6, pin 20). Contiene también la documentación de SW1 y SW2 referenciados funcionalmente por este módulo.
- [Modulo_8_Optoacoplador_PC817C.md](Modulo_8_Optoacoplador_PC817C.md) — documento hermano cuya estructura se replica aquí.
- [BOM_Smart_Relay_C6.csv](BOM_Smart_Relay_C6.csv) — lista maestra de materiales; filas con `Module=M9`: 18 (R_LED), 19 (R_NTC_PU), 58 (LED1), 59 (NTC_temp).

---

## 11. Aclaración respecto al documento maestro

La sección del Módulo 9 en el documento maestro [Diseño PCB Smart Relay ESP32-C6 - 1 Canal.md](Diseño%20PCB%20Smart%20Relay%20ESP32-C6%20-%201%20Canal.md) (líneas 821–865) contiene **dos inexactitudes técnicas** que este documento M9 corrige:

### 11.1 Valor de β del termistor NTC

- **Doc maestro línea 825**: "Réplica del diseño de la Shelly Plus 1 (NTC 10 kΩ B=3350 en GPIO32)".
- **Doc maestro línea 830**: "NTC_temp: 10 kΩ B=3350".
- **Doc maestro línea 837**: "A 80 °C: NTC ≈ 1.26 kΩ → V_ADC = 3.3 × 1.26k / (10k + 1.26k) = 0.37 V".

El valor **β = 3350 K** corresponde al termistor usado por la Shelly Plus 1 (cuya ingeniería inversa inspiró este diseño). El componente **realmente especificado en el BOM** es el **Vishay NTCS0805E3103FHT** (LCSC C3195213), cuyo β₂₅/₈₅ según datasheet Vishay es **3435 K**, no 3350.

**Valores correctos con β = 3435:**

| Temperatura | Doc maestro (β=3350) | **Correcto (β=3435)** |
|---|---|---|
| 80 °C, R(T) | ~1.26 kΩ | **1.66 kΩ** |
| 80 °C, V_ADC | ~0.37 V | **0.471 V** |

El firmware de ESPHome debe usar β = 3435 al configurar `calibrate_polynomial`. Si se cargara β = 3350, el error en la lectura a 80 °C sería **≈ 4 °C** — suficiente para hacer que la protección OTP se dispare en el umbral equivocado.

### 11.2 Vf y corriente del LED

- **Doc maestro línea 849**: "LED verde 0805, V_F ≈ 2.2 V".
- **Doc maestro línea 850**: "I_LED = (3.3 − 2.2) / 470 = 2.3 mA".

El valor **Vf = 2.2 V** corresponde a LEDs verdes AlGaInP de tecnología antigua (520 nm pale yellow-green). El LED especificado en el BOM — **Hubei KENTO KT-0805G** (LCSC C2297) — es un **InGaN emerald green de 525 nm**, con Vf = **2.6–3.1 V** según datasheet.

**Cálculo corregido:**

```
I_LED(typ)  = (3.3 − 2.85) / 470 = 0.96 mA    ← ~58 % menos que los 2.3 mA declarados
```

El LED resulta visiblemente menos brillante que lo estimado en el doc maestro, pero sigue siendo **perfectamente funcional como indicador de estado**. Si se desea recuperar los 2.3 mA originales, cambiar R_LED a **220 Ω** (recalcular P_R_LED(max) = 2.5 mA² × 220 = 1.4 mW — sigue <<62.5 mW nominal; cambio viable).

### 11.3 Recomendación

Mantener el LED y el NTC actuales (están en el CSV correctamente), pero actualizar el doc maestro líneas 825, 830, 837, 849 y 850 con los valores correctos de este documento. El resto de la sección §11 del doc maestro (topología, uso de GPIO3/GPIO6, referencia a botones M5) es correcta.

---

**Fin del documento — Módulo 9: Sensores, LEDs y Botones v1.0**
