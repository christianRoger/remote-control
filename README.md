# Documentazione Tecnica — Tech3D Remote Control

## 1. Panoramica del Progetto

**Tech3D Remote Control** è un pannello di controllo remoto wireless sviluppato con un microcontrollore ESP32 e progettato per funzionare come interfaccia remota per il sistema **ESP32 Smart Irrigation**.

Il dispositivo comunica con il controller principale dell'impianto di irrigazione tramite **ESP-NOW** e dispone di un'interfaccia grafica locale basata su un **display TFT 320×240 con touch resistivo**.

L'obiettivo principale è fornire un'interfaccia remota dedicata per il monitoraggio dello stato del sistema e l'accesso a funzioni selezionate dell'impianto di irrigazione, senza dover interagire direttamente con il controller principale.

Il progetto integra:

* microcontrollore ESP32
* display TFT 320×240
* interfaccia touch resistiva XPT2046
* comunicazione wireless ESP-NOW
* interfaccia grafica
* navigazione touch
* monitoraggio dello stato del sistema
* monitoraggio dell'irrigazione
* informazioni su acqua / pozzo
* informazioni sulla programmazione
* informazioni meteorologiche
* informazioni sul sistema SMART
* informazioni sulla rete Wi-Fi
* configurazione audio e display

> **Nota:** Il codice sorgente non è incluso in questo repository. Il progetto è pubblicato come **portfolio tecnico**, con l'obiettivo di documentare l'architettura, l'interfaccia, il sistema di comunicazione e le soluzioni tecniche utilizzate durante lo sviluppo.

---

# 2. Relazione con il Progetto Principale

Il Remote Control è progettato come dispositivo complementare al controller principale **ESP32 Smart Irrigation**.

L'architettura generale può essere rappresentata come segue:

```text
┌─────────────────────────────────┐
│      ESP32 SMART IRRIGATION     │
│                                 │
│  Sensori                        │
│  Pompe                          │
│  Logica di irrigazione          │
│  SMART                          │
│  Meteo                          │
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
│  Touch XPT2046                  │
│  Stato del sistema              │
│  Acqua / Pozzo                  │
│  Programmazione                 │
│  Meteo                          │
│  SMART                          │
│  Audio / Display                │
│  Wi-Fi                          │
└─────────────────────────────────┘
```

Il controller principale rimane responsabile della gestione dell'impianto di irrigazione, mentre il Remote Control fornisce un'interfaccia dedicata per il monitoraggio e l'interazione remota.

---

# 3. Hardware

## 3.1 Controller Principale

Il Remote Control è basato su un microcontrollore ESP32.

Componenti principali:

| Componente          | Funzione                    |
| ------------------- | --------------------------- |
| ESP32               | Microcontrollore principale |
| TFT 320×240         | Interfaccia grafica         |
| XPT2046             | Controller touch resistivo  |
| ESP-NOW             | Comunicazione wireless      |
| Indicatore batteria | Stato dell'alimentazione    |
| Indicatori di stato | Stato comunicazione/sistema |

---

# 4. Display e Interfaccia Touch

L'interfaccia grafica utilizza un **display TFT 320×240** con controller touch resistivo **XPT2046**.

Il display è organizzato in:

* Header
* Area principale dei contenuti
* Area di navigazione
* Informazioni di stato

L'header fornisce informazioni di sistema come:

* livello della batteria;
* schermata corrente;
* ora;
* stato della comunicazione radio.

Il footer visualizza lo stato della comunicazione radio.

Esempio:

```text
┌──────────────────────────────────────┐
│ BATTERIA  SCHERMATA     HH:MM ●●●    │
├──────────────────────────────────────┤
│                                      │
│                                      │
│          CONTENUTO PRINCIPALE        │
│                                      │
│                                      │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

---

# 5. Navigazione

L'interfaccia supporta due principali modalità di navigazione.

## 5.1 Navigazione tramite Swipe

Le gesture swipe orizzontali possono essere utilizzate per cambiare schermata.

Parametri principali:

```text
SWIPE_THRESHOLD = 60 px
SWIPE_MAX_MS    = 800 ms
```

La navigazione segue la seguente sequenza:

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

L'interfaccia è composta da sette schermate principali.

---

## 5.2 Navigazione tramite Touch

L'interfaccia touch viene utilizzata anche per l'interazione diretta con i controlli.

A seconda della schermata, il touch può essere utilizzato per:

* selezionare una funzione;
* attivare o modificare un parametro;
* controllare funzioni di irrigazione;
* modificare il volume;
* modificare la luminosità del display;
* attivare o disattivare la modalità SMART;
* navigare tra gli elementi dell'interfaccia.

La navigazione tramite swipe e le azioni touch vengono gestite separatamente, in modo da evitare che una gesture di navigazione attivi accidentalmente un pulsante.

---

# 6. Schermata 0 — STATO SISTEMA

La schermata principale fornisce una panoramica dello stato dell'impianto di irrigazione.

Le informazioni visualizzate includono:

* livello della batteria;
* stato del sistema;
* ora corrente;
* stato Pompa 1;
* stato Pompa 2;
* stato del pozzo;
* temperatura ambiente;
* umidità ambientale;
* condizioni meteorologiche;
* stato del terreno;
* stato della comunicazione radio.

Le schede delle pompe forniscono un feedback visivo quando l'irrigazione è attiva.

L'interfaccia può visualizzare un'indicazione animata della pompa/flusso durante il funzionamento.

### Principali aree di interazione

```text
Scheda Pompa 1 → Informazioni/azione Pompa 1
Scheda Pompa 2 → Informazioni/azione Pompa 2
Indicatore Pozzo → Informazioni/azione Pozzo
```

---

# 7. Schermata 1 — POZZO / ACQUA

Questa schermata è dedicata alle informazioni relative all'acqua e al pozzo.

L'interfaccia visualizza:

* stato del pozzo;
* livello del serbatoio;
* disponibilità dell'acqua;
* informazioni sul serbatoio;
* stato della pompa del pozzo.

Il livello del serbatoio viene rappresentato graficamente insieme alla percentuale di acqua disponibile.

Esempio:

```text
┌──────────────────────────────────────┐
│          POZZO / ACQUA               │
├──────────┬───────────────────────────┤
│          │ LIVELLO SERBATOIO         │
│  ACQUA   │                           │
│          │ 73%                       │
│          │ ACQUA DISPONIBILE         │
│          │                           │
│          │ POMPA POZZO: ON           │
├──────────┴───────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

Il controllo della pompa del pozzo può essere effettuato tramite l'area touch dedicata quando la modalità operativa corrispondente lo consente.

---

# 8. Schermata 2 — PROGRAMMAZIONE

La schermata di programmazione permette di accedere alla configurazione del programma di irrigazione.

L'interfaccia consente di selezionare:

```text
[POMPA 1]     [POMPA 2]
```

Per ciascuna pompa sono configurabili:

* ora di inizio;
* ora di fine;
* stato attivo/disattivo.

Esempio:

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
│             [ SALVA ]                │
└──────────────────────────────────────┘
```

I controlli touch includono:

* scheda Pompa 1;
* scheda Pompa 2;
* regolazione dell'ora di inizio;
* regolazione dell'ora di fine;
* salvataggio della configurazione.

---

# 9. Schermata 3 — METEO

La schermata meteo visualizza le informazioni meteorologiche ricevute dal sistema di irrigazione.

L'interfaccia può visualizzare:

* condizioni meteorologiche attuali;
* descrizione del tempo;
* temperatura;
* umidità;
* informazioni meteorologiche ricevute dal servizio configurato.

Esempio:

```text
┌──────────────────────────────────────┐
│               METEO                  │
├──────────────────────────────────────┤
│                                      │
│        METEO — Tempo sereno          │
│                                      │
│      TEMP.              UMIDITÀ      │
│       24°C                55%        │
│                                      │
├──────────────────────────────────────┤
│ RADIO: ONLINE                        │
└──────────────────────────────────────┘
```

Questa schermata ha principalmente una funzione informativa.

---

# 10. Schermata 4 — SMART IA

La schermata SMART IA fornisce informazioni sul sistema di irrigazione adattivo.

L'interfaccia visualizza:

* stato SMART;
* stato di attivazione;
* efficienza dell'irrigazione;
* informazioni sullo score;
* stato del sistema adattivo.

Esempio:

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

La modalità SMART si basa su una strategia di irrigazione adattiva/euristica implementata nel controller principale.

Il Remote Control visualizza lo stato e le informazioni fornite dal sistema principale.

---

# 11. Schermata 5 — AUDIO E SCHERMO

Questa schermata permette di configurare l'interfaccia locale.

Sono disponibili due parametri principali:

### Volume

```text
[ - ]      100%      [ + ]
```

### Luminosità del display

```text
[ - ]      100%      [ + ]
```

I controlli touch permettono di aumentare o diminuire i valori corrispondenti.

Questa schermata è dedicata esclusivamente all'interfaccia locale e non rappresenta la logica di irrigazione.

---

# 12. Schermata 6 — RETE WI-FI

La schermata Wi-Fi fornisce informazioni relative alla connessione di rete.

Le informazioni visualizzate includono:

* SSID della rete;
* intensità del segnale;
* indirizzo IP;
* indirizzo MAC;
* stato della rete.

Esempio:

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

Questa schermata è informativa e permette di verificare lo stato della rete del Remote Control.

---

# 13. Comunicazione ESP-NOW

La comunicazione tra il Remote Control e il controller principale dell'irrigazione utilizza **ESP-NOW**.

L'architettura della comunicazione è:

```text
ESP32-S3
Smart Irrigation
     │
     │ ESP-NOW
     ▼
ESP32
Remote Control
     │
     ▼
