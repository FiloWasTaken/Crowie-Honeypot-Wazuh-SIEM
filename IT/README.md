Documentazione tecnica del setup di un honeypot SSH/Telnet basato su **Cowrie**, monitorato tramite **Wazuh SIEM**, esposto su una VPS pubblica per la raccolta e l'analisi di attacchi reali provenienti da internet.

> **Disclaimer**: questo progetto è a scopo di ricerca/apprendimento in ambito cybersecurity (threat intelligence, detection engineering). L'honeypot non fornisce accesso a sistemi di produzione e non contiene dati sensibili propri; contiene però dati reali raccolti da attacchi effettivi (IP sorgente, credenziali tentate, comandi eseguiti dagli attaccanti), essendo esposto su una VPS pubblica su internet.

---

## Indice

- [Obiettivo del progetto](#obiettivo-del-progetto)
- [Architettura](#architettura)
- [Stack tecnologico](#stack-tecnologico)
- [Setup — Cowrie](#setup--cowrie)
- [Setup — Wazuh](#setup--wazuh)
- [Integrazione log Cowrie → Wazuh](#integrazione-log-cowrie--wazuh)
- [Regole e Decoder](#regole-e-decoder)
- [Verifica del funzionamento](#verifica-del-funzionamento)
- [Lookup IP e analisi dei comandi eseguiti](#lookup-ip-e-analisi-dei-comandi-eseguiti)
- [Analisi degli attacchi raccolti](#analisi-degli-attacchi-raccolti)
- [Considerazioni di sicurezza](#considerazioni-di-sicurezza)
- [Lezioni apprese / Limiti](#lezioni-apprese--limiti)
- [Possibili sviluppi futuri](#possibili-sviluppi-futuri)

---

## Obiettivo del progetto

L'obiettivo di questo progetto è simulare un servizio SSH/Telnet vulnerabile per:

- Osservare e raccogliere tentativi di accesso non autorizzato provenienti da internet (credential stuffing, brute force, comandi post-login).
- Centralizzare e analizzare i log dell'honeypot tramite un SIEM (Wazuh), per abituarsi a un flusso realistico di **detection engineering** (log ingestion → parsing → alerting).
- Costruire un piccolo caso di studio da poter mostrare in un contesto di portfolio/CV.

---

## Architettura

Cowrie e Wazuh (Manager + Dashboard) sono installati **sulla stessa macchina/VM**, una VPS esposta pubblicamente su internet.

```
                        INTERNET (attaccanti reali)
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   VPS pubblica          │
                    │                         │
                    │  ┌───────────────────┐  │
                    │  │ Cowrie (honeypot) │  │
                    │  │ SSH / Telnet      │  │
                    │  └────────┬──────────┘  │
                    │           │ log JSON     │
                    │           ▼              │
                    │  ┌───────────────────┐  │
                    │  │ Wazuh Manager     │  │
                    │  │ (localfile)       │  │
                    │  └────────┬──────────┘  │
                    │           ▼              │
                    │  ┌───────────────────┐  │
                    │  │ Wazuh Dashboard   │  │
                    │  └───────────────────┘  │
                    └─────────────────────────┘
```
Wazuh-Manager:
<br>
<img width="1354" height="567" alt="image" src="https://github.com/user-attachments/assets/9bb60f23-4fae-45cc-a1a4-278016a927cc" />

Cowrie:
<br>
<img width="569" height="84" alt="image" src="https://github.com/user-attachments/assets/b3eb4c87-7dc2-41b2-b57f-999e80cea6e9" />


---

## Stack tecnologico

| Componente | Ruolo |
|---|---|
| **Cowrie** | Honeypot SSH/Telnet a media interazione, simula una shell e un filesystem falsi |
| **Wazuh Manager** | Ingestion, parsing (decoder/rule) e generazione alert dai log di Cowrie |
| **Wazuh Dashboard** | Visualizzazione centralizzata degli alert e dei log |
| **VPS pubblica** | Hosting di entrambi i servizi, esposta su internet per raccogliere attacchi reali |

Stack volutamente minimale: nessuna integrazione aggiuntiva (no TheHive, no Shuffle, no notifiche esterne) — focus su honeypot → SIEM.

---

## Setup — Cowrie

*(Cowrie è già installato e configurato su questa macchina. Sezione descrittiva del setup esistente, non uno step-by-step da rieseguire.)*

- Installazione in ambiente virtuale Python dedicato (utente non privilegiato, best practice raccomandata da Cowrie per limitare l'impatto in caso di escape dal sandbox).
- Configurazione principale in `cowrie.cfg` (porte in ascolto, hostname/fake system finto, filesystem simulato).
- Log applicativi generati in formato **JSON** (`var/log/cowrie/cowrie.json`), oltre al log testuale classico (`cowrie.log`).
- Porta SSH reale della VPS spostata su una porta non standard (2222), con Cowrie in ascolto sulla porta 22 "esca" per attirare gli scanner automatici e i bot che scansionano il default.

<img width="1740" height="64" alt="image" src="https://github.com/user-attachments/assets/da37add9-4ce7-4bd4-8fc8-ad6c9f8ac258" />

Output di `cowrie.json` che mostra un tentativo di login catturato (username, password, IP sorgente).
</screenshot>

---

## Setup — Wazuh

Gurdare dei log JSON su un terminale è abbasstanza brutto, quindi ci installiamo un SIEM 

*(Anche Wazuh è già installato e configurato. Sezione descrittiva, non guida all'installazione.)*

- Wazuh Manager e Dashboard installati sulla stessa VPS dell'honeypot.
- Configurazione centrale in `/var/ossec/etc/ossec.conf`.
- Agente locale (self-monitoring) attivo di default sul manager stesso, usato anche per leggere i log di Cowrie tramite `localfile`.

Regole di creazione per gli alert di Cowrie (Nuovo accesso e comandi eseguiti una volta dentro)
<screenshot>
Wazuh Dashboard che mostra l'agent/host online e sano (stato "Active").
</screenshot>

---

## Integrazione log Cowrie → Wazuh

L'integrazione è stata realizzata tramite direttiva **`<localfile>`** nel file `ossec.conf` di Wazuh, puntata direttamente al file di log JSON generato da Cowrie:

<img width="588" height="98" alt="image" src="https://github.com/user-attachments/assets/a49cdeb0-33fd-4437-9f5c-f5fdf2a86d21" />


Questo permette a Wazuh di leggere ogni nuova riga del log Cowrie in tempo reale e passarla alla pipeline di parsing/decoding interna, senza bisogno di agenti aggiuntivi (Filebeat) o forwarding syslog: essendo Cowrie e Wazuh sulla stessa macchina, la lettura diretta del file è la soluzione più semplice ed efficace.

<screenshot>
Estratto di `ossec.conf` con il blocco `<localfile>` configurato (conferma della configurazione realmente in uso).
</screenshot>

---

Durante la configurazione del tutto ho riscontrato un problema ovvero che i log di cowrie venivano letti dal SIEM ma non riuscivo a vederli sulla dashboard perchè non c'era nessuna regola per quanto 
riguardava gli alert dato che eventi e alert sono due cose ben diverse, quindi ho creato due regole per poter generare gli eventi una per il login avvenuto con successo (Cowrie accetta ogni credenziale) 
ed una per vedere che comandi sono stati eseguiti una volta che il bot è entrato nell honeypot, dico bot perchè di solito sono dei computer/VPS infetti con script automatici che fanno brute force, una volta dentro eseguono uno o due comandi e questo informazioni vengono mandate all'attaccante vero e proprio.

Regola per login:
<br>
<img width="689" height="169" alt="image" src="https://github.com/user-attachments/assets/aa207c8b-ed7d-4143-b0bb-5be1ecad6ac7" />

Regola per comando eseguito:
<br>
<img width="829" height="252" alt="image" src="https://github.com/user-attachments/assets/4d79dfb5-949b-4bd0-a998-5e490fbb7008" />


Dopo aver lasciato qualche giorno la porta 22 aperta possiamo vedere che abbiamo ricevuto all'incrica 12,900 alerts:

<img width="1486" height="512" alt="image" src="https://github.com/user-attachments/assets/0d78662b-39ec-42ed-935c-a2fb8411f2cb" />


---

## Verifica del funzionamento

Questa sezione documenta le prove che l'intera pipeline (Cowrie → log → Wazuh → alert) funziona correttamente end-to-end, con attacchi reali intercettati dall'honeypot esposto su internet.

### 1. Tentativi di connessione intercettati da Cowrie


Log che mostra una sessione SSH in ingresso da un IP esterno reale (con timestamp e IP sorgente visibili).
<br>
<img width="1465" height="532" alt="image" src="https://github.com/user-attachments/assets/f9b054b5-4941-48aa-9a03-6bf90aad2dd4" />

### 2. Comandi eseguiti dall'attaccante nella shell finta

Andiamo a filtrare i log per i comandi eseguiti da questo indirizzo IP con qeusta query: rule.id: 100200 and data.src_ip: "141.11.88.243"
dove rule.id va a specificare la regola che abbiamo creato prima e il secondo campo filtra per l'indirizzo IP che ci interessa


### 4. Alert generato su Wazuh Dashboard
I timestamp potrebbero non combaciare perchè il bot ha provate molte volte ad entrare nel server.
<img width="1482" height="475" alt="image" src="https://github.com/user-attachments/assets/4885728d-a758-42b3-9fec-4bb59dbde9d5" />


## Lookup IP e analisi dei comandi eseguiti

Una volta raccolti gli eventi da Cowrie, ho svolto un'attività manuale di analisi su alcuni IP e comandi ritenuti più interessanti, per capire meglio la natura degli attacchi ricevuti.

### 1. Lookup degli indirizzi IP attaccanti

Prendiamo l'indirizzo IP che abbiamo negli screenshot, facendo un lookup su AbuseIPDB possiamo vedere che questo indirizzo è malevolo:
<br>
<img width="1556" height="786" alt="image" src="https://github.com/user-attachments/assets/2eb47663-981a-42c1-b7fa-5ca48a578125" />


### 2. Comandi eseguiti dagli attaccanti nella shell simulata

Analisi dei comandi effettivamente digitati dagli attaccanti una volta "entrati" nella shell finta di Cowrie (utile per capire l'intento: reconnaissance, tentativo di download di malware, tentativo di persistenza, ecc.).
Possiamo vedere che il comando eseguito è uname -s -v -n -r -m, comando usato per reperire informazioni sulla macchina appena "bucata" molto probabilmente poi queste informazioni verranno mandate all attacante.

### 3. Ricerca di un comando specifico eseguito da un IP specifico

Nel caso di prima il comando eseguito è stato solamente uname -s -v -n -r -m, ma se vogliamo vedere dei comandi specifici ?
La seguente query ce lo permette: rule.id: 100200 and <Comando>,
ad esempio:
rule.id: 100200 and whoami ci restituisce il seguente risultato:
<img width="1509" height="446" alt="image" src="https://github.com/user-attachments/assets/caaa2b3c-5316-41e2-bb3d-d78f0cd8269f" />

---

## Considerazioni di sicurezza

- L'honeypot è isolato logicamente dal resto del sistema (utente dedicato non privilegiato, sandbox Cowrie).
- La porta SSH reale della VPS è stata spostata su una porta non standard per separarla da quella "esca" di Cowrie.
- Wazuh, essendo sulla stessa macchina esposta, rappresenta un potenziale punto di attacco secondario: va monitorato l'accesso al Dashboard (autenticazione, firewall sulle porte 443/55000) per evitare che diventi esso stesso un vettore di compromissione.
- Nessun dato reale o credenziale valida è presente sulla macchina.
