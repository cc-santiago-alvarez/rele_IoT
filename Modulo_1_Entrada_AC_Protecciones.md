# Módulo 1: Entrada AC y Protecciones — Documento Técnico

## Proyecto: Smart Relay ESP32-C3/C6 de 1 Canal

**Versión:** 1.0  
**Fecha:** 2026-04-16  
**Autor:** Equipo Domotica  
**Herramienta EDA:** EasyEDA Pro Edition  

---

## 1. Propósito del Módulo

El Módulo 1 es la primera línea de defensa del Smart Relay. Su función es:

1. **Recibir** 110/220 VAC de la red doméstica colombiana (o exportación)
2. **Proteger** contra sobrecorrientes, transientes de red (surges/spikes) y sobretensiones
3. **Limitar** la corriente de inrush al encendido
4. **Filtrar** EMI conducido (emisiones de modo diferencial)
5. **Proteger contra incendio** como última barrera térmica

Las salidas del módulo (`AC_L_OUT` y `AC_N_OUT`) alimentan directamente el **HLK-5M05** (Módulo 2, fuente AC-DC aislada).

---

## 2. Teoría de Operación — Explicación a Fondo

### 2.1 Flujo de la señal AC

La corriente AC recorre el siguiente camino desde la red hasta la fuente HLK-5M05:

```
RED ELÉCTRICA
     │
     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  MÓDULO 1: ENTRADA AC Y PROTECCIONES                                      │
│                                                                            │
│  L ──[J_AC pin1]──[F1]──[TF1]──[NTC1]──┬──[L1 bob.A]── AC_L_OUT ──►M2   │
│                                         │                                  │
│                                      [MOV1]  (L↔N)                        │
│                                      [CX1]   (L↔N)                        │
│                                      [R_disc] (L↔N)                       │
│                                         │                                  │
│  N ──[J_AC pin2]────────────────────────┴──[L1 bob.B]── AC_N_OUT ──►M2   │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
     │
     ▼
  HLK-5M05 (Módulo 2)
  Pin 1 = AC_L_OUT
  Pin 2 = AC_N_OUT
```

### 2.2 Función de cada componente

#### J_AC — Terminal Block de Entrada

Punto de conexión física entre el cableado de la vivienda y la PCB. Terminal de tornillo de 2 posiciones con pitch de 7.5 mm para garantizar creepage (distancia de fuga) mínima de 5.5 mm entre Línea y Neutro, conforme a RETIE/IEC 60950-1 para tensiones hasta 300 VAC.

- **Pin 1:** Línea (L) — cable vivo de la red
- **Pin 2:** Neutro (N) — referencia de la red

#### F1 — Fusible Slow-Blow (Protección contra Sobrecorriente)

Fusible de acción lenta (slow-blow / time-delay) de **1 A a 250 V** en formato 5×20 mm, montado en portafusible BLX-A soldado a la PCB.

**Por qué slow-blow y no fast-blow:**
- La fuente HLK-5M05 genera un pico de corriente de arranque (inrush) de ~10-30A durante los primeros 2-5 ms al cargar sus capacitores internos. Un fusible fast-blow se abriría innecesariamente con estos picos transitorios.
- El fusible slow-blow tolera picos breves (I²t alto) pero se abre ante sobrecorrientes sostenidas.
- Valor de 1 A: el consumo nominal del circuito es ~100-200 mA RMS. 1 A da margen suficiente para transitorios de arranque pero protege contra cortocircuitos.

**Modo de falla del MOV:** Si el varistor MOV1 falla en cortocircuito (su modo de falla esperado por diseño), F1 se sacrifica y desconecta la línea, evitando un incendio. Esta es la razón por la que **F1 siempre debe estar antes del MOV** en la cadena serie.

#### TF1 — Fusible Térmico (Protección contra Incendio)

Dispositivo de corte térmico de una sola acción (no resetteable) que se abre permanentemente cuando su temperatura interna alcanza **115 °C**. Se monta en contacto físico directo con la carcasa del HLK-5M05.

**Principio de operación:**
- Contiene una pastilla de aleación eutéctica o cera termosensible
- Al alcanzar 115 °C, la pastilla se funde y un resorte interno separa los contactos permanentemente
- Es la **última defensa contra incendio**: si el HLK-5M05 se sobrecalienta por una falla interna (ej: capacitor en corto, componente en thermal runaway), TF1 desconecta toda la alimentación AC

