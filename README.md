# SIEM Lab Semplificato con ELK Stack e Docker

Questo repository illustra la configurazione di una pipeline SIEM (Security Information and Event Management) di base. Utilizzeremo **Elasticsearch** e **Kibana** containerizzati con Docker, e **Filebeat** per raccogliere e inviare i log di sistema Linux direttamente a Elasticsearch per l'analisi.

L'obiettivo di questo laboratorio è comprendere il flusso di dati in un SIEM: **raccolta**, **archiviazione** e **visualizzazione** dei log per scopi di monitoraggio e sicurezza.

---
Indice
---

Architettura del Lab

Prerequisiti

Setup del Lab

1. Prepara l'Ambiente Docker

2. Crea il File docker-compose.yml

3. Avvia lo Stack ELK

4. Installa e Configura Filebeat

5. Configurazione Iniziale in Kibana

6. Esplora i Log in Kibana

## Architettura del Lab

Il setup prevede i seguenti componenti:

* **Elasticsearch:** Il motore di ricerca distribuito e il database che indicizza e archivia i log.
* **Kibana:** L'interfaccia utente web per visualizzare, analizzare e creare dashboard dai log archiviati in Elasticsearch.
* **Filebeat:** Un "data shipper" leggero installato sulla macchina da cui raccogliere i log, che li invia direttamente a Elasticsearch.
* **Docker & Docker Compose:** Strumenti usati per containerizzare e orchestrare Elasticsearch e Kibana, garantendo una configurazione rapida e isolata.

---

## Prerequisiti

Assicurati di avere i seguenti strumenti installati sulla tua macchina:

* **Docker:** Versione 20.10.0 o successiva.
* **Docker Compose:** Versione v2.0.0 o successiva.
* **Accesso `sudo`:** Necessario per l'installazione e la configurazione di Filebeat.

---

## Setup del Lab

Segui questi passaggi per configurare il tuo ambiente SIEM:
```
1. Prepara l'Ambiente Docker

Crea una directory per il tuo progetto e naviga al suo interno:

[bash]
mkdir elk-siem-lab
cd elk-siem-lab

2. Crea il File docker-compose.yml

Crea un file chiamato docker-compose.yml all'interno della directory elk-siem-lab e incolla il seguente contenuto:

YAML

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.0
    container_name: elasticsearch
    environment:
      - xpack.security.enabled=false # Disabilita la sicurezza per semplicità del lab
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m # Alloca RAM, puoi aumentare se hai più disponibilità
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data # Persistenza dei dati
    ports:
      - "9200:9200" # Porta per l'API di Elasticsearch
    networks:
      - elk-network

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.0 # Stessa versione di Elasticsearch
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601" # Porta per l'interfaccia web di Kibana
    networks:
      - elk-network
    depends_on:
      - elasticsearch # Kibana dipende da Elasticsearch

volumes:
  esdata: # Volume per i dati di Elasticsearch

networks:
  elk-network:
    driver: bridge

3. Avvia lo Stack ELK

Esegui questo comando nella stessa directory dove hai creato il file docker-compose.yml:

[Bash]
docker compose up -d

Verifica che entrambi i container siano attivi:

[Bash]
docker compose ps

Dovresti vedere elasticsearch e kibana con stato Up.

4. Installa e Configura Filebeat

Installiamo Filebeat sulla macchina da cui vuoi raccogliere i log (la macchina host):

a. Installa Filebeat:

[Bash]
curl -L -O [https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb)
sudo dpkg -i filebeat-8.14.0-amd64.deb

b. Configura Filebeat:

Modifica il file di configurazione principale di Filebeat:

[Bash]
sudo nano /etc/filebeat/filebeat.yml

Assicurati che la sezione output.elasticsearch sia decommentata e configurata per puntare a localhost:9200. La sezione output.logstash deve essere commentata o rimossa.

YAML

# ---------------------------- Outputs -----------------------------

output.elasticsearch:
  hosts: ["localhost:9200"] # Punta al tuo Elasticsearch in Docker
  #username: "elastic" # Decommenta se abiliti la sicurezza di Elasticsearch
  #password: "changeme" # Decommenta se abiliti la sicurezza di Elasticsearch

# output.logstash: # Lascia questa sezione COMMENTATA per questo lab
#   hosts: ["localhost:5044"]

c. Abilita il Modulo system di Filebeat:
Questo modulo raccoglie automaticamente i log di sistema Linux (es. /var/log/auth.log, /var/log/syslog).

[Bash]
sudo filebeat modules enable system

d. Correggi i Permessi (Passaggio Cruciale):

Per prevenire problemi di scrittura, forza i permessi sulle directory chiave di Filebeat:

[Bash]
sudo mkdir -p /var/lib/filebeat/registry/filebeat
sudo mkdir -p /var/log/filebeat
sudo chown -R root:root /etc/filebeat /var/lib/filebeat /var/log/filebeat
sudo chmod -R 755 /etc/filebeat /var/lib/filebeat /var/log/filebeat

e. Avvia Filebeat:

[Bash]
sudo systemctl daemon-reload # Ricarica la configurazione di systemd
sudo systemctl start filebeat
sudo systemctl enable filebeat # Per avviarlo automaticamente all'avvio del sistema
Verifica che Filebeat sia attivo e senza errori:

[Bash]
sudo systemctl status filebeat
sudo journalctl -u filebeat --since "2 minutes ago" # Controlla i log per errori

5. Configurazione Iniziale in Kibana

Rendi i log visibili in Kibana:

a. Apri il browser e vai su http://localhost:5601.
b. Nel menu di sinistra, clicca sull'icona dell'ingranaggio (Management) e poi su Stack Management -> Index Patterns (o Data Views nelle versioni più recenti di Kibana).
c. Clicca su Create data view.
d. Nel campo "Index pattern name", digita filebeat-*.
e. Seleziona @timestamp dal menu a discesa per il "Time field".
f. Clicca Create data view.

6. Esplora i Log in Kibana

Vai nel menu di sinistra e clicca sull'icona della bussola (Analytics) -> Discover.
Dovresti vedere i tuoi log di sistema fluire in Kibana! Usa il selettore temporale in alto a destra per visualizzare i log più recenti.
