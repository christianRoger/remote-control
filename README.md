# Technical Documentation — Tech3D Remote Control

## 1. Project Overview

**Tech3D Remote Control** is a wireless remote control panel developed around an ESP32 microcontroller and designed to operate as a remote interface for the **ESP32 Smart Irrigation** system.

The device communicates with the main irrigation controller through **ESP-NOW** and provides a local graphical interface through a **320×240 TFT display with resistive touch**.

The main objective is to provide a dedicated remote interface for monitoring system status and accessing selected irrigation and system functions without requiring direct interaction with the main controller.

The project integrates:

* ESP32 microcontroller
* TFT 320×240 display
* XPT2046 resistive touch interface
* ESP-NOW wireless communication
* Graphical user interface
* Touch navigation
* System status monitoring
* Irrigation monitoring
* Water / well information
* Programming information
* Weather information
* SMART irrigation information
* Wi-Fi network information
* Audio and display configuration

> **Note:** The source code is not included in this repository. The project is published as a **technical portfolio**, documenting the architecture, interface, communication system and technical solutions used during development.

---

# 2. Relationship with the Main Project

The Remote Control is designed as a complementary device to the main **ESP32 Smart Irrigation** controller.

The overall architecture can be represented as:

```text
┌─────────────────────────────────┐
│      ESP32 SMART IRRIGATION     │
│                                 │
│  Sensors                        │
│  Pumps                          │
│  Irrigation Logic               │
│  SMART                          │
│  Weather                        │
│  Web Server                     │
│  Telegram                       │
└───────────────┬─────────────────┘
                │
                │ ESP-NOW
                │
                ▼
┌─────────────────────────────────┐
│       TECH3D REMOTE CONTROL     │
│                                 │
│  TFT 320×240                    │
│  XPT2046 Touch                  │
│  System Status                  │
│  Water / Well                   │
│  Programming                    │
│  Weather                        │
│  SMART                          │
│  Audio / Display                │
│  Wi-Fi                          │
└─────────────────────────────────┘
```

The main controller remains responsible for the irrigation system itself, while the Remote Control provides a dedicated user interface for remote interaction and monitoring.

---

# 3. Hardware

## 3.1 Main Controller

The remote control is based on an ESP32 microcontroller.

Main components:

| Component         | Function                    |
| ----------------- | --------------------------- |
| ESP32             | Main microcontroller        |
| TFT 320×240       | Graphical interface         |
| XPT2046           | Resistive touch controller  |
| ESP-NOW           | Wireless communication      |
| Battery indicator | Power status                |
| Status indicators | Communication/system status |

---

# 4. Display and Touch Interface

The graphical interface uses a **320×240 TFT display** with a resistive **XPT2046 touch controller**.

The display is organized into:

* Header
* Main content area
* Navigation area
* Status information

The header provides system information such as:

* battery level;
* current screen;
* time;
* radio/communication status.

The footer displays the radio status.

Example:

```text
┌──────────────────────────────────────┐
│ BATTERY   SCREEN NAME     HH:MM ●●●  │
├──────────────────────────────────────┤
│                                      │
│                                      │
│            MAIN CONTENT              │
│                                      │
│                                      │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

---

# 5. Navigation

The interface supports two main navigation methods.

## 5.1 Swipe Navigation

Horizontal swipe gestures can be used to change screens.

Main parameters:

```text
SWIPE_THRESHOLD = 60 px
SWIPE_MAX_MS    = 800 ms
```

The navigation follows the following sequence:

```text
STATO SISTEMA
      ↓
POZZO / ACQUA
      ↓
PROGRAMMAZIONE
      ↓
METEO
      ↓
SMART IA
      ↓
AUDIO E SCHERMO
      ↓