**Por qué 115 °C:**
- La temperatura máxima de operación del HLK-5M05 es 70 °C (ambiente) + ~20 °C (autoheating) = ~90 °C carcasa
- 115 °C está 25 °C por encima del peor caso normal → solo se dispara en condición de falla real
- Muy por debajo de la temperatura de ignición de FR4 (~300 °C) → actúa antes de que la PCB se dañe

#### NTC1 — Termistor NTC (Limitador de Inrush)

Resistencia de coeficiente térmico negativo que limita el pico de corriente en el arranque (inrush current).

**Principio de operación:**
1. **En frío (arranque):** NTC1 presenta **5 Ω** de resistencia, limitando la corriente pico:
   - A 110 V: I_inrush = 155 V_pico / 5 Ω = **~31 A** (vs. >100 A sin protección)
   - A 220 V: I_inrush = 311 V_pico / 5 Ω = **~62 A** (vs. >200 A sin protección)
2. **En régimen (caliente):** el NTC se autocalentado por la corriente que circula, su resistencia cae a **< 0.5 Ω**, disipando muy poca potencia
3. **Limitación:** si el equipo se apaga brevemente y se re-enciende antes de que el NTC se enfríe, la protección es reducida (el NTC aún está caliente y tiene baja resistencia)

#### MOV1 — Varistor de Óxido Metálico (Protección contra Transientes)

Dispositivo de protección contra sobretensiones transitorias (surges, spikes, lightning induced surges).

**Principio de operación:**
1. **En operación normal:** MOV1 presenta una impedancia extremadamente alta (> 1 MΩ) — es prácticamente invisible en el circuito
2. **Durante un transiente:** cuando el voltaje entre L y N supera el voltaje de clampeo, la resistencia del MOV cae abruptamente a < 1 Ω, absorbiendo la energía del transiente y limitando el voltaje a un nivel seguro
3. **Después del transiente:** el MOV vuelve a su estado de alta impedancia

**Selección según tensión de red:**

| Parámetro | 110 V (Colombia) | 220 V (Exportación) |
|---|---|---|
| Modelo | **10D241K** | **14D431K** |
| MCOV (Max Continuous Operating Voltage) | 150 VAC | 275 VAC |
| V_varistor @ 1 mA | 240 V | 430 V |
| V_clampeo (8/20 µs) | ~395 V | ~700 V |
| Energía surge (8/20 µs) | ~2.5 kA | ~4.5 kA |
| Diámetro disco | 10 mm | 14 mm |

**Nota importante:** Para el mercado colombiano doméstico (110 VAC), el **10D241K** ofrece protección más agresiva (clamp ~395 V vs ~700 V). El 14D431K es para diseños que necesitan operar en ambas tensiones.

#### CX1 — Capacitor de Seguridad X2 (Filtro EMI Diferencial)

Capacitor de polipropileno metalizado (MKP) clase **X2** de **100 nF a 275 VAC**.

**Principio de operación:**
- Conectado entre L y N, forma un filtro paso-bajo para interferencias electromagnéticas conducidas de modo diferencial
- Frecuencia de corte: junto con la impedancia de la red (~50 Ω) → f_c ≈ 1 / (2π × 50 × 100nF) ≈ **31.8 kHz**
- Atenúa ruido de alta frecuencia generado por la fuente switching HLK-5M05 hacia la red y viceversa

**Por qué clase X2:**
- Los capacitores clase X están diseñados específicamente para conexión L↔N (across-the-line)
- X2 soporta pulsos de voltaje de hasta 2.5 kV (categoría II IEC 60384-14)
- Es **self-healing**: si ocurre una perforación dieléctrica localizada, la metalización alrededor del defecto se evapora, aislando el punto de falla sin cortocircuito permanente
- Falla en **circuito abierto** (modo de falla seguro): si el capacitor falla, simplemente pierde capacitancia sin crear un cortocircuito

#### R_disc — Resistencia de Descarga (Seguridad IEC)

Resistencia de **1 MΩ, 1 W** en paralelo con CX1.

