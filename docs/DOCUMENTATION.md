# Documentazione Tecnica — Tech3D Remote Control

Controllo remoto wireless basato su **ESP32** e comunicazione **ESP-NOW**, progettato come interfaccia remota per l'unità principale del sistema **ESP32 Smart Irrigation**.

Il Remote Control utilizza un display **TFT 240×320 in orientamento landscape**, con area logica di **320×240 pixel**, e un pannello touch resistivo basato su **XPT2046**.

Il dispositivo replica localmente l'interfaccia principale e consente di visualizzare lo stato del sistema e, dove previsto, inviare comandi per pompe, pozzo, modalità SMART IA, programmazione dell'irrigazione e configurazione dell'interfaccia.

> **Nota:** il firmware sorgente non è incluso nel repository pubblico. Questa documentazione descrive l'architettura, il funzionamento, il protocollo di comunicazione e le soluzioni tecniche adottate durante lo sviluppo del progetto.

---

# 1. Panoramica del sistema

| Voce                | Descrizione                                 |
| ------------------- | ------------------------------------------- |
| Microcontrollore    | ESP32-S3                                    |
| Display             | TFT 240×320 fisico, utilizzato in landscape |
| Canvas logico       | 320×240 pixel                               |
| Touch               | Resistivo XPT2046 / TFT_Touch               |
| Comunicazione       | ESP-NOW                                     |
| Modalità ESP-NOW    | Broadcast, senza crittografia               |
| Interfaccia         | GUI embedded touch                          |
| Navigazione         | Touch + swipe orizzontale                   |
| Numero di schermate | 7                                           |
| Firmware            | C++ / Arduino                               |
| Librerie principali | TFT_eSPI, TFT_Touch, WiFi, esp_now          |

Tutte le coordinate grafiche e di interazione descritte in questa documentazione fanno riferimento al sistema logico:

```text
X: 0 – 319
Y: 0 – 239
```

Il display fisico è un **240×320**, ma viene ruotato tramite:

```cpp
setRotation(1);
```

ottenendo una superficie logica di:

```text
320 × 240 pixel
```

---

# 2. Relazione con il sistema ESP32 Smart Irrigation

Il Remote Control è un nodo embedded separato che lavora insieme all'unità principale **ESP32 Smart Irrigation**.

L'unità principale rimane responsabile della gestione fisica del sistema di irrigazione.

Il Remote Control svolge invece il ruolo di interfaccia locale wireless.

```text
┌─────────────────────────────────────┐
│       ESP32 SMART IRRIGATION        │
│          UNITÀ PRINCIPALE           │
│                                     │
│  Sensori                            │
│  Pompe                              │
│  Logica irrigazione                 │
│  SMART IA                           │
│  Meteo                              │
│  Wi-Fi                              │
│  Web Server                         │
│  Telegram                           │
└──────────────────┬──────────────────┘
                   │
                   │ ESP-NOW
                   │
                   ▼
┌─────────────────────────────────────┐
│       TECH3D REMOTE CONTROL         │
│                                     │
│  ESP32-S3                           │
│  TFT 320×240                        │
│  Touch XPT2046                      │
│  Interfaccia grafica                │
│  Comandi remoti                     │
└─────────────────────────────────────┘
```

La separazione permette di mantenere indipendenti:

* la logica di controllo dell'impianto;
* l'interfaccia utente;
* la comunicazione wireless;
* la gestione del display e del touch.

---

# 3. Mappatura dei pin

Il Remote Control utilizza i seguenti GPIO:

| GPIO | Definizione | Funzione                      |
| ---: | ----------- | ----------------------------- |
|   39 | `RTP_DOUT`  | Touch resistivo DOUT          |
|   32 | `RTP_DIN`   | Touch resistivo DIN           |
|   25 | `RTP_SCK`   | Touch resistivo SCK           |
|   33 | `RTP_CS`    | Touch resistivo CS            |
|   36 | `RTP_IRQ`   | Interrupt touch               |
|    4 | `AUDIO_EN`  | Enable amplificatore audio    |
|   26 | `AUDIO_OUT` | Uscita audio / buzzer         |
|   22 | `LED_R`     | LED rosso stato radio         |
|   16 | `LED_G`     | LED verde stato radio         |
|   17 | `LED_B`     | LED blu stato radio           |
|   34 | `PIN_BAT`   | Lettura batteria tramite ADC  |
|   21 | `PIN_BL`    | Backlight display tramite PWM |

