# Ingeniería inversa avanzada de la Shelly Plus 1

La Shelly Plus 1 (SKU **SNSW-001X16EU**, PCB fechada 2021‑08‑06) es un relé inteligente basado en **ESP32‑U4WDH single‑core a 160 MHz** con **fuente offline no aislada tipo buck** que alimenta simultáneamente una bobina de relé a 12 V y la lógica digital a 3.3 V. Su arquitectura combina un convertidor offline probable **LinkSwitch‑TN (LNK304)** con un segundo buck step‑down 12 V → 3.3 V (marcado **"S478"**), un **relé monoestable Panasonic AJVN5241F** de 16 A / 250 V AC y una red de protección compuesta por **MOV 10D471K**, resistencia fusible y diodos Schottky **SS14**. Esta combinación permite tolerar **85–265 V AC** en un volumen de 37 × 42 × 16 mm pero **deja toda la electrónica a potencial de red**: flashear firmware sin aislamiento galvánico destruye el adaptador USB‑UART y puede electrocutar al operador. El dispositivo es plenamente compatible con **Tasmota (binario `tasmota32solo1.factory.bin`)** y **ESPHome** sobre ESP32 single‑core con `CONFIG_FREERTOS_UNICORE=y`, y existe un sensor **NTC externo de 10 kΩ en GPIO32** que habilita la protección por sobre‑temperatura que el firmware Shelly ya implementa y que debe replicarse en cualquier firmware propio.

---

## 1. Mapeo completo de componentes y su función eléctrica

### 1.1 Microcontrolador — ESP32‑U4WDH

El nomenclador Espressif descompone en: **U** = flash embebida en el package, **4** = 4 MB, **WD** = Wi‑Fi + Bluetooth Dual‑mode (Classic BR/EDR + BLE), **H** = calificación alta temperatura (PCN‑2021‑021). Package **QFN‑48 de 5 × 5 mm** (no 6 × 6). Arquitectura **Xtensa LX6 a 160 MHz**: aunque los silicios fabricados después de finales de 2021 son físicamente dual‑core, Shelly y ESPHome los ejecutan deliberadamente en modo **single‑core (`CONFIG_FREERTOS_UNICORE=y`)** a 160 MHz — esto es crítico porque determina qué binario de Tasmota funciona. SRAM 520 kB, ROM 448 kB, radio Wi‑Fi 802.11 b/g/n 2.4 GHz hasta +20 dBm, Bluetooth 4.2. **Consumos típicos**: TX Wi‑Fi ≈ 240 mA pico, RX ≈ 100 mA, modem‑sleep ≈ 30 mA, deep‑sleep ≈ 10 µA. La tensión operativa del chip es 3.0–3.6 V (abs. max 3.6 V); el rango 2.3–3.6 V que aparece en el pin VDD del header es el límite del pin de I/O, no del core.

El **cristal Y1 de 40.000 MHz** es la referencia para el PLL interno. El ESP32 admite 24/26/40 MHz, pero 40 MHz es el valor recomendado por Espressif porque genera derivaciones más limpias para el PHY Wi‑Fi. La PLL multiplica ×4 para 160 MHz y ×6 para 240 MHz; Shelly corre a ×4. Condensadores de carga típicos 8–12 pF, estabilidad requerida ≤ ±10 ppm para cumplir máscaras de espectro Wi‑Fi.

### 1.2 Relé — Panasonic AJVN5241F (serie JV‑N, 10.9 mm de altura)

La lectura literal del datasheet Panasonic corrige varios supuestos habituales: **SPST‑NO (1 Form A), monoestable con retorno por resorte** (no biestable latching). Bobina: **720 Ω ±10 % a 20 °C**, corriente nominal **16.7 mA**, potencia **200 mW**. Tensión de pickup ≤ 9 V (75 % de 12 V); drop‑out ≥ 0.6 V; Vmax 18 V. **Tiempo de operación máximo 12 ms; tiempo de liberación máximo 20 ms con diodo de rueda libre** — esto justifica la presencia del SS14 contra la bobina. Contactos AgSnO₂ con doble certificación **16 A / 250 V AC cos φ = 1.0 y 16 A / 250 V AC cos φ = 0.4** (VDE 40055712, UL E43028), lo que significa que el relé **sí está físicamente calificado para cargas inductivas** a 16 A, aunque Shelly no publica ese número en su hoja técnica pública. Vida mecánica 20 M operaciones; vida eléctrica 10 k ops a 16 A/125 V AC, cayendo a 50 k a 10 A y 100 k a 8 A. Rigidez dieléctrica 2500 Vrms contacto‑bobina — **este es el único aislamiento galvánico real de toda la placa** y es lo que convierte el par I/O en un "dry contact" a pesar de que el resto del circuito esté caliente.