**Principio de operación:**
- Cuando el equipo se desconecta de la red, CX1 puede quedar cargado a tensión peligrosa (hasta V_pico = 155 V @ 110 VAC o 311 V @ 220 VAC)
- R_disc descarga CX1 de forma segura:
  - τ = R × C = 1 MΩ × 100 nF = **100 ms**
  - Tiempo de descarga a < 34 V (seguro al tacto): 5τ = **500 ms < 1 segundo**
- **Requerido por IEC 60950-1 / IEC 62368-1:** el voltaje en los terminales debe caer por debajo de 34 V en menos de 1 segundo después de desconectar

**Disipación en operación:**
- P = V²_rms / R = (110)² / 1M = **12.1 mW** (110 V) o (220)² / 1M = **48.4 mW** (220 V)
- Muy por debajo del rating de 1 W → amplio margen térmico

#### L1 — Choke de Modo Común (Filtro EMI Común)

Inductor con dos bobinados acoplados sobre un núcleo de ferrita **UU9.8**, con **10 mH** de inductancia de modo común.

**Principio de operación:**
1. **Modo diferencial (señal útil AC):** la corriente entra por el pin 1 (dot) del bobinado A y por el pin 4 (dot) del bobinado B. Los flujos magnéticos se **cancelan** → impedancia prácticamente nula → la señal AC de 50/60 Hz pasa sin atenuación
2. **Modo común (ruido EMI):** la corriente de ruido fluye en la misma dirección en ambos bobinados. Los flujos magnéticos se **suman** → impedancia de 10 mH → atenúa fuertemente el ruido

**Pinout UU9.8:**
```
         ┌────────────┐
Pin 1 ●──┤ Bobinado A ├──● Pin 2
  (dot)  │   ══════   │
Pin 4 ●──┤ Bobinado B ├──● Pin 3
  (dot)  └────────────┘

Pin 1 (dot) = Entrada L  (nodo L_in, post-NTC1)
Pin 2       = Salida L   (AC_L_OUT → HLK-5M05 pin 1)
Pin 4 (dot) = Entrada N  (nodo N_in, post-J_AC pin 2)
Pin 3       = Salida N   (AC_N_OUT → HLK-5M05 pin 2)
```

**Nota para prototipo:** L1 es **opcional en el prototipo inicial**. Para omitirlo, puentear pin 1↔pin 2 y pin 3↔pin 4 con jumpers de alambre. El choke de modo común mejora la compatibilidad electromagnética (EMC) pero no es crítico para la función básica del circuito.

---

## 3. Diagrama de Conexión Pin a Pin

```
                         MÓDULO 1 — ENTRADA AC Y PROTECCIONES
                         ════════════════════════════════════

  RED DOMÉSTICA                                                          → MÓDULO 2
  ─────────────                                                            (HLK-5M05)

                    F1 Holder        F1 Fuse
                   ┌─────────┐    ┌─────────┐
  J_AC             │  BLX-A  │    │ 1A 250V │     TF1             NTC1
 ┌──────┐         │ C3131   │    │ C142715 │   ┌──────┐        ┌──────┐
 │ pin1 ├─(L)────►│pin1  pin2├───►│pin1  pin2├──►│pin1 pin2├───►│pin1 pin2├──►─┐
 │  L   │         └─────────┘    └─────────┘   └──────┘        └──────┘    │
 │      │                                                                   │
 │      │                                                            NODO_L │
 │      │                                                         (común)   │
 │      │         MOV1              CX1            R_disc                   │
 │      │        ┌──────┐        ┌──────┐        ┌──────┐                  │
 │      │        │14D431│        │100nF │        │ 1MΩ  │                  │
 │      │     ┌──┤pin1  │     ┌──┤pin1  │     ┌──┤pin1  │       ┌─────────┤
 │      │     │  │pin2  │     │  │pin2  │     │  │pin2  │       │         │
 │      │     │  └──┬───┘     │  └──┬───┘     │  └──┬───┘       │         │
 │      │     │     │         │     │         │     │           │         │
 │      │     └─────┴─────────┴─────┴─────────┴─────┘           │         │
 │      │           │                                            │   L1 (UU9.8)
 │      │           │         NODO_N                             │  ┌───────────┐
 │      │           │        (común)                             │  │           │
 │ pin2 ├─(N)──────►┴───────────────────────────────────────────►├─►│P1(●) → P2├──► AC_L_OUT
 │  N   │                                                        │  │           │     (→ HLK pin1)
 └──────┘                                                        └─►│P4(●) → P3├──► AC_N_OUT
                                                                    │           │     (→ HLK pin2)
                                                                    └───────────┘
```