### Stato iniziale

Durante l'avvio:

```text
PIN_BL     → luminosità sysBright (255)
LED_R      → HIGH
LED_G      → HIGH
LED_B      → HIGH
AUDIO_EN   → LOW
```

---

# 4. Touchscreen

Il pannello touch è di tipo resistivo.

La calibrazione utilizzata è:

```cpp
setCal(495, 3398, 721, 3448, 320, 240, 1);
```

Il sistema utilizza il touch per:

* selezionare funzioni;
* modificare parametri;
* attivare o disattivare funzioni;
* navigare tra le schermate;
* eseguire swipe orizzontali.

La gestione del touch distingue intenzionalmente tra:

```text
TAP
 │
 └── comando / interazione

SWIPE
 │
 └── navigazione tra schermate
```

Questa separazione evita che un gesto di navigazione possa attivare accidentalmente un comando.

---

# 5. Comunicazione ESP-NOW

La comunicazione tra Remote Control e unità principale utilizza **ESP-NOW**.

```text
┌──────────────────────────────┐
│ ESP32-S3                     │
│ Smart Irrigation             │
└──────────────┬───────────────┘
               │
               │ ESP-NOW
               │
               ▼
┌──────────────────────────────┐
│ ESP32-S3                     │
│ Tech3D Remote Control        │
└──────────────────────────────┘
```

La comunicazione utilizza un peer broadcast e non richiede una connessione TCP/IP tra i due dispositivi.

Configurazione principale:

```text
WiFi Mode : WIFI_STA
ESP-NOW   : Broadcast
Channel   : 0
Encryption: Disabilitata
```

---

# 6. Struttura dei comandi

Il Remote Control invia alla unità principale una struttura `struct_command`.

```cpp
typedef struct struct_command {

    int cmdP1;
    int cmdP2;
    int cmdPozzo;
    int cmdResetWifi;

    int setProgPompa;

    int pStartH;
    int pStartM;

    int pEndH;
    int pEndM;

    int cmdSmartMode;

    int magicNumber;

} struct_command;
```

## Significato dei campi

| Campo          | Funzione                               |
| -------------- | -------------------------------------- |
| `cmdP1`        | Comando Pompa 1                        |
| `cmdP2`        | Comando Pompa 2                        |
| `cmdPozzo`     | Comando pompa pozzo                    |
| `cmdResetWifi` | Richiesta reset Wi-Fi                  |
| `setProgPompa` | Pompa interessata dalla programmazione |
| `pStartH`      | Ora di inizio                          |
| `pStartM`      | Minuto di inizio                       |
| `pEndH`        | Ora di fine                            |
| `pEndM`        | Minuto di fine                         |
| `cmdSmartMode` | Comando modalità SMART IA              |
| `magicNumber`  | Valore di validazione `12345`          |

I comandi che devono essere eseguiti una sola volta vengono riportati a `-1` dopo l'invio.

Questo evita la ripetizione involontaria dello stesso comando.

---

# 7. Struttura della telemetria

L'unità principale invia al Remote Control la struttura `struct_telemetry`.

```cpp
typedef struct struct_telemetry {

    int sens1;
    int sens2;

    int percentAcqua;

    int temp;
    int umid;

    bool statoP1;
    bool statoP2;
    bool statoPozzo;

    char ipStr[16];
    char ssid[32];
    char mac[18];
    char ora[6];

    int prog1_Start;
    int prog1_End;
    int prog2_Start;
    int prog2_End;

    int rssi;

    char meteo[48];

    int pioggia;
    int meteoEnabled;

    int smartMode;

    float efficienzaP1;
    float efficienzaP2;

    int scoreP1;
    int scoreP2;

    int magicNumber;

} struct_telemetry;
```