### 1.3 Red de diodos A7

El marcaje **"A7"** en encapsulado SMA/DO‑214AC corresponde al **SS14: Schottky 40 V / 1 A, V_F ≈ 0.5 V a 1 A, I_FSM 25–30 A**. En la Plus 1 cumple dos roles distintos: **D19 actúa como diodo de rueda libre (flyback) a través de la bobina del relé** — imprescindible para absorber la energía del colapso magnético cuando el transistor driver corta (200 mW almacenados en la inductancia de bobina). **D12/D14 pertenecen a la etapa de buck offline**: uno es el **rectificador de salida del buck LinkSwitch‑TN** (conmuta a 66 kHz, de ahí la necesidad de Schottky rápido) y el otro puede ser el rectificador de entrada de media onda desde la red (topología típica del LNK304 en modo buck directo low‑side).

### 1.4 Fuente conmutada offline no aislada — topología y ICs

Esta es la parte más crítica para entender la seguridad. La Shelly Plus 1 **NO usa flyback aislado** (no hay transformador con aislamiento primario‑secundario, solo un inductor SMD ferrítico) y **NO usa capacitive dropper** (el dropper no toleraría los 16 A del relé ni los picos de corriente Wi‑Fi del ESP32). Es un **convertidor buck non‑isolated offline** en dos etapas:

**Primera etapa — AC→12 V**: el candidato más probable es el **Power Integrations LNK304DN** (LinkSwitch‑TN, MOSFET 700 V integrado, 66 kHz, 120 mA en topología buck con realimentación low‑side). Es el mismo IC confirmado en la Shelly 1 Gen1 y en múltiples Shellys de la familia. Algunas revisiones recientes de la placa Plus podrían haber migrado al **BPSemi BP2522 / BP2525** (SOT‑23‑5, también 700 V integrado) por costo, como ya se observó en la Plus 2PM. Este tema tiene **incertidumbre**: los teardowns públicos del Plus 1 rara vez identifican el IC primario por encontrarse bajo pegote térmico o con marcado borrado.

**Segunda etapa — 12 V→3.3 V**: un segundo convertidor buck marcado **"S478"** en SOT‑23‑6, **no un LDO AMS1117**. El motivo es térmico: un LDO dropando 12 V→3.3 V con corrientes pico del ESP32 de 240 mA disiparía ~2 W, imposible en el volumen de 37 × 42 × 16 mm. El datasheet del S478 no es público (probable rebrand chino equivalente a MP2143 o SY8113).

**Etapa de entrada AC**: en orden desde los terminales L/N hacia dentro, hay: resistencia fusible (actúa como fuse no renovable y limita inrush), **MOV 10D471K** (disco cerámico azul de 10 mm, V_N = 470 V @ 1 mA, V_clamp = 775 V a 25 A, energía 60–70 J, surge máx 2.5 kA — clampa transientes > 470 V y en caso de fallo catastrófico se cortocircuita para quemar el fusible), rectificación probable **de media onda con un solo diodo** (típica de LNK304 buck directo) seguida del **condensador electrolítico azul de bus DC** (450 V class, ~4.7–10 µF) que almacena la energía entre ciclos. El inductor SMD es el inductor del buck primario (~2.2 mH para LNK304 a 66 kHz). El **condensador marrón/negro a la salida** filtra los 12 V antes de alimentar la bobina del relé y la entrada del S478.

**La consecuencia de seguridad es absoluta**: el nodo "GND" del header de programación está galvánicamente unido al retorno de la red (Línea activa cuando la red está invertida — algo común en instalaciones residenciales donde los polos no están marcados). Esto convierte el chasis del ESP32 en una superficie potencialmente peligrosa.