### 3.1 Tabla de Nets (Conexiones Eléctricas)

| Net Name | Nodos conectados | Descripción |
|---|---|---|
| `AC_LINE_IN` | J_AC pin 1, F1_holder pin 1 | Línea viva de la red |
| `AC_FUSED` | F1_holder pin 2 / F1 pin 2, TF1 pin 1 | Post-fusible |
| `AC_THERMAL` | TF1 pin 2, NTC1 pin 1 | Post-fusible térmico |
| `NODO_L` | NTC1 pin 2, MOV1 pin 1, CX1 pin 1, R_disc pin 1, L1 pin 1 | Nodo común Línea (post-protecciones serie) |
| `NODO_N` | J_AC pin 2, MOV1 pin 2, CX1 pin 2, R_disc pin 2, L1 pin 4 | Nodo común Neutro |
| `AC_L_OUT` | L1 pin 2, HLK-5M05 pin 1 | Línea filtrada hacia fuente |
| `AC_N_OUT` | L1 pin 3, HLK-5M05 pin 2 | Neutro filtrado hacia fuente |

### 3.2 Diagrama de Conexión Pin a Pin Detallado

```
J_AC (KF128-7.5-2P)
  Pin 1 (L) ─────────────────────────────────► F1_holder Pin 1
  Pin 2 (N) ─────────────────────────────────► NODO_N

F1_holder (BLX-A)
  Pin 1 ◄──── J_AC Pin 1 (L)
  Pin 2 ─────────────────────────────────────► TF1 Pin 1
  [Internamente aloja el fusible F1 de 5×20 mm]

TF1 (Fusible Térmico 115 °C)
  Pin 1 ◄──── F1_holder Pin 2
  Pin 2 ─────────────────────────────────────► NTC1 Pin 1

NTC1 (5D-9, 5 Ω)
  Pin 1 ◄──── TF1 Pin 2
  Pin 2 ─────────────────────────────────────► NODO_L

MOV1 (14D431K / 10D241K)
  Pin 1 ◄──── NODO_L
  Pin 2 ◄──── NODO_N

CX1 (100 nF 275 VAC X2)
  Pin 1 ◄──── NODO_L
  Pin 2 ◄──── NODO_N

R_disc (1 MΩ 1 W)
  Pin 1 ◄──── NODO_L  (en paralelo con CX1)
  Pin 2 ◄──── NODO_N  (en paralelo con CX1)

L1 (UU9.8, 10 mH, Choke de Modo Común)
  Pin 1 (●) ◄── NODO_L           [Bobinado A entrada]
  Pin 2     ──► AC_L_OUT          [Bobinado A salida → HLK-5M05 pin 1]
  Pin 4 (●) ◄── NODO_N           [Bobinado B entrada]
  Pin 3     ──► AC_N_OUT          [Bobinado B salida → HLK-5M05 pin 2]
```

---

## 4. Secuencia de Protección ante Eventos

### 4.1 Arranque Normal (Power-On)

```
t=0ms    Red se conecta
         │
         ▼
t=0-5ms  NTC1 presenta 5 Ω → limita inrush a ~31 A (110V) / ~62 A (220V)
         F1 slow-blow NO se abre (tolera pico breve)
         │
         ▼
t>100ms  NTC1 se calienta → resistencia cae a < 0.5 Ω
         Circuito en régimen estable
         HLK-5M05 arranca y produce 5 VDC
```

### 4.2 Transiente de Red (Surge/Spike)

```
Evento: Spike de 1-4 kV en la línea (ej: rayo cercano, switching de motor)
         │
         ▼
         MOV1 detecta sobretensión → resistencia cae a < 1 Ω
         Corriente del transiente se desvía por MOV1 a Neutro
         Voltaje se limita a V_clamp (~395 V para 110V / ~700 V para 220V)
         │
         ▼
         Transiente pasa → MOV1 vuelve a alta impedancia
         Circuito sigue operando normalmente
```

### 4.3 Falla del MOV (Cortocircuito)

