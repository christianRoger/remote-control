
# Documentazione Tecnica — Tech3D Remote Control (CYD / ESP32-S3)

Controllo remoto tramite ESP-NOW per l'unità principale di irrigazione `ESP32S3_Irrigatore_ST7789`.

Il firmware viene eseguito su un ESP32-S3 con display TFT touch resistivo (CYD) e **replica visivamente** le schermate dell'unità principale, consentendo di attivare/disattivare le pompe, il pozzo e la modalità Smart IA, programmare gli orari e regolare audio e luminosità.

> **Nota:** Il codice sorgente non è incluso nel repository pubblico. Questa documentazione descrive l'architettura tecnica, il protocollo di comunicazione, l'interfaccia utente e le principali soluzioni implementative del progetto.

---

## 1. Panoramica

| Elemento      | Descrizione                                                                                             |
| ------------- | ------------------------------------------------------------------------------------------------------- |
| Scheda        | ESP32-S3 (CYD — Cheap Yellow Display)                                                                   |
| Display       | TFT 240×320 **fisico**, utilizzato in modalità landscape (`setRotation(1)`) → canvas logico **320×240** |
| Touch         | Resistivo (`TFT_Touch`), calibrato con `setCal(495, 3398, 721, 3448, 320, 240, 1)`                      |
| Comunicazione | ESP-NOW (broadcast), senza conferma di consegna                                                         |
| Schermate     | 7 schermate navigabili tramite swipe orizzontale                                                        |
| Librerie      | `TFT_eSPI`, `TFT_Touch`, `WiFi`, `esp_now`                                                              |

> ⚠️ **Il canvas logico è 320×240 (landscape).** Tutte le coordinate riportate in questo documento utilizzano il sistema di coordinate logico `(x: 0–319, y: 0–239)`.

---

## 2. Mappatura dei Pin

| Pin | Definizione | Funzione                       |
| --: | ----------- | ------------------------------ |
|  39 | `RTP_DOUT`  | Touch resistivo DOUT           |
|  32 | `RTP_DIN`   | Touch resistivo DIN            |
|  25 | `RTP_SCK`   | Touch resistivo SCK            |
|  33 | `RTP_CS`    | Touch resistivo CS             |
|  36 | `RTP_IRQ`   | Touch IRQ (interruzione touch) |
|   4 | `AUDIO_EN`  | Enable amplificatore audio     |
|  26 | `AUDIO_OUT` | Uscita audio (buzzer/click)    |
|  22 | `LED_R`     | LED rosso (stato radio)        |
|  16 | `LED_G`     | LED verde (stato radio)        |
|  17 | `LED_B`     | LED blu (stato radio)          |
|  34 | `PIN_BAT`   | Lettura batteria (ADC)         |
|  21 | `PIN_BL`    | Backlight display (PWM)        |

Stato iniziale:

* `PIN_BL` viene attivato con la luminosità `sysBright` (255);
* tutti i LED vengono impostati su `HIGH`;
* `AUDIO_EN` viene impostato su `LOW`.

---

## 3. Comunicazione ESP-NOW

### 3.1 Strutture Dati

Le strutture dati devono essere **identiche a quelle utilizzate dall'unità principale**.

### Dati inviati

`struct_command` → unità principale:

```c
typedef struct struct_command {

  int cmdP1;        // 1 attiva / 0 disattiva Pompa 1

  int cmdP2;        // 1 attiva / 0 disattiva Pompa 2

  int cmdPozzo;     // 1 attiva / 0 disattiva Pozzo

  int cmdResetWifi; // -1 = inattivo; altro valore = reset Wi-Fi

  int setProgPompa; // pompa target della programmazione (1 o 2)

  int pStartH;      // ora inizio (0–23)

  int pStartM;      // minuto inizio

  int pEndH;        // ora fine

  int pEndM;        // minuto fine

  int cmdSmartMode; // -1 = inattivo; 1 = attiva; 0 = disattiva modalità IA

  int magicNumber;  // sempre 12345

} struct_command;
```

### Dati ricevuti

`struct_telemetry` ← unità principale:

