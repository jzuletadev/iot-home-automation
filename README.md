# IOT Home Automation

Sistema de domótica residencial de arquitectura centralizada. Esta guía reproduce el sistema completo desde cero, paso a paso.

> Repositorio privado. El nombre comercial del producto está sin definir — el nombre del repo es un placeholder.

## Premisa

**Home Assistant es el backend. Las apps son el frontend y son el producto.**

No se construye backend propio. Home Assistant ya resuelve el broker, el motor de automatizaciones, la persistencia y 3,000+ integraciones — replicarlo tomaría meses y saldría peor. Lo que sí se construye es la **capa de presentación con marca propia**: apps que consumen la API de HA y presentan la experiencia bajo una identidad propia.

El cliente final nunca ve Home Assistant. Ve la app. Ese es el diferenciador frente a cualquier otro integrador que también use HA.

## Arquitectura

```
Apps propias (React Native + Expo)          ← el producto
        │
        │ REST API + WebSocket
        ▼
Raspberry Pi — Home Assistant OS             ← el backend, no se toca
├── HA Core (puerto 80 en esta instalación)
│   ├── State Machine · Event Bus
│   ├── Automation Engine · Recorder
└── ESPHome Builder (add-on)
        │
        │ API nativa de ESPHome sobre WiFi   ← cifrada, con autodescubrimiento
        ▼
Nodos ESP8266 / ESP32
```

**Sin MQTT.** La API nativa de ESPHome reemplaza al broker: HA descubre los nodos solo, el canal va cifrado sin configurar TLS, la latencia es menor y el OTA viaja por el mismo canal. Mosquitto solo haría falta si algo **fuera** de HA necesitara consumir los datos.

### API que consumen las apps

```
REST:      http://<ip-del-pi>:<puerto>/api/
WebSocket: ws://<ip-del-pi>:<puerto>/api/websocket

GET  /api/states                  → todas las entidades y su estado
GET  /api/states/light.bedroom_light
POST /api/services/light/turn_on  → { "entity_id": "light.bedroom_light" }
POST /api/services/switch/turn_off
```

Autenticación con *long-lived access token* (perfil de usuario en HA → abajo del todo).

> **Puerto:** HA usa `8123` por defecto, pero **esta instalación responde en el puerto 80** (por eso la URL del navegador no lleva `:8123`). Verificado el 2026-08-25: el 8123 está cerrado y el 80 sirve la API. Las apps deben apuntar al 80 mientras siga así.
>
> Vale la pena decidirlo a conciencia: el 80 es cómodo (URL sin puerto), pero choca con un reverse proxy futuro — Cloudflared, NGINX y la mayoría de guías de HA asumen 8123. Confirmado por `ha core info`: en HAOS el puerto de Core lo administra el **Supervisor**, no `configuration.yaml` — ese archivo no tiene bloque `http:`. Cambiarlo con `ha core options --port 8123` **no persiste**: el ajuste se guarda pero Core vuelve al 80 en cada reinicio. Origen sin identificar — ver nota abajo.

---

## Requisitos previos

| Componente | Detalle |
|---|---|
| Raspberry Pi 4 | 2GB mínimo, 4GB recomendado |
| MicroSD | 32GB clase 10 A2 |
| Fuente | USB-C oficial 5V/3A |
| ESP8266 | NodeMCU v2 o Wemos D1 mini |
| Módulo relay | 5V, 1 canal, con optoacoplador |
| PC | Para flasheo inicial por USB y desarrollo de las apps |

Driver USB-serial en Windows: **CH340** (mayoría de NodeMCU genéricas) o **CP2102**. Sin él la placa no aparece como puerto COM.

---

## Paso 1 — Home Assistant en el Raspberry Pi

- [x] Flashear **Home Assistant OS** al MicroSD con Raspberry Pi Imager
- [x] Primer arranque (la primera vez tarda 10–20 min)
- [x] Onboarding en `http://homeassistant.local`
- [x] Crear las áreas iniciales
- [ ] **IP fija del Pi** — pendiente: solo se resuelve bien con reserva DHCP en el Velop (Opción A)

> La IP fija no es opcional. Si la IP del Pi cambia, las apps pierden el backend y hay que reconfigurar cada cliente. Hacerlo antes de que haya nodos instalados.

### Datos de red

Relevados el 2026-08-25 desde la PC de desarrollo y confirmados por SSH en el Pi.