### Dati ricevuti

| Campo          | Informazione                          |
| -------------- | ------------------------------------- |
| `sens1`        | Umidità sensore S1                    |
| `sens2`        | Umidità sensore S2                    |
| `percentAcqua` | Livello del serbatoio                 |
| `temp`         | Temperatura                           |
| `umid`         | Umidità dell'aria                     |
| `statoP1`      | Stato Pompa 1                         |
| `statoP2`      | Stato Pompa 2                         |
| `statoPozzo`   | Stato pompa pozzo                     |
| `ipStr`        | Indirizzo IP dell'unità principale    |
| `ssid`         | SSID della rete                       |
| `mac`          | MAC address                           |
| `ora`          | Ora del sistema                       |
| `prog1_Start`  | Inizio programmazione P1              |
| `prog1_End`    | Fine programmazione P1                |
| `prog2_Start`  | Inizio programmazione P2              |
| `prog2_End`    | Fine programmazione P2                |
| `rssi`         | RSSI Wi-Fi dell'unità principale      |
| `meteo`        | Descrizione condizioni meteorologiche |
| `pioggia`      | Previsione di pioggia                 |
| `meteoEnabled` | Stato configurazione meteo            |
| `smartMode`    | Stato SMART IA                        |
| `efficienzaP1` | Efficienza P1                         |
| `efficienzaP2` | Efficienza P2                         |
| `scoreP1`      | Score P1                              |
| `scoreP2`      | Score P2                              |
| `magicNumber`  | Valore di validazione `54321`         |

---

# 8. Validazione della telemetria

La telemetria ricevuta viene accettata solamente quando:

```text
len == sizeof(struct_telemetry)
```

e:

```text
magicNumber == 54321
```

Questo permette di evitare l'elaborazione di pacchetti non compatibili o non validi.

> **Regola critica:** `struct_telemetry` nel Remote Control e `struct_telemetry` nell'unità principale devono avere **gli stessi campi, nello stesso ordine e con gli stessi tipi**.

Una modifica della struttura richiede la ricompilazione e il caricamento del firmware su entrambe le unità.

Una differenza nella dimensione della struttura può impedire la corretta ricezione della telemetria e portare allo stato:

```text
RADIO: DISCONNESSO
```

---

# 9. Ricezione e invio

### Inizializzazione

La funzione:

```cpp
initESPNOW()
```

si occupa di:

* configurare `WIFI_STA`;
* inizializzare ESP-NOW;
* registrare i callback;
* configurare il peer broadcast;
* preparare la comunicazione.

### Ricezione

La funzione:

```cpp
OnDataRecv()
```

verifica la dimensione del pacchetto e il `magicNumber`.

Quando la telemetria è valida:

```text
Pacchetto
   ↓
Controllo dimensione
   ↓
Controllo magicNumber
   ↓
Aggiornamento telemetria
   ↓
Aggiornamento lastTeleReceived
```

### Invio

La funzione:

```cpp
enviaComando()
```

trasmette `cmdData`.

Dopo l'invio vengono azzerati i comandi single-fire:

```text
cmdResetWifi  → -1
setProgPompa  → -1
cmdSmartMode  → -1
```

---

# 10. Riconnessione e channel hopping

Se il Remote Control non riceve telemetria per più di **6 secondi**, viene attivata una procedura di ricerca del canale.

```text
Nessuna telemetria
       │
       │ > 6 s
       ▼
Channel Hop
       │
       ▼
Canali 1–13
       │
       ▼
Tentativo di riconnessione
```

Il cambio di canale viene eseguito ogni:

```text
500 ms
```

Questo permette al Remote Control di tentare nuovamente la sincronizzazione con l'unità principale.

---

# 11. Lettura della batteria

La batteria viene letta attraverso:

```cpp
PIN_BAT
```

La conversione utilizza l'intervallo ADC:

```text
2000 → 0%
2800 → 100%
```

Il valore risultante viene utilizzato nell'header dell'interfaccia.

---

# 12. Architettura della User Interface

La GUI è organizzata attraverso alcune variabili globali principali.

