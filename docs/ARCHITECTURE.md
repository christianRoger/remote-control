# Architettura del Sistema — Tech3D Remote Control

## 1. Panoramica

Il **Tech3D Remote Control** è un nodo di interfaccia embedded dedicato, progettato per funzionare insieme al sistema **ESP32 Smart Irrigation**.

L'architettura separa la logica di controllo dell'irrigazione dall'interfaccia utente.

Il controller principale è responsabile dei sensori, delle pompe, della logica di irrigazione, delle funzioni SMART, dei dati meteorologici e dei servizi del sistema.

Il Remote Control fornisce un'interfaccia locale dedicata per il monitoraggio e l'interazione con alcune funzioni del sistema.

La comunicazione tra i due dispositivi avviene tramite **ESP-NOW**.

---

## 2. Architettura ad Alto Livello

```text
                    ESP32 SMART IRRIGATION
                       CONTROLLER PRINCIPALE
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
    Sensori                 Pompe              Logica Sistema
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              │
                              │
                       COLLEGAMENTO ESP-NOW
                              │
                              ▼
                    TECH3D REMOTE CONTROL
                         NODO ESP32
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
          TFT 320×240      XPT2046          Logica UI
           Display          Touch
```

Il Remote Control non sostituisce il controller principale dell'impianto di irrigazione.

Funziona come nodo di interfaccia dedicato all'interno del sistema embedded distribuito.

---

## 3. Controller Principale

Il controller **ESP32 Smart Irrigation** è responsabile delle funzioni principali dell'impianto di irrigazione.

Le sue responsabilità includono:

* monitoraggio dell'umidità del terreno;
* monitoraggio dell'acqua e del pozzo;
* controllo delle pompe;
* programmazione dell'irrigazione;
* logica di irrigazione SMART;
* informazioni meteorologiche;
* connettività Wi-Fi;
* interfaccia web;
* integrazione Telegram;
* gestione dello stato del sistema.

Il controller principale rimane la fonte principale delle informazioni relative allo stato dell'impianto di irrigazione.

---

## 4. Nodo Remote Control

Il Remote Control è basato su un ESP32 separato.

Le sue principali responsabilità sono:

* ricevere le informazioni dal sistema;
* visualizzare lo stato dell'impianto;
* elaborare gli input touch;
* gestire la navigazione tra le schermate;
* inviare i comandi utente selezionati;
* visualizzare lo stato della comunicazione;
* fornire la configurazione dell'interfaccia locale.

Il dispositivo è progettato attorno a un **display TFT 320×240 con controller touch resistivo XPT2046**.

---

## 5. Comunicazione Wireless

Il livello di comunicazione utilizza **ESP-NOW**.

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

ESP-NOW fornisce un canale di comunicazione wireless diretto tra i dispositivi embedded.

Questo permette al Remote Control di scambiare informazioni sul sistema e determinati comandi di controllo senza dipendere da una connessione TCP/IP tradizionale tra i due dispositivi.

---

## 6. Flusso dei Dati

Il flusso generale dei dati può essere rappresentato come segue:

```text
Sensori / Logica Sistema
          │
          ▼
   Controller Principale
          │
          │ dati sistema
          ▼
       ESP-NOW
          │
          ▼
   Remote Control
          │
          ▼
      Display TFT
```

Per l'interazione dell'utente:

```text
Input Touch
     │
     ▼
Remote Control
     │
     │ comando
     ▼
  ESP-NOW
     │
     ▼
Controller Principale
     │
     ▼
Impianto di Irrigazione
```

Questa struttura crea un'architettura di comunicazione bidirezionale.

---

## 7. Architettura dell'Interfaccia

L'interfaccia utente è suddivisa in **sette schermate principali**:

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

La navigazione avviene tramite interazione touch e gesture swipe orizzontali.

Ogni schermata possiede una specifica responsabilità funzionale.

---

## 8. Livelli dell'Interfaccia Utente

L'interfaccia grafica utilizza una struttura comune:

```text
┌──────────────────────────────────────┐
│ Header / Informazioni di Sistema     │
├──────────────────────────────────────┤
│                                      │
│                                      │
│          Contenuto Schermata         │
│                                      │
│                                      │
├──────────────────────────────────────┤
│ Stato Comunicazione / Radio          │
└──────────────────────────────────────┘
```

### Header

Fornisce informazioni come:

* livello della batteria;
* schermata corrente;
* ora;
* indicatori di comunicazione.

