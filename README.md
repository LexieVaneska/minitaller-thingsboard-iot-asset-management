
# Guía de configuración: ThingsBoard Cloud + Wokwi
## Mini-taller — Gestión de activos IoT: Cuarto frío inteligente

---

## ¿Qué vas a configurar?

Al seguir esta guía tendrás:

- Un **device** registrado en ThingsBoard Cloud que representa el ESP32
- Un **dashboard** con widgets de temperatura, alarma, enfriamiento e historial
- Dos **botones RPC** para activar y desactivar el enfriamiento de forma remota
- Un **proyecto Wokwi** con ESP32 + DHT22 + 3 LEDs enviando telemetría en tiempo real

---

## Parte 1 — Crear cuenta en ThingsBoard Cloud

Entra a:

```
https://thingsboard.cloud
```

Puedes registrarte con Google, GitHub o Apple, o crear una cuenta con correo y contraseña. Una vez dentro verás el panel principal (Home) con el menú lateral izquierdo.

---

## Parte 2 — Crear el device

El device representa el ESP32 dentro de ThingsBoard. Es la entidad digital que recibe la telemetría.

En el menú lateral:

```
Entities → Devices → + Add device → Add new device
```

En el formulario:

- **Name:** `ESP32-Cuarto-Frio-TuNombre`  
  (ejemplo: `ESP32-Cuarto-Frio-Alex`)
- **Device profile:** `default`
- Todo lo demás: dejar en blanco

Haz click en **Add**.

---

## Parte 3 — Copiar el Access Token

El Access Token es la contraseña que usa el ESP32 para autenticarse con ThingsBoard vía MQTT.

En la lista de devices, haz click sobre el device que acabas de crear. Se abre un panel lateral. Verás el botón:

```
Copy access token
```

Cópialo y guárdalo en un bloc de notas. Lo necesitarás en el código de Wokwi.

> **Importante:** no compartas este token. Cualquiera que lo tenga puede enviar datos a tu device.

---

## Parte 4 — Configurar Wokwi

### 4.1 Crear cuenta