| Variabile          | Tipo            | Funzione                      |
| ------------------ | --------------- | ----------------------------- |
| `currentPage`      | `int`           | Schermata attiva, 0–6         |
| `forceRedraw`      | `bool`          | Richiede ridisegno completo   |
| `touchStartX/Y`    | `int`           | Coordinate iniziali del touch |
| `editP`            | `int`           | Pompa in modifica             |
| `eS_mins`          | `int`           | Ora di inizio modificabile    |
| `eE_mins`          | `int`           | Ora di fine modificabile      |
| `animFrame`        | `int`           | Contatore animazioni          |
| `sysBright`        | `int`           | Luminosità                    |
| `sysVolume`        | `int`           | Volume                        |
| `isSleeping`       | `bool`          | Stato sleep del display       |
| `lastActivity`     | `unsigned long` | Ultima attività touch         |
| `lastTeleReceived` | `long`          | Ultima telemetria valida      |

---

# 13. Ciclo di rendering

La funzione principale dell'interfaccia è:

```cpp
updateUI()
```

Questa funzione:

1. reimposta il testo a dimensione `1`;
2. identifica la pagina corrente;
3. richiama la relativa funzione `drawPageN()`;
4. aggiorna header e footer;
5. gestisce il ridisegno dell'interfaccia.

La struttura generale è:

```text
updateUI()
     │
     ├── drawPage0()
     ├── drawPage1()
     ├── drawPage2()
     ├── drawPage3()
     ├── drawPage4()
     ├── drawPage5()
     └── drawPage6()
```

Le animazioni vengono aggiornate ogni circa:

```text
300 ms
```

tramite:

```text
animFrame++
```

---

# 14. Header

L'header occupa la parte superiore del display:

```text
Y = 0–25
```

Contiene:

* icona batteria;
* percentuale batteria;
* ora;
* titolo della schermata;
* indicatori di comunicazione;
* 7 punti di navigazione.

I punti permettono di identificare visivamente la schermata attiva.

```text
● ○ ○ ○ ○ ○ ○
```

---

# 15. Footer

Il footer occupa la parte inferiore:

```text
Y = 220–239
```

Mostra lo stato della comunicazione radio:

```text
RADIO: ONLINE
```

oppure:

```text
RADIO: DISCONNESSO
```

Lo stato viene determinato in base al tempo trascorso dall'ultima telemetria valida.

La soglia utilizzata è:

```text
6000 ms
```

I LED RGB vengono inoltre utilizzati per rappresentare lo stato della comunicazione.

---

# 16. Funzioni grafiche principali

L'interfaccia utilizza diverse funzioni grafiche riutilizzabili.

| Funzione              | Descrizione                   |
| --------------------- | ----------------------------- |
| `drawRoundButton()`   | Pulsante arrotondato          |
| `drawPill()`          | Indicatore P1/P2 e ON/OFF     |
| `drawGearSmall()`     | Ingranaggio animato           |
| `drawStylizedCloud()` | Nuvola meteorologica          |
| `drawThinkingBrain()` | Cervello animato per SMART IA |
| `drawBar()`           | Barra percentuale             |
| `drawRssiTower()`     | Indicatore segnale Wi-Fi      |
| `formatTime()`        | Conversione minuti → HH:MM    |

L'utilizzo di funzioni grafiche riutilizzabili permette di mantenere coerente l'aspetto dell'interfaccia tra le diverse schermate.

---

# 17. Le 7 schermate

Il Remote Control dispone di **7 schermate principali**, numerate da `0` a `6`.

```text
0 — STATO SISTEMA
1 — POZZO / ACQUA
2 — PROGRAMMAZIONE
3 — METEO
4 — SMART IA
5 — AUDIO E SCHERMO
6 — RETE WI-FI
```

---

# 18. Schermata 0 — STATO SISTEMA

La schermata principale replica le informazioni essenziali dello stato dell'impianto.

Visualizza:

* Pompa 1;
* Pompa 2;
* percentuale di umidità;
* stato delle pompe;
* temperatura;
* umidità dell'aria;
* stato del pozzo;
* informazioni meteorologiche;
* stato della comunicazione.

