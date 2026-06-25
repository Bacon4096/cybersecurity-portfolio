# Relazione Tecnica di Audit e Troubleshooting: Progetto "Complex Network"

Questo report descrive dettagliatamente lo scopo e i processi del progetto "Complex Network", realizzato personalmente e reso pubblico alla consultazione (fare riferimento al file `README.md` per le modalità di accesso).

La rete "Complex Network", concepita originariamente come progetto in sede di stage aziendale, è stata sviluppata con lo scopo di applicare comandi avanzati sui dispositivi di rete al fine di interconnetterli in modo efficiente. L'infrastruttura integra dispositivi IoT gestibili da qualsiasi workstation aziendale previo collegamento al server dedicato, diversificando la topologia in molteplici VLAN per mitigare i colli di bottiglia causati dai messaggi di broadcast.

A distanza di un anno, il progetto è stato riesaminato approfonditamente, riscontrando notevoli criticità di sicurezza e falle logiche che, in un contesto reale, minerebbero l'integrità strutturale dell'azienda se sfruttate da un malintenzionato (es. possibilità di movimento laterale dal segmento Wi-Fi Ospiti verso le workstation di produzione, con conseguente accesso non autorizzato al server e ai dispositivi IoT). Sebbene l'ambiente Cisco Packet Tracer presenti evidenti limitazioni intrinseche in alcune funzioni avanzate — quali la generazione di chiavi RSA stabili e la gestione della crittografia — lo strumento si rivela estremamente efficace per lo studio delle telecomunicazioni e delle reti, consentendo l'esame dettagliato del percorso dei pacchetti di dati.

Questo report si propone di analizzare le criticità del sistema precedente e di illustrare il procedimento risolutivo applicato per rendere la rete sicura, resiliente e frammentata in modo efficiente (fare riferimento al file `config.me` per la consultazione delle configurazioni correnti dei dispositivi di rete e topologia).

---

## 1. Analisi dello Stato del Firewall Perimetrale (Cisco ASA 5506-X)

L'analisi dell'apparato perimetrale tramite il comando `show interface ip brief` ha permesso di mappare le zone di sicurezza fisiche e di evidenziare una vulnerabilità strutturale nel posizionamento della rete ospiti.

### 1.1 Tratte di Collegamento Attive
Il firewall opera su due interfacce fisiche principali:
* **GigabitEthernet1/1 (IP 192.168.254.1):** Interfaccia logica interna (`inside`). Funge da Next-Hop per la default route del router centrale.
* **GigabitEthernet1/2 (IP 209.165.200.226):** Interfaccia logica esterna (`outside`), instradata verso la rete pubblica (Modem/Internet).

### 1.2 Evidenze di Audit: Mancanza di una Zona Perimetrale per gli Ospiti
Dall'output emerge che le interfacce dalla `GigabitEthernet1/3` alla `GigabitEthernet1/8` risultano disattivate e prive di indirizzamento IP. 

* **Criticità Rilevata:** La rete ospiti (`GUEST_WIFI` - VLAN 60) non è attestata su un'interfaccia fisica o logica dedicata del firewall (es. una zona `guest` con Security Level intermedio). Di conseguenza, l'ASA non ha visibilità diretta sul traffico proveniente dai laptop ospiti se questo è diretto verso le altre sottoreti aziendali (VLAN 10, VLAN 40).
* **Impatto sulla Sicurezza:** Il controllo degli accessi e l'isolamento degli ospiti gravano interamente sul router centrale. Se sul centro stella non sono applicate ACL restrittive, un attaccante sulla rete Guest può effettuare movimenti laterali e ricognizione sulla LAN aziendale, bypassando completamente i controlli statali e le policy d'ispezione del firewall ASA.

---

## 2. Analisi delle Policy di Sicurezza (Audit delle Access Control List)

L'ispezione delle regole di filtraggio sull'ASA tramite `show access-list` ha identificato il motivo tecnico per cui i servizi di rete (come la risoluzione DNS di `cisco.com`) risultavano interrotti, a fronte di un protocollo ICMP (Ping) perfettamente funzionante.