### 1.5 Frontend de detección AC — entrada SW

El terminal **SW no es una entrada de contacto seco**. Detecta presencia de tensión de red mediante una red divisora de resistencias de alto valor (centenas de kΩ a MΩ) que escala los 120–240 V AC a un nivel que un GPIO del ESP32 puede muestrear de forma rectificada/integrada. **R29 marcada "473" (47 kΩ)** forma parte de esta cadena de atenuación o actúa como pull‑down de referencia. En la instalación AC, el interruptor externo conecta la Línea al terminal SW; en modo DC, conecta SW a masa. El ESP32 muestrea la señal por **GPIO4** con debounce de 50 ms en firmware. Esta topología implica que **el estado lógico del bit en GPIO4 está en fase con la red**, con ondulación de 50/60 Hz superpuesta que se filtra por software.

### 1.6 Sensores auxiliares

A diferencia de lo habitualmente asumido, **sí hay un NTC externo**: un termistor 10 kΩ con coeficiente β ≈ 3350 conectado al **GPIO32 (ADC1)** en configuración downstream con pull‑up de 10 kΩ. Permite al firmware Shelly (y también a ESPHome) implementar la protección por **sobre‑temperatura** con umbral típico a 80 °C. Además, **GPIO33 lee la tensión de suministro del relé escalada ×8** mediante otro divisor, permitiendo detectar fallos en el buck primario. La Plus 1 **no tiene medición de corriente** (esa es la Plus 1PM, que usa un ADE7953), por lo que la protección por sobrecorriente debe provenir del disyuntor aguas arriba.

---

## 2. Especificaciones oficiales y uso en red de 110 V

La ficha oficial de Shelly confirma **entrada 110–240 V AC 50/60 Hz y también 24–48 V DC o 12 V DC ±10 %**, consumo en reposo < 1.2 W, capacidad de conmutación **16 A AC / 10 A DC** (variante Shelly Plus 1 UL para EE. UU. limitada a 15 A por certificación UL E504925, FCC ID **2ALAY‑SHELLYPLUS1**). Rango operativo ambiental **−20 °C a +40 °C**, humedad 30–70 %, altitud máxima 2000 m. **En 120 V AC residencial la potencia máxima resistiva es 16 A × 120 V ≈ 1920 W**; en 230 V sube a 3680 W.

### 2.1 Cableado en 110 V (US/LATAM)

Los terminales son **L (línea), N (neutro), SW (sensado de interruptor externo), I (común del contacto), O (salida del contacto al consumo)** más tres pads opcionales para alimentación DC (+12 V, +, ⊥). La configuración clásica residencial 110 V:

```
  Breaker ──► L terminal (alimenta fuente + via I al contacto)
  Neutral ──► N terminal
         ┌── I (puentear desde L en la mayoría de instalaciones)
  Relay ─┤
         └── O ──► consumo ──► Neutral
  External switch: un extremo a L, otro a SW
```

**Requisito crítico**: L de alimentación y L del interruptor SW deben estar en la misma fase (no es un dispositivo trifásico). Para instalaciones 3‑vías (US "three‑way switch") con dos puntos de control se usa un interruptor momentary cableado convencionalmente al terminal SW más controles app/API.

### 2.2 Modos de operación del switch

Shelly expone cinco modos por firmware configurables desde la UI web o API RPC Gen2: **Toggle** (el relé imita el estado del interruptor, ideal para interruptores basculantes tradicionales), **Momentary** (pulsador que alterna), **Edge** (cada cambio de estado del SW alterna — útil para remplazar interruptores basculantes sin recablear), **Detached** (el SW solo reporta estado a la nube/HA sin mandar el relé — para escenarios de smart‑bulbs donde no se quiere cortar la alimentación de la bombilla), **Activation** (cualquier flanco enciende y rearma un temporizador auto‑off, estilo sensor de movimiento). Plus comportamiento por power‑on: On/Off/Restore‑last/Match‑input.

### 2.3 Restricciones de carga

