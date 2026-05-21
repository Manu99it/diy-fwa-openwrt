# Collegamento Point-to-Point (P2P) Wireless Outdoor in Modalità WDS Trasparente

Questo documento descrive la progettazione e la configurazione del ponte radio a corto raggio (5 GHz) utilizzato per estendere la connettività LAN dall'abitazione principale verso le aree esterne: la **piscina** e la **dependance** adiacente.

L'installazione di cavi in rame (Ethernet) o fibra ottica interrati non era praticabile a causa della pavimentazione esterna esistente e dei costi di scavo. La soluzione ottimale è stata la creazione di un bridge wireless dedicato ad alte prestazioni.

---

## 📡 WDS (Wireless Distribution System) vs Client Mode Tradizionale

Per fare in modo che i dispositivi situati nella dependance e vicino alla piscina (telecamere di sicurezza, sensori IoT, smartphone di ospiti) facessero parte della **stessa identica rete (subnet)** della casa principale, era fondamentale implementare un bridge trasparente a Layer 2 (Data Link).

```
[Casa Principale]                                                     [Piscina / Dependance]
  Cudy WR3000H                                                            Switch Esterno
       │                                                                        │
  [MikroTik SXTsq]  . . . . (Link Wireless 5GHz DFS - 80 MHz) . . . . .   [MikroTik SXTsq]
  (AP Master WDS)                                                         (Client Slave WDS)
```

### Il limite del Wi-Fi standard (3-Address Frame)
Nello standard Wi-Fi standard (802.11), i pacchetti trasmessi contengono solo 3 indirizzi MAC (Sorgente, Destinazione, Access Point). Se un router client convenzionale si collega a un AP, non può far transitare i pacchetti dei dispositivi ad esso collegati mantenendo intatti i loro indirizzi MAC originali. Questo richiede l'uso di:
* **NAT (Network Address Translation):** crea una sottorete separata, bloccando i protocolli di scoperta locale (mDNS, Chromecast, AirPlay, stampanti di rete).
* **Relayd (ARP Spoofing pseudo-bridge):** un hack software che simula un bridge ma introduce instabilità, overhead di CPU ed errori con alcuni client DHCP.

### La soluzione: WDS a 4 Indirizzi (4-Address Mode)
Abilitando l'opzione **WDS** (chiamata anche *4-address mode*) sia sull'AP trasmittente (Master) e sul router ricevente (Slave) in OpenWrt, i frame Wi-Fi includono un quarto indirizzo MAC (quello del trasmettitore wireless intermedio). 

Questo trasforma il collegamento wireless in un **cavo Ethernet virtuale**. Tutti i pacchetti Ethernet transitano intatti, preservando i MAC address originali di ogni dispositivo client all'esterno.
* Il server DHCP sul router principale (Cudy) assegna direttamente gli IP ai dispositivi della piscina.
* I servizi di rete locali (es. Home Assistant, integrazioni domotiche, telecamere smart) funzionano senza bisogno di routing aggiuntivo o configurazione di porte.

---

## 🗼 Scelta dell'Hardware: MikroTik SXTsq 5 ac e Flashing OpenWrt

Per la realizzazione fisica del link ho scelto due antenne outdoor **MikroTik SXTsq 5 ac**, apparati compatti e resistenti alle intemperie con antenna integrata da 16dBi a 5GHz.

### Il bypass delle limitazioni di Licenza RouterOS
Di fabbrica, questi dispositivi sono venduti con il sistema operativo **RouterOS** e una **licenza di Livello 3 (CPE)**. 
Nelle policy di MikroTik, una licenza di Livello 3 limita l'utilizzo dell'interfaccia wireless: il dispositivo può fungere solo da client (station) verso un altro AP, oppure effettuare un bridge punto-punto proprietario, ma **non può agire come Access Point standard** per creare un link WDS aperto o servire più dispositivi a meno di non acquistare una licenza superiore (Livello 4), con costi aggiuntivi.

Per aggirare questo vincolo software e standardizzare l'infrastruttura su un unico stack open-source, ho deciso di **flashare OpenWrt** su entrambi i dispositivi MikroTik.

### Il processo di Flashing su MikroTik
L'installazione di OpenWrt su hardware MikroTik richiede passaggi più complessi rispetto a un classico router consumer:
1. **Configurazione TFTP / Netboot:** Ho configurato sul PC una scheda di rete con un server DHCP/TFTP temporaneo per caricare l'immagine initramfs di OpenWrt tramite boot da rete (PXE).
2. **Bootloader Access (RouterBOOT):** All'avvio del MikroTik, tenendo premuto il tasto di reset, ho forzato il dispositivo ad entrare in modalità bootloader di rete per cercare un server DHCP e scaricare il kernel via TFTP.
3. **Flashing Permanente:** Una volta che il dispositivo ha eseguito il boot temporaneo in RAM di OpenWrt, ho effettuato l'accesso SSH su `/tmp`, eseguito il backup della partizione di configurazione originale e flashato l'immagine sysupgrade squashfs definitiva sulla memoria flash interna (`mtd write`).

Grazie a questa modifica, i due MikroTik SXTsq 5 ac beneficiano del driver open-source `mac80211` integrato nel kernel Linux di OpenWrt, permettendo la configurazione nativa di WDS AP e WDS Client senza alcun costo di licenza o vincolo proprietario.

---

## 🔧 Parametri di Configurazione e Ottimizzazione Radio

Per garantire un collegamento stabile, a bassa latenza e ad alta velocità, sono stati impostati i seguenti parametri radio nella frequenza a 5 GHz:

1. **Selezione della Frequenza (Canali DFS):**
   * Ho configurato il link su canali DFS (Dynamic Frequency Selection, canali dal 52 al 140). Questi canali consentono una potenza di trasmissione (EIRP) massima superiore (fino a 1000mW / 30dBm in Europa) rispetto ai canali non-DFS (200mW), fondamentale per superare l'attenuazione dell'aria e della vegetazione esterna.
   * *Nota di sicurezza:* OpenWrt gestisce automaticamente lo scan radar obbligatorio prima di trasmettere su frequenze DFS per evitare interferenze con i radar meteorologici o aerei.

2. **Ampiezza del Canale (Channel Width):**
   * Impostata a **80 MHz**. Questo consente di ottenere tassi di allineamento fisici (PHY Rate) fino a 866 Mbps (su standard 802.11ac) o superiori (su standard 802.11ax/Wi-Fi 6), traducendosi in una velocità reale di trasferimento dati (throughput) di oltre **450-500 Mbps simmetrici**.

3. **Isolamento Wireless (Client Isolation):**
   * Disabilitato sul link P2P per permettere la comunicazione tra i nodi, ma abilitato sull'Access Point locale della piscina per i dispositivi ospiti, garantendo la sicurezza informatica dei dati aziendali/personali della casa principale.

---

## 📶 Risultati e Latenza

* **Latenza del Link (Ping verso il Gateway):** < 1.8 ms stabili, senza jitter significativo.
* **Resistenza agli Agenti Atmosferici:** La stabilità del segnale (RSSI mantenuto intorno a -60 dBm) garantisce zero perdita di pacchetti anche in caso di forte pioggia o vento.
* **Trasparenza Totale:** Le telecamere smart TP-Link Tapo e i sensori IoT installati nell'area piscina comunicano in modo trasparente con i relativi server locali e cloud, garantendo flussi video stabili e reattività istantanea dei comandi domotici senza alcuna perdita di frame.