Le pompe sono rappresentate tramite card dedicate.

Quando una pompa è attiva, l'interfaccia può visualizzare un'animazione con ingranaggio/flusso.

### Interazione

Le aree touch principali sono:

```text
┌─────────────────────┬─────────────────────┐
│                     │                     │
│       POMPA 1       │       POMPA 2       │
│                     │                     │
└─────────────────────┴─────────────────────┘
```

Con:

```text
x < 160 → Pompa 1
x ≥ 160 → Pompa 2
```

Per il pozzo:

```text
y = 150–190
x > 214
```

Il comando viene inviato all'unità principale e lo stato locale viene aggiornato in modo ottimistico.

---

# 19. Schermata 1 — POZZO / ACQUA

Questa schermata è dedicata alla gestione e visualizzazione delle informazioni relative all'acqua.

Visualizza:

* livello del serbatoio;
* percentuale disponibile;
* stato dell'acqua;
* stato della pompa pozzo;
* animazione del livello dell'acqua;
* onde e bolle;
* indicazione del flusso quando il pozzo è attivo.

### Interazione

Il pulsante:

```text
POMPA POZZO
```

può essere utilizzato per alternare lo stato della pompa.

Area principale:

```text
y = 162–184
x = 128–308
```

---

# 20. Schermata 2 — PROGRAMMAZIONE

La schermata di programmazione permette di configurare l'orario di funzionamento delle pompe.

Sono disponibili due schede:

```text
[POMPA 1]       [POMPA 2]
```

Per ogni pompa è possibile modificare:

* ora di inizio;
* ora di fine;
* salvare la programmazione.

La regolazione avviene con incrementi di:

```text
15 minuti
```

con ritorno ciclico nell'intervallo:

```text
0–1439 minuti
```

### Selezione pompa

```text
x = 20–150   → Pompa 1
x = 170–300  → Pompa 2
```

### Modifica inizio

```text
y = 82–116

x = 100–140 → diminuisce
x = 240–280 → aumenta
```

### Modifica fine

```text
y = 130–164
```

con la stessa logica dei pulsanti `–` e `+`.

### Salvataggio

Il pulsante:

```text
SALVARE
```

invia:

```text
setProgPompa
pStartH
pStartM
pEndH
pEndM
```

e visualizza temporaneamente:

```text
SALVATO!
```

> **Nota:** il protocollo attuale supporta una programmazione per pompa. Non sono presenti tre slot indipendenti per ogni pompa.

---

# 21. Schermata 3 — METEO

La schermata meteorologica visualizza i dati ricevuti dall'unità principale.

Informazioni principali:

* descrizione del tempo;
* temperatura;
* umidità;
* eventuale previsione di pioggia;
* stato del servizio meteorologico.

L'icona meteorologica viene determinata sulla base di:

```text
teleData.meteo
+
ora del sistema
```

Sono supportate rappresentazioni grafiche per condizioni come:

* sole;
* luna;
* nuvoloso;
* pioggia;
* temporale;
* neve.

Se OpenWeather non è configurato nel sistema principale, viene visualizzato un messaggio informativo.

### Interazione

La schermata è esclusivamente informativa:

```text
Touch → nessuna azione
```

---

# 22. Schermata 4 — SMART IA

La schermata SMART IA visualizza lo stato del sistema di irrigazione adattivo.

Visualizza:

* stato SMART;
* stato attivato/disattivato;
* efficienza Pompa 1;
* efficienza Pompa 2;
* score Pompa 1;
* score Pompa 2.

L'interfaccia utilizza un'animazione grafica del cervello per rappresentare il sistema SMART.

### Interazione

Il tocco nell'area principale può alternare:

```text
SMART IA
ATTIVATO
      ↕
DISATTIVATO
```

Il comando viene inviato tramite:

```text
cmdSmartMode
```

Il sistema utilizza inoltre una protezione specifica per distinguere il tocco da uno swipe.

Questo evita che lo scorrimento verso una schermata adiacente possa modificare accidentalmente la modalità SMART.

---

# 23. Schermata 5 — AUDIO E SCHERMO