```c
typedef struct struct_telemetry {

  int sens1, sens2;        // umidità dei sensori S1/S2 (%)

  int percentAcqua;        // livello del serbatoio (%)

  int temp;                // temperatura (°C)

  int umid;                // umidità dell'aria (%)

  bool statoP1, statoP2, statoPozzo; // stati pompe/pozzo

  char ipStr[16], ssid[32], mac[18], ora[6];

  int prog1_Start, prog1_End, prog2_Start, prog2_End; // programmazione (minuti)

  int rssi;                // segnale Wi-Fi dell'unità principale

  char meteo[48];          // descrizione meteo (OpenWeather)

  int pioggia;             // 1 = previsione di pioggia

  int meteoEnabled;        // 1 = OpenWeather configurato

  int smartMode;           // 1 = modalità IA attiva

  float efficienzaP1, efficienzaP2; // efficienza (%/min) — provenienti dall'unità principale

  int scoreP1, scoreP2;    // score della modalità IA (P1/P2)

  int magicNumber;         // sempre 54321

} struct_telemetry;
```

> 🔴 **Regola critica:** `struct_telemetry` del Remote Control e `struct_telemetry` (`main/app.h`) devono contenere **esattamente gli stessi campi, nello stesso ordine e con gli stessi tipi**.
>
> Qualsiasi modifica a una delle strutture richiede la ricompilazione e la programmazione di **entrambi** i firmware. In caso contrario, la dimensione della struttura può cambiare e la comunicazione radio può interrompersi, causando lo stato `RADIO: DISCONNESSO`.

### 3.2 Invio e Ricezione

`initESPNOW()`:

* imposta `WiFi.mode(WIFI_STA)`;
* registra i callback;
* aggiunge il peer broadcast;
* utilizza `channel = 0`;
* non utilizza crittografia.

`OnDataRecv()`:

* copia il pacchetto in `teleData` quando `len == sizeof(struct_telemetry)`;
* verifica `magicNumber == 54321`;
* aggiorna `lastTeleReceived`.

`enviaComando()`:

* invia `cmdData`;
* azzera i comandi single-fire dopo l'invio:

  * `cmdResetWifi`;
  * `setProgPompa`;
  * `cmdSmartMode` → `-1`;
* evita quindi la trasmissione ripetuta dello stesso comando.

`lerBateriaPct()`:

* legge `PIN_BAT`;
* converte il valore ADC nell'intervallo 0–100%;
* utilizza il range `2000–2800`.

### 3.3 Canale e Riconnessione

Nel `loop()`, se non viene ricevuta telemetria per più di **6 secondi**, il firmware avvia una procedura di ricerca del canale.

Il sistema effettua il **channel hop** sui canali 1–13, cambiando canale ogni 500 ms per tentare di ritrovare l'unità principale.

---

## 4. Palette dei Colori

| Nome             | Hex      | Utilizzo                                 |
| ---------------- | -------- | ---------------------------------------- |
| `COLOR_BG`       | `0x0821` | Sfondo                                   |
| `COLOR_ACCENT`   | `0x07E0` | Evidenziazione verde acqua               |
| `COLOR_BLUE`     | `0x041F` | Blu                                      |
| `COLOR_RED`      | `0xF800` | Rosso / critico                          |
| `COLOR_TEXT`     | `0xFFFF` | Testo bianco                             |
| `COLOR_DARK`     | `0x2124` | Barre superiore/inferiore e sfondi scuri |
| `COLOR_WARN`     | `0xFFE0` | Avviso giallo                            |
| `COLOR_SUCCESS`  | `0x07E4` | Successo                                 |
| `COLOR_CRITICAL` | `0xF800` | Critico                                  |
| `COLOR_WARNING`  | `0xFEA0` | Avviso arancione                         |
| `COLOR_INFO`     | `0x07FF` | Informazione ciano (Smart IA)            |
| `COLOR_MUTED`    | `0x8C71` | Testo secondario grigio                  |
| `COLOR_WATER`    | `0x1CFF` | Blu acqua                                |
| `COLOR_FLOWER_R` | `0xF9A0` | Arancione (sole)                         |
| `COLOR_CARD`     | `0x0861` | Sfondo delle schede                      |
| `COLOR_TOUCH`    | `0x5D7F` | Blu touch                                |

---

## 5. Architettura dell'Interfaccia Utente

