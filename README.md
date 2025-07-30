# SIEM Lab Semplificato con ELK Stack e Docker

Questo repository illustra la configurazione di una pipeline SIEM (Security Information and Event Management) di base. Utilizzeremo **Elasticsearch** e **Kibana** containerizzati con Docker, e **Filebeat** per raccogliere e inviare i log di sistema Linux direttamente a Elasticsearch per l'analisi.

L'obiettivo di questo laboratorio è comprendere il flusso di dati in un SIEM: **raccolta**, **archiviazione** e **visualizzazione** dei log per scopi di monitoraggio e sicurezza.

---

## Architettura del Lab

Il setup prevede i seguenti componenti:

* **Elasticsearch:** Il motore di ricerca distribuito e il database che indicizza e archivia i log.
* **Kibana:** L'interfaccia utente web per visualizzare, analizzare e creare dashboard dai log archiviati in Elasticsearch.
* **Filebeat:** Un "data shipper" leggero installato sulla macchina da cui raccogliere i log, che li invia direttamente a Elasticsearch.
* **Docker & Docker Compose:** Strumenti usati per containerizzare e orchestrare Elasticsearch e Kibana, garantendo una configurazione rapida e isolata.

```mermaid
graph LR
    A[Macchina Host - Filebeat] --> B(Docker Network)
    B --> C[Elasticsearch Container]
    B --> D[Kibana Container]
    C --> D