Questa schermata gestisce i parametri dell'interfaccia locale.

Sono disponibili due controlli:

```text
VOLUME SUONO
```

e:

```text
LUMINOSITÀ SCHERMO
```

### Volume

Intervallo:

```text
0–100
```

Passo:

```text
±10
```

Area touch:

```text
x = 60–110 → diminuisce
x = 210–260 → aumenta
```

### Luminosità

Intervallo:

```text
10–255
```

Passo:

```text
±25
```

Il valore viene applicato direttamente al backlight tramite:

```cpp
analogWrite(PIN_BL, sysBright);
```

---

# 24. Schermata 6 — RETE WI-FI

La schermata di rete visualizza le informazioni della connessione dell'unità principale.

Visualizza:

* SSID;
* livello RSSI;
* indirizzo IP;
* MAC address;
* stato della rete.

Il livello RSSI viene rappresentato graficamente tramite una torre di segnale.

### Interazione

La schermata è informativa:

```text
Touch → nessuna azione
```

---

# 25. Navigazione tramite Swipe

La navigazione tra le sette schermate utilizza gesti orizzontali.

Parametri:

```text
SWIPE_THRESHOLD = 60 px
SWIPE_MAX_MS    = 800 ms
```

Sequenza:

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

Lo swipe viene riconosciuto confrontando:

```text
touchStartX
```

con la posizione finale del dito.

Quando:

```text
|touchStartX - endX| > 60
```

il sistema interpreta il gesto come navigazione.

La pagina viene mantenuta nell'intervallo:

```text
0–6
```

---

# 26. Gestione TAP vs SWIPE

La logica di interazione segue questo flusso:

```text
Touch rilevato
      │
      ▼
Lettura X/Y
      │
      ▼
Memorizzazione touchStartX
      │
      ▼
Gestione comando pagina
      │
      ▼
Rilevazione movimento
      │
      ▼
|ΔX| > 60 ?
   │          │
  NO         SI
   │          │
   ▼          ▼
  TAP        SWIPE
   │          │
   ▼          ▼
Comando     Cambio pagina
```

Nel caso specifico della schermata SMART IA, il comando viene prima memorizzato come:

```text
pendingSmartToggle
```

e viene eseguito solamente quando viene confermato che il gesto non era uno swipe.

Questo meccanismo impedisce l'attivazione involontaria dello SMART durante la navigazione.

---

# 27. Modalità Sleep

Il display dispone di una modalità di risparmio energetico.

Dopo:

```text
30 secondi
```

senza interazione touch:

```text
lastActivity
      ↓
timeout
      ↓
riduzione graduale backlight
      ↓
isSleeping = true
```

Il backlight viene ridotto progressivamente fino allo spegnimento.

Un nuovo tocco:

```text
Touch
 ↓
Wake-up
 ↓
Ripristino display
```

---

# 28. Sistema audio

Il Remote Control dispone di feedback sonoro.

## Click

La funzione:

```cpp
playClick()
```

genera un tono di:

```text
3000 Hz
```

La durata dipende dal volume configurato.

Quando:

```text
sysVolume = 0
```

il sistema rimane silenzioso.

## Suono di avvio

La funzione:

```cpp
playBootSound()
```

riproduce una sequenza di quattro toni durante l'avvio del dispositivo.

---

# 29. Sequenza di Boot

La procedura di avvio segue una sequenza definita.

```text
1. Inizializzazione GPIO
        ↓
2. Inizializzazione Serial
        ↓
3. Configurazione backlight
        ↓
4. Configurazione LED
        ↓
5. Inizializzazione audio
        ↓
6. Inizializzazione TFT
        ↓
7. setRotation(1)
        ↓
8. Visualizzazione splash screen
        ↓
9. Calibrazione touch
        ↓
10. Inizializzazione ESP-NOW
        ↓
11. Inizializzazione comandi
        ↓
12. Avvio interfaccia
```

Lo splash screen utilizza il logo definito in:

```text
logo.h
```

---

# 30. Palette grafica

L'interfaccia utilizza una palette definita per mantenere una rappresentazione coerente dello stato del sistema.