RETE WI-FI
```

The interface contains seven main pages.

---

## 5.2 Touch Navigation

The touch interface is also used for direct interaction with controls.

Depending on the screen, touch actions can be used to:

* select a function;
* activate or change a parameter;
* control irrigation functions;
* change volume;
* change display brightness;
* switch SMART mode;
* navigate through interface elements.

Swipe navigation and touch actions are handled separately so that a swipe gesture does not unintentionally trigger a button action.

---

# 6. Screen 0 — STATO SISTEMA

The main screen provides an overview of the irrigation system.

Displayed information includes:

* battery level;
* system status;
* current time;
* Pump 1 status;
* Pump 2 status;
* well status;
* ambient temperature;
* ambient humidity;
* weather condition;
* soil status;
* radio communication status.

The pump cards provide visual feedback when irrigation is active.

The interface may display an animated gear/flow indication while a pump is operating.

### Main interaction areas

```text
Pump 1 card     → Pump 1 information/action
Pump 2 card     → Pump 2 information/action
Well badge      → Well information/action
```

---

# 7. Screen 1 — POZZO / ACQUA

This screen is dedicated to water and well information.

The interface displays:

* well status;
* reservoir level;
* water availability;
* reservoir information;
* well pump status.

The reservoir level is represented visually together with the available percentage.

Example structure:

```text
┌──────────────────────────────────────┐
│          POZZO / ACQUA               │
├──────────┬───────────────────────────┤
│          │ LIVELLO SERBATOIO         │
│  WATER   │                           │
│  LEVEL   │ 73%                       │
│          │ ACQUA DISPONIBILE         │
│          │                           │
│          │ POMPA POZZO: ON           │
├──────────┴───────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

The well pump control can be accessed through the dedicated touch area when the corresponding operating mode permits it.

---

# 8. Screen 2 — PROGRAMMAZIONE

The programming screen provides access to the irrigation schedule.

The interface allows selection between:

```text
[POMPA 1]     [POMPA 2]
```

Each pump has configurable:

* start time;
* end time;
* active/inactive state.

Example:

```text
┌──────────────────────────────────────┐
│          PROGRAMMAZIONE              │
├──────────────┬───────────────────────┤
│   POMPA 1    │       POMPA 2         │
├──────────────────────────────────────┤
│ INIZIO       [ - ] 08:00 [ + ]       │
│                                      │
│ FINE         [ - ] 09:00 [ + ]       │
├──────────────────────────────────────┤
│             [ SALVARE ]              │
└──────────────────────────────────────┘
```

Touch controls include:

* Pump 1 tab;
* Pump 2 tab;
* start time adjustment;
* end time adjustment;
* save operation.

---

# 9. Screen 3 — METEO

The weather screen displays meteorological information received from the irrigation system.

The interface can display:

* current weather condition;
* weather description;
* temperature;
* humidity;
* weather information received from the configured service.

Example:

```text
┌──────────────────────────────────────┐
│               METEO                  │
├──────────────────────────────────────┤
│                                      │
│        METEO — Tempo sereno          │
│                                      │
│      TEMP.              UMIDITA      │
│       24°C                55%        │
│                                      │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

This screen is primarily informational.

---

# 10. Screen 4 — SMART IA

The SMART IA screen provides information about the adaptive irrigation system.

The interface displays:

* SMART status;
* activation state;
* irrigation efficiency;
* score information;
* adaptive system status.

Example:

```text
┌──────────────────────────────────────┐
│              SMART IA                │
├──────────────────────────────────────┤
│                                      │
│          STATO SISTEMA IA            │
│                                      │
│             [ATTIVATO]               │
│                                      │
│  Efficienza P1: 2.5%                 │
│  Efficienza P2: 2.5%                 │
│  SCORE (P1/P2): 12/8                 │
│                                      │
│     Touch → SMART ON/OFF             │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

The SMART mode is based on an adaptive/heuristic irrigation strategy implemented by the main irrigation controller.

The Remote Control displays the status and information provided by the main system.

---

# 11. Screen 5 — AUDIO E SCHERMO

This screen provides local interface configuration.

Two main parameters are available:

### Volume

```text
[ - ]      100%      [ + ]
```

### Display brightness

```text
[ - ]      100%      [ + ]
```

Touch controls allow the user to increase or decrease the corresponding values.

This screen is dedicated to the local user interface and does not represent irrigation logic.

---

# 12. Screen 6 — RETE WI-FI

The Wi-Fi screen provides information about the network connection.

Displayed information includes:

* network SSID;
* signal strength;
* IP address;
* MAC address;
* network status.

Example:

```text
┌──────────────────────────────────────┐
│              RETE WI-FI              │
├──────────────────────────────────────┤
│                                      │
│ STATO RETE PRINCIPALE                │
│                                      │
│ SSID Rete: Casa_IoT                  │
│ Segnale:   ▮▮▮▮                   │
│ Indirizzo IP: 192.168.1.50           │
│ MAC Address: AABBCC...               │
│                                      │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

This screen is informational and allows the user to verify the network state of the remote controller.

---

# 13. ESP-NOW Communication

Communication between the Remote Control and the main irrigation controller is based on **ESP-NOW**.

The communication architecture is:

```text
Main ESP32-S3
     │
     │ ESP-NOW
     ▼
Remote ESP32
     │
     ▼