### 2.1 Analisi della ACL "ACL_PER_NAT" (Traffico Outbound)
L'elenco contiene le direttive per consentire l'accesso outbound a tutte le subnet della rete aziendale:
* Line 1 (VLAN 10): `permit ip 192.168.1.0 255.255.255.0 any` -> `hitcnt=0`
* Line 2 (VLAN 20): `permit ip 172.16.1.0 255.255.255.0 any` -> `hitcnt=0`
* Line 3 (VLAN 30): `permit ip 10.1.1.0 255.255.255.0 any` -> `hitcnt=0`
* Line 4 (VLAN 40): `permit ip 192.168.40.0 255.255.255.0 any` -> `hitcnt=0`
* Line 5 (VLAN 60): `permit ip 192.168.60.0 255.255.255.0 any` -> `hitcnt=0`

**Evidenza di Audit:** Il contatore `hitcnt=0` su tutte le linee dimostra che questa lista non sta filtrando attivamente il traffico in ingresso sull'interfaccia interna, confermando che il posizionamento delle regole di sicurezza perimetrali è disallineato rispetto al flusso reale dei pacchetti.

### 2.2 Analisi della ACL "OUTSIDE_IN" (Traffico Inbound)
L'interfaccia perimetrale esterna ha una sola regola attiva:
* `access-list OUTSIDE_IN line 1 extended permit icmp any any (hitcnt=11)`

**La Causa del Disservizio:** Il contatore attivo (`hitcnt=11`) dimostra che il firewall permette esclusivamente il transito dei pacchetti ICMP. Qualsiasi altro pacchetto di risposta proveniente da Internet — come i pacchetti di ritorno delle query DNS (porta 53 UDP) o le risposte HTTP/HTTPS — viene implicitamente scartato dall'ASA all'interfaccia `outside`, provocando il timeout sistematico dei comandi come `nslookup cisco.com` lato client.

### 2.3 Verifica del Modular Policy Framework (MPF) ed Ispezione Stateful
Il comando `show service-policy` non ha restituito output o policy attive sull'apparato perimetrale.

* **Evidenza di Audit:** L'assenza di una `global_policy` attiva indica che il firewall non sta eseguendo l'ispezione dinamica del traffico (Stateful Inspection) per i protocolli applicativi critici, in particolare il DNS (`inspect dns`) e l'HTTP/HTTPS (`inspect http`).
* **Conseguenza Tecnica:** Senza l'ispezione stateful, l'ASA non è in grado di tracciare le sessioni UDP/TCP outbound generate dai client interni. Di conseguenza, i pacchetti di risposta legittimi provenienti dai server esterni vengono considerati come traffico inbound non richiesto e vengono droppati implicitamente all'interfaccia Outside, confermando il blocco totale della risoluzione dei nomi sul dominio `cisco.com`.

---

## 3. Analisi della Configurazione Lato Client (End-Point Audit)

L'audit finale è stato condotto direttamente sull'host della LAN tramite il comando `ipconfig /all`, al fine di verificare la correttezza dei parametri distribuiti ai client della rete.

### 3.1 Parametri di Rete Rilevati
* **IPv4 Address:** `192.168.1.7` (Subnet `255.255.255.0`) -> L'host è correttamente instradato all'interno della VLAN 10.
* **Default Gateway:** `192.168.1.1` -> Il puntamento al centro stella per l'inter-VLAN routing è valido e attivo.
* **DNS Servers:** `208.67.220.220` -> Il client punta direttamente all'IP pubblico del server esterno per la risoluzione dei domini.

### 3.2 Valutazione del Flusso Logico del Disservizio
L'host presenta tutti i parametri nominali corretti per navigare. Quando l'utente tenta di raggiungere `cisco.com`:
1. Il client genera una query DNS verso l'IP pubblico `208.67.220.220`.
2. Il pacchetto viene inoltrato al gateway `192.168.1.1` (Router1) e da lì instradato via default route verso l'interfaccia interna dell'ASA (`192.168.254.1`).
3. Il firewall effettua il NAT e inoltra la richiesta verso la WAN.
4. Il server esterno risponde correttamente, ma la risposta UDP viene scartata all'interfaccia esterna dell'ASA a causa dell'assenza di regole di `permit` per la porta 53 nella ACL `OUTSIDE_IN` e della mancanza di un meccanismo di ispezione dinamica globale (`show service-policy` vuoto).

