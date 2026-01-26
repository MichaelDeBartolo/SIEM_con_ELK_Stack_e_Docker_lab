# SIEM Lab semplificato con ELK Stack e Docker

Questo repository illustra la configurazione di una pipeline **SIEM** (Security Information and Event Management) di base. Utilizziamo **Elasticsearch** e **Kibana** containerizzati con Docker, e **Filebeat** per raccogliere e inviare i log di sistema Linux direttamente a Elasticsearch per l'analisi.

**Obiettivo del laboratorio:** comprendere il flusso di dati in un SIEM attraverso tre fasi principali:
1. **Raccolta** dei log
2. **Archiviazione** centralizzata
3. **Visualizzazione** per monitoraggio e sicurezza

---

## 📑 Indice

- [Architettura del Lab](#architettura-del-lab)
- [Prerequisiti](#prerequisiti)
- [Setup del Lab](#setup-del-lab)
  - [1. Prepara l'Ambiente Docker](#1-prepara-lambiente-docker)
  - [2. Crea il File docker-compose.yml](#2-crea-il-file-docker-composeyml)
  - [3. Avvia lo Stack ELK](#3-avvia-lo-stack-elk)
  - [4. Installa e Configura Filebeat](#4-installa-e-configura-filebeat)
  - [5. Configurazione Iniziale in Kibana](#5-configurazione-iniziale-in-kibana)
  - [6. Esplora i Log in Kibana](#6-esplora-i-log-in-kibana)

---

## 🏗️ Architettura del Lab

Il setup prevede i seguenti componenti:

| Componente | Descrizione |
|-----------|-------------|
| **Elasticsearch** | Motore di ricerca distribuito e database che indicizza e archivia i log |
| **Kibana** | Interfaccia utente web per visualizzare, analizzare e creare dashboard dai log |
| **Filebeat** | Data shipper leggero che raccoglie i log dalla macchina host e li invia a Elasticsearch |
| **Docker & Docker Compose** | Strumenti per containerizzare e orchestrare i servizi |

```
┌─────────────┐
│  Filebeat   │ (Log Collector)
└──────┬──────┘
       │
       │ Invia log
       │
       ▼
┌──────────────────┐        ┌─────────┐
│ Elasticsearch    │◄──────►│ Kibana  │
│ (Index & Store) │        │ (UI)    │
└──────────────────┘        └─────────┘
```

---

## ✅ Prerequisiti

Assicurati di avere installato sulla tua macchina:

- **Docker** — versione 20.10.0 o successiva
- **Docker Compose** — versione v2.0.0 o successiva
- **Accesso `sudo`** — necessario per l'installazione di Filebeat

---

## 🚀 Setup del Lab

Segui questi passaggi per configurare il tuo ambiente SIEM.

### 1. Prepara l'Ambiente Docker

Crea una directory per il progetto e accedici:

```bash
mkdir elk-siem-lab
cd elk-siem-lab
```

---

### 2. Crea il File docker-compose.yml

Crea un file denominato `docker-compose.yml` all'interno della directory `elk-siem-lab`:

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

**Variabili principali:**
- `xpack.security.enabled=false`: Disabilita la sicurezza (solo per lab)
- `discovery.type=single-node`: Elasticsearch in modalità single-node
- `ES_JAVA_OPTS`: Allocazione memoria JVM (512MB min e max)

---

### 3. Avvia lo Stack ELK

Esegui questo comando nella directory dove hai creato `docker-compose.yml`:

```bash
docker compose up -d
```

**Verifica che i container siano attivi:**

```bash
docker compose ps
```

Dovresti vedere `elasticsearch` e `kibana` con stato **Up**.

---

### 4. Installa e Configura Filebeat

Installa Filebeat sulla macchina host (dove vuoi raccogliere i log).

#### a. Installa Filebeat

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb
sudo dpkg -i filebeat-8.14.0-amd64.deb
```

#### b. Configura Filebeat

Modifica il file di configurazione principale:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Assicurati che la sezione `output.elasticsearch` sia decommentata e configurata per puntare a `localhost:9200`. La sezione `output.logstash` deve essere commentata:

```yaml
# ========================== Outputs ==========================

output.elasticsearch:
  hosts: ["localhost:9200"]
  # username: "elastic"
  # password: "changeme"

# output.logstash:
#   hosts: ["localhost:5044"]
```

#### c. Abilita il Modulo system di Filebeat

Questo modulo raccoglie automaticamente i log di sistema Linux (es. `/var/log/auth.log`, `/var/log/syslog`):

```bash
sudo filebeat modules enable system
```

#### d. Correggi i Permessi (Cruciale!)

Per prevenire problemi di scrittura, imposta i permessi corretti:

```bash
sudo mkdir -p /var/lib/filebeat/registry/filebeat
sudo mkdir -p /var/log/filebeat

sudo chown -R root:root /etc/filebeat /var/lib/filebeat /var/log/filebeat
sudo chmod -R 755 /etc/filebeat /var/lib/filebeat /var/log/filebeat
```

#### e. Avvia Filebeat

```bash
sudo systemctl daemon-reload
sudo systemctl start filebeat
sudo systemctl enable filebeat
```

**Verifica lo stato di Filebeat:**

```bash
sudo systemctl status filebeat
sudo journalctl -u filebeat --since "2 minutes ago"
```

---

### 5. Configurazione Iniziale in Kibana

Rendi i log visibili in Kibana seguendo questi passaggi:

1. Apri il browser e accedi a **http://localhost:5601**
2. Nel menu di sinistra, fai clic sull'icona dell'ingranaggio ⚙️ (**Management**)
3. Seleziona **Stack Management** → **Index Patterns** (o **Data Views** nelle versioni più recenti)
4. Clicca su **Create data view**
5. Nel campo *"Index pattern name"*, digita: `filebeat-*`
6. Seleziona `@timestamp` dal dropdown per il *"Time field"*
7. Clicca su **Create data view**

---

### 6. Esplora i Log in Kibana

1. Nel menu di sinistra, clicca sull'icona della bussola 🧭 (**Analytics**)
2. Seleziona **Discover**

Dovresti vedere i tuoi log di sistema fluire in Kibana! Usa il selettore temporale in alto a destra per visualizzare i log più recenti.

---

## 🔍 Troubleshooting

### I log non compaiono in Kibana

- Verifica che Filebeat sia in esecuzione: `sudo systemctl status filebeat`
- Controlla i log di Filebeat: `sudo journalctl -u filebeat -n 50`
- Assicurati che Elasticsearch sia accessibile da Filebeat: `curl http://localhost:9200`

### Container Docker non avviati

```bash
docker compose logs elasticsearch
docker compose logs kibana
```

### Errori di permessi in Filebeat

Ripeti il passaggio **4.d** per correggere i permessi delle directory di Filebeat.

---

## 📚 Risorse Aggiuntive

- [Documentazione Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Documentazione Kibana](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Documentazione Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)

---

## 📝 Note

- Questo setup è **semplificato** e pensato per scopi didattici
- Non abilitare in produzione senza implementare proper security (SSL/TLS, autenticazione, etc.)
- Per ambienti di produzione, consulta la documentazione ufficiale di Elastic

---

**Creato per scopi educativi di SIEM Lab** 🔐