### 5.1 Variabili Globali

| Variabile                | Tipo            | Funzione                                                     |
| ------------------------ | --------------- | ------------------------------------------------------------ |
| `currentPage`            | `int`           | Schermata attiva (0–6)                                       |
| `forceRedraw`            | `bool`          | Forza il ridisegno completo al prossimo `updateUI()`         |
| `touchStartX/Y`          | `int`           | Coordinate iniziali del touch per il rilevamento dello swipe |
| `editP`                  | `int`           | Pompa in modifica nella Programmazione (1 o 2)               |
| `eS_mins`, `eE_mins`     | `int`           | Orario di inizio/fine modificabile in minuti                 |
| `animFrame`              | `int`           | Contatore delle animazioni                                   |
| `sysBright`, `sysVolume` | `int`           | Luminosità (10–255) e volume (0–100)                         |
| `isSleeping`             | `bool`          | Display in modalità sleep                                    |
| `lastActivity`           | `unsigned long` | Ultimo touch, timeout sleep di 30 s                          |
| `lastTeleReceived`       | `long`          | Ultima telemetria valida ricevuta                            |

### 5.2 Ciclo di Rendering

`updateUI()`:

* reimposta `setTextSize(1)`;
* richiama `drawPageN()` in base a `currentPage`.

Ogni `drawPageN()`:

1. esegue `fillScreen(COLOR_BG)`;
2. richiama `drawHeader(...)` quando `forceRedraw` è `true`;
3. disegna il contenuto della schermata;
4. richiama `drawFooter()`.

L'animazione viene aggiornata ogni **300 ms**:

```text
animFrame++
     ↓
updateUI()
```

Il ridisegno è quindi limitato alle aree e agli elementi necessari.

`drawHeader(titolo)`:

* barra superiore da `y=0` a `y=25`;
* icona batteria + percentuale;
* orologio (`teleData.ora`, TR);
* titolo centrale;
* **7 indicatori di navigazione**, con quello corrente evidenziato.

`drawFooter()`:

* barra inferiore da `y=220` a `y=240`;
* visualizza `RADIO: ONLINE` in ciano oppure `RADIO: DISCONNESSO` in rosso;
* determina lo stato in base a `lastTeleReceived < 6000 ms`;
* gestisce anche i LED di stato.

### 5.3 Funzioni Grafiche Ausiliarie

| Funzione                                      | Descrizione                                                             |
| --------------------------------------------- | ----------------------------------------------------------------------- |
| `drawRoundButton(x,y,w,h,label,bg,fg,active)` | Pulsante arrotondato generico                                           |
| `drawPill(x,y,w,h,lbl,on)`                    | Indicatore "pill" P1/P2/ON-OFF utilizzato nella schermata STATO SISTEMA |
| `drawGearSmall(cx,cy,r,ang,c)`                | Ingranaggio animato                                                     |
| `drawStylizedCloud(...)`                      | Nuvola stilizzata nella schermata METEO                                 |
| `drawThinkingBrain(cx,cy,active)`             | Cervello animato nella schermata SMART IA                               |
| `drawBar(x,y,w,h,percent,color,label)`        | Barra di avanzamento                                                    |
| `drawRssiTower(x,y,rssi)`                     | Indicatore del segnale Wi-Fi nella schermata RETE                       |
| `formatTime(m)`                               | Conversione da minuti a formato `"HH:MM"`                               |

---

# 6. Le 7 Schermate

## 6.1 Schermata 0 — `STATO SISTEMA`

Funzioni:

`drawPage0()` → `drawStatoSistema()`

La schermata replica la schermata principale del sistema.

Visualizza:

* scheda **POMPA 1**;
* scheda **POMPA 2**;
* stato delle pompe;
* percentuale di umidità;
* barra/indicatore;
* animazione dell'ingranaggio;
* pannello **AMBIENTE**;
* temperatura;
* umidità dell'aria;
* stato **POZZO**;
* informazioni meteorologiche.

### Interazione Touch

**Schede pompe — `y=30–136`:**

```text
x < 160  → alterna Pompa 1
x ≥ 160  → alterna Pompa 2
```

**Badge POZZO — `y=150–190`, `x > 214`:**

```text
→ alterna stato Pozzo
```

