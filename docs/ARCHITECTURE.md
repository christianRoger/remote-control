# System Architecture — Tech3D Remote Control

## 1. Overview

The Tech3D Remote Control is a dedicated embedded interface node designed to operate together with the **ESP32 Smart Irrigation** system.

The architecture separates the irrigation control logic from the user interface.

The main controller is responsible for sensors, pumps, irrigation logic, SMART functions, weather data and system services.

The Remote Control provides a dedicated local interface for monitoring and interacting with selected system functions.

Communication between the two devices is handled through **ESP-NOW**.

---

## 2. High-Level Architecture

```text
                    ESP32 SMART IRRIGATION
                         MAIN CONTROLLER
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
    Sensors                 Pumps              System Logic
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              │
                              │
                         ESP-NOW LINK
                              │
                              ▼
                    TECH3D REMOTE CONTROL
                         ESP32 NODE
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
          TFT 320×240      XPT2046          UI Logic
           Display          Touch
```

The Remote Control does not replace the main irrigation controller.

It acts as a dedicated interface node within the distributed embedded system.

---

## 3. Main Controller

The **ESP32 Smart Irrigation** controller is responsible for the core functions of the irrigation system.

Its responsibilities include:

* soil monitoring;
* water and well monitoring;
* pump control;
* irrigation scheduling;
* SMART irrigation logic;
* weather information;
* Wi-Fi connectivity;
* web interface;
* Telegram integration;
* system state management.

The main controller remains the authoritative source for irrigation-related system information.

---

## 4. Remote Control Node

The Remote Control is based on a separate ESP32.

Its primary responsibilities are:

* receiving system information;
* displaying system status;
* processing touch input;
* handling screen navigation;
* sending selected user commands;
* displaying communication status;
* providing local interface configuration.

The device is designed around a **320×240 TFT display with an XPT2046 resistive touch controller**.

---

## 5. Wireless Communication

The communication layer uses **ESP-NOW**.

```text
┌──────────────────────────┐
│ ESP32-S3                 │
│ Smart Irrigation         │
└────────────┬─────────────┘
             │
             │ ESP-NOW
             │
             ▼
┌──────────────────────────┐
│ ESP32                    │
│ Remote Control           │
└──────────────────────────┘
```

ESP-NOW provides a direct wireless communication channel between the embedded devices.

This allows the Remote Control to exchange system information and selected control commands without relying on a traditional TCP/IP connection between the two devices.

---

## 6. Data Flow

The general data flow can be represented as:

```text
Sensors / System Logic
          │
          ▼
   Main Controller
          │
          │ system data
          ▼
       ESP-NOW
          │
          ▼
   Remote Control
          │
          ▼
      TFT Display
```

For user interaction:

```text
Touch Input
     │
     ▼
Remote Control
     │
     │ command
     ▼
  ESP-NOW
     │
     ▼
Main Controller
     │
     ▼
Irrigation System
```

This creates a bidirectional communication architecture.

---

## 7. Interface Architecture

The user interface is divided into seven main screens:

```text
┌─────────────────────────────┐
│ 0  STATO SISTEMA            │
├─────────────────────────────┤
│ 1  POZZO / ACQUA            │
├─────────────────────────────┤
│ 2  PROGRAMMAZIONE           │
├─────────────────────────────┤
│ 3  METEO                    │
├─────────────────────────────┤
│ 4  SMART IA                 │
├─────────────────────────────┤
│ 5  AUDIO E SCHERMO          │
├─────────────────────────────┤
│ 6  RETE WI-FI               │
└─────────────────────────────┘
```

Navigation is performed through touch interaction and horizontal swipe gestures.

Each screen has a defined functional responsibility.

---

## 8. User Interface Layers

The graphical interface follows a common layout:

```text
┌──────────────────────────────────────┐
│ Header / System Information          │
├──────────────────────────────────────┤
│                                      │
│                                      │
│          Screen Content              │
│                                      │
│                                      │
├──────────────────────────────────────┤
│ Communication / Radio Status         │
└──────────────────────────────────────┘
```

### Header

Provides information such as:

* battery level;
* current screen;
* time;
* communication indicators.

### Main Content

Displays the information and controls associated with the selected screen.

### Footer

Provides the radio communication status.

---

## 9. Navigation Architecture

Two different interaction mechanisms are used.

### Swipe Navigation

Horizontal gestures are used to move between screens.

```text
STATO SISTEMA
      │
      ▼
POZZO / ACQUA
      │
      ▼
PROGRAMMAZIONE
      │
      ▼
METEO
      │
      ▼
SMART IA
      │
      ▼
AUDIO E SCHERMO
      │
      ▼
RETE WI-FI
```

The gesture parameters are designed for the physical limitations of the 320×240 touchscreen:

```text
SWIPE_THRESHOLD = 60 px
SWIPE_MAX_MS    = 800 ms
```

### Touch Controls

Touch interaction is handled independently from swipe detection.

This prevents a navigation gesture from unintentionally activating a control.

---

## 10. Distributed Embedded System

The complete system can be considered a distributed embedded architecture:

```text
                  ┌──────────────────────┐
                  │  SMART IRRIGATION    │
                  │                      │
                  │  Sensors             │
                  │  Pumps               │
                  │  Control Logic       │
                  │  SMART               │
                  │  Weather             │
                  │  Network Services    │
                  └──────────┬───────────┘
                             │
                             │ ESP-NOW
                             │
                  ┌──────────▼───────────┐
                  │   REMOTE CONTROL     │
                  │                      │
                  │   ESP32              │
                  │   Communication      │
                  │   Touch Processing   │
                  │   UI Logic           │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    TFT + XPT2046     │
                  │                      │
                  │    User Interface    │
                  └──────────────────────┘
```

The separation of responsibilities allows the system to scale without placing the complete user interface and irrigation control logic on the same device.

---

## 11. Design Principles

The architecture was developed around several principles:

### Separation of Responsibilities

The irrigation controller manages the physical system while the Remote Control manages the local user interface.

### Wireless Communication

ESP-NOW provides a lightweight communication channel between the embedded nodes.

### Local Interface

The Remote Control remains usable as a dedicated physical control panel without requiring a computer or smartphone.

### Visual Feedback

System state and communication status are continuously presented through the TFT interface.

### Modular Architecture

The Remote Control is maintained as a separate project from the main irrigation firmware.

---

## 12. Repository Relationship

The two projects are maintained independently:

```text
┌───────────────────────────────┐
│ esp32-smart-irrigation        │
│                               │
│ Main irrigation controller    │
└───────────────┬───────────────┘
                │
                │ ESP-NOW
                │
                ▼
┌───────────────────────────────┐
│ remote-control                │
│                               │
│ Wireless user interface       │
└───────────────────────────────┘
```

This repository documents the Remote Control as an independent embedded project while preserving its relationship with the main Smart Irrigation system.

---

## 13. Technical Summary

| Layer                  | Technology                      |
| ---------------------- | ------------------------------- |
| Microcontroller        | ESP32                           |
| Main System            | ESP32-S3                        |
| Wireless Communication | ESP-NOW                         |
| Display                | TFT 320×240                     |
| Touch                  | XPT2046                         |
| Firmware               | C++ / Arduino                   |
| Interface              | Embedded GUI                    |
| Navigation             | Touch + Swipe                   |
| Architecture           | Distributed Embedded System     |
| Project Type           | Embedded / IoT / Remote Control |