Resistiva pura hasta 16 A. Para **cargas inductivas Shelly exige un snubber RC externo de 0.1 µF + 100 Ω / ½ W / 600 V AC en paralelo al consumo** — motores, electroválvulas, transformadores ferromagnéticos, ventiladores. Las **cargas LED/CFL son tratadas como inductivas** por el inrush extremo de los drivers (cientos de amperios en µs); se recomienda el mismo snubber. Capacitivas permitidas. No hay medición de corriente, así que la protección contra sobrecarga depende del breaker exterior.

---

## 3. Protecciones térmicas y de seguridad

El diseño combina tres capas: **hardware pasivo** (MOV 10D471K clampando transientes > 470 V, resistencia fusible sacrificable, creepage/clearance del PCB conforme LVD 2014/35/EU entre la zona mains y la sección low‑voltage de MCU, rigidez dieléctrica del relé de 2500 Vrms), **sensado activo** (NTC 10 kΩ β=3350 en GPIO32, lectura de raíl del relé en GPIO33) y **firmware** (corte automático del relé al superar ~80 °C con notificación cloud/API, watchdog hardware del ESP32 activo). No existe protección por sobrecorriente en esta variante — esa función se delega al disyuntor del tablero. Las pistas de la sección AC tienen ancho incrementado para 16 A continuos y la separación AC–DC en el PCB está diseñada para 2000 m de altitud y grado de polución 2.

El fabricante insiste explícitamente en que el dispositivo **debe instalarse dentro de una caja de empotrar u otro encerramiento protector**: los terminales tornillo son accesibles y energizados. El SW es una entrada **sensada a tensión de red**, no un contacto seco, y por eso nunca debe conectarse a una salida lógica 3.3/5 V externa ni a un pulsador pasando por un cable de baja tensión junto a mains.

---

## 4. Proceso detallado de flasheo con firmware propio

### 4.1 Validación del pinout del header de 7 pines

El pinout reportado por el usuario (1=GND, 2=GPIO0, 3=EN, 4=VDD, 5=RX, 6=TX, 7=GPIO19) coincide con la documentación de **devices.esphome.io/devices/shelly-plus-1** y con el template de blakadder. Existe **una discrepancia documentada**: el blog de RevK (2022) reportó GPIO16 en el pin 7 en lugar de GPIO19, posiblemente por un error de medida o una revisión anterior del PCB. Como el pin 38 del QFN‑48 del ESP32‑U4WDH es efectivamente GPIO19, la asignación del usuario es la más probable. **Recomendación**: antes de conectar cualquier sensor externo al pin 7, verificar con multímetro en modo continuidad desde el pin 38 del chip hasta el pad del header.

### 4.2 Hardware requerido

