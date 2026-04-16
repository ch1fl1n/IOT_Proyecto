# SmartJoy — Joystick Inteligente Inalámbrico IoT
 
> **Proyecto de IoT** | Ingeniería de Sistemas  
> **Sensor principal:** KY-023 Dual-Axis Joystick + ESP32  
 
---
 
## Tabla de Contenido
 
1. [Visión del Proyecto](#visión-del-proyecto)
2. [Usuarios y Contexto](#usuarios-y-contexto)
3. [Arquitectura del Sistema](#arquitectura-del-sistema)
4. [Diagrama de Bloques](#diagrama-de-bloques)
5. [Restricciones de Hardware y Presupuesto](#restricciones-de-hardware-y-presupuesto)
6. [Reporte del Spike](#reporte-del-spike)
7. [MVP — Definición del Producto Mínimo Viable](#mvp--definición-del-producto-mínimo-viable)
8. [Backlog de Features](#backlog-de-features)
9. [Cronograma de Sprints](#cronograma-de-sprints)
10. [Equipo](#equipo)
---
 
## Visión del Proyecto
 
### Problema que Resuelve
 
Los sistemas de control remoto actuales para robótica educativa, domótica y automatización básica presentan tres limitaciones críticas:
 
1. **Alta latencia o dependencia de línea de visión** (controles IR o RF básicos).
2. **Sin retroalimentación de estado** — el operador no sabe si el comando fue recibido, ni el nivel de batería del dispositivo controlado.
3. **Costo elevado** o ecosistemas propietarios que dificultan el aprendizaje y la personalización.
### Solución Propuesta
 
**SmartJoy** es un joystick inteligente, inalámbrico y basado en IoT que permite controlar distintos dispositivos de forma remota, segura y en tiempo real mediante Wi-Fi/MQTT. El sistema es bidireccional: no solo envía comandos sino que también **recibe telemetría del dispositivo controlado** (nivel de batería, estado de conexión, confirmación de comando), mejorando la experiencia del operador y reduciendo fallos operativos.
 
### Propuesta de Valor
 
```
Joystick KY-023 → ESP32 (transmisor) ──MQTT/Wi-Fi──→ ESP32 (receptor) → Dispositivo controlado
                                            ↑                    |
                                            └────── Telemetría ──┘
```
 
---
 
##  Usuarios y Contexto
 
| Segmento | Caso de uso | Beneficio |
|---|---|---|
| Estudiantes de robótica | Control de robot educativo móvil | Aprendizaje de IoT sin hardware costoso |
| Hobbyistas / Makers | Control de brazo robótico DIY | Interfaz personalizable y extensible |
| Docentes | Demostraciones en clase | Sistema plug-and-play con feedback visual |
| Automatización hogareña | Control de actuadores domóticos | Bajo costo, integración con MQTT/Home Assistant |
 
---
 
##  Arquitectura del Sistema
 
### Stack Tecnológico
 
| Capa | Tecnología | Justificación |
|---|---|---|
| **Hardware entrada** | KY-023 (joystick) + ESP32 DevKit | Bajo costo, ADC integrado, Wi-Fi nativo |
| **Protocolo IoT** | MQTT (broker: Mosquitto / HiveMQ Cloud) | Ligero, baja latencia, pub/sub ideal para tiempo real |
| **Transporte** | Wi-Fi 802.11 b/g/n | Disponible en ESP32, sin infraestructura extra |
| **Firmware** | Arduino IDE / PlatformIO (C++) | Ecosistema maduro, amplia documentación |
| **Dashboard (nice-to-have)** | Node-RED o MQTT Explorer | Visualización de telemetría |
| **Alimentación** | Batería Li-Po 3.7V + regulador 3.3V | Portabilidad |
 
### Flujo de Datos
 
```
[KY-023 VRx/VRy] ──ADC──→ [ESP32 Tx]
[KY-023 SW btn]  ──GPIO──→ [ESP32 Tx] ──MQTT publish──→ [Broker] ──MQTT subscribe──→ [ESP32 Rx] ──→ [Actuador]
                                ↑                                                           |
                           [Display OLED]  ←──── MQTT publish (telemetría) ────────────────┘
```
 
---
 
##  Diagrama de Bloques
 
```
┌─────────────────────────────────────────────────────────────────────┐
│                        UNIDAD TRANSMISORA                           │
│                                                                     │
│  ┌──────────┐   Analog    ┌──────────────────────────────────────┐  │
│  │ KY-023   │ ──VRx/VRy──▶│                                      │  │
│  │ Joystick │             │         ESP32 DevKit                 │  │
│  │          │ ──SW btn───▶│   (ADC + GPIO + Wi-Fi + MQTT)        │  │
│  └──────────┘             │                                      │  │
│                           │  ┌──────────┐   ┌────────────────┐  │  │
│  ┌──────────┐             │  │  OLED    │   │  Indicadores   │  │  │
│  │ Batería  │ ──3.3V─────▶│  │  Display │   │  LED Estado    │  │  │
│  │ Li-Po    │             │  └──────────┘   └────────────────┘  │  │
│  └──────────┘             └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                              Wi-Fi / MQTT
                            (topic: smartjoy/cmd)
                                    │
                    ┌───────────────▼───────────────┐
                    │        BROKER MQTT             │
                    │   (HiveMQ Cloud / Mosquitto)   │
                    └───────────────┬───────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │        UNIDAD RECEPTORA        │
                    │                                │
                    │  ┌──────────────────────────┐  │
                    │  │      ESP32 DevKit        │  │
                    │  │  (MQTT sub + control)    │  │
                    │  └──────────┬───────────────┘  │
                    │             │                   │
                    │  ┌──────────▼───────────┐       │
                    │  │   Driver de Motores   │       │
                    │  │  (L298N / L293D)      │       │
                    │  └──────────┬────────────┘      │
                    │             │                   │
                    │  ┌──────────▼───────────┐       │
                    │  │  Dispositivo Target  │       │
                    │  │  (Robot / Actuador)  │       │
                    │  └──────────────────────┘       │
                    └────────────────────────────────┘
                                    │
                    (topic: smartjoy/telemetry) → Tx ESP32
```
 
---
 
##  Restricciones de Hardware y Presupuesto
 
### Especificaciones del Sensor KY-023
 
| Parámetro | Valor |
|---|---|
| Voltaje de operación | 3.3V – 5V |
| Ejes analógicos | X (VRx) y Y (VRy) — salida 0 a VCC |
| Posición central | ~1.65V @ 3.3V / ~2.5V @ 5V |
| Potenciómetros | 2× 10kΩ |
| Botón (Z-axis) | Digital, activo en LOW con pull-up |
| Resolución ADC ESP32 | 12 bits (0–4095) |
| Interfaz | 5 pines: VCC, GND, VRx, VRy, SW |
| Dimensiones | ~3.2 × 3.5 cm, peso ~11g |
| Precio estimado (Colombia) | COP $3,000 – $6,000 |
 
### Restricciones del ESP32 DevKit
 
| Recurso | Disponible | Uso estimado proyecto |
|---|---|---|
| Flash | 4 MB | ~500 KB (firmware + MQTT) |
| RAM (SRAM) | 520 KB | ~80 KB en operación |
| Pines ADC | 18 canales (12-bit) | 2 (VRx, VRy) |
| Pines GPIO | 34 usables | ~6 (joystick + LEDs + OLED) |
| Consumo activo Wi-Fi | ~160–260 mA | Requiere fuente estable ≥500 mA |
| Consumo sleep | ~10 µA (deep sleep) | N/A en MVP |
| Frecuencia CPU | 240 MHz (doble núcleo) | Núcleo 0: MQTT / Núcleo 1: lectura sensor |
 
### Presupuesto Estimado
 
| Componente | Cantidad | Precio Unit. (COP) | Total |
|---|---|---|---|
| ESP32 DevKit v1 | 2 | $18,000 | $36,000 |
| KY-023 Joystick | 1 | $4,500 | $4,500 |
| Driver L298N / L293D | 1 | $8,000 | $8,000 |
| OLED 0.96" I2C (SSD1306) | 1 | $10,000 | $10,000 |
| Batería Li-Po 3.7V 1000mAh | 1 | $15,000 | $15,000 |
| Módulo cargador TP4056 | 1 | $3,000 | $3,000 |
| Cables, protoboard, LEDs | — | $8,000 | $8,000 |
| **Total estimado** | | | **~$84,500 COP** |
 
>  Costo aproximado en USD: ~$20 USD. Todos los componentes disponibles en mercado local (TecnoSur, uElectronics, MercadoLibre Colombia).
 
---
 
##  Reporte del Spike
 
### Spike #1 — Viabilidad de Comunicación MQTT entre dos ESP32 con KY-023
 
**Objetivo del spike:** Verificar que es técnicamente posible leer el joystick KY-023 con un ESP32 y transmitir los valores en tiempo real por MQTT a un segundo ESP32 con latencia aceptable (<100ms).
 
**Fecha de ejecución:** Sprint 1 (Semanas 1–2)
 
**Resultado esperado / criterio de aceptación:**
- [ ] ESP32 lee VRx y VRy del KY-023 con ADC de 12 bits sin ruido excesivo.
- [ ] Los valores se publican como JSON en tópico MQTT `smartjoy/cmd` cada 50ms.
- [ ] ESP32 receptor recibe y parsea el mensaje correctamente.
- [ ] Latencia end-to-end medida < 150ms en red local.
- [ ] Se documenta el esquema de pines y el payload JSON acordado.
**Riesgos identificados:**
1. **ADC no lineal en ESP32** — El ADC del ESP32 tiene distorsión conocida entre 0.1V–0.3V y 3.0V–3.3V. Mitigación: aplicar curva de calibración o usar librería `esp_adc_cal`.
2. **Jitter de Wi-Fi** — Las interrupciones del stack Wi-Fi pueden afectar la lectura periódica del ADC. Mitigación: leer en Core 1, gestionar MQTT en Core 0 (FreeRTOS).
3. **Disponibilidad del broker en red de la universidad** — Algunos routers bloquean el puerto 1883. Mitigación: usar HiveMQ Cloud (TLS 8883) como fallback.
**Formato del payload MQTT definido:**
```json
{
  "x": 2048,
  "y": 1980,
  "btn": 0,
  "ts": 1718500000
}
```
 
 
##  MVP — Definición del Producto Mínimo Viable
 
El MVP de SmartJoy consiste en **dos ESP32 comunicados por MQTT** donde el joystick KY-023 controla en tiempo real un motor DC (o servo) en el dispositivo receptor, con confirmación visual de conexión.
 
###  Must-Have (necesario para que el sistema funcione)
 
- Lectura analógica de ejes X/Y del KY-023 con ESP32
- Publicación de comandos por MQTT (tópico `smartjoy/cmd`)
- Suscripción y recepción de comandos en ESP32 receptor
- Control de motor/servo según valores del joystick
- Indicador LED de estado de conexión Wi-Fi/MQTT
- Reconexión automática ante pérdida de conexión
###  Nice-to-Have (mejoras opcionales)
 
- Display OLED con nivel de batería y estado de conexión
- Telemetría bidireccional (receptor publica estado al transmisor)
- Modos de operación (velocidad máxima, modo preciso)
- Dashboard en Node-RED para monitoreo
- Encriptación TLS en la conexión MQTT
- App móvil básica de monitoreo
- Modo de ahorro de energía (deep sleep cuando inactivo)
---
 
##  Backlog de Features
 
> Gestionado en **GitHub Projects** — ver tablero [aquí](../../projects)
 
### Release 1 — Fundamentos y Viabilidad (Semanas 1–2)
 
| ID | Feature | Tipo | Sprint | Estado |
|---|---|---|---|---|
| F-01 | Configurar repositorio GitHub y estructura de proyecto | Must-have | Sprint 1 |  Done |
| F-02 | **[SPIKE]** Validar lectura KY-023 + transmisión MQTT entre dos ESP32 | Must-have | Sprint 1 |  In Progress |
| F-03 | Esquema de pines y payload JSON documentado | Must-have | Sprint 1 |  In Progress |
| F-04 | README.md inicial con arquitectura y backlog | Must-have | Sprint 1 |  Done |
| F-05 | Definir backlog priorizado en GitHub Projects | Must-have | Sprint 2 |  To Do |
 
### Release 2 — Control Básico Funcional (Semanas 3–4)
 
| ID | Feature | Tipo | Sprint | Estado |
|---|---|---|---|---|
| F-06 | Firmware transmisor: lectura KY-023 + publicación MQTT | Must-have | Sprint 3 |  To Do |
| F-07 | Firmware receptor: suscripción MQTT + control motor/servo | Must-have | Sprint 3 |  To Do |
| F-08 | Indicador LED estado conexión (transmisor) | Must-have | Sprint 4 |  To Do |
| F-09 | Reconexión automática Wi-Fi/MQTT | Must-have | Sprint 4 |  To Do |
| F-10 | Pruebas de latencia y ajuste de frecuencia de muestreo | Must-have | Sprint 4 |  To Do |
 
### Release 3 — Telemetría y Robustez (Semanas 5–6)
 
| ID | Feature | Tipo | Sprint | Estado |
|---|---|---|---|---|
| F-11 | Telemetría bidireccional (receptor → transmisor) | Nice-to-have | Sprint 5 |  To Do |
| F-12 | Display OLED: estado y telemetría en transmisor | Nice-to-have | Sprint 5 |  To Do |
| F-13 | Calibración ADC y filtro de ruido (media móvil) | Must-have | Sprint 6 |  To Do |
| F-14 | Pruebas de campo (distancia, obstáculos, interferencia) | Must-have | Sprint 6 |  To Do |
 
### Release 4 — Pulido y Entrega Final (Semanas 7–8)
 
| ID | Feature | Tipo | Sprint | Estado |
|---|---|---|---|---|
| F-15 | Feature Freeze — corrección de bugs | Must-have | Sprint 7 |  To Do |
| F-16 | Documentación final (README, esquemático, video demo) | Must-have | Sprint 7 |  To Do |
| F-17 | Preparación demo y pitch | Must-have | Sprint 8 |  To Do |
| F-18 | Dashboard Node-RED (opcional si el tiempo lo permite) | Nice-to-have | Sprint 8 |  To Do |
 
---
 
##  Cronograma de Sprints
 
```
SEMANA  │  SPRINT   │  RELEASE   │  ACTIVIDAD PRINCIPAL
────────┼───────────┼────────────┼──────────────────────────────────────────────
  1     │  Sprint 1 │            │  Setup repo · Spike KY-023 + MQTT · README
  2     │  Sprint 2 │ Release 1  │  Resultados Spike · Backlog priorizado
        │           │   Retro  │  Retrospectiva con profesor
────────┼───────────┼────────────┼──────────────────────────────────────────────
  3     │  Sprint 3 │            │  Firmware transmisor · Firmware receptor
  4     │  Sprint 4 │ Release 2  │  LEDs · Reconexión automática · Pruebas
        │           │   Retro  │  Retrospectiva con profesor
────────┼───────────┼────────────┼──────────────────────────────────────────────
  5     │  Sprint 5 │            │  Telemetría bidireccional · OLED
  6     │  Sprint 6 │ Release 3  │  Calibración ADC · Pruebas de campo
        │           │   Retro  │  Retrospectiva con profesor
────────┼───────────┼────────────┼──────────────────────────────────────────────
  7     │  Sprint 7 │            │  Feature Freeze · Bug fixing · Documentación
  8     │  Sprint 8 │ Release 4  │  Demo · Pitch · Entrega final
        │           │   Final  │  Evaluación final del curso
────────┴───────────┴────────────┴──────────────────────────────────────────────
```
 
---
 
##  Equipo
 
| Nombre | Rol |
|---|---|
| [Nombre 1] | Firmware / Hardware |
| [Nombre 2] | Comunicaciones IoT / MQTT |
| [Nombre 3] | Documentación / Testing |
 
---
 
##  Estructura del Repositorio
 
```
smartjoy-iot/
├── README.md                   ← Este archivo
├── firmware/
│   ├── transmitter/            ← Código ESP32 con KY-023
│   └── receiver/               ← Código ESP32 receptor
├── hardware/
│   ├── schematic.png           ← Esquemático de conexiones
│   └── bom.md                  ← Bill of Materials
├── docs/
│   ├── spike-report.md         ← Reporte detallado del Spike
│   └── architecture.md         ← Decisiones de arquitectura (ADRs)
└── .github/
    └── ISSUE_TEMPLATE/         ← Templates para issues
```
 
---
 
##  Referencias
 
- [Datasheet KY-023 — ArduinoModules.info](https://arduinomodules.info/ky-023-joystick-dual-axis-module/)
- [ESP32 ADC Calibration — Espressif Docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc_calibration.html)
- [HiveMQ Cloud — Free MQTT Broker](https://www.hivemq.com/mqtt-cloud-broker/)
- [PlatformIO + ESP32 + MQTT](https://platformio.org/)
---
 
 