| Nome             | Hex      | Utilizzo              |
| ---------------- | -------- | --------------------- |
| `COLOR_BG`       | `0x0821` | Sfondo principale     |
| `COLOR_ACCENT`   | `0x07E0` | Colore di evidenza    |
| `COLOR_BLUE`     | `0x041F` | Blu                   |
| `COLOR_RED`      | `0xF800` | Rosso / stato critico |
| `COLOR_TEXT`     | `0xFFFF` | Testo                 |
| `COLOR_DARK`     | `0x2124` | Barre e fondi scuri   |
| `COLOR_WARN`     | `0xFFE0` | Avviso                |
| `COLOR_SUCCESS`  | `0x07E4` | Successo              |
| `COLOR_CRITICAL` | `0xF800` | Critico               |
| `COLOR_WARNING`  | `0xFEA0` | Avviso arancione      |
| `COLOR_INFO`     | `0x07FF` | Informazioni SMART    |
| `COLOR_MUTED`    | `0x8C71` | Testo secondario      |
| `COLOR_WATER`    | `0x1CFF` | Acqua                 |
| `COLOR_FLOWER_R` | `0xF9A0` | Sole / arancione      |
| `COLOR_CARD`     | `0x0861` | Card                  |
| `COLOR_TOUCH`    | `0x5D7F` | Interazione touch     |

---

# 31. Principi di progettazione

Il progetto è stato sviluppato seguendo alcuni principi fondamentali.

### Separazione delle responsabilità

L'unità principale gestisce l'impianto mentre il Remote Control gestisce l'interfaccia utente locale.

### Comunicazione wireless

ESP-NOW permette una comunicazione diretta tra i nodi embedded.

### Interfaccia locale

Il dispositivo funziona come pannello di controllo dedicato senza richiedere PC o smartphone.

### Feedback visivo

Lo stato delle pompe, della rete e della comunicazione viene rappresentato graficamente.

### Feedback sonoro

Le interazioni dell'utente possono produrre un feedback acustico.

### Modularità

Il Remote Control è mantenuto come progetto separato rispetto al firmware principale dell'irrigazione.

---

# 32. Considerazioni di manutenzione

## Compatibilità delle strutture

Quando vengono modificate:

```text
struct_command
struct_telemetry
```

è necessario aggiornare entrambi i firmware.

```text
MAIN
 │
 ├── modifica struttura
 │
 ▼
REMOTE
 │
 └── stessa struttura
```

I due dispositivi devono essere ricompilati e aggiornati insieme.

---

## Dimensione del testo

L'header e il footer utilizzano esplicitamente:

```cpp
setTextSize(1);
```

Le schermate che utilizzano dimensioni maggiori devono ripristinare la dimensione del testo quando necessario.

Questa gestione evita che una dimensione di testo impostata da una schermata venga mantenuta accidentalmente nelle schermate successive.

---

## Coordinate touch

Le coordinate delle aree touch devono rimanere allineate alle coordinate utilizzate durante il rendering grafico.

Una modifica grafica deve quindi essere verificata anche nella relativa logica touch.

---

## Programmazione

Il protocollo attuale supporta:

```text
1 programmazione per Pompa 1
1 programmazione per Pompa 2
```

La schermata modifica una pompa alla volta tramite:

```text
editP
```

---

## Magic Number

I valori:

```text
12345 → comandi
54321 → telemetria
```

vengono utilizzati come identificatori di validazione del protocollo e non devono essere modificati indipendentemente sui due dispositivi.

---

# 33. Estensione futura dell'interfaccia

L'architettura è stata predisposta per poter aggiungere ulteriori schermate.

Per aggiungere una nuova schermata sarebbe necessario:

1. creare una nuova funzione `drawPageN()`;
2. aggiungere la relativa gestione in `updateUI()`;
3. aggiungere la gestione touch;
4. aggiornare il limite della navigazione;
5. aggiornare il numero dei punti di navigazione nell'header;
6. aggiungere eventuali nuovi dati alla struttura di telemetria;
7. aggiornare e ricompilare entrambi i firmware se il protocollo cambia.