Entra a [wokwi.com](https://wokwi.com) y regístrate. Es gratuito y no requiere instalación.

### 4.2 Crear proyecto

```
New Project → ESP32
```

Se abren dos archivos: `diagram.json` y `main.py`.

### 4.3 Circuito

Reemplaza todo el contenido de `diagram.json` con el siguiente JSON. Define el ESP32, el sensor DHT22 y los tres LEDs:

```json
{
  "version": 1,
  "author": "Mini-taller ThingsBoard",
  "editor": "wokwi",
  "parts": [
    { "type": "board-esp32-devkit-v1", "id": "esp32", "top": 100, "left": 100, "attrs": {} },
    { "type": "wokwi-dht22", "id": "dht1", "top": 0, "left": 550, "attrs": {} },
    { "type": "wokwi-led", "id": "ledVerde", "top": 150, "left": 550, "attrs": { "color": "green" } },
    { "type": "wokwi-led", "id": "ledRojo", "top": 250, "left": 550, "attrs": { "color": "red" } },
    { "type": "wokwi-led", "id": "ledAzul", "top": 350, "left": 550, "attrs": { "color": "blue" } }
  ],
  "connections": []
}
```

### 4.4 Conexiones manuales

Conecta los componentes en Wokwi de la siguiente forma:

| Desde | Hacia | Color de cable |
|---|---|---|
| DHT22 VCC | ESP32 3V3 | Rojo |
| DHT22 GND | ESP32 GND | Negro |
| DHT22 SDA | ESP32 D4 | Verde |
| LED Verde ánodo (+) | ESP32 D26 | Verde |
| LED Verde cátodo (-) | ESP32 GND | Negro |
| LED Rojo ánodo (+) | ESP32 D25 | Rojo |
| LED Rojo cátodo (-) | ESP32 GND | Negro |
| LED Azul ánodo (+) | ESP32 D33 | Azul |
| LED Azul cátodo (-) | ESP32 GND | Negro |

> El ánodo es la pata más larga del LED (+). El cátodo es la pata más corta (-).

### 4.5 Código

Reemplaza todo el contenido de `main.py` con el siguiente código. Antes de correr, sustituye `PEGAR_TOKEN_AQUI` con tu Access Token real:

```python
import network
import time
from machine import Pin
from umqtt.simple import MQTTClient
import dht
import json

# ── Configuración ──────────────────────────────────────
WIFI_SSID    = "Wokwi-GUEST"
WIFI_PASS    = ""
MQTT_BROKER  = "thingsboard.cloud"
MQTT_PORT    = 1883
ACCESS_TOKEN = "PEGAR_TOKEN_AQUI"

TOPIC_TEL    = b"v1/devices/me/telemetry"
TOPIC_RPC    = b"v1/devices/me/rpc/request/+"

SETPOINT     = 4.0       # °C — límite máximo cuarto frío de refrigeración
PIN_DHT      = 4
PIN_VERDE    = 26
PIN_ROJO     = 25
PIN_AZUL     = 33

# ── Hardware ───────────────────────────────────────────
sensor    = dht.DHT22(Pin(PIN_DHT))
led_verde = Pin(PIN_VERDE, Pin.OUT)
led_rojo  = Pin(PIN_ROJO,  Pin.OUT)
led_azul  = Pin(PIN_AZUL,  Pin.OUT)

# ── Estado global ──────────────────────────────────────
cooling_active = False

# ── Callback RPC (botón desde ThingsBoard) ─────────────
def on_rpc(topic, msg):
    global cooling_active
    print("RPC recibido:", msg)
    try:
        data = json.loads(msg)
        method = data.get("method", "")
        if method == "activarEnfriamiento":
            cooling_active = True
            print("Enfriamiento activado por operario")
        elif method == "desactivarEnfriamiento":
            cooling_active = False
            print("Enfriamiento desactivado por operario")
    except:
        print("Error al leer RPC")

# ── WiFi ───────────────────────────────────────────────
def conectar_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASS)
    print("Conectando WiFi", end="")
    while not wlan.isconnected():
        print(".", end="")
        time.sleep(0.5)
    print(" OK")

# ── MQTT ───────────────────────────────────────────────
def conectar_mqtt():
    client = MQTTClient(
        "esp32_cuarto_frio", MQTT_BROKER,
        port=MQTT_PORT, user=ACCESS_TOKEN, password=""
    )
    client.set_callback(on_rpc)
    client.connect()
    client.subscribe(TOPIC_RPC)
    print("MQTT conectado y suscrito a RPC")
    return client

# ── Temperatura sintética ──────────────────────────────
# Simula cuarto frío de refrigeración: oscila entre 1°C y 7°C
# Setpoint en 4°C — cuando sube, se activa la alarma
temp_sim = 1.0
subiendo = True

def siguiente_temp():
    global temp_sim, subiendo
    if subiendo:
        temp_sim += 0.3
        if temp_sim >= 7.0:
            subiendo = False
    else:
        temp_sim -= 0.3
        if temp_sim <= 1.0:
            subiendo = True
    return round(temp_sim, 1)

# ── Loop principal ─────────────────────────────────────
def main():
    global cooling_active

    conectar_wifi()
    client = conectar_mqtt()

    while True:
        client.check_msg()
        temperature = siguiente_temp()

        alarm_active = temperature > SETPOINT

        if not alarm_active:
            cooling_active = False

        led_verde.value(1 if not alarm_active else 0)
        led_rojo.value(1 if alarm_active else 0)
        led_azul.value(1 if cooling_active else 0)

        payload = json.dumps({
            "temperature":   temperature,
            "setpoint":      SETPOINT,
            "alarmStatus":   alarm_active,
            "coolingStatus": cooling_active
        })
        client.publish(TOPIC_TEL, payload)
        print(payload)

        time.sleep(2)

main()
```

Presiona ▶ para iniciar la simulación.

---

## Parte 5 — Verificar telemetría

Antes de crear el dashboard, confirma que los datos están llegando correctamente.

En ThingsBoard:

```
Entities → Devices → tu device → Latest telemetry
```

Deben aparecer estas variables con valores actualizados cada 2 segundos:

| Variable | Descripción |
|---|---|
| `temperature` | Temperatura actual del cuarto frío (°C) |
| `setpoint` | Límite máximo permitido (4.0°C) |
| `alarmStatus` | `true` si hay alarma, `false` si todo está bien |
| `coolingStatus` | `true` si el enfriamiento está activo |

Si no aparecen datos, revisa la sección de Troubleshooting al final de este documento.

---

## Parte 6 — Crear el dashboard

En el menú lateral:

```
Dashboards → + Add dashboard → Create new dashboard
```

Nombra el dashboard:

```
CuartoFrio-Dashboard-TuNombre
```

Abre el dashboard. Estará en modo edición automáticamente.

---

## Parte 7 — Agregar widgets

Haz click en **+ Add widget** para cada uno de los siguientes.

### Widget 1 — Gauge de temperatura

```
Analogue gauges → Gauge
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **Data key:** `temperature`
- **Title:** `Temperatura`
- **Min:** `−5` / **Max:** `15`

### Widget 2 — Estado de alarma

```
Cards → Label & value card
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **Data key:** `alarmStatus`
- **Title:** `Alarma`
- **Units:** vacío


### Widget 3 — Estado del enfriamiento

```
Cards → Label & value card
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **Data key:** `coolingStatus`
- **Title:** `Enfriamiento`
- **Units:** vacío
- **Background:** azul `#2196F3`

### Widget 4 — Historial de temperatura

```
Charts → Time series chart
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **Data key:** `temperature`
- **Title:** `Historial de temperatura`

### Widget 5 — Botón activar enfriamiento

```
Buttons → Command button
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **On click → Action:** `Execute RPC`
- **Method:** `activarEnfriamiento`
- **Parameters:** dejar en `None`
- **Label:** `Activar Enfriamiento`

### Widget 6 — Botón desactivar enfriamiento

```
Buttons → Command button
```

- **Device:** ESP32-Cuarto-Frio-TuNombre
- **On click → Action:** `Execute RPC`
- **Method:** `desactivarEnfriamiento`
- **Parameters:** dejar en `None`
- **Label:** `Desactivar Enfriamiento`

Haz click en **Save** para guardar el dashboard.

---

## Parte 8 — Probar el sistema

Con Wokwi corriendo y el dashboard abierto, observa:

1. La temperatura sube gradualmente de 1°C a 7°C y baja de vuelta, en pasos de 0.3°C cada 2 segundos.
2. Cuando supera 4°C (el setpoint), el LED rojo se enciende y `alarmStatus` pasa a `true`.
3. El LED verde se apaga.
4. Presiona **Activar Enfriamiento** en el dashboard — el LED azul se enciende y `coolingStatus` pasa a `true`.
5. Cuando la temperatura vuelve a bajar de 4°C, el sistema apaga automáticamente el enfriamiento y vuelve al LED verde.

---

## Significado de los LEDs

| LED | Color | Estado del sistema |
|---|---|---|
| Verde | Encendido | Temperatura dentro del setpoint — sistema normal |
| Rojo | Encendido | Temperatura superó el setpoint — alarma activa |
| Azul | Encendido | Sistema de enfriamiento activado |


---

## Troubleshooting

### El device no aparece activo en ThingsBoard

- Verifica que Wokwi está corriendo (botón ▶ presionado)
- Verifica que el Access Token en el código es el correcto
- Espera 10-15 segundos después de iniciar — el ESP32 necesita conectarse al WiFi y luego al broker

### No aparecen datos en Latest telemetry

- Confirma que el device está activo (punto verde en la lista)
- Revisa que el topic MQTT en el código sea exactamente: `v1/devices/me/telemetry`
- Verifica que las claves en el JSON coincidan exactamente con las que buscas en el dashboard

### Error MQTT en la consola de Wokwi

El error más común es `MQTTException: 5`, que significa autenticación fallida. Causas:

- El Access Token está mal copiado (verifica que no tenga espacios extra)
- El broker no corresponde — para ThingsBoard Cloud debe ser exactamente: `thingsboard.cloud`

### Los botones RPC no tienen efecto

- Verifica que el método en el botón coincide exactamente con el nombre en el código:  
  `activarEnfriamiento` y `desactivarEnfriamiento` (sin espacios, sensible a mayúsculas)
- Confirma que el device está activo en el momento de presionar el botón

---

## Referencia rápida

| Parámetro | Valor |
|---|---|
| Broker MQTT | `thingsboard.cloud` |
| Puerto | `1883` |
| Topic telemetría | `v1/devices/me/telemetry` |
| Topic RPC | `v1/devices/me/rpc/request/+` |
| Setpoint | `4.0°C` |
| Rango temperatura sintética | `1°C – 7°C` |
| Intervalo de envío | `2 segundos` |
| Pin DHT22 | `D4` |
| Pin LED Verde | `D26` |
| Pin LED Rojo | `D25` |
| Pin LED Azul | `D33` |