TFT + Touch Interface
```

ESP-NOW fornisce un meccanismo di comunicazione wireless diretto tra dispositivi ESP.

Il Remote Control agisce come nodo di interfaccia dedicato all'interno dell'architettura complessiva del sistema di irrigazione.

---

# 14. Informazioni di Sistema

L'interfaccia fornisce informazioni visive relative allo stato della comunicazione.

Il footer contiene un indicatore dello stato radio:

```text
RADIO: ONLINE
```

Anche l'header fornisce indicatori relativi allo stato e alla comunicazione.

Questo permette all'utente di verificare rapidamente se il Remote Control sta comunicando correttamente con il sistema.

---

# 15. Progettazione dell'Interfaccia Utente

L'interfaccia è stata progettata attorno alle dimensioni compatte del display 320×240.

Il design privilegia:

* gerarchia chiara delle informazioni;
* aree touch di dimensioni adeguate;
* navigazione semplice;
* visibilità dello stato del sistema;
* accesso rapido alle funzioni utilizzate più frequentemente;
* feedback visivo.

L'utilizzo di un display touch resistivo permette al dispositivo di funzionare come pannello di controllo embedded dedicato senza la necessità di dispositivi di input esterni.

---

# 16. Architettura Tecnica

Il Remote Control può essere considerato un nodo di interfaccia embedded all'interno di un sistema distribuito.

```text
                     SISTEMA PRINCIPALE
┌──────────────────────────────────────────┐
│ ESP32-S3 Smart Irrigation                │
│                                          │
│ Sensori                                  │
│ Controllo irrigazione                    │
│ Pompe                                    │
│ Logica SMART                             │
│ Meteo                                    │
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
│ Comunicazione                            │
│ Logica interfaccia                       │
│ Gestione touch                           │
│                                          │
│ ┌──────────────────────────────────────┐ │
│ │ TFT 320×240                          │ │
│ │                                      │ │
│ │ Stato Sistema                        │ │
│ │ Acqua / Pozzo                        │ │
│ │ Programmazione                       │ │
│ │ Meteo                                │ │
│ │ SMART                                │ │
│ │ Audio / Display                      │ │
│ │ Wi-Fi                                │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

---

# 17. Approccio allo Sviluppo

Il Remote Control è stato sviluppato come interfaccia embedded dedicata al sistema di irrigazione.

Il processo di sviluppo ha coinvolto:

```text
Comunicazione ESP32
        ↓
Integrazione ESP-NOW
        ↓
Integrazione del display
        ↓
Interfaccia touch
        ↓
Sviluppo delle schermate
        ↓
Navigazione
        ↓
Informazioni di sistema
        ↓
Interazione remota
```

L'interfaccia è stata sviluppata tenendo conto dei limiti fisici di un display embedded 320×240, richiedendo un'organizzazione precisa delle informazioni e delle aree di interazione touch.

---

# 18. Competenze Tecniche Dimostrate

Questo progetto dimostra esperienza pratica nei seguenti ambiti:

### Sistemi Embedded

* sviluppo ESP32;
* C/C++;
* programmazione di microcontrollori;
* interfacce utente embedded;
* integrazione GPIO e periferiche.

### Comunicazione Wireless

* ESP-NOW;
* comunicazione ESP32-ESP32;
* sistemi embedded distribuiti;
* monitoraggio dello stato wireless.

### Human-Machine Interface

* display TFT;
* touch resistivo;
* interfacce grafiche;
* navigazione touch;
* gesture swipe;
* feedback visivo.

### Integrazione di Sistema

* integrazione con la piattaforma ESP32 Smart Irrigation;
* monitoraggio remoto;
* comunicazione tra nodi embedded;
* progettazione di interfacce per sistemi embedded.

---

# 19. Struttura del Repository

Il repository è organizzato come portfolio tecnico:

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
    └── remote-06.jpeg
```

Il firmware sorgente non è incluso nel repository pubblico.

---

# 20. Stato del Progetto

**Piattaforma:** ESP32
**Display:** TFT 320×240
**Controller Touch:** XPT2046
**Comunicazione:** ESP-NOW
**Firmware:** C++ / Arduino
**Interfaccia:** Embedded TFT Touch UI
**Architettura:** Sistema Embedded Distribuito
**Tipologia:** Embedded / IoT / Remote Control

---

# 21. Relazione con ESP32 Smart Irrigation

Il Remote Control fa parte del più ampio ecosistema **ESP32 Smart Irrigation**.

I due progetti vengono mantenuti come repository separati per documentare in modo chiaro le rispettive responsabilità tecniche.

```text
ESP32 Smart Irrigation
        │
        │ ESP-NOW
        ▼
Tech3D Remote Control
```

La separazione permette di presentare il Remote Control come un progetto embedded indipendente, mantenendo allo stesso tempo una chiara relazione tecnica con il controller principale dell'impianto di irrigazione.