```
Evento: MOV1 falla en cortocircuito (después de absorber muchos surges)
         │
         ▼
         Corriente excesiva fluye L → MOV1 → N
         │
         ▼
         F1 (1A slow-blow) se abre → desconecta toda la línea AC
         │
         ▼
         Circuito completamente desconectado (seguro)
         Usuario reemplaza F1 y MOV1
```

### 4.4 Sobrecalentamiento del HLK-5M05

```
Evento: Falla interna del HLK-5M05 → temperatura sube descontroladamente
         │
         ▼
         TF1 (montado en contacto con HLK-5M05) detecta T > 115 °C
         │
         ▼
         Aleación interna de TF1 se funde → contactos se abren permanentemente
         │
         ▼
         Toda la alimentación AC se corta → incendio prevenido
         TF1 es de un solo uso: requiere reemplazo del módulo
```

---

## 5. BOM Completo — Módulo 1 con Códigos LCSC

Todos los componentes verificados para importación desde LCSC/JLCPCB y compatibles con EasyEDA Pro.

### 5.1 Componentes en Serie (Cadena L)

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 1 | J_AC | Terminal block tornillo | 2P, 7.5 mm pitch, 300 V / 15 A | THT 7.5 mm | **C474954** | Cixi Kefa Elec | KF128-7.5-2P | $0.10 |
| 2 | F1_holder | Portafusible PCB | 5×20 mm, tipo BLX-A | THT | **C3131** | Xucheng Elec | 5×20 BLX-A XC-7 | $0.04 |
| 3 | F1 | Fusible slow-blow | 1 A, 250 V, 5×20 mm vidrio | Axial 5×20 mm | **C142715** | Littelfuse | 0215001.MXP | $0.06 |
| 4 | TF1 | Fusible térmico | 115 °C, 15 A, 250 V (1-shot) | Axial THT | **C20448209** | SETsafe/SETfuse | T115 | $0.11 |
| 5 | NTC1 | Termistor NTC inrush | 5 Ω @ 25 °C, 3 A max, 9 mm disco | Radial THT P=7.5 mm | **C3789** | RUILON | NTC 5D-9 | $0.02 |

### 5.2 Componentes en Paralelo (L ↔ N)

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 6a | MOV1 | Varistor (110 V) | 10D241K, MCOV 150 VAC, clamp ~395 V | 10 mm disco radial | **C190241** | Brightking | 241KD10 | $0.03 |
| 6b | MOV1 | Varistor (220 V alt.) | 14D431K, MCOV 275 VAC, clamp ~700 V | 14 mm disco radial | **C273566** | VDR | VDR-14D431K | $0.05 |
| 7 | CX1 | Capacitor X2 safety | 100 nF (104), 275 VAC, MKP film | Film THT P=15 mm | **C434188** | Jimson | MKP104K275A07 | $0.03 |
| 8 | R_disc | Resistencia descarga | 1 MΩ, 1 W, ±1%, metal film | Axial THT D3.3×L9 mm | **C433678** | TyoHM | RN 1WS 1M F T/B A1 | $0.01 |

### 5.3 Filtro EMI (Salida del Módulo)

| # | Ref | Componente | Valor / Specs | Encapsulado | LCSC Code | Fabricante | MPN | Precio aprox. |
|---|---|---|---|---|---|---|---|---|
| 9 | L1 | Choke modo común | 10 mH @ 1 kHz, 0.5 A | DIP-4 THT, UU9.8 | **C148474** | FH (Guangdong Fenghua) | UU9.8-10mH | $0.22 |

### 5.4 Componentes Alternativos / Equivalentes

Para cada componente, si el código LCSC primario no está disponible al momento del pedido:

| Ref | Primario | Alternativa 1 | Alternativa 2 | Notas |
|---|---|---|---|---|
| J_AC | C474954 (Kefa) | C8463 (Dinkle DT-128-7.5-02P) | — | Verificar pitch 7.5 mm y rating 300 V |
| F1_holder | C3131 (Xucheng) | — | — | BLX-A es estándar, ampliamente disponible |
| F1 | C142715 (Littelfuse) | C908063 (Hollyland 1A 250V T) | — | Verificar que sea slow-blow (T = time delay) |
| TF1 | C20448209 (SETfuse T115) | C7499892 (SETfuse RT167) | — | Ambos 115 °C; verificar form factor |
| NTC1 | C3789 (RUILON 5D-9) | C11277 (Nanjing Shiheng MF72 5D9) | C332361 (Hongzhi) | Mismo valor 5 Ω / 9 mm |
| MOV1 (220V) | C273566 (VDR 14D431K) | C1527438 (Bourns MOV-14D431K) | C137830 (Hongzhi 14D431K) | Bourns es premium/certificado |
| MOV1 (110V) | C190241 (Brightking 241KD10) | C136818 (Hongzhi 10D241K) | C3263663 (Bourns MOV-10D241K) | Brightking tiene datasheets formales |
| CX1 | C434188 (Jimson) | C883957 (KNSCHA MKP-X2) | — | Verificar certificación X2 y pitch 15 mm |
| R_disc | C433678 (TyoHM 1M 1W) | C176397 (YAGEO MFR1WSJT-52-1M) | — | Cualquier metal film 1 MΩ ≥ 1 W |
| L1 | C148474 (FH UU9.8-10mH) | C75192 (FH UU9.8Y-10mH) | — | Verificar current rating ≥ 0.5 A |

---

## 6. Notas de Diseño para EasyEDA Pro

### 6.1 Importación de Componentes

Para importar cada componente en EasyEDA Pro:

1. Abrir **Library** → **LCSC Parts**
2. Buscar por código LCSC (ej: `C474954`)
3. Verificar que el footprint coincida con el encapsulado listado
4. Colocar en el esquemático y asignar la referencia (J_AC, F1, etc.)

### 6.2 Reglas de Layout Críticas

| Regla | Valor | Razón |
|---|---|---|
| Creepage L↔N | ≥ 5.5 mm | IEC 60950-1 / RETIE para 300 VAC |
| Creepage AC↔DC | ≥ 6 mm (slot fresado 2 mm recomendado) | Aislamiento 3000 VAC del HLK-5M05 |
| Ancho de traza AC | ≥ 1.0 mm (recomendado 1.5 mm) | Corriente nominal ~200 mA + margen térmico |
| Clearance MOV1 | ≥ 3 mm a componentes vecinos | El MOV puede calentarse durante absorción de surges |
| Montaje TF1 | En contacto mecánico con HLK-5M05 | Debe detectar sobrecalentamiento de la fuente |
| Zona MOV/CX1 | Lo más cerca posible de J_AC | Minimizar longitud de trazas expuestas a transientes |

### 6.3 Consideraciones de Seguridad para PCB

- **No rutear trazas de AC debajo del slot de aislamiento** entre zona caliente y zona SELV
- **Usar cobre de 2 oz** en las trazas de AC si es posible, o ensanchar a 2 mm
- **Marcar en silkscreen** la zona AC con símbolo de alto voltaje ⚡ y "DANGER: HIGH VOLTAGE"
- **Thermal relief** en pads THT conectados a planos de cobre para facilitar soldadura manual

---

## 7. Resumen de Costo del Módulo 1

| Componente | Costo unitario (aprox.) |
|---|---|
| J_AC (C474954) | $0.10 |
| F1_holder (C3131) | $0.04 |
| F1 fuse (C142715) | $0.06 |
| TF1 (C20448209) | $0.11 |
| NTC1 (C3789) | $0.02 |
| MOV1 — 241KD10 (C190241) | $0.03 |
| CX1 (C434188) | $0.03 |
| R_disc (C433678) | $0.01 |
| L1 (C148474) | $0.22 |
| **TOTAL Módulo 1** | **~$0.64 USD** |

*Precios de LCSC en cantidades ≥ 10 unidades. Sujetos a cambio.*

---

## 8. Checklist de Verificación Pre-Fabricación

- [ ] Verificar disponibilidad de cada LCSC code antes de ordenar
- [ ] Confirmar selección de MOV: 10D241K (110 V Colombia) vs 14D431K (dual 110/220 V)
- [ ] Decidir si incluir L1 en prototipo o puentear (recomendación: puentear en proto v1)
- [ ] Verificar que TF1 hace contacto físico con carcasa del HLK-5M05 en el layout
- [ ] Confirmar creepage ≥ 5.5 mm entre pads de J_AC (L↔N) en el PCB
- [ ] Revisar que F1 está ANTES del MOV en la cadena serie (protección contra falla MOV)