Questo disallineamento sul firewall perimetrale rappresenta la causa primaria del timeout applicativo riscontrato lato client.

---

## 4. Piano di Remediation e Ripristino dei Servizi

Di seguito i processi necessari al Ripristino dei sistemi:

### 4.1 Ripristino della Risoluzione DNS e Navigazione Web

Per consentire ai pacchetti di risposta `DNS (porta 53)` e `Web (80/443)` di rientrare verso i client interni senza esporre la rete a minacce esterne, è stata riattivata l'ispezione stateful sull'`ASA`, integrando le Access Control List di rientro per i server autorizzati.

Configurazione applicata sulla CLI dell'**ASA 5506-X**:

   * ciscoasa# configure terminal

1. Ripristino dell'Ispezione Stateful Globale (`Modular Policy Framework`)

   * ciscoasa(config)# `policy-map global_policy`
   * ciscoasa(config-pmap)# class `inspection_default`
   * ciscoasa(config-pmap-c)# inspect `dns`
   * ciscoasa(config-pmap-c)# inspect `http`
   * ciscoasa(config-pmap-c)# inspect `https`
   * ciscoasa(config-pmap-c)# exit
   * ciscoasa(config-pmap)# exit

2. Aggiornamento della `ACL` esterna per consentire il rientro del traffico `DNS`
   * ciscoasa(config)# access-list `OUTSIDE_IN` extended permit udp host `208.67.220.220` any eq 53
   * ciscoasa(config)# access-list `OUTSIDE_IN` extended permit tcp host `208.67.220.220` any eq 53
   * ciscoasa(config)# access-group `OUTSIDE_IN` in interface outside
   * ciscoasa(config)# write memory

### 4.2 Hardening e Isolamento della Rete Ospiti (VLAN 60)

In attesa di un refactoring fisico che attesti la `VLAN 60` su una porta dedicata del firewall, si interviene direttamente sul Router Centrale (centro stella) tramite `Access Control List` estese per bloccare alla radice i tentativi di movimento laterale verso i segmenti di produzione.

Configurazione applicata sulla CLI del Router1:

1. Router1# configure terminal

* access-list 160 deny ip `192.168.60.0 0.0.0.255` `192.168.1.0 0.0.0.255`
* access-list 160 deny ip `192.168.60.0 0.0.0.255` `192.168.40.0 0.0.0.255`
* access-list 160 permit ip any any
* interface Vlan60
* ip access-group 160 in
* exit
* write memory

### 4.3 Risultati Attesi Post-Intervento

Il comando `nslookup` cisco.com eseguito dalle postazioni interne risolve nominalmente i domini esterni senza interruzioni, grazie alla tracciabilità dinamica delle sessioni introdotta dal Modular Policy Framework.
I tentativi di scansione o ricognizione (`Ping` o `Port Scan`) originati dal segmento `GUEST_WIFI` verso le macchine aziendali vengono intercettati e scartati all'ingresso del router, azzerando la superficie di attacco interna.

## 5. Nota di Controllo Architetturale: Flusso WAN vs Infrastruttura Locale

    Un'analisi superficiale della topologia visiva in Packet Tracer potrebbe indurre a pensare che l'host interno e il server esterno comunichino semplicemente perché posizionati all'interno dello stesso spazio di lavoro virtuale. Dal punto di vista logico e sistemistico, questa assunzione è errata, vediamo come:

### 5.1 Dimostrazione del Transito Geografico (WAN Path)

La connettività `end-to-end` è garantita esclusivamente dall'attraversamento di una catena di `routing` e sicurezza che simula una vera infrastruttura geografica (`Wide Area Network`):

* `Isolamento` degli Indirizzamenti: Il client opera su uno spazio privato (`192.168.1.0/24`), non instradabile sulla rete pubblica globale. Il server risiede su una rete pubblica remota (`208.67.220.0/24`).

* Meccanismo di Frontiera (`NAT/PAT`): Il `firewall AS`A intercetta il pacchetto privato e ne modifica l'`header`, traducendo l'`IP sorgente` nell'`IP pubblico` della tratta outside (`209.165.200.226`), rendendolo idoneo al transito su Internet.