La red tiene **doble NAT**: el módem del ISP (KAON, `192.168.1.1`) alimenta al mesh Linksys, que crea su propia red `10.85.1.0/24`. **El Pi y todos los nodos viven en la red del Linksys** — el KAON no los ve y configurarlo a él no sirve de nada.

```
Internet → KAON (ISP)        192.168.1.1   ← no tocar
             └→ Linksys Velop Pro 6E  10.85.1.1   ← acá se configura todo
                  ├─ Raspberry Pi (cable)  10.85.1.74
                  ├─ PC de desarrollo      10.85.1.22
                  └─ nodos ESP (futuro)
```

| Dato | Valor |
|---|---|
| MAC del Pi | `D8:3A:DD:A5:A7:EE` |
| IP actual del Pi | `10.85.1.74` (por cable) |
| Router / servidor DHCP | `10.85.1.1` — Linksys Velop Pro 6E (MX62 / MX6200) |
| Subred | `10.85.1.0/24` |
| Puerto de HA | `80` |

El prefijo `D8:3A:DD` es OUI de Raspberry Pi Ltd — sirve para identificar el Pi en la lista de clientes, donde suele aparecer con un nombre poco descriptivo.

### Opción A — Reserva DHCP en el Velop

Interfaz web local: `http://10.85.1.1` (**no** `192.168.1.1`). La contraseña es la de administrador del router definida en el setup, no la del WiFi.

**Connectivity → pestaña Local Network → DHCP Reservations →** seleccionar la MAC `D8:3A:DD:A5:A7:EE` de la lista de clientes → **Save**.

Desde la **app Linksys** (más confiable en Velop, que empuja la administración a la app): Menú → *Advanced Settings* → *Local Network Settings* → **DHCP Reservations** → agregar el Pi. También suele poderse desde la lista de dispositivos, tocando el Pi.

> Reservar **la IP que ya tiene** (`10.85.1.74`) y no una nueva: ya está arrendada a esa MAC, así que no puede chocar con otro equipo y no obliga a reconfigurar nada de lo que ya funciona.

### Opción B — IP estática en el propio Pi *(solo si se conoce el pool DHCP)*

> **No usar sin verificar el pool primero.** Las IPs que el Velop repartió por DHCP observadas el 2026-08-25 (`.22`, `.74`, `.100`, `.138`, `.233`, `.240`) indican un pool de al menos `.22`–`.240`, así que `10.85.1.74` cae adentro. Fijarla como estática en el Pi crea el riesgo de que el router entregue esa misma IP a otro equipo. Se intentó y se revirtió con `ha network update end0 --ipv4-method auto`.
>
> Esta opción solo sirve con una IP confirmada fuera del rango del pool.

Si el acceso al router se complica, se resuelve desde HA sin tocar el Velop. El Pi está por cable, así que la config va en el adaptador ethernet:

**Settings → System → Network →** adaptador ethernet → **IPv4 → Static**

```
IP address:  10.85.1.74/24
Gateway:     10.85.1.1
DNS:         10.85.1.1
```

**Por CLI** — vía SSH, add-on *Terminal & SSH*. La interfaz por cable de este Pi es **`end0`**, no `eth0` (los kernels recientes de Raspberry Pi renombraron el ethernet integrado):

```bash
ha network update end0 \
  --ipv4-method static \
  --ipv4-address 10.85.1.74/24 \
  --ipv4-gateway 10.85.1.1 \
  --ipv4-nameserver 10.85.1.1 \
  --ipv4-nameserver 1.1.1.1

ha network info    # verificar: method debe pasar de "auto" a "static"
```

`1.1.1.1` como DNS secundario es deliberado: si el Velop se reinicia o se traba, el Pi sigue resolviendo nombres en vez de quedarse sin actualizaciones ni integraciones cloud.

Volver a DHCP: `ha network update end0 --ipv4-method auto`

Fijar **la misma IP que ya tiene** hace que la sesión SSH no se corte al aplicar. Con otra IP, se cae.

> Riesgo a tener presente: sin ver el pool DHCP del Velop no se sabe si `10.85.1.74` está dentro del rango que reparte. Si lo está, el router podría entregarle esa IP a otro equipo y generar un conflicto. Es poco probable —.74 ya está arrendada al Pi— pero la Opción A no tiene ese riesgo. Conviene mirar el rango del pool cuando se logre entrar al router.

### Verificar