### Contenuto Principale

Visualizza le informazioni e i controlli associati alla schermata selezionata.

### Footer

Visualizza lo stato della comunicazione radio.

---

## 9. Architettura della Navigazione

Vengono utilizzati due differenti meccanismi di interazione.

### Navigazione tramite Swipe

Le gesture orizzontali vengono utilizzate per passare da una schermata all'altra.

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

I parametri della gesture sono stati definiti considerando le limitazioni fisiche del touchscreen 320×240:

```text
SWIPE_THRESHOLD = 60 px
SWIPE_MAX_MS    = 800 ms
```

### Controlli Touch

L'interazione touch viene gestita separatamente dal rilevamento dello swipe.

Questo impedisce che una gesture di navigazione possa attivare accidentalmente un controllo.

---

## 10. Sistema Embedded Distribuito

Il sistema completo può essere considerato un'architettura embedded distribuita:

```text
                  ┌──────────────────────┐
                  │  SMART IRRIGATION    │
                  │                      │
                  │  Sensori             │
                  │  Pompe               │
                  │  Logica di controllo │
                  │  SMART               │
                  │  Meteo               │
                  │  Servizi di rete     │
                  └──────────┬───────────┘
                             │
                             │ ESP-NOW
                             │
                  ┌──────────▼───────────┐
                  │   REMOTE CONTROL     │
                  │                      │
                  │   ESP32              │
                  │   Comunicazione      │
                  │   Gestione Touch     │
                  │   Logica UI          │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    TFT + XPT2046     │
                  │                      │
                  │    Interfaccia Utente│
                  └──────────────────────┘
```

La separazione delle responsabilità permette al sistema di essere sviluppato in modo modulare, evitando di concentrare la gestione completa dell'interfaccia utente e della logica di irrigazione sullo stesso dispositivo.

---

## 11. Principi di Progettazione

L'architettura è stata sviluppata secondo alcuni principi fondamentali.

### Separazione delle Responsabilità

Il controller dell'irrigazione gestisce il sistema fisico, mentre il Remote Control gestisce l'interfaccia utente locale.

### Comunicazione Wireless

ESP-NOW fornisce un canale di comunicazione leggero e diretto tra i nodi embedded.

### Interfaccia Locale

Il Remote Control può essere utilizzato come pannello di controllo fisico dedicato senza richiedere un computer o uno smartphone.

### Feedback Visivo

Lo stato del sistema e della comunicazione viene visualizzato attraverso l'interfaccia TFT.

### Architettura Modulare

Il Remote Control viene mantenuto come progetto separato rispetto al firmware principale del sistema di irrigazione.

---

## 12. Relazione tra i Repository

I due progetti vengono mantenuti separatamente:

```text
┌───────────────────────────────┐
│ esp32-smart-irrigation        │
│                               │
│ Controller principale         │
│ dell'impianto di irrigazione  │
└───────────────┬───────────────┘
                │
                │ ESP-NOW
                │
                ▼
┌───────────────────────────────┐
│ remote-control                │
│                               │
│ Interfaccia utente wireless   │
└───────────────────────────────┘
```

Questo repository documenta il Remote Control come progetto embedded indipendente, mantenendo allo stesso tempo la relazione tecnica con il sistema principale Smart Irrigation.

---

## 13. Riepilogo Tecnico

| Livello                | Tecnologia                      |
| ---------------------- | ------------------------------- |
| Microcontrollore       | ESP32                           |
| Sistema Principale     | ESP32-S3                        |
| Comunicazione Wireless | ESP-NOW                         |
| Display                | TFT 320×240                     |
| Touch                  | XPT2046                         |
| Firmware               | C++ / Arduino                   |
| Interfaccia            | GUI Embedded                    |
| Navigazione            | Touch + Swipe                   |
| Architettura           | Sistema Embedded Distribuito    |
| Tipologia di Progetto  | Embedded / IoT / Remote Control |

---

## 14. Nota sul Repository

Il presente repository è stato realizzato come **portfolio tecnico**.

Il firmware completo del dispositivo non viene pubblicato. La documentazione si concentra sull'architettura del sistema, sull'interfaccia utente, sulla comunicazione wireless e sulle soluzioni tecniche adottate durante lo sviluppo.

L'obiettivo è mostrare l'approccio progettuale e le competenze tecniche applicate nella realizzazione di un'interfaccia embedded wireless integrata con un sistema IoT reale.