* Infrastruttura dell'`ISP` (`Router0`): Il pacchetto attraversa il `modem DSL` e la `nuvola Internet` per essere preso in carico dal router del provider (`Router0`). Questo apparato instrada il traffico verso il `server` e, grazie alle sue rotte statiche di ritorno, sa come re-instradare le risposte verso l'`IP pubblico` del firewall aziendale.

La disattivazione dell'interfaccia esterna dell'`ASA` o la rimozione delle rotte sul `Router0` provocherebbero l'`isolamento` immediato del `server`, confermando l'`indipendenza` logica e geografica delle due reti.

### 6. Implementazione e Logica della Vera DMZ (DeMilitarized Zone)

Per elevare i requisiti di sicurezza dell'infrastruttura a uno standard `Enterprise`, è stata introdotta una terza area di sicurezza isolata (`DMZ fisica`) direttamente sul Firewall `ASA 5506-X`, attestata sull'interfaccia fisica `GigabitEthernet1/3`.
## 6.1 Configurazione Logica della DMZ

* Subnet Dedicata: 192.168.50.0/24 (IP Gateway attestato sull'interfaccia ASA: 192.168.50.1).

* Livello di Sicurezza (Security Level): Configurato a 50. Questa impostazione gerarchica posiziona la DMZ in una zona intermedia di transizione: possiede un livello di fiducia inferiore rispetto alla LAN interna (security-level 100), ma superiore rispetto al perimetro pubblico di Internet (security-level 0).

* Host Attestati: Un server dedicato (Server_DMZ) configurato con indirizzamento IP statico 192.168.50.2/24 e Default Gateway 192.168.50.1.

## 6.2 Politica di Accesso Selettivo (Access Control Policy)

In conformità con il principio del minimo privilegio (`Least Privilege`), il transito inter-zona non è stato aperto indiscriminatamente all'intera infrastruttura aziendale. L'accesso alle risorse residenti in `DMZ` è stato mappato tramite direttive esplicite sul `firewall`:

* Segmenti Autorizzati: Esclusivamente la Core `LAN` (`VLAN 10`, subnet `192.168.1.0/24`), consentendo all'endpoint di gestione usr:jakelafuria (`192.168.1.7`) l'amministrazione e l'interrogazione dei servizi applicativi.

* Isolamento dei Segmenti Non Autorizzati: Le restanti `VLAN` aziendali (Uffici, Ospiti, IoT) non possiedono regole di permit esplicite verso la rete `192.168.50.0/24`. I tentativi di connessione vengono pertanto scartati implicitamente dall'`ASA` (`Implicit Deny`), riducendo drasticamente la superficie di attacco interna.

* Meccanismo di Ritorno `Stateful`: L'attivazione della `global_policy` e dei meccanismi di ispezione dinamica sull'`ASA` permette il transito automatico dei pacchetti di risposta dal `Server DMZ` verso la `VLAN 10`, escludendo la necessità di configurare `Access Control List` bidirezionali permissive e potenzialmente pericolose.

### 7. Risoluzione Anomalie (Troubleshooting DMZ)

    Sono state riscontrate numerose anomalie, molte delle quali a causa delle limitazioni del software Cisco Packet Tracer stesso, ma garantisco che è stato fatto il possibile per rendere l'esperienza quanto più simulativa possibile.

## 7.1 Sintomatologia Rilevata

Successivamente all'attestazione del nodo `Server_DMZ` sull'interfaccia `GigabitEthernet1/3` dell'`ASA`, i tentativi di connessione `HTTP/HTTPS `originati dal client `usr:jakelafuria` (`192.168.1.7`) verso l'IP della `DMZ` (`192.168.50.2`) fallivano sistematicamente in `timeout` applicativo, nonostante l'avvenuta compilazione delle regole di classificazione del traffico.

## 7.2 Root Cause Analysis (Analisi delle Cause)

L'analisi dei flussi logici e dei registri dell'`ASA` ha evidenziato due problematiche architetturali concorrenti:

* Asimmetria di `Routing Interno`: L'ASA non possedeva nel proprio thin client di instradamento una rotta di ritorno esplicita per raggiungere la subnet della `Core LAN` (`192.168.1.0/24`), i cui segmenti non sono direttamente connessi al firewall ma attestati dietro il centro stella (Router1 con IP di Next-Hop `192.168.254.2`). Questo provocava lo scarto sistematico del traffico di ritorno.

* Interferenza delle Direttive di `Egress NAT`: La policy di `NAT dinamico globale` configurata sull'interfaccia `outside` per consentire l'uscita su Internet intercettava erroneamente anche i flussi destinati al segmento `dmz`. La conseguente alterazione degli `header` dei pacchetti comprometteva l'integrità del flusso e la coerenza delle tabelle dello `stateful inspection`.

## 7.3 Soluzione Applicata (NAT Identity / Exempt e Routing Fix)

Per risolvere l'anomalia, è stata implementata una regola di `NAT Twice `(denominata `NAT Exempt o NAT Identity`) per escludere il traffico `LAN-to-DMZ` da qualsiasi processo di traduzione, parallelamente all'inserimento della corretta rotta di `Next-Hop` interna.

Sintassi di configurazione applicata sulla CLI dell'ASA 5506-X:

1. ciscoasa# configure terminal

* object network `obj-lan`
* subnet `192.168.1.0 255.255.255.0`
* exit

* object network `obj-dmz`
* subnet `192.168.50.0 255.255.255.0`
* exit

* nat (`inside,dmz`) source static obj-lan obj-lan destination static obj-dmz obj-dmz

* route inside `192.168.1.0` `255.255.255.0` `192.168.254.2`

* write memory

### Appendice: Limitazioni dello Strumento di Emulazione e Anomalie Software

Durante le fasi di `testing avanzato` e l'integrazione della terza zona di sicurezza (`DMZ`), sono state riscontrate diverse incongruenze logiche non imputabili a errori macroscopici di configurazione, bensì a limitazioni strutturali e bug noti del software di simulazione Cisco Packet Tracer.
1. Instabilità del Modulo Cisco ASA (Legacy Emulation)

    Saturazione delle Tabelle di Stato: L'engine di ispezione stateful dell'ASA simulato tende a corrompere la tabella delle connessioni attive a seguito di ripetute modifiche sequenziali sulle ACL d'interfaccia. Tale fenomeno genera drop arbitrari sul traffico legittimo HTTP o DNS, anche in presenza di direttive permit esplicite.

    Incongruenze della Sintassi CLI: Alcune release software dell'apparato ASA implementate in Packet Tracer rifiutano costrutti standard di diagnostica perimetrale (come il comando packet-tracer input) o interpretano in modo errato la precedenza delle righe nelle regole di NAT Twice avanzate, limitando di fatto gli strumenti di troubleshooting a disposizione dell'operatore.

2. Corruzione dei Servizi Applicativi (DNS e HTTP) post-Power Cycle

    Mancata Sincronizzazione dei Processi: Al ripristino dei file di configurazione o in seguito a un riavvio hardware globale dell'ambiente virtuale (Power Cycle Devices), i demoni software integrati nei Server simulati (come il servizio DNS su cisco.com) mostrano uno stato grafico nominale impostato su ON nella GUI, pur non rispondendo alle reali richieste di risoluzione nomi inoltrate dai client della LAN.

    Workaround e Risoluzione: Tali anomalie software si bypassano esclusivamente forzando lo svuotamento manuale della cache locale dei resolver sui client (tramite il comando ipconfig /flushdns) o accelerando artificialmente il tempo di convergenza dell'ambiente virtuale tramite la funzione di Fast Forward Time.

3. Conclusioni Tecniche dell'Analizzatore

L'incremento della complessità topologica (introduzione di `VLAN multiple`, `instradamento asimmetrico` e `firewalling multilivello`) evidenzia il limite prettamente didattico dello strumento Packet Tracer.

In uno scenario reale operante su hardware fisico, o all'interno di ambienti di emulazione professionale basati su `hypervisor` (come EVE-NG o GNS3), i protocolli di routing e le tabelle di stato del firewall avrebbero mantenuto la coerenza logica end-to-end, senza la necessità di applicare rollback di configurazione o soluzioni di ripiego sulla cache dei client.

Vi ringrazio per l'attenzione, spero che la consultazione di questo materiale didattico vi sia utile.

Sayonara.