Reiniciar el Pi y comprobar que vuelve con la misma IP:

```powershell
Test-Connection 10.85.1.74 -Count 2
(Invoke-WebRequest http://10.85.1.74/api/ -SkipHttpErrorCheck).StatusCode  # 401 = HA vivo
```

HAOS y no Docker: la Add-on Store solo existe en HAOS, y ahí es donde vive ESPHome Builder.

## Paso 2 — ESPHome Builder

- [x] Instalar el add-on **ESPHome Builder** desde la Add-on Store
- [ ] Crear `secrets.yaml` desde el ícono de llave en ESPHome Builder

Usar [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) como plantilla. La clave de cifrado de la API se genera en la [documentación de ESPHome](https://esphome.io/components/api.html).

> No hace falta instalar Mosquitto. Ver *Arquitectura*.

## Paso 3 — Primer nodo

### 3.1 Cableado

**Nunca alimentar la bobina del relay desde el pin de 3.3V del ESP** — el regulador no da la corriente y resetea la placa.

| ESP8266 | Módulo relay |
|---|---|
| `VIN` (NodeMCU) o `5V` (D1 mini) | `VCC` |
| `GND` | `GND` |
| `D1` = **GPIO5** | `IN` |

Pines seguros en ESP8266: **GPIO5 (D1)**, GPIO4 (D2), GPIO12/13/14 (D6/D7/D5). Evitar D3, D4 y D8 — definen el modo de arranque y un módulo *active-LOW* conectado ahí impide que la placa bootee.

**Si el módulo trae jumper JD-VCC** (4 pines de control): quitarlo, `VCC` a **3.3V** y `JD-VCC` a **5V**. Así el optoacoplador aísla de verdad, y se evita que el relay no apague limpio — con `VCC` a 5V y señal de 3.3V quedan 1.7V que a veces alcanzan para mantener encendido el LED del opto.

Lado de carga: `COM` + `NO`, **interrumpiendo la línea viva (fase), nunca el neutro**. Con el flipón abajo. Estándar Guatemala: 120V 60Hz.

### 3.2 Firmware

Los archivos viven en **`/config/esphome/`** dentro del Pi. El dashboard de ESPHome Builder **solo lista los `.yaml` que están en la raíz** de esa carpeta — los subdirectorios existen para packages e includes, y sus archivos no aparecen como dispositivos. Por eso `bedroom.yaml` va en la raíz y `base.yaml` en `common/`.

```
/config/esphome/
├── secrets.yaml        credenciales — nunca sale del Pi
├── common/base.yaml    paquete compartido, no es un dispositivo
└── bedroom.yaml        ← aparece como tarjeta en el dashboard
```

Este repo es un espejo de esa carpeta: copiar es 1:1, sin reescribir rutas.

**1. Generar la clave de cifrado de la API** (por SSH, en el Pi):

```bash
head -c 32 /dev/urandom | base64
```

**2. Cargar los secretos** — en ESPHome Builder, ícono de **llave** arriba a la derecha. Usar [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) como plantilla, con la clave del paso anterior.

**3. Crear los archivos** por SSH, pegando el contenido del repo:

```bash
mkdir -p /config/esphome/common
cat > /config/esphome/common/base.yaml <<'EOF'
# ...contenido de common/base.yaml...
EOF
cat > /config/esphome/bedroom.yaml <<'EOF'
# ...contenido de bedroom.yaml...
EOF
```

> **Al renombrar un nodo, el OTA necesita la IP explícita.** Cambiar `esphome.name` rompe el descubrimiento mDNS: el dashboard busca el nombre nuevo y el hardware todavía anuncia el viejo, así que el nodo figura *Offline* y el OTA no tiene destino. En el diálogo de troubleshoot: **Set the address manually** → IP actual del nodo → **Install**. Después del reinicio el nombre nuevo resuelve solo y la dirección manual deja de hacer falta.

**4. Compilar** — refrescar el dashboard. `bedroom` aparece como tarjeta nueva → menú `⋮` → **Install**.

- [x] `secrets.yaml` cargado con la clave de API generada
- [x] `common/base.yaml` y `bedroom.yaml` en `/config/esphome/`
- [x] Compila sin errores

> El nodo usa un paquete compartido: `common/base.yaml` aporta WiFi, API cifrada, OTA, AP de rescate y entidades de diagnóstico. Cada nodo nuevo solo declara sus substitutions, su board y sus periféricos.

### 3.3 Flasheo inicial

El primer flasheo es por USB; todos los siguientes son OTA por WiFi.

> **Gotcha:** la opción *"Plug into this computer"* de ESPHome Builder usa la Web Serial API, que **exige contexto seguro**. Como HA corre sobre `http://` sin TLS (da igual el puerto), esa opción falla o no aparece.

Ruta que sí funciona:

1. ESPHome Builder → **Install → Manual download → Factory format (.bin)**
2. Abrir **https://web.esphome.io** (es HTTPS, ahí sí hay Web Serial)
3. Conectar el ESP por USB → **Connect** → elegir el `.bin` → Install

### 3.4 Validación en banco (sin 120V todavía)

- [x] El nodo aparece en **Settings → Devices & Services → ESPHome**
- [x] Toggle desde el dashboard → se oye el *click* y enciende el LED del módulo
- [x] Apaga limpio — el módulo tolera la señal de 3.3V, no hizo falta el jumper JD-VCC
- [ ] 24–48h conectado sin desconexiones

### 3.5 Instalación física — 120V

> **Diagrama visual completo:** https://claude.ai/code/artifact/a7c26fa4-7e7b-4a1a-8f36-e130a5b85605

El HLK-PM01 se conecta **directo a la línea viva**: no hay transformador de aislamiento aguas arriba. Sus pines de entrada están a 120V respecto a tierra mientras el circuito esté energizado, incluso con el ESP apagado. Si no hay experiencia con línea viva, esta parte la hace un electricista — el resto del sistema se arma y prueba con 5V.

**Cortar el flipón y verificar con detector de tensión sin contacto sobre los conductores**, no confiar en la etiqueta del tablero.

#### Conexiones — lado 120V

| Desde | Hasta | Nota |
|---|---|---|
| Línea (L, negro) | `HLK-PM01 AC-L` | Alimenta la fuente |
| Línea (L, negro) | `Relay COM` | Empalme. Se interrumpe la línea, **nunca el neutro** |
| `Relay NO` | Ventilador L | **NO**, no NC: sin alimentación el ventilador queda apagado |
| Neutro (N, blanco) | `HLK-PM01 AC-N` | |
| Neutro (N, blanco) | Ventilador N | Empalme, pasa directo |
| Tierra (G, verde) | Ventilador G | Directo y entero, sin pasar por el relay |

#### Conexiones — lado 5V

| Desde | Hasta | Nota |
|---|---|---|
| `HLK-PM01 +Vo` | `NodeMCU VIN` | VIN acepta 5V; el regulador de la placa baja a 3.3V |
| `HLK-PM01 +Vo` | `Relay VCC` | La bobina nunca se alimenta del pin 3.3V del ESP |
| `HLK-PM01 -Vo` | `NodeMCU GND` + `Relay GND` | Masa común obligatoria |
| `NodeMCU D1` (GPIO5) | `Relay IN` | Módulo active-HIGH, sin `inverted` |

**Condensador de 470–1000 µF** entre `+Vo` y `-Vo`, cerca del NodeMCU. El HLK-PM01 da 600mA; el ESP8266 pica ~300mA en cada transmisión WiFi y la bobina consume ~70mA. Sin él, los picos hunden la tensión y la placa se reinicia sola — un fallo intermitente que aparece recién con todo montado en la pared.

#### Orden de armado

- [ ] Probar todo el lado de 5V con cargador USB, antes de tocar 120V
- [ ] Cortar el flipón y verificar ausencia de tensión
- [ ] Cablear el lado de 120V con borneras de tornillo, nunca cables torcidos a mano
- [ ] Conectar la salida DC al circuito de control — **desconectando el USB primero**, alimentar por ambos retroalimenta el bus de 5V
- [ ] Cerrar el gabinete antes de energizar
- [ ] Verificar ≥6mm de separación entre el lado 120V y el lado 5V dentro de la caja

#### Carga inductiva

Un ventilador genera un pico de tensión al abrir el contacto, que arquea y desgasta el relay — con el tiempo puede soldarlo cerrado, dejando el ventilador encendido e ignorando a HA. Un módulo de 10A/250V sobra para un ventilador residencial (<1A), pero un **snubber RC** (100 Ω + 100 nF en serie, capacitor clase X2) en paralelo con los contactos absorbe el arco y multiplica la vida útil del relay.

Para aire acondicionado o motores grandes el relay no alcanza: va un contactor externo, con el relay accionando solo su bobina.

## Paso 4 — Automatizaciones

- [ ] "Sin movimiento por 30 min → apagar luz"
- [ ] "Temperatura > 27°C → encender ventilador"
- [ ] Notificación push de movimiento en horario inusual
- [ ] Verificar en **History** que el Recorder está guardando

Criterio de salida: 2 automatizaciones corriendo una semana sin intervención.

## Paso 5 — Expansión de nodos

Repetir el ciclo del Paso 3 por cada nodo. Costos y componentes en [`hardware/bom.csv`](hardware/bom.csv).

- [ ] Living Room — ESP32, 2 luces + ventilador, DHT22 + LDR
- [ ] Kitchen — ESP8266, extractor, MQ135
- [ ] Entrance — ESP32, portón/timbre, PIR + LDR
- [ ] Garden — ESP32, bomba de riego, sensor de humedad de suelo

## Paso 6 — Acceso remoto

- [ ] Instalar add-on **Cloudflared**
- [ ] Crear el tunnel en Cloudflare Zero Trust → Networks → Tunnels
- [ ] Apuntar un subdominio al tunnel
- [ ] Agregar `trusted_proxies` en `configuration.yaml`
- [ ] Probar desde datos móviles

Alternativa: **Tailscale** (gratis, más simple, pero requiere instalar el cliente en cada dispositivo — sirve para administración, no para clientes finales).

Nunca port forwarding.

## Paso 7 — App móvil

> Este paso va **después** de tener el sistema físico funcionando y usado con la app oficial de HA. Diseñar una interfaz para un sistema que no se ha vivido lleva a decisiones equivocadas.

- [ ] `npx create-expo-app` en `apps/mobile`
- [ ] Expo Router para navegación, Zustand para estado global
- [ ] Generar long-lived token en HA
- [ ] Cliente REST (`GET /api/states`) con Axios
- [ ] WebSocket con reconexión automática
- [ ] Pantalla de setup (URL + token), dashboard por áreas, detalle de área
- [ ] Toggles y sliders conectados a `/api/services/...`
- [ ] Expo Notifications
- [ ] Identidad visual propia

---

## Estructura del repositorio

```
esphome/
  secrets.yaml.example    plantilla de credenciales (secrets.yaml va gitignored)
  common/base.yaml        paquete compartido: wifi, api, ota, diagnóstico
  <nodo>.yaml             un archivo por nodo, en la raíz (el dashboard solo lista este nivel)
homeassistant/            config versionada de HA (ver nota abajo)
  dashboards/
apps/                     frontends propios — el producto
hardware/
  bom.csv                 componentes y costos por nodo
  wiring/                 diagramas de cableado
```

**Nota sobre `homeassistant/`:** la config real vive en el Pi (`/config`). Este directorio guarda solo lo que vale la pena versionar — `configuration.yaml`, `automations.yaml`, `scripts.yaml`, dashboards en YAML. El resto (`.storage/`, la base de datos, logs) está en `.gitignore`. La sincronización es manual, vía el add-on Samba o Studio Code Server; no hay sync automático.

Lo mismo con `esphome/`: ESPHome Builder edita en `/config/esphome/` del Pi. Este directorio es la copia versionada.

## Convenciones

**Entidades y áreas en inglés**, en la capa de HA:

```
light.bedroom_light        switch.kitchen_extractor
sensor.living_room_temp    binary_sensor.entrance_motion
```

**Cómo ESPHome deriva el `entity_id`.** La regla es:

```
entity_id = <esphome.name> + "_" + <nombre del componente>
```

Por eso el nodo se llama `bedroom` (no `node-bedroom`) y la luz se llama `Light` (no `Bedroom Light`): juntos dan `light.bedroom_light`. Nombrar la luz "Bedroom Light" dentro de un nodo llamado `node-bedroom` produce `light.node_bedroom_bedroom_light`, con el área repetida.

`friendly_name` en el nodo controla solo el nombre visible: `Bedroom` + `Light` se muestra como "Bedroom Light", que es lo que ve el usuario, mientras el `entity_id` queda limpio.

Las apps traducen a español en la capa de presentación. El backend queda consistente con ESPHome, la documentación de HA y el código; el idioma del usuario final es decisión del frontend, no del backend.

**Documentación en español.** Código, entidades y nombres de archivo en inglés.