L'interfaccia utilizza un aggiornamento ottimistico locale:

```text
teleData.statoP1 = !teleData.statoP1
```

seguito dall'invio del comando all'unità principale.

---

## 6.2 Schermata 1 — `POZZO / ACQUA`

Funzione:

`drawPage1()` → `drawWaterScreen()`

La schermata visualizza:

* serbatoio animato;
* onde dell'acqua;
* bolle;
* getto del pozzo quando attivo;
* percentuale dell'acqua;
* stato `ACQUA DISPONIBILE`;
* stato `LIVELLO CRITICO`;
* stato della **POMPA POZZO**.

### Interazione Touch

Pulsante **POMPA POZZO**:

```text
y = 162–184
x = 128–308
```

L'interazione alterna lo stato del pozzo.

---

## 6.3 Schermata 2 — `PROGRAMMAZIONE`

Funzione:

`drawPage2()`

La schermata contiene:

* scheda **POMPA 1**;
* scheda **POMPA 2**;
* orario **INIZIO**;
* orario **FINE**;
* pulsanti `–` e `+`;
* incremento di 15 minuti;
* orari visualizzati in formato grande;
* pulsante **SALVA**.

### Selezione della Pompa

Area:

```text
y = 30–56

x = 20–150   → modifica Pompa 1
x = 170–300  → modifica Pompa 2
```

Quando viene selezionata una pompa, vengono caricati i relativi valori:

```text
prog1_Start / prog1_End
```

oppure:

```text
prog2_Start / prog2_End
```

### Modifica INIZIO

Area:

```text
y = 82–116
```

```text
x = 100–140 → diminuisce
x = 240–280 → aumenta
```

Passo:

```text
±15 minuti
```

Intervallo:

```text
0–1439 minuti
```

con gestione circolare (`wrap`).

### Modifica FINE

Area:

```text
y = 130–164
```

Con la stessa logica:

```text
x = 100–140 → diminuisce
x = 240–280 → aumenta
```

### Salvataggio

Area:

```text
y = 170–200
```

L'operazione invia:

* `setProgPompa`;
* `pStartH`;
* `pStartM`;
* `pEndH`;
* `pEndM`.

Dopo il salvataggio viene visualizzato:

```text
SALVATO!
```

per circa 1 secondo.

> ⚠️ **Il protocollo supporta una sola programmazione per pompa.**
>
> La schermata Remote Control gestisce quindi un unico intervallo di programmazione per Pompa 1 e un unico intervallo per Pompa 2.

---

## 6.4 Schermata 3 — `METEO`

Funzione:

`drawPage3()` → `drawMeteoScreen()`

La schermata visualizza le informazioni meteorologiche ricevute dall'unità principale.

L'icona meteorologica viene animata in base a:

* `teleData.meteo`;
* ora corrente.

Sono supportate rappresentazioni come:

* sole;
* luna;
* nuvola;
* pioggia;
* temporale;
* neve.

Vengono inoltre visualizzati:

* temperatura;
* umidità.

Se OpenWeather non è configurato, viene visualizzato:

```text
Non configurato nel sistema
```

### Interazione Touch

Nessuna.

La schermata è esclusivamente informativa.

---

## 6.5 Schermata 4 — `SMART IA`

Funzione:

`drawPage4()` → `drawSmartScreen()`

La schermata visualizza:

* animazione del cervello;
* stato del sistema IA;
* badge **ATTIVATO / DISATTIVATO**;
* efficienza Pompa 1;
* efficienza Pompa 2;
* score Pompa 1;
* score Pompa 2.

I valori delle metriche provengono dall'unità principale.

### Interazione Touch

Qualsiasi punto dell'area principale può essere utilizzato per alternare la modalità IA:

```text
y = 28–218
x = 5–315
```

Il sistema utilizza un aggiornamento ottimistico:

```text
teleData.smartMode = !teleData.smartMode
```

Il comando viene successivamente inviato all'unità principale.

L'azione è protetta contro le gesture swipe, come descritto nella sezione 7.

---

## 6.6 Schermata 5 — `AUDIO E SCHERMO`

Funzione:

`drawPage5()`

La schermata contiene due gruppi di controllo.

### Volume Suono