TFT + Touch Interface
```

ESP-NOW provides a direct wireless communication mechanism between ESP devices without requiring the Remote Control to communicate with the main controller through a traditional TCP/IP connection.

The Remote Control acts as a dedicated interface node within the overall irrigation system.

---

# 14. System Information

The interface provides visual information regarding the communication state.

The footer contains a radio status indicator:

```text
RADIO: ONLINE
```

The header also provides communication/status indicators.

This allows the user to quickly determine whether the remote controller is communicating with the system.

---

# 15. User Interface Design

The interface was designed around a compact 320×240 display.

The design prioritizes:

* clear information hierarchy;
* large touch areas;
* simple navigation;
* status visibility;
* rapid access to frequently used functions;
* visual feedback.

The use of a resistive touch display allows the device to operate as a dedicated embedded control panel without requiring external input devices.

---

# 16. Technical Architecture

The Remote Control can be considered a distributed embedded interface node.

```text
                     MAIN SYSTEM
┌──────────────────────────────────────────┐
│ ESP32-S3 Smart Irrigation                │
│                                          │
│ Sensors                                  │
│ Irrigation Control                       │
│ Pumps                                    │
│ SMART Logic                              │
│ Weather                                  │
│ Web Server                               │
│ Telegram                                 │
└─────────────────────┬────────────────────┘
                      │
                      │ ESP-NOW
                      │
                      ▼
┌──────────────────────────────────────────┐
│ Tech3D Remote Control                    │
│                                          │
│ ESP32                                    │
│ Communication                            │
│ UI Logic                                 │
│ Touch Processing                         │
│                                          │
│ ┌──────────────────────────────────────┐ │
│ │ TFT 320×240                          │ │
│ │                                      │ │
│ │ System Status                        │ │
│ │ Water / Well                         │ │
│ │ Programming                          │ │
│ │ Weather                              │ │
│ │ SMART                                │ │
│ │ Audio / Display                      │ │
│ │ Wi-Fi                                │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

---

# 17. Development Approach

The Remote Control was developed as a dedicated embedded interface for the irrigation system.

The development process involved:

```text
ESP32 communication
        ↓
ESP-NOW integration
        ↓
Display integration
        ↓
Touch interface
        ↓
Screen development
        ↓
Navigation
        ↓
System information
        ↓
Remote interaction
```

The interface was developed around the constraints of an embedded 320×240 display, requiring careful organization of information and touch interaction areas.

---

# 18. Technical Skills Demonstrated

This project demonstrates practical experience in:

### Embedded Systems

* ESP32 development;
* C/C++;
* microcontroller programming;
* embedded user interfaces;
* GPIO and peripheral integration.

### Wireless Communication

* ESP-NOW;
* ESP32-to-ESP32 communication;
* distributed embedded systems;
* wireless status monitoring.

### Human-Machine Interface

* TFT displays;
* resistive touch;
* graphical interfaces;
* touch navigation;
* swipe gestures;
* visual feedback.

### System Integration

* integration with the ESP32 Smart Irrigation platform;
* remote monitoring;
* communication between embedded nodes;
* interface design for embedded systems.

---

# 19. Project Structure

The repository is organized as a technical portfolio:

```text
remote-control/
│
├── README.md
│
├── docs/
│   ├── DOCUMENTATION.md
│   ├── ARCHITECTURE.md
│   └── architecture.svg
│
└── images/
    ├── remote-00.jpeg
    ├── remote-01.jpeg
    ├── remote-02.jpeg
    ├── remote-03.jpeg
    ├── remote-04.jpeg
    ├── remote-05.jpeg
    ├── remote-06.jpeg
    └── remote-07.jpeg
```

The source firmware is not included in the public repository.

---

# 20. Project Status

**Platform:** ESP32
**Display:** TFT 320×240
**Touch Controller:** XPT2046
**Communication:** ESP-NOW
**Firmware:** C++ / Arduino
**Interface:** Embedded TFT Touch UI
**Architecture:** Distributed Embedded System
**Project Type:** Embedded / IoT / Remote Control

---

# 21. Relationship with ESP32 Smart Irrigation

The Remote Control is part of the broader **ESP32 Smart Irrigation** ecosystem.

The two projects are maintained as separate repositories to clearly document their individual technical responsibilities.

```text
ESP32 Smart Irrigation
        │
        │ ESP-NOW
        ▼
Tech3D Remote Control
```

The separation allows the Remote Control to be presented as an independent embedded project while maintaining a clear technical relationship with the main irrigation controller.
