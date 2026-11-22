# SIEM Lab Semplificato con ELK Stack e Docker — Guida Passo Passo

Guida pratica di **Security Information and Event Management (SIEM)** e centralizzazione dei log su Linux con **Elasticsearch**, **Kibana** (in container Docker) e **Filebeat** (sull'host).  
L'obiettivo di questo laboratorio è mostrare come raccogliere i log di sistema Linux, inviarli a un motore di ricerca centralizzato e visualizzarli in una dashboard per monitoraggio, analisi e triage di sicurezza.

> **Nota sul perimetro e sulla sicurezza:** Questo setup disabilita la sicurezza (`xpack.security.enabled=false`) per semplificare l'apprendimento in laboratorio locale. Non utilizzare mai questa configurazione in produzione o su reti esposte: in ambienti reali vanno abilitati autenticazione, TLS/HTTPS e controllo degli accessi.

---

## Cosa fa in breve un SIEM (e perché si parte da qui)

Immagina di dover investigare un tentativo di accesso sospetto o un errore di sistema su una macchina Linux: se i log restano sparsi in file diversi e senza un punto di consultazione unificato, la ricerca diventa lenta e frammentata.
* **Filebeat (raccolta):** legge i file di log dalla macchina host (come autenticazione e log di sistema) e li spedisce in modo affidabile.
* **Elasticsearch (archiviazione e indicizzazione):** riceve, indicizza e conserva i log per poterli cercare e filtrare velocemente.
* **Kibana (visualizzazione):** offre l'interfaccia web per esplorare gli eventi, filtrare per intervallo temporale e costruire viste/dashboard.

Partire con uno stack locale semplificato permette di capire il flusso completo dei dati in un SIEM senza perdersi subito nella complessità di un cluster multi-nodo di produzione.

---

## Architettura e Prerequisiti

### Prerequisiti
* **Docker:** versione `20.10.0` o successiva.
* **Docker Compose:** versione `v2.0.0` o successiva.
* **Permessi `sudo`:** necessari per installare e avviare Filebeat sull'host.

### Componenti del laboratorio

| Componente | Ruolo | Funzione nel laboratorio |
| :--- | :--- | :--- |
| **Elasticsearch** | Motore di ricerca e storage | Indicizza e archivia i log ricevuti (`8.14.0`, porta `9200`) |
| **Kibana** | Interfaccia web | Consente di esplorare i log e creare Data View (`8.14.0`, porta `5601`) |
| **Filebeat** | Log shipper sull'host | Raccoglie i log di sistema Linux e li invia a Elasticsearch (`8.14.0`) |
| **Docker & Compose** | Orchestrazione container | Gestisce rete, volumi e ciclo di vita di Elasticsearch e Kibana |

### Schema del flusso dati

```text
┌─────────────┐
│  Filebeat   │ (Log Collector sull'host)
└──────┬──────┘
       │
       │ Invia log (HTTP / 9200)
       ▼
┌──────────────────┐        ┌─────────┐
│ Elasticsearch    │◄──────►│ Kibana  │
│ (Index & Store)  │        │ (UI)    │
└──────────────────┘        └─────────┘
```
## Passo 1 — Preparare la Directory e Creare `docker-compose.yml`

### 1.1 Crea la cartella del progetto
```bash
mkdir elk-siem-lab
cd elk-siem-lab
```

### 1.2 Crea il file `docker-compose.yml`
Crea il file `docker-compose.yml` all'interno della cartella `elk-siem-lab`:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.0
    container_name: elasticsearch
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk-network

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - elk-network
    depends_on:
      - elasticsearch

volumes:
  esdata:

networks:
  elk-network:
    driver: bridge
```

### Come leggere le variabili principali
* **`xpack.security.enabled=false`**: disabilita le funzionalità di sicurezza di X-Pack per semplificare i test locali.
* **`discovery.type=single-node`**: avvia Elasticsearch in modalità a singolo nodo.
* **`ES_JAVA_OPTS=-Xms512m -Xmx512m`**: imposta memoria minima e massima della JVM a `512 MB` per contenere l'uso risorse in laboratorio.

---

## Passo 2 — Avviare lo Stack ELK

Dalla directory `elk-siem-lab`, avvia i container in background:

```bash
docker compose up -d
```

### Verifica del passaggio
Controlla lo stato dei servizi con:

```bash
docker compose ps
```
Entrambi i container (`elasticsearch` e `kibana`) devono risultare attivi (`Up`).

---
## Passo 3 — Installare, Configurare e Avviare Filebeat

Filebeat viene installato sulla macchina host per leggere i log di sistema e inviarli a Elasticsearch.

### 3.1 Installazione del pacchetto `.deb`
```bash
curl -L -O [https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb)
sudo dpkg -i filebeat-8.14.0-amd64.deb
```

### 3.2 Configurazione dell'output in `/etc/filebeat/filebeat.yml`
Apri `/etc/filebeat/filebeat.yml` con un editor (ad esempio `sudo nano /etc/filebeat/filebeat.yml`) e verifica che la sezione `output.elasticsearch` punti a `localhost:9200`, lasciando commentato `output.logstash`:

```yaml
# ========================== Outputs ==========================
output.elasticsearch:
  hosts: ["localhost:9200"]
  # username: "elastic"
  # password: "changeme"

# output.logstash:
#   hosts: ["localhost:5044"]
```

### 3.3 Abilitazione del modulo `system`
Abilita il modulo dedicato ai log di sistema Linux:

```bash
sudo filebeat modules enable system
```

### 3.4 Preparazione directory e permessi
Crea le cartelle necessarie e allinea proprietà e permessi:

```bash
sudo mkdir -p /var/lib/filebeat/registry/filebeat
sudo mkdir -p /var/log/filebeat

sudo chown -R root:root /etc/filebeat /var/lib/filebeat /var/log/filebeat
sudo chmod -R 755 /etc/filebeat /var/lib/filebeat /var/log/filebeat
```

### 3.5 Avvio e verifica del servizio Filebeat
```bash
sudo systemctl daemon-reload
sudo systemctl start filebeat
sudo systemctl enable filebeat
```

### Verifica del passaggio
Controlla stato e log recenti del servizio:

```bash
sudo systemctl status filebeat
sudo journalctl -u filebeat --since "2 minutes ago"
```

---

## Passo 4 — Configurare la Data View e Esplorare i Log in Kibana

### 4.1 Creare la Data View in Kibana
1. Apri il browser all'indirizzo `http://localhost:5601`.
2. Nel menu laterale sinistro, apri **Management** (icona dell'ingranaggio) → **Stack Management** → **Data Views** (o **Index Patterns** nelle versioni precedenti).
3. Clicca su **Create data view**.
4. In **Name / Index pattern**, inserisci `filebeat-*`.
5. Seleziona `@timestamp` come **Timestamp field**.
6. Conferma cliccando su **Create data view**.

### 4.2 Visualizzare i log in Discover
1. Nel menu laterale sinistro, apri **Analytics** (icona della bussola) → **Discover**.
2. Seleziona la data view `filebeat-*` e imposta l'intervallo temporale in alto a destra (es. ultimi 15 minuti) per osservare i log di sistema in arrivo.

---

## Troubleshooting Rapido

* **I log non compaiono in Kibana:**
  * Controlla lo stato di Filebeat: `sudo systemctl status filebeat`
  * Ispeziona i log di Filebeat: `sudo journalctl -u filebeat -n 50`
  * Verifica che Elasticsearch risponda su `9200`: `curl http://localhost:9200`
* **Problemi con i container Docker:**
  * Controlla i log dei servizi: `docker compose logs elasticsearch` e `docker compose logs kibana`
* **Errori legati a permessi o directory di Filebeat:**
  * Riesegui i comandi del **Passo 3.4** per ripristinare directory, proprietà e permessi.

---

## Competenze Dimostrate in Questo Laboratorio

* Orchestrazione di un ambiente di analisi log tramite **Docker Compose** con Elasticsearch e Kibana.
* Configurazione di **Filebeat** sull'host con modulo `system` e invio degli eventi verso Elasticsearch.
* Creazione di una Data View su `@timestamp` e esplorazione dei log in **Kibana Discover** per attività di monitoraggio e triage.

---