```text
[ - ]      100%      [ + ]
```

Area:

```text
y = 80–120

x = 60–110  → diminuisce
x = 210–260 → aumenta
```

Passo:

```text
±10
```

Intervallo:

```text
0–100
```

### Luminosità Schermo

```text
[ - ]      100%      [ + ]
```

Area:

```text
y = 160–200

x = 60–110  → diminuisce
x = 210–260 → aumenta
```

Passo:

```text
±25
```

Intervallo:

```text
10–255
```

La luminosità viene applicata direttamente al backlight tramite:

```text
analogWrite(PIN_BL, ...)
```

---

## 6.7 Schermata 6 — `RETE WI-FI`

Funzione:

`drawPage6()`

La schermata visualizza le informazioni di rete dell'unità principale:

* **SSID**;
* intensità del segnale tramite indicatore RSSI;
* **IP**;
* **MAC Address**.

### Interazione Touch

Nessuna.

La schermata è esclusivamente informativa.

---

# 7. Logica Touch — Tap vs Swipe

Nel `loop()`, quando viene rilevato:

```text
touch.Pressed() && !RTP_IRQ
```

viene eseguita la seguente sequenza:

### 1. Risveglio del Display

Se il display è in modalità sleep, viene riattivato.

### 2. Lettura del Touch

Vengono lette le coordinate:

```text
x, y
```

e viene memorizzata:

```text
touchStartX
```

### 3. Gestione della Schermata

Viene eseguito l'handler associato alla schermata corrente, utilizzando le aree touch definite nella sezione 6.

### 4. Rilevamento Swipe

Durante il mantenimento del touch viene eseguito un ciclo `while` che legge `endX` fino al rilascio del dito.

Se:

```text
|touchStartX - endX| > 60
```

viene riconosciuta una gesture swipe.

La schermata viene quindi modificata:

```text
currentPage
```

limitandola all'intervallo:

```text
0–6
```

e l'interfaccia viene ridisegnata.

### 5. Protezione SMART IA

Nella schermata 4, il touch non attiva immediatamente il cambio di modalità.

Viene prima impostato:

```text
pendingSmartToggle
```

Il cambio di stato viene eseguito solamente quando:

```text
!didSwipe
```

In questo modo uno swipe tra le schermate non può accidentalmente attivare o disattivare la modalità SMART IA.

---

# 8. Modalità Sleep

Dopo **30 secondi** senza interazione touch:

```text
lastActivity
```

il sistema entra gradualmente in modalità sleep.

Il backlight viene ridotto progressivamente fino a:

```text
0
```

e:

```text
isSleeping = true
```

Qualsiasi nuovo touch:

* riattiva il display;
* ripristina il backlight;
* aggiorna l'attività dell'utente.

---

# 9. Sistema Audio

Il Remote Control dispone di feedback sonoro locale.

### `playClick()`

Genera un bip a:

```text
3000 Hz
```

La durata è proporzionale a:

```text
sysVolume
```

Quando:

```text
sysVolume = 0
```

l'audio viene disattivato.

### `playBootSound()`

Durante l'avvio viene riprodotta una sequenza composta da **4 toni**.

---

# 10. Sequenza di Avvio — `setup()`

Durante l'avvio del sistema vengono eseguite le seguenti operazioni:

### 1. Inizializzazione Hardware

* inizializzazione dei pin;
* seriale;
* backlight;
* LED.

### 2. Inizializzazione Display

```text
tft.init()
setRotation(1)
```

Successivamente viene visualizzata la schermata iniziale tramite:

```text
renderSplash()
```

utilizzando il logo definito in `logo.h`.

### 3. Touch e Comunicazione

* calibrazione del touch;
* inizializzazione ESP-NOW.

```text
initESPNOW()
```

### 4. Inizializzazione dei Comandi

Tutti i comandi vengono inizializzati a:

```text
-1
```

in modo da evitare l'invio accidentale di comandi durante l'avvio.

---

# 11. Punti di Attenzione e Manutenzione

### 1. Sincronizzazione dei Firmware

Quando viene modificata:

```text
struct_telemetry
```

o:

```text
struct_command
```

è necessario aggiornare e programmare **entrambi i firmware**:

```text
ESP32 Smart Irrigation
        +
Tech3D Remote Control
```

