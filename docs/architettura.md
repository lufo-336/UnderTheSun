# UnderTheSun — architettura (bozza)

Bozza da validare con il team dati prima del SAL 1 (20 nov). Modello di calcolo: CER realistico (vedi [team-dati.md](team-dati.md)). Contratto API: [api.md](api.md).

I diagrammi sono in Mermaid (GitHub li disegna). Versioni PNG in [img/](img/).

## 1. Architettura della piattaforma

```mermaid
flowchart LR
  subgraph Utenti
    S[Socio]
    A[Operatore / Admin]
  end

  subgraph Piattaforma["Piattaforma (docker compose)"]
    FE[Frontend React]
    BE["Backend Node + Express<br/>API REST, auth JWT"]
    DB[(PostgreSQL)]
  end

  subgraph Dati["Pipeline dati - Python, team DA"]
    GEN[Generazione dataset]
    ML["Batch: previsioni,<br/>anomalie, ripartizione CER"]
  end

  EXT["PVGIS, Open-Meteo,<br/>GSE, ISPRA"]
  LLM["LLM API<br/>classificazione testi"]

  S & A --> FE
  FE -->|HTTP JSON| BE
  BE --> DB
  BE -->|nuova segnalazione| LLM
  EXT --> GEN
  GEN -->|CSV + script di caricamento| DB
  DB --> ML
  ML -->|scrive risultati| DB
```

Chi fa cosa:
- **Team DA**: tutto il riquadro "Pipeline dati" + prompt/categorie per l'LLM.
- **WSA**: riquadro "Piattaforma" (DB fisico, API, auth, frontend, docker).
- **Contratto** (congelato al SAL 1): schema tabelle + data dictionary + endpoint API con esempi JSON.

## 2. Modello dati (bozza)

Una sola CER. Le serie orarie hanno chiave composta (id + ts).
Le tabelle marcate *derivata* le riempie il batch Python, non l'utente.

```mermaid
erDiagram
  UTENTE {
    int id PK
    string email
    string password_hash
    string ruolo "socio, operatore, admin"
    int socio_id FK "null per i dipendenti"
  }
  SOCIO {
    int id PK
    string nome
    string tipo "famiglia, impresa, ente"
    string comune
    date data_adesione
  }
  IMPIANTO {
    int id PK
    string nome
    string comune
    float lat
    float lon
    float kwp
    date data_attivazione
  }
  PRODUZIONE_ORARIA {
    int impianto_id FK
    timestamp ts
    float kwh
  }
  METEO_ORARIO {
    int impianto_id FK
    timestamp ts
    float irraggiamento
    float temperatura
    float nuvolosita
  }
  CONSUMO_ORARIO {
    int socio_id FK
    timestamp ts
    float kwh
  }
  BOLLETTA {
    int id PK
    int socio_id FK
    date mese
    float kwh
    float importo_eur
  }
  PREVISIONE {
    int impianto_id FK
    timestamp ts
    float kwh_previsti "derivata"
  }
  ANOMALIA {
    int id PK
    int impianto_id FK
    timestamp inizio
    timestamp fine
    string tipo "guasto, sporcizia, stringa, neve"
    float score "derivata"
  }
  RIPARTIZIONE_MENSILE {
    int socio_id FK
    date mese
    float kwh_condivisi "derivata"
    float incentivo_eur "derivata"
    float co2_evitata_kg "derivata"
  }
  SEGNALAZIONE {
    int id PK
    int socio_id FK
    int impianto_id FK "opzionale"
    timestamp ts
    string canale "email, portale"
    string testo
    string categoria "da LLM"
    string urgenza "da LLM"
    string stato "aperta, in_corso, chiusa"
  }
  VERBALE {
    int id PK
    date data_assemblea
    string testo
    string sintesi "da LLM"
  }
  PARAMETRO {
    string chiave PK "es. fattore_co2, tariffa_gse"
    float valore
    string fonte
  }

  SOCIO |o--o| UTENTE : "account"
  SOCIO ||--o{ CONSUMO_ORARIO : "consuma"
  SOCIO ||--o{ BOLLETTA : "riceve"
  SOCIO ||--o{ RIPARTIZIONE_MENSILE : "incassa"
  SOCIO |o--o{ SEGNALAZIONE : "invia"
  IMPIANTO ||--o{ PRODUZIONE_ORARIA : "produce"
  IMPIANTO ||--o{ METEO_ORARIO : "ha"
  IMPIANTO ||--o{ PREVISIONE : "ha"
  IMPIANTO ||--o{ ANOMALIA : "ha"
  IMPIANTO |o--o{ SEGNALAZIONE : "riguarda"
```

Domande aperte per il team DA:
- Serve la cabina primaria (in una CER reale la condivisione vale per cabina)? Proposta: tutti sotto la stessa, campo aggiunto solo se serve.
- La tariffa GSE dipende dal prezzo zonale: serve una tabella `prezzo_orario` o basta un valore medio in `PARAMETRO`?
- Le "comunicazioni guasti" sono segnalazioni con `canale = interno` o una tabella a parte?

## 3. Flusso di una segnalazione

```mermaid
sequenceDiagram
  actor S as Socio
  participant FE as Frontend
  participant BE as Backend
  participant LLM as LLM API
  participant DB as PostgreSQL
  actor O as Operatore

  S->>FE: compila segnalazione
  FE->>BE: POST /api/segnalazioni
  BE->>LLM: testo + prompt del team DA
  LLM-->>BE: categoria, urgenza, impianto
  BE->>DB: INSERT segnalazione classificata
  BE-->>FE: 201 Created
  Note over BE,LLM: se l'LLM non risponde si salva senza categoria e si riclassifica dopo
  O->>FE: apre coda segnalazioni
  FE->>BE: GET /api/segnalazioni?stato=aperta
  BE->>DB: SELECT ordinate per urgenza
  DB-->>BE: righe
  BE-->>FE: JSON
```

## 4. Pagine (prima stesura)

| Ruolo | Pagina | Dati |
|---|---|---|
| Socio | Simulatore di risparmio | consumi orari del socio + formula CER |
| Socio | Dashboard personale | consumi, kWh condivisi, incentivo, CO2 evitata |
| Socio | Le mie segnalazioni | apri nuova, vedi stato |
| Operatore | Impianti | produzione vs previsione, anomalie |
| Operatore | Coda segnalazioni | classificate e ordinate per urgenza |
| Admin | KPI comunità | energia condivisa, incentivi totali, CO2, soci |
| Admin | Assemblee | verbali + sintesi |
