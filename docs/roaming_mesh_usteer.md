# Wireless Roaming Multi-AP con OpenWrt: 802.11k/v/r e Usteer

Questo documento descrive la progettazione e l'ottimizzazione del sistema Wi-Fi multi-piano realizzato nella mia abitazione. L'obiettivo era ottenere una copertura totale su tre piani (divisi da solette in cemento armato che attenuano fortemente il segnale) e garantire che dispositivi in movimento (smartphone, portatili) passassero da un piano all'altro senza disconnessioni o interruzioni di chiamate VoIP o streaming.

L'infrastruttura utilizza tre nodi fisici con firmware **OpenWrt**:
1. **Cudy WR3000H** (Main Router e AP al Piano Terra)
2. **TP-Link Archer C6 v2/v3** (Access Point al Primo Piano, connesso via cavo Ethernet)
3. **TP-Link Archer C6 v2/v3** (Access Point al Secondo Piano, connesso via cavo Ethernet)

---

## ⚠️ Il Problema: I "Client Appiccicosi" (Sticky Clients)

Nelle reti Wi-Fi tradizionali con lo stesso nome di rete (SSID) e password su più AP, la decisione di cambiare AP (roaming) spetta **interamente al client** (lo smartphone o il PC). 
I client sono tipicamente "pigri": tendono a rimanere connessi all'AP iniziale anche se il segnale scende a livelli critici (es. -82 dBm), ignorando un AP adiacente che eroga un segnale eccellente (es. -50 dBm). Questo causa:
* Latenza elevata e forti perdite di pacchetti.
* Degradazione delle prestazioni Wi-Fi complessive per via dei bassi tassi di modulazione (MCS) utilizzati dai client distanti.
* Mancata connettività effettiva pur mostrando l'icona Wi-Fi attiva.

---

## 🛠️ La Soluzione Tecnologica: Standard 802.11k, 802.11v, 802.11r

Per risolvere il problema, ho configurato gli standard di roaming assistito nativi su OpenWrt (gestiti da `hostapd`).

```
                                  [ Roaming 802.11k/v/r ]
   Client (Piano 1)   -------->   Richiede Neighbor Report (802.11k)   -------->   AP (Piano 1)
   Client (Piano 1)   <--------   Invia lista AP vicini (Canali/BSSID)  <--------   AP (Piano 1)
   
                      *Il client si sposta verso il Piano Terra*
   
   usteer Daemon      -------->   Soglia RSSI superata (-75dBm).
                                  Invia BSS Transition Request (802.11v) -------> Client
   Client             -------->   Esegue Handshake FT rapido (802.11r)  -------> AP (Piano Terra) [Latenza < 10ms]
```

### 1. 802.11r (Fast BSS Transition)
Permette al client di pre-autenticarsi sul nuovo AP prima di effettuare il passaggio vero e proprio. Nello schema WPA2/WPA3-Personal standard, il processo di associazione richiede un handshake a 4 vie che può impiegare dai 100 ai 500 ms (tempo percepibile come disconnessione). 
Con l'abilitazione di **802.11r (FT)**, parte delle chiavi crittografiche viene memorizzata localmente e distribuita tra gli AP della stessa rete ("Mobility Domain"). Il tempo di transizione scende a **meno di 10 millisecondi**, rendendo il passaggio impercettibile anche durante chiamate vocali bidirezionali (es. WhatsApp, Microsoft Teams).

### 2. 802.11k (Radio Resource Measurement)
Aiuta i client a scoprire rapidamente gli AP vicini. Quando il segnale scende, il client chiede all'AP corrente un "Neighbor Report". L'AP risponde con una lista dei BSSID vicini e i relativi canali radio. In questo modo il client non deve scansionare tutte le frequenze radio (processo lento che consuma batteria), ma scansiona direttamente solo i canali dove sa che ci sono AP validi.

### 3. 802.11v (Wireless Network Management)
Consente all'infrastruttura di rete di suggerire attivamente al client di spostarsi. Se l'AP rileva che il segnale del client è scarso, può inviargli un pacchetto di "BSS Transition Management Request", indicando qual è l'AP preferibile in quel momento. I client moderni (iOS, Android, Windows) rispettano questa richiesta ed eseguono immediatamente il roaming.

---

## ⚡ Ottimizzazione Dinamica con Usteer

Nonostante 802.11k/v/r siano standard fantastici, alcuni dispositivi non li implementano perfettamente o ignorano le raccomandazioni. Per questo motivo ho introdotto **Usteer (Micro-steering Daemon)** su OpenWrt.

`usteer` è un demone che gira su ciascuno dei tre nodi e scambia informazioni in tempo reale (tramite una porta UDP dedicata sulla LAN locale, di default `16720`). Esso crea una tabella condivisa di tutti i client connessi e del loro segnale (RSSI) visto da ciascun AP.

### Logica di Steering Implementata:
1. **Monitoraggio costante:** Ciascun AP misura il segnale (in dBm) di tutti i client associati e di quelli che inviano "Probe Request" nelle vicinanze.
2. **Identificazione dello sbilanciamento:** Se il segnale di un client su un AP scende sotto la soglia di **-75 dBm** e un altro AP rileva lo stesso client con un segnale migliore di almeno **10 dBm**, `usteer` interviene.
3. **Fase 1 - Steering Dolce (802.11v):** `usteer` invia una richiesta 802.11v invitando il client a migrare verso l'AP migliore.
4. **Fase 2 - Steering Aggressivo (Kick):** Se il client ignora la richiesta 802.11v o non supporta lo standard, `usteer` disassocia il client dall'AP debole e blocca temporaneamente le sue risposte di associazione (Probe Response) su quell'AP per circa 5 secondi. Il client è così costretto a fare una scansione e a connettersi all'AP con il segnale forte, l'unico che in quel momento risponde alle sue richieste.

---

## 🔧 Configurazione Tipica

L'implementazione pratica ha richiesto la modifica manuale dei file di configurazione `/etc/config/wireless` e `/etc/config/usteer` per sintonizzare fine le soglie.

* **Frequenze separate e SSID unificato:** Le bande 2.4 GHz e 5 GHz condividono lo stesso SSID per permettere il band-steering (spostare i client capaci sulla banda a 5 GHz, più veloce).
* **Controllo della Potenza di Trasmissione (Tx Power):** Ho ridotto leggermente la potenza di trasmissione della banda a 2.4 GHz su tutti gli AP per fare in modo che la cella a 2.4 GHz non fosse eccessivamente grande rispetto a quella a 5 GHz, incoraggiando i client a preferire la banda a 5 GHz quando si trovano nella stessa stanza.

Le configurazioni reali commentate sono disponibili nella cartella principale del progetto:
* [`/configs/wireless_roaming.conf`](../configs/wireless_roaming.conf): Abilitazione di 802.11k/v/r e chiavi di mobilità (`nasid`, `mobility_domain`).
* [`/configs/usteer.conf`](../configs/usteer.conf): Soglie RSSI, tempi di blocco probe e intervalli di aggiornamento del demone `usteer`.