---

### 2. Gestione della Dimensione del Testo

L'header e il footer utilizzano esplicitamente:

```text
setTextSize(1)
```

Le schermate che utilizzano dimensioni di testo maggiori devono considerare che:

```text
updateUI()
```

riporta la dimensione a `1`.

Questo evita che una dimensione di testo impostata in una schermata venga mantenuta accidentalmente nelle schermate successive.

Questa gestione è stata introdotta anche per evitare un problema precedente in cui alcuni testi venivano visualizzati con dimensioni eccessive.

---

### 3. Allineamento delle Coordinate Touch

Le coordinate utilizzate nel `loop()` devono rimanere sempre coerenti con le posizioni grafiche definite nelle rispettive funzioni:

```text
drawPageN()
```

Una modifica alla posizione di un pulsante deve quindi essere accompagnata dall'aggiornamento della relativa area touch.

---

### 4. Programmazione

Il Remote Control supporta:

```text
1 programmazione per Pompa 1
1 programmazione per Pompa 2
```

La schermata Programmazione modifica una pompa alla volta tramite:

```text
editP
```

---

### 5. Magic Number

Il protocollo utilizza due valori di validazione:

```text
Comando:
12345

Telemetria:
54321
```

Questi valori vengono utilizzati per verificare che i pacchetti ricevuti appartengano al protocollo previsto.

Non devono essere rimossi o modificati senza aggiornare entrambi i dispositivi.

---

# 12. Estensione del Sistema — Aggiunta di una 8ª Schermata

L'architettura è stata progettata in modo da poter essere estesa.

Per aggiungere una nuova schermata:

### 1. Creare `drawPage7()`

Seguire il pattern esistente:

```text
if(forceRedraw) {
    fillScreen(COLOR_BG);
    drawHeader(...);
    forceRedraw = false;
}

...

drawFooter();
```

### 2. Aggiornare `updateUI()`

Aggiungere:

```text
case 7:
    drawPage7();
    break;
```

### 3. Aggiungere il Gestore Touch

Nel `loop()`:

```text
else if (currentPage == 7) {
    ...
}
```

### 4. Aggiornare la Navigazione

Modificare il limite:

```text
currentPage > 6
```

in:

```text
currentPage > 7
```

e aggiornare il numero degli indicatori nel `drawHeader()`:

```text
i < 7
```

in:

```text
i < 8
```

### 5. Aggiornare il Protocollo

Se la nuova schermata richiede dati provenienti dall'unità principale, sarà necessario aggiornare:

```text
struct_telemetry
```

e successivamente ricompilare e programmare **entrambi i firmware**.

---

# 13. Riepilogo Tecnico

| Caratteristica      | Implementazione                    |
| ------------------- | ---------------------------------- |
| MCU                 | ESP32-S3                           |
| Display             | TFT 240×320 fisico                 |
| Canvas logico       | 320×240 landscape                  |
| Touch               | Resistivo XPT2046 / TFT_Touch      |
| Comunicazione       | ESP-NOW                            |
| Modalità wireless   | Broadcast                          |
| Crittografia        | Non utilizzata                     |
| Schermate           | 7                                  |
| Navigazione         | Touch + Swipe                      |
| Feedback            | Display + LED + Audio              |
| Backlight           | PWM                                |
| Batteria            | ADC                                |
| Firmware            | C++ / Arduino                      |
| Librerie principali | TFT_eSPI, TFT_Touch, WiFi, esp_now |
| Architettura        | Sistema Embedded Distribuito       |

---

## 14. Finalità del Repository

Il repository **Tech3D Remote Control** è stato realizzato come **portfolio tecnico**.

Il firmware completo non viene pubblicato. La documentazione permette di analizzare:

* l'architettura del sistema;
* la comunicazione ESP-NOW;
* il protocollo dati;
* la gestione del display;
* il sistema touch;
* la navigazione tramite swipe;
* la progettazione delle 7 schermate;
* la gestione degli stati;
* l'integrazione con il sistema Smart Irrigation.

L'obiettivo è dimostrare competenze pratiche nello sviluppo di **sistemi embedded, interfacce HMI, comunicazione wireless e integrazione di dispositivi ESP32**.