Un **adaptador USB‑UART exclusivamente 3.3 V**: CP2102, FT232RL con jumper de 3.3 V, o CH340 3.3 V. **NUNCA 5 V** — el ESP32‑U4WDH no es 5 V tolerante y la señal RX del ESP32 recibirá el pico TX del adaptador directamente sobre su pin. Muchos adaptadores FDTI no suministran los 240 mA pico del ESP32 en TX Wi‑Fi y causan brownouts; se recomienda **fuente de banco externa 3.3 V / ≥ 500 mA** y alimentar solo por allí. El paso del header es **1.27 mm (0.05")** con pines cuadrados de 0.4 mm — se encuentran headers AliExpress de este paso, o como hack rápido funcionan agujas de costura finas presionadas contra los pads con cinta aislante.

### 4.3 Advertencia de seguridad no negociable

La fuente es **no aislada**. El pin GND del header está eléctricamente unido al retorno de la red cuando la placa se alimenta por L/N. Conectar el USB del PC a ese GND con la placa enchufada significa **conectar el chasis del PC a la línea**: destrucción segura del adaptador UART, daño probable del puerto USB, riesgo real de electrocución si se toca simultáneamente el chasis del PC y tierra. **Protocolo obligatorio**: desconectar totalmente la placa de la red AC (terminales L y N al aire), alimentar únicamente por el pin 4 (VDD 3.3 V) del header desde el adaptador, y recién entonces conectar las líneas UART.

### 4.4 Conexionado y entrada al modo bootloader

```
USB‑UART        Shelly Plus 1 (header)
GND      ──►    Pin 1 (GND)
3V3      ──►    Pin 4 (VDD)
TXD      ──►    Pin 5 (RX Shelly)
RXD      ──►    Pin 6 (TX Shelly)
GND jumper ──►  Pin 2 (GPIO0)  [mantener durante power‑up]
```

Para entrar en bootloader: aplicar jumper GPIO0→GND, luego energizar VDD. Opcionalmente togglear EN (pin 3) a GND por un instante hace un reset sin tocar la alimentación, muy útil para re‑entradas rápidas. Si el LED rojo queda encendido fijo (no parpadeando), el chip está en ROM download mode.

### 4.5 Backup obligatorio del firmware original

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
    read_flash 0x0 0x400000 shelly_plus1_backup.bin
```

Guardar este `.bin` de 4 MB en lugar seguro — es el único camino para volver al firmware Shelly original sin re‑comprar el dispositivo. Verificar checksum SHA‑256 después del read.

### 4.6 Flasheo de Tasmota (la decisión del binario es crítica)

Debido al modo single‑core, **se debe usar `tasmota32solo1.factory.bin`, NO `tasmota32.factory.bin`**. El binario `.factory` incluye bootloader + tabla de particiones + safeboot + aplicación y va escrito desde 0x0. El binario sin sufijo `.factory` es solo para OTA y fallará al iniciar tras flasheo serial. Este punto está discutido en detalle en `github.com/arendst/Tasmota/discussions/22802`.

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 erase_flash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
    write_flash -z 0x0 tasmota32solo1.factory.bin
```

Tras flashear, desconectar el jumper GPIO0→GND, ciclar la alimentación, buscar el AP `tasmota-XXXX` (192.168.4.1), configurar Wi‑Fi, luego en la UI **Configuration → Auto-configuration → "Shelly Plus 1"**. Esto aplica automáticamente el template de blakadder:

```
{"NAME":"Shelly Plus 1 ","GPIO":[288,0,0,0,192,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,32,224,0,0,0,0,0,4736,4705,0,0,0,0,0,0],"FLAG":0,"BASE":1}
```

### 4.7 Mapeo GPIO confirmado de la Shelly Plus 1

| Función | GPIO | Notas eléctricas |
|---|---|---|
| LED de estado (rojo interno) | 0 | Activo en bajo (invertido) |
| Entrada del switch externo (terminal SW) | 4 | Pull‑down interno; pin con strapping, atender boot‑time |
| Pin 7 del header (reservado para sensor externo) | 19 | Libre |
| Botón físico del encapsulado (reset/factory) | 25 | Activo en bajo con pull‑up |
| Control del relé | 26 | Activo en alto, drivea driver de bobina |
| NTC 10 kΩ (β=3350) | 32 | ADC1, pull‑up 10 kΩ, downstream |
| Sensado tensión raíl del relé | 33 | ADC1, factor ×8 |

### 4.8 Configuración ESPHome lista para usar

```yaml
substitutions:
  device_name: "shelly-plus-1"
  friendly_name: "Shelly Plus 1"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  platformio_options:
    board_build.f_cpu: 160000000L

esp32:
  variant: esp32
  board: esp32doit-devkit-v1
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_FREERTOS_UNICORE: y
      CONFIG_ESP32_DEFAULT_CPU_FREQ_160: y
      CONFIG_ESP32_DEFAULT_CPU_FREQ_MHZ: "160"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Shelly-Plus-1 Fallback"
    password: !secret ap_password

captive_portal:
logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

output:
  - platform: gpio
    id: relay_output
    pin: GPIO26

switch:
  - platform: output
    id: relay
    name: "${friendly_name} Relay"
    output: relay_output
    restore_mode: RESTORE_DEFAULT_OFF

binary_sensor:
  - platform: gpio
    name: "${friendly_name} Switch"
    pin: GPIO4
    filters:
      - delayed_on_off: 50ms
    on_press:
      then:
        - switch.toggle: relay

  - platform: gpio
    name: "${friendly_name} Button"
    pin:
      number: GPIO25
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

status_led:
  pin:
    number: GPIO0
    inverted: true

sensor:
  - platform: adc
    id: temp_adc
    pin: GPIO32
    attenuation: 12db
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

  - platform: adc
    name: "${friendly_name} Relay Supply Voltage"
    pin: GPIO33
    attenuation: 12db
    filters:
      - multiply: 8
```

La clave de encriptación de la API se genera en esphome.io/components/api.html (32 bytes base64). La compilación con framework `esp-idf` falla en Raspberry Pi ARM; usar x86_64 o cambiar a `arduino` si se compila en ARM.

### 4.9 Alternativa OTA — solo si el firmware Shelly original es ≤ 1.4.x

La herramienta **`mgos32-to-tasmota32`** (github.com/tasmota/mgos32-to-tasmota32, **archivado en agosto 2025**) permitía subir Tasmota desde la UI Shelly sin tocar hardware. Descarga `mgos32-to-tasmota32-Plus1.zip`, subir por Settings → Firmware → drag&drop (NO usar "update by URL"). **Bloqueado en firmware Shelly ≥ 1.5**, que para un dispositivo comprado en 2026 es casi seguro ya. No hay backdoor en la API RPC Gen2: `Shelly.Update` solo acepta firmware firmado. Tuya‑Convert no aplica (Shelly usa Mongoose OS, no Tuya). La migración **Tasmota → ESPHome por OTA brickea** en Tasmota ≥ v12, así que si se quiere terminar en ESPHome, flashearlo directamente por serial desde Shelly stock.

### 4.10 Integración con Home Assistant post‑flasheo

Con **Tasmota**: configurar MQTT en la consola (`Backlog MqttHost ip; MqttUser u; MqttPassword p; Topic shelly_plus_1_x`), luego activar la integración Tasmota de HA que auto‑descubre por prefijo `tasmota/discovery/`. Con **ESPHome**: HA auto‑detecta por mDNS; agregar en Settings → Devices → ESPHome pegando IP y la clave de encriptación.

---

## 5. Correcciones a supuestos comunes y puntos de incertidumbre

Varios datos populares en internet sobre este dispositivo son incorrectos y vale aclararlos. **La resistencia de bobina del relé es 720 Ω, no ~400 Ω**; el tiempo de operación máximo es 12 ms y el de liberación 20 ms (con diodo), no 10/5 ms. El **sensor de temperatura no es el del die del ESP32** (sin calibrar): es un NTC externo 10 kΩ β=3350 en GPIO32. El package del ESP32‑U4WDH es **QFN‑48 5×5 mm**, no 6×6. La conversión 12 V→3.3 V no usa un LDO AMS1117 sino un **buck switching S478** por razones térmicas. Sobre el IC primario offline, el candidato más fuerte es **LNK304** por herencia de familia pero no está confirmado sin desoldar pegote térmico en cada revisión — podría ser BP2522/BP2525 en placas más recientes. Y el pin 7 del header es **GPIO19** según ESPHome devices y coherente con la pinout del QFN‑48, aunque un blog (RevK 2022) reportó GPIO16; probar con continuímetro antes de usar el pin.

---

## Conclusión

La Shelly Plus 1 es una lección de ingeniería de densidad extrema: ocho componentes activos clave resuelven conmutación de 3.6 kW, tolerancia a 265 V AC con surge de 2.5 kA, y computación Wi‑Fi/BLE en 24 cm³. El costo de esa densidad es el **diseño no aislado**, que convierte el flasheo casero en una operación que exige desconexión total de la red y alimentación exclusivamente por el pin VDD del header. Una vez sorteada esa barrera, la plataforma es un ESP32 single‑core perfectamente domesticable desde Tasmota (`tasmota32solo1.factory.bin`) o ESPHome con YAML estándar, con acceso completo al NTC para replicar la protección térmica que el firmware Shelly implementa por defecto. La Plus 1 no mide corriente — esa diferenciación la reserva Shelly para la 1PM con ADE7953 — por lo que cualquier protección por sobrecorriente en una instalación residencial 110/120 V debe depender del disyuntor del tablero, nunca del firmware. Para un reemplazo confiable de un interruptor tradicional detrás de la placa, con lógica Home Assistant y temporización local incluso sin internet, la Plus 1 sigue siendo, cinco años después de su lanzamiento, el punto de referencia en relación prestaciones/volumen/precio del sector.