L'aggiunta di una nuova schermata deve quindi mantenere la coerenza tra:

```text
UI
│
├── Rendering
├── Touch
├── Navigazione
└── Protocollo dati
```

---

# 34. Competenze tecniche dimostrate

Il progetto dimostra esperienza pratica nei seguenti ambiti.

### Embedded Systems

* ESP32 / ESP32-S3;
* programmazione C++;
* Arduino framework;
* GPIO;
* ADC;
* PWM;
* gestione periferiche;
* interfacce embedded.

### Comunicazione

* ESP-NOW;
* comunicazione ESP32-to-ESP32;
* pacchetti strutturati;
* validazione tramite `magicNumber`;
* gestione timeout;
* channel hopping;
* architetture embedded distribuite.

### Human-Machine Interface

* TFT 320×240;
* touch resistivo;
* XPT2046;
* GUI embedded;
* touch interaction;
* swipe gesture;
* feedback visivo;
* feedback acustico.

### System Integration

* integrazione con sistema di irrigazione;
* telemetria wireless;
* comandi remoti;
* visualizzazione dati;
* gestione stato;
* integrazione tra nodi embedded.

---

# 35. Struttura del progetto

Il repository è organizzato come documentazione tecnica di portfolio.

```text
remote-control/
│
├── README.md
│
├── docs/
│   ├── DOCUMENTAZIONE.md
│   ├── ARCHITETTURA.md
│   └── architettura.svg
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

Il firmware sorgente non è incluso nel repository pubblico.

La documentazione e le immagini hanno lo scopo di mostrare:

* architettura;
* interfaccia;
* comunicazione;
* progettazione embedded;
* integrazione del sistema;
* soluzioni tecniche adottate.

---

# 36. Stato del progetto

| Parametro     | Valore                          |
| ------------- | ------------------------------- |
| Piattaforma   | ESP32-S3                        |
| Display       | TFT 240×320 fisico              |
| Canvas        | 320×240 landscape               |
| Touch         | XPT2046 resistivo               |
| Comunicazione | ESP-NOW                         |
| Firmware      | C++ / Arduino                   |
| GUI           | TFT Touch Embedded UI           |
| Navigazione   | Touch + Swipe                   |
| Schermate     | 7                               |
| Architettura  | Sistema Embedded Distribuito    |
| Categoria     | Embedded / IoT / Remote Control |
| Repository    | Portfolio tecnico               |

---

# 37. Collegamento con ESP32 Smart Irrigation

Il Remote Control fa parte dell'ecosistema **ESP32 Smart Irrigation**, ma viene mantenuto in un repository separato.

```text
┌──────────────────────────────────┐
│      ESP32 Smart Irrigation      │
│                                  │
│      Unità principale            │
│      Controllo irrigazione       │
└────────────────┬─────────────────┘
                 │
                 │ ESP-NOW
                 ▼
┌──────────────────────────────────┐
│       Tech3D Remote Control      │
│                                  │
│       Interfaccia wireless       │
│       TFT + Touch                │
└──────────────────────────────────┘
```

La separazione dei repository consente di documentare in modo indipendente le responsabilità tecniche dei due dispositivi, mantenendo allo stesso tempo evidente la loro integrazione all'interno dello stesso sistema.

---

# 38. Sintesi tecnica

Il **Tech3D Remote Control** rappresenta un esempio di interfaccia embedded wireless progettata per interagire con un sistema IoT distribuito.

Il progetto combina:

```text
ESP32-S3
   +
TFT 320×240
   +
Touch resistivo
   +
ESP-NOW
   +
Telemetria
   +
Comandi bidirezionali
   +
GUI embedded
   +
Gestione touch/swipe
   +
Feedback visivo e acustico
```

L'architettura separa il controllo dell'impianto dall'interfaccia utente, permettendo di realizzare un pannello di controllo dedicato, compatto e indipendente da PC o smartphone.

Il progetto è presentato come **portfolio tecnico**, con particolare attenzione alle competenze di progettazione embedded, comunicazione wireless, interfacce uomo-macchina e integrazione di sistemi.
