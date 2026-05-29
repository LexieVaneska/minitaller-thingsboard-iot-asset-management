# Cuarto Frío Inteligente — ThingsBoard + ESP32

Demo de monitoreo y control IoT para el Taller de Sistemas Embebidos, ITCR 2026.  
Un ESP32 con sensor DHT22 publica telemetría vía MQTT a ThingsBoard. Desde el dashboard se puede activar el enfriamiento de forma remota mediante RPC, o dejar que el firmware actúe de forma autónoma con histéresis de 1°C.

---

## Hardware

| Componente | Función |
|---|---|
| ESP32 | Microcontrolador principal |
| DHT22 | Temperatura y humedad |
| LED Rojo | Alarma — temp > 4°C |
| LED Azul | Enfriamiento activo (RPC o AUTO) |
| LED Verde | Sistema en estado normal |

## Modos de operación

**Manual** — El operario observa la alarma en el dashboard y activa el enfriamiento con un botón RPC.  
**Automático** — El firmware activa el enfriamiento si `temp > 4.0°C` y lo desactiva si `temp ≤ 3.0°C`.

## Parámetros clave

```
Setpoint:   4.0°C
Histéresis: 1.0°C
Ciclo:      ~16 s  (1°C → 7°C → 1°C en Wokwi)
Protocolo:  MQTT — topic v1/devices/me/telemetry
```

## Archivos

```
main.py          Firmware MicroPython para ESP32
diagram.json     Circuito simulado en Wokwi
dashboard.json   Dashboard importable en ThingsBoard
```

## Simulación

El proyecto corre directamente en [Wokwi](https://wokwi.com) sin hardware físico. Solo se necesita una cuenta en ThingsBoard Cloud (plan gratuito) y configurar el Access Token en `main.py`.

---

Taller de Sistemas Embebidos · ITCR · 2026
