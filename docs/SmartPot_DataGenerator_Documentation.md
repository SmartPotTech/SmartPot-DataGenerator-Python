<!-- portada
eyebrow: Component Technical Documentation
titulo: SmartPot-DataGenerator-Python (Legacy Stack A)
acento: DataGenerator
subtitulo: Legacy Crop Telemetry Simulator
bajada: Virtual hydroponic crop simulations producing MQTT v1 telemetry: physical thermal and moisture models, diurnal cycles, environmental perturbations, actuator feedback, and internal control APIs.
documento: SmartPot-DataGenerator-Python Documentation
version: 1.1 · October 2026
equipo: SmartPotTech
proyecto: smartpot.app
-->

# SmartPot-DataGenerator-Python (Legacy Stack A)

> [!WARNING]
> **Componente Legado (Stack A)**: Este simulador monolítico en Python ha sido reemplazado en producción por el simulador de alto rendimiento en Rust ([`SmartPot-DataGenerator`](https://github.com/SmartPotTech/SmartPot-DataGenerator)). Este documento describe la implementación original en Python como registro histórico.

## Document Summary

| Field                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Proyecto                       | SmartPot · [smartpot.app](https://smartpot.app)                                                                                                                                                                                                                                                                                                                                                                                                         |
| Componente                     | [SmartPot-DataGenerator](https://github.com/SmartPotTech/SmartPot-DataGenerator)                                                                                                                                                                                                                                                                                                                                                                        |
| Versión                        | 1.1 · octubre 2026                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Alcance                        | Cultivos virtuales, cultivos fijos para demo y QA, modelo físico, API de control, configuración, pruebas y operación                                                                                                                                                                                                                                                                                                                                    |
| Platform Documentation | [Technical Documentation](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Technical_Documentation.md), [Project Journey](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Project_Journey.md), [Software Lifecycle](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Software_Lifecycle.md), and [Global Diagrams](https://github.com/SmartPotTech/.github/blob/main/docs/README.md#global-diagrams) |
| Maintenance                  | Generated from `docs/` in this repository via `.github/docs/tools`; updated with every component change                                                                                                                                                                                                                                                                                                                   |

<!-- parte: PART I | Component Overview -->

## 1. Purpose

### In Plain Terms

El simulador da vida a los **cultivos virtuales**: los que una persona crea en SmartPot sin hardware. Corre siempre,
junto a la plataforma, y hace lo mismo que un ESP32 con el firmware: publica lecturas por MQTT con la cuenta del
cultivo, obedece las órdenes y las confirma. Solo la API le habla, por la red interna; la persona nunca ve la clave.

| Caso                               | Quién lo pide                                                               | Qué es para la plataforma                                                           |
|------------------------------------|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| Cultivo virtual                    | La API, al crear un cultivo `VIRTUAL` o al cambiar o reanudar su simulación | Un cultivo virtual: sus lecturas no entran al aprendizaje                           |
| Cultivo fijo (`SIMULATOR_DEVICES`) | La configuración del contenedor (demo y QA)                                 | Un cultivo real que el simulador hace publicar con su clave, como lo haría un ESP32 |

Simular en Wokwi es otra cosa: Wokwi ejecuta el firmware real
de [SmartPot-IoT](https://github.com/SmartPotTech/SmartPot-IoT) en un ESP32 del navegador y cuenta como cultivo real.

## 2. Component Architecture

<!-- diagrama: SmartPot_DataGenerator_Global_Component | titulo=SmartPot-DataGenerator Component Architecture | lamina=H -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}, "layout": "elk", "elk": {"nodePlacementStrategy": "BRANDES_KOEPF", "mergeEdges": false, "cycleBreakingStrategy": "GREEDY"}}}%%
flowchart LR
  api["SmartPot-API-Java"]
  meteo["Open-Meteo API"]
  broker["SmartPot-Broker-Mosquito"]
  subgraph sim["SmartPot-DataGenerator-Python · Python 3.13"]
    direction TB
    main["__main__.py<br/>Async startup & worker threads"]
    control["api.py · FastAPI :8081<br/>REST management"]
    manager["pots.py · PotManager<br/>Virtual pot registry"]
    env["environment.py<br/>Physics & noise model"]
    weather["weather.py<br/>Weather cache & geocoding"]
    device["device.py · SimulatedDevice<br/>MQTT client"]
  end
  api -->|"REST control"| control
  main --> control & manager
  control --> manager
  manager --> env & device
  env --> weather
  weather -->|"HTTPS"| meteo
  device <-->|"MQTT"| broker
  classDef core fill:#067A52,stroke:#0B3D2B,color:#FFFFFF
  classDef leaf fill:#DDF5EA,stroke:#067A52,color:#17261F
  classDef water fill:#E3F2FB,stroke:#1F6FA0,color:#17261F
  class api,broker core
  class meteo water
  class sim,main,control,manager,env,weather,device leaf
```

<!-- parte: PART II | Biological Simulation Engine -->

## 3. Virtual Crop Lifecycle & Telemetry

<!-- diagrama: SmartPot_DataGenerator_01_Virtual_Crop_Lifecycle | titulo=Vida de un cultivo virtual en el simulador -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
sequenceDiagram
  autonumber
  participant A as SmartPot-API
  participant C as api.py (FastAPI :8081)
  participant M as PotManager
  participant D as SimulatedDevice
  participant B as Mosquitto Broker
  A->>C: PUT /v1/pots/{cropId} (key, species, mode, location, interval)
  C->>M: Create or update virtual pot
  M->>D: Connect MQTT client with crop credentials
  D->>B: CONNECT with LWT offline status
  D->>B: Publish status online (retained)
  loop Every sampling interval
    M->>M: Step environment model & add sensor noise
    D->>B: Publish telemetry JSON
  end
  B->>D: Dispatch command {id, actuator, action, durationSeconds}
  D->>M: Apply actuator effect to environment
  D->>B: Publish commands/ack EXECUTED
  A->>C: GET /v1/pots/{cropId}
```

La API decide cuándo existe una simulación: la crea con el cultivo, la retira al pausarla o al borrar el cultivo y cada
minuto reconcilia lo que el simulador tiene con lo que dice `virtual_devices`. Así, tras un reinicio del simulador, los
cultivos virtuales activos vuelven en menos de un minuto.

## 4. Modelo físico

<!-- diagrama: SmartPot_DataGenerator_02_Environment_Model | titulo=Cómo se calcula cada lectura | lamina=H -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
flowchart LR
  subgraph target["Target Environment by Mode"]
    direction TB
    auto["AUTO<br/>species baseline curve with day/night cycle"]
    manual["MANUAL<br/>user defined sliders"]
    weather["WEATHER<br/>ambient temp, humidity, solar radiation"]
  end
  subgraph actuators["Actuator Responses"]
    direction TB
    pump["Water Pump: +1.2% moisture/sec"]
    fan["Fan: -0.35°C & -1.5% RH/min"]
    uv["UV Lamp: +900 lux & +0.05°C/min"]
    hum["Humidifier: +2.5% RH/min"]
    ph["pH Doser: -0.12 pH/sec"]
    nut["Nutrient Doser: +70 ppm TDS/sec"]
  end
  place["Crop Placement<br/>indoor: buffered weather & no rain<br/>outdoor: direct weather impact"]
  step["Simulation Step<br/>drift toward targets & natural evaporation"]
  noise["Sensor Noise<br/>Gaussian jitter within scale limits"]
  out(["Telemetry Output<br/>temperature · humidity · brightness<br/>ph · tds · soilMoisture · atmosphere"])
  target --> step
  actuators --> step
  place --> step
  step --> noise --> out
  classDef core fill:#067A52,stroke:#0B3D2B,color:#FFFFFF
  classDef leaf fill:#DDF5EA,stroke:#067A52,color:#17261F
  classDef water fill:#E3F2FB,stroke:#1F6FA0,color:#17261F
  class target,actuators leaf
  class place,step,noise water
  class out core
```

| Modo      | Qué refleja                                                                                                                                                                                                                                                                                                |
|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `AUTO`    | Día y noche típicos de la especie alrededor de su línea base, en la hora local (`SIMULATOR_UTC_OFFSET`)                                                                                                                                                                                                    |
| `MANUAL`  | Los medidores que mueve la persona; los actuadores siguen actuando encima                                                                                                                                                                                                                                  |
| `WEATHER` | El clima actual del lugar con [Open-Meteo](https://open-meteo.com), abierto y sin clave: la lluvia moja el sustrato y el sol y el aire seco lo secan más rápido. El lugar lo filtra: bajo techo la temperatura y la humedad se amortiguan y no llueve; la media sombra y la sombra bajan la luz y el calor |

Los actuadores mueven las lecturas en los tres modos. Encendido sin duración, un actuador sigue así hasta que se apaga;
los dosificadores sueltan una dosis de 3 s si no la traen. Tras cada orden ejecutada el cultivo publica una lectura a los
2 s, para que el efecto se vea sin esperar el intervalo. Cada orden responde con un mensaje en español que dice la
duración como se lee («Ventilador encendido por 10 min»); un actuador que no existe responde `FAILED` («El actuador X no
existe en este cultivo»).

<!-- parte: PART III | Operations & Verification -->

## 5. API de control

Interna, con `Authorization: Bearer <SIMULATOR_TOKEN>`; sin token responde 503.

| Método | Ruta                            | Descripción                                                                                              |
|--------|---------------------------------|----------------------------------------------------------------------------------------------------------|
| GET    | `/health`                       | Cultivos simulados y conexiones                                                                          |
| GET    | `/v1/pots`, `/v1/pots/{cropId}` | Estado: modo, lectura, medidores, clima, actuadores encendidos y último comando                          |
| PUT    | `/v1/pots/{cropId}`             | Crea o cambia: `key`, `cropType`, `mode`, `manual`, `location`, `setting`, `exposure`, `intervalSeconds` |
| DELETE | `/v1/pots/{cropId}`             | Retira el cultivo simulado y publica `offline`                                                           |
| GET    | `/v1/places?q=`, `/v1/weather`  | Lugares para el modo clima y clima actual de un punto                                                    |

Los cultivos de `SIMULATOR_DEVICES` no se pueden cambiar por la API.

## 6. Configuration & Environment

| Variable                                             | Por defecto         | Uso                                                       |
|------------------------------------------------------|---------------------|-----------------------------------------------------------|
| `MQTT_HOST`, `MQTT_PORT`, `MQTT_TLS`, `MQTT_CA_FILE` | `localhost`, `1883` | Conexión al broker                                        |
| `MQTT_TOPIC_PREFIX`                                  | `smartpot/v1`       | Prefijo del contrato                                      |
| `SIMULATOR_DEVICES`                                  | —                   | Cultivos fijos `cropId:clave:ESPECIE` separados por comas |
| `SIMULATOR_INTERVAL_SECONDS`                         | `30`                | Segundos entre lecturas de los cultivos fijos             |
| `SIMULATOR_UTC_OFFSET`                               | `-5`                | Hora local para el día y la noche                         |
| `SIMULATOR_TOKEN`, `SIMULATOR_PORT`                  | —, `8081`           | Token y puerto de la API de control                       |

## 7. Automated Testing Suite

`uv run ruff check .` y `uv run pytest`: 34 pruebas sobre el modelo físico, el efecto de los actuadores en los tres
modos, los actuadores sin duración, el lugar y el clima, la lluvia y el sol, la caché del clima, el contrato de tópicos,
los comandos con su ACK y su mensaje, la lectura tras una orden y la API de control.

## 8. Production Operations

| Tarea      | Cómo                                                                                               |
|------------|----------------------------------------------------------------------------------------------------|
| Imagen     | `ghcr.io/smartpottech/smartpot-datagenerator`: usuario `1000`, solo lectura y chequeo de salud     |
| Red        | Red interna para la API y el broker; `public` solo para consultar el clima; sin puertos publicados |
| Despliegue | Cada cambio en `main` pasa por el CI, publica la imagen y pide el despliegue central de `.github`  |